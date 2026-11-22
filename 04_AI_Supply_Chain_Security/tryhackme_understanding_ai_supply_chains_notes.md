# AI Supply Chain Security — Understanding AI Supply Chains

> **TryHackMe learning notes**
>
> Simple explanations, practical examples, memory tricks, and quick revision.

---

## 1. What Is a Supply Chain?

A **supply chain** is the network of people, companies, tools, and components involved in creating and delivering something.

### The house analogy 🏠

Imagine building a house:

- Bricks come from one supplier.
- Timber comes from another.
- Electrical wiring comes from a third.
- Plumbing materials come from somewhere else.

You depend on all these suppliers. If even one supplier gives you a faulty or dangerous component, the whole house can be affected.

**Software works the same way.**

A Python application may depend on dozens or hundreds of third-party packages. When you run:

```bash
pip install some-package
```

you may also install packages that `some-package` depends on.

These automatically installed packages are called **transitive dependencies**.

### Direct vs transitive dependency

- **Direct dependency:** A package you deliberately add to your project.
- **Transitive dependency:** A package installed because one of your direct dependencies needs it.

**Remember:** You may choose the package directly, but you also inherit trust in its dependencies.

### Why supply chain attacks work

Supply chain attacks exploit **trust**. Instead of attacking your application directly, an attacker compromises something your application already trusts.

| Incident | What happened | Impact described in the room |
|---|---|---|
| SolarWinds (2020) | Malicious code was inserted into the Orion build process. Customers installed a legitimate-looking update containing a backdoor. | Around 18,000 organisations were compromised, including US government agencies. |
| Log4Shell (2021) | A critical vulnerability in the widely used Log4j logging library enabled remote code execution. | Millions of Java applications were affected. |

**Why this is powerful:** One compromised component can affect many organisations that depend on it.

> **Memory trick:** Don't just ask, “Do I trust this application?” Ask, “Do I trust everything this application trusts?”

---

## 2. What Makes an AI Supply Chain Different?

Traditional software supply chains mainly involve **code**: libraries, frameworks, and packages.

AI supply chains add another important artefact: **trained models**.

A trained model is made up of more than a program. It involves:

1. **Architecture** — the structure of the neural network.
2. **Training data** — the examples used to teach the model.
3. **Training process** — the settings, methods, and hardware used during training.
4. **Serialised weights** — the learned parameters saved into a file.

### Model weights: think of a game save file 🎮

A game save file stores your progress: your character, inventory, and completed missions.

Model weights are similar: they store what a model learned during training.

Now imagine downloading a save file from a stranger. It loads and appears to work normally. But you don't know exactly how it was created or whether something harmful was hidden inside it.

That is the trust problem with downloading a model from an unknown source.

**Important:** Some model formats, especially pickle-based files, can contain serialised objects that execute code when loaded. Loading an untrusted model can therefore be dangerous.

---

## 3. Transfer Learning and Fine-Tuning

Most teams do not train every AI model from scratch. They often download an existing model and adapt it for their own task. This is called **transfer learning**.

A common method is **fine-tuning**:

- Start with a pre-trained model.
- Train it further using a smaller, task-specific dataset.
- Adjust its weights so it performs better for the new task.

### Example

Imagine downloading a general language model and fine-tuning it to answer questions about company policies.

You save time because the model already learned general language patterns. But you also inherit trust in its original training data, training process, and weights.

### The inherited-backdoor problem

A backdoor inserted during pre-training may survive fine-tuning.

That means:

1. An attacker poisons a popular base model.
2. Many teams download that model.
3. Those teams fine-tune it for different applications.
4. The hidden behaviour may remain in the downstream models.

**One compromised base model can affect many later applications.**

### What is LoRA?

**LoRA (Low-Rank Adaptation)** is a method that lets teams adapt a model using relatively small additional files called **adapters**.

Think of it like adding an attachment to a device:

- Base model = the main device.
- LoRA adapter = an add-on that changes how it behaves.

This creates two dependencies:

- The base model must be trusted.
- The adapter must also be trusted.

A safe base model plus a malicious adapter can still result in a compromised system.

> **Memory trick:** Fine-tuning changes the model's behaviour; it does not automatically erase everything the model inherited.

---

## 4. The Four Components of an AI Supply Chain

An AI system depends on four major component groups.

| Component | What it means | Example sources / examples |
|---|---|---|
| **Models** | Pre-trained weights and architectures | Hugging Face Hub, TensorFlow Hub, PyTorch Hub; `bert-base-uncased`, `gpt2` |
| **Datasets** | Data used for training and evaluation | Hugging Face Datasets, Kaggle, academic repositories; ImageNet, Common Crawl |
| **Frameworks** | Libraries used to train and run models | PyPI, conda, GitHub; PyTorch, TensorFlow, scikit-learn |
| **Dependencies** | Supporting packages required by frameworks and projects | PyPI, npm, conda-forge; NumPy, SciPy, Pillow, tokenizers |

### Easy way to remember

- **Model:** The brain.
- **Dataset:** The study material.
- **Framework:** The machinery used to build and run the brain.
- **Dependencies:** The supporting tools and parts.

Every connection between these components is a **trust relationship**.

For example:

- Your application trusts its ML framework.
- The framework depends on packages.
- The package manager obtains packages from a registry.
- The registry contains packages uploaded by publishers.

If one important trust relationship is compromised, the larger system may be affected.

---

## 5. Two Ways Organisations Consume AI

There are two common ways to use AI models. They create different supply-chain risks.

### Paradigm 1: Downloading model files

You download a model from a repository and run it on your own infrastructure.

Examples of file extensions include:

- `.pkl`
- `.pt`
- `.bin`
- `.safetensors`
- `.h5`
- `.gguf`

The model file crosses your trust boundary and is loaded by your own system.

#### Why file formats matter

| Format | What the room highlights |
|---|---|
| `.pkl` | Uses Python pickle serialisation; loading an untrusted file can execute arbitrary code. |
| `.pt` / `.bin` | The room groups these with pickle-based model files that can carry load-time code-execution risk. |
| `.safetensors` | Stores raw weight values without executable pickle code, reducing serialisation-level risk. |
| `.h5` | Keras format; not pickle-based, but may contain executable architecture-level code in model layers. |
| `.gguf` | Common for local LLMs such as LLaMA, Mistral, and Qwen. It is not pickle-based, but weight-level backdoors can still apply. |

**Crucial distinction:** A safer file format does not guarantee a safe model.

For example, `.safetensors` helps prevent pickle-based code execution, but it does not prove that the model's learned weights are benign.

#### What is quantisation?

Large models can require a lot of storage and memory.

**Quantisation** reduces the precision used to store model weights. For example, weights may be converted from 32-bit floating-point values to lower-precision 8-bit or 4-bit values.

This can reduce model size and help it run on consumer hardware.

But if another party performs quantisation, that transformation becomes another supply-chain step you need to trust.

**Think of it like compressing a file:** you are trusting both the original file and the process that transformed it.

### Paradigm 2: Calling a model through an API

Instead of downloading a model, you send prompts to a hosted provider and receive responses.

Examples mentioned in the room include OpenAI, Anthropic, Google, and aggregators such as OpenRouter.

You do not handle the model file yourself, but you still depend on the provider's supply chain.

| Supply-chain element | Downloading a model | Using an API |
|---|---|---|
| Model weights | You can inspect or scan the files you download. | Provider controls them; you cannot inspect them directly. |
| Training data | May be described in a model card. | May be undisclosed or only vaguely described. |
| Fine-tuning | You may control it yourself. | Provider may fine-tune without notice. |
| System prompts | Not part of loading a local model file in the same way. | Templates from untrusted sources may affect behaviour. |
| Versioning | You can pin a specific file hash. | Provider may change the model behind the same endpoint. |
| Hosting | Your infrastructure. | Provider's infrastructure, potentially shared and multi-tenant. |

**Key insight:** An API does not remove supply-chain risk. It moves much of that risk behind the provider's service.

> **Memory trick:**
> - Download model = you handle the package.
> - API model = provider handles the package, but you trust the provider.

---

## 6. The Four AI Supply-Chain Attack Layers

For the model-download approach, the room identifies four attack layers.

### Layer 1: Model layer 🧠

Attackers can target the model file or the model's internal behaviour.

The room describes three levels:

| Attack level | What happens | Simple way to picture it |
|---|---|---|
| **Serialisation-level** | Malicious code is hidden in the file's serialised structure and executes when the model is loaded. | A dangerous surprise inside the package when you open it. |
| **Architecture-level** | Malicious logic is embedded in the model's layers and can run during prediction. | A hidden instruction built into the machine. |
| **Weights-level** | Learned values are subtly changed to cause hidden behaviour for particular inputs. | A tiny change in the brain's learned associations. |

**Important lesson:** Removing executable code from a model file only addresses serialisation-level attacks. Architecture-level and weights-level attacks still need to be considered.

### Layer 2: Dependency layer 📦

This layer covers packages and libraries used by your ML project.

Attack techniques include:

- **Dependency confusion:** An attacker publishes a package with a name that causes a package manager to select the attacker's package instead of the intended internal package.
- **Typosquatting:** An attacker publishes a package with a name very similar to a legitimate package. A developer may mistype the name and install the malicious one.
- **Malicious packages:** A package may look professional and useful while hiding harmful behaviour.

**Example:** You intend to install a trusted package but accidentally install a similarly named malicious package.

### Layer 3: Data layer 📊

AI models learn from data. If attackers can poison the training data, they may influence the model's behaviour.

**Data poisoning** means inserting or changing data so that the trained model learns unwanted behaviour.

The TryHackMe room reports research showing that:

- Replacing as little as **0.1%** of a training dataset with crafted samples can introduce a reliable backdoor without measurably affecting accuracy on clean data.
- For denial-of-service objectives, the effective threshold may be as low as **0.001%**.

These figures are claims reported by the room's cited research; they are not universal thresholds for every model or dataset.

**Why it is hard to spot:** A poisoned model may still perform normally on ordinary test examples.

### Layer 4: Infrastructure layer 🏗️

This layer includes the systems used to host, distribute, build, and manage AI artefacts.

Attackers may:

- Compromise model or package repositories.
- Inject malicious steps into CI/CD build pipelines.
- Steal maintainer credentials.
- Push malicious updates while pretending to be a trusted publisher.

**Example:** If an attacker steals a maintainer's account, they may publish a malicious update under a name developers already trust.

---

## 7. How Attack Layers Can Combine

The four layers do not have to be attacked separately.

Imagine this chain:

1. **Infrastructure layer:** An attacker compromises an account on a model-hosting platform.
2. **Model layer:** They upload a model containing malicious pickle code or a backdoor.
3. **Dependency layer:** They include a `requirements.txt` file that references typosquatted packages.
4. **Data layer:** They distribute a poisoned dataset alongside the model.

A developer downloads what appears to be one useful AI project, but several parts of the supply chain may be malicious.

> **Remember:** One download can bring several kinds of risk into your environment.

---

## 8. Real-World AI and Software Supply-Chain Incidents

The TryHackMe room describes five incidents from 2022–2025. Together, they show how attackers target packages, model repositories, build systems, maintainer accounts, and security scanners.

### Incident 1: PyTorch dependency confusion — December 2022

An attacker published a malicious package named `torchtriton` on PyPI.

PyTorch nightly builds depended on an internal package with the same name. Package resolution caused the public malicious package to be installed.

The malicious package collected information including:

- SSH keys
- Git configuration
- `/etc/passwd`
- Up to 1,000 files from home directories

The information was exfiltrated using encrypted DNS queries. The package remained available for five days before discovery.

**Attack layer:** Dependency layer.

**Lesson:** Internal package names and public package registries can create dependency-confusion risks.

### Incident 2: Hugging Face Hub issues — 2023–2024

The room describes two sets of findings:

- In November 2023, Lasso Security found more than **1,500 exposed API tokens**, including **655 with write permissions**. These could allow malicious actors to push changes to legitimate repositories.
- In 2024, JFrog identified approximately **100 malicious pickle-based models** that looked legitimate and had real ML functionality, but silently executed attacker code when loaded.

**Attack layers:** Infrastructure layer (exposed tokens and repository access) and model layer (malicious pickle files).

**Lesson:** A professional model description or working demo does not prove that a model file is safe.

### Incident 3: Ultralytics build-pipeline compromise — December 2024

Attackers injected malicious code into the GitHub Actions build workflow for Ultralytics, the organisation behind the YOLO object-detection library.

A cryptominer was included in published PyPI packages. Developers who installed the affected package could receive the malicious version.

The attack targeted the **build infrastructure**, rather than simply changing the source code developers expected.

**Attack layer:** Infrastructure layer.

**Lesson:** Even if source code looks legitimate, a compromised build pipeline can produce malicious releases.

### Incident 4: `@solana/web3.js` compromise — December 2024

Attackers stole a maintainer's npm credentials and published two backdoored versions of a widely used package.

The malicious versions attempted to steal cryptocurrency private keys from wallet holders. The room gives estimated losses of **$130,000–$184,000** before detection.

This was not an AI-specific incident, but the same supply-chain pattern applies to AI dependencies.

**Attack layer:** Dependency and infrastructure layers.

**Lesson:** A trusted package can become dangerous if an attacker's update is published using a maintainer's stolen identity.

### Incident 5: NullifAI scanner evasion on Hugging Face — February 2025

ReversingLabs researchers found deliberately corrupted pickle files that could bypass Hugging Face's automated security scanners while still executing malicious code when loaded.

The technique abused edge cases in how scanners parsed malformed pickle structures.

**Attack layer:** Model layer, with a security-scanning weakness involved.

**Lesson:** Passing an automated scanner does not guarantee that a model is safe. Scanner coverage and the way files are parsed matter.

---

## 9. Quick Revision Table

| Concept | Remember this |
|---|---|
| Supply chain | Everything your application depends on, directly or indirectly. |
| Transitive dependency | A dependency installed because another package needs it. |
| AI supply chain | Models + datasets + frameworks + dependencies. |
| Model weights | Saved learned parameters of a trained model. |
| Transfer learning | Reusing a pre-trained model for a new task. |
| Fine-tuning | Further training a pre-trained model on task-specific data. |
| LoRA | A lightweight adaptation method that uses adapter files. |
| Quantisation | Reducing weight precision to make models smaller and easier to run. |
| Serialisation-level attack | Malicious code executes when a model file is loaded. |
| Architecture-level attack | Malicious logic is embedded in model layers. |
| Weights-level attack | Model weights are altered to create hidden behaviour. |
| Dependency confusion | A package manager selects an unintended package, potentially from a public registry. |
| Typosquatting | A malicious package imitates a legitimate package name. |
| Data poisoning | Malicious or crafted data influences model training. |
| Infrastructure attack | Compromising repositories, credentials, or build pipelines. |

---

## 10. Security Researcher's Checklist 🔍

When reviewing an AI project or model, think about each layer.

### Model
- Where did the model come from?
- Is the file format safe to load?
- Is the model's architecture or behaviour inspectable?
- Is there evidence of model provenance and integrity?

### Dependencies
- Are package names correct?
- Are dependencies pinned to known versions?
- Could a public package conflict with an internal package?
- Are packages coming from trusted sources?

### Data
- Where did the training or evaluation data come from?
- Is its origin documented?
- Could untrusted data have been added?
- Are there signs of suspicious behaviour on specific inputs?

### Infrastructure
- Who can publish or modify model artefacts?
- Are maintainer credentials protected?
- Are build pipelines protected and reviewed?
- Are model and package updates verified before use?

### One important rule

Do not trust a model only because it:

- Has many downloads or stars.
- Has a professional description.
- Produces correct answers in a quick test.
- Passed one automated scanner.

Those signals may be useful, but they do not prove the entire supply chain is safe.

---

## 11. The Big Picture — Remember the Four Layers

Use this mnemonic:

**M-D-D-I**

- **M**odel — Can the model file, architecture, or weights be malicious?
- **D**ependency — Can a package or library be compromised?
- **D**ata — Can poisoned data influence what the model learns?
- **I**nfrastructure — Can repositories, credentials, or build pipelines be compromised?

And remember the four main AI supply-chain components:

**Models + Datasets + Frameworks + Dependencies**

The central lesson of this room:

> **You are not just trusting an AI model. You are trusting the entire chain that created, modified, packaged, distributed, and runs it.**

---

## 12. Quick Self-Test 🧠

Try answering before checking the answers.

1. What is a transitive dependency?
2. What is the difference between a model and its weights?
3. What is transfer learning?
4. Why can fine-tuning fail to remove a backdoor?
5. What does a LoRA adapter add to the supply chain?
6. Which file format stores raw weights without executable pickle code?
7. Does using a hosted model API remove supply-chain risk?
8. What are the four AI supply-chain attack layers?
9. What is the difference between typosquatting and dependency confusion?
10. Which layer is targeted when an attacker compromises a CI/CD build pipeline?

### Answers

1. A package installed because another dependency requires it.
2. The model includes its architecture and learned parameters; weights are the learned parameter values.
3. Reusing a pre-trained model as the starting point for a new task.
4. A backdoor learned during pre-training may remain after further training.
5. It adds another file and publisher that must be trusted, alongside the base model.
6. `.safetensors`.
7. No. It moves much of the supply chain behind the provider's service.
8. Model, dependency, data, and infrastructure.
9. Typosquatting imitates a legitimate package name; dependency confusion exploits package-source or name-resolution behaviour to get an unintended package selected.
10. Infrastructure layer.

---

*Notes based on the supplied TryHackMe module content: “Understanding AI Supply Chains.”*
