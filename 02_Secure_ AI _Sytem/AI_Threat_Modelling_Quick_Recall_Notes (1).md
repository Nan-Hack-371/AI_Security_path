# AI Threat Modelling — Quick Recall Notes

> **Big idea:** AI threat modelling means identifying what can go wrong across an AI system — its data, model, integrations, users and decisions — then choosing controls before attackers exploit them.

---

## 1. Why AI needs threat modelling

Imagine MegaCorp uses AI for three jobs:
- A chatbot answers customer questions.
- A recommendation engine suggests products.
- A fraud model approves or flags transactions.

A traditional security review may check servers, databases, APIs and passwords. But an AI system has extra things to protect: training data, model weights, embeddings, prompts and model versions.

AI also behaves differently:
- **Non-deterministic:** the same input may produce different outputs.
- **Black box:** it can be difficult to explain exactly why a model produced a result.

So, defenders must test observable behaviour and failure modes — not just inspect code.

**Recall:** AI system = traditional application assets + AI-specific assets + unpredictable behaviour.

---

## 2. AI-specific assets — what must we protect?

| Asset | Simple meaning | Why attackers care |
|---|---|---|
| Training data | Examples used to teach the model | Poison it and the model may learn harmful or incorrect patterns |
| Model weights / parameters | Values that store what the model has learned | Steal them and an attacker may obtain a working copy of the model |
| Embeddings | Numeric representations used to compare or retrieve information | Manipulation can change which information the system retrieves |
| System prompts | Hidden instructions guiding the model | Leakage can reveal internal rules and business logic |
| Feature stores | Prepared data supplied to models at prediction time | Tampered features can distort decisions without changing the model |
| Model registry / artifacts | Stored, versioned models ready for deployment | An attacker may replace a trusted model with a backdoored one |

**Quick memory hooks:**
- Training data = what the model learns from.
- Weights = what the model has learned.
- Embeddings = how information is represented for similarity/retrieval.
- System prompt = how the model is instructed.
- Feature store = what the model is shown at prediction time.
- Registry = which model gets deployed.

**Important:** A stolen model is not fixed by simply rotating a password. A poisoned training set may require investigation and retraining with trusted data.

---

## 3. The AI data supply chain — follow the data

Think of the model as a product moving through a factory. Every stage can be attacked.

<text>Raw data → Clean/label → Train → Validate/package → Deploy/inference</text>

### Stage 1 — Collection
Data arrives from websites, internal systems, users, vendors or purchased datasets.

**Risk:** An attacker contributes malicious or misleading data at its source.

### Stage 2 — Cleaning and labelling
Data is filtered, transformed and tagged with labels.

**Risk:** Incorrect labels quietly teach the model the wrong relationship. The dataset may look normal even though its labels are corrupted.

### Stage 3 — Training
The model learns patterns and stores them in its weights.

**Risk:** Poison that survives earlier checks can become part of the trained model. The impact may persist until a clean model is trained.

### Stage 4 — Validation and packaging
The model is tested, versioned and placed in a registry.

**Risk:** An attacker swaps the approved model for a backdoored one. Normal tests may miss it if they do not include the secret trigger.

### Stage 5 — Inference / production
The model receives live inputs and returns predictions. RAG systems may also retrieve documents from vector databases at this point.

**Risk:** Malicious live inputs or poisoned retrieved documents influence the output.

**MegaCorp example:** Fraud models are retrained monthly. An attacker gradually injects crafted transactions into the training data. The model learns to overlook a particular fraud pattern. The compromise may only become visible after retraining and deployment.

**Recall:** Data poisoning can be delayed — the bad input enters today, but the harmful behaviour may appear after a future training cycle.

---

## 4. Why STRIDE needs an AI upgrade

STRIDE is a familiar threat-modelling framework. It asks six questions:

| Letter | Threat | Security property | Basic meaning |
|---|---|---|---|
| S | Spoofing | Authenticity | Can someone pretend to be a trusted user/source? |
| T | Tampering | Integrity | Can data, code or instructions be changed? |
| R | Repudiation | Accountability | Can an action be denied because evidence is missing? |
| I | Information Disclosure | Confidentiality | Can protected information be exposed? |
| D | Denial of Service | Availability | Can the service or resources be exhausted? |
| E | Elevation of Privilege | Authorisation | Can someone gain capabilities they should not have? |

STRIDE is still useful, but AI changes how each threat appears:
- Poisoning training data is not like editing one database row; the effects can be hidden inside model behaviour.
- Adversarial inputs may cause misclassification or bypass safeguards, crossing multiple STRIDE categories.
- An AI agent's permissions may include tools such as email, databases or code execution.
- Stealing model weights or reproducing a model can expose an organisation's expensive AI capability.

**Recall:** Don't discard STRIDE. Adapt it to AI assets, data pipelines, model behaviour and tool permissions.

---

## 5. STRIDE adapted for AI — MegaCorp examples

### S — Spoofing: fake trusted data or identity

**Story:** MegaCorp's chatbot uses RAG to answer questions from an internal policy library. An attacker adds a fake policy document. The chatbot retrieves it and treats the false information as trusted.

Other examples:
- Fake AI API endpoint pretending to be the real service.
- Inputs crafted to fool face or voice recognition.

**Defender question:** Can the system verify where its data and model responses came from?

**Recall:** Spoofing = make malicious content look like a trusted source.

### T — Tampering: poisoning or changing what the model uses

**Story:** Crafted transactions enter MegaCorp's monthly fraud-training pipeline. Over time, the model learns the wrong patterns and misses a specific fraud type.

Other examples:
- Changing model weights or replacing a registry artifact.
- Manipulating prompts or retrieved context.
- Altering feature-store values.

**Defender question:** Can an attacker change training data, model files, features or inference context?

**Recall:** Tampering = corrupt the information or instructions the AI relies on.

### R — Repudiation: decisions cannot be reconstructed

**Story:** A regulator asks why the fraud model approved a suspicious transaction three weeks ago. MegaCorp has the result but did not record the model version, input features or decision context.

Useful audit evidence includes:
- Input and output (with sensitive data handled appropriately).
- Model and configuration version.
- Relevant prompt and retrieved context.
- Tool calls and actions taken.
- Timestamp, actor and decision threshold.

**Defender question:** Can we explain and reproduce what happened?

**Recall:** Repudiation = no reliable evidence trail for an AI decision.

### I — Information Disclosure: model or data gets exposed

**Story:** A competitor sends thousands of carefully selected queries to MegaCorp's recommendation API and collects the responses. They use these examples to train a model that imitates MegaCorp's recommendation logic.

Other examples:
- Training data extraction.
- System prompt leakage.
- Reconstructing source information from embeddings.

**Defender question:** Can queries or outputs reveal data, prompts, embeddings or model behaviour?

**Recall:** Information disclosure = expose the organisation's data or AI intellectual property.

### D — Denial of Service: inference becomes too expensive

**Story:** An attacker sends huge prompts and requests long responses. The chatbot stays online, but its inference bill increases dramatically.

Other examples:
- GPU exhaustion through high-volume or costly requests.
- Specially crafted inputs that consume excessive compute (“sponge” inputs).
- Flooding or disrupting the training pipeline.

**Defender question:** Can someone exhaust compute, capacity or budget?

**Recall:** DoS does not always mean the service goes offline. It can mean the service remains available but becomes financially unsustainable.

**Related OWASP risk:** LLM10 — Unbounded Consumption.

### E — Elevation of Privilege: the AI gains or uses too much power

**Story:** A customer chatbot is jailbroken. It also has a database tool with broad permissions. The attacker manipulates the bot into querying customer records it should never access.

Other examples:
- Bypassing model guardrails (jailbreaking).
- Abusing tools such as web browsing, email or code execution.
- Using one plugin or integration to influence a more privileged one.

**Defender question:** If the model is manipulated, what is the maximum damage its tools can cause?

**Recall:** Elevation of privilege = turn limited AI access into unauthorised capabilities.

### STRIDE-AI summary

| STRIDE | Main AI manifestation | MegaCorp example |
|---|---|---|
| Spoofing | Data-source impersonation | Fake policy document enters RAG |
| Tampering | Data/model/feature poisoning | Fraud model learns to miss a pattern |
| Repudiation | Missing decision audit trail | Cannot explain an old approval |
| Information Disclosure | Model extraction or data leakage | Competitor imitates recommendation engine |
| Denial of Service | Inference/resource cost abuse | Expensive prompts drive up cloud bill |
| Elevation of Privilege | Jailbreak / excessive agency | Chatbot abuses an over-permissioned database tool |

**STRIDE does not cover everything neatly.** Bias, fairness, unexpected emergent behaviour and some adversarial examples may need additional AI-specific assessment.

---

## 6. MITRE ATLAS — the AI attack technique catalogue

Think of:
- **MITRE ATT&CK** = structured knowledge of attacks against traditional IT.
- **MITRE ATLAS** = structured knowledge of attacks against AI/ML systems.

ATLAS helps turn a broad threat into a recognisable attack technique and defensive action.

### ATLAS vocabulary

| Term | Question it answers | Example |
|---|---|---|
| Tactic | Why is the attacker doing this? | ML Attack Staging |
| Technique | How are they doing it? | Data Poisoning |
| Sub-technique | What specific variation? | A particular way of crafting adversarial data |
| Mitigation | What can defenders do? | Validate inputs and track data provenance |

### Five techniques to recognise

| Technique | What it means | STRIDE connection |
|---|---|---|
| **AML.T0020 — Data Poisoning** | Insert malicious data into training so the model learns harmful patterns | Tampering |
| **AML.T0024 — Model Extraction** | Query a model and use its outputs to recreate its behaviour | Information Disclosure |
| **AML.T0015 — Evade ML Model** | Craft inputs that cause a model to misclassify or fail to detect something | Can span Tampering, Spoofing and Elevation of Privilege |
| **AML.T0051 — LLM Prompt Injection** | Manipulate model behaviour through malicious instructions | Often Tampering; context-dependent |
| **AML.T0018 — Backdoor ML Model** | Hide a trigger in a model so it behaves maliciously only when that trigger appears | Tampering |

**Direct vs indirect prompt injection:**
- **Direct:** Attacker types malicious instructions into the chat.
- **Indirect:** Malicious instructions are hidden in content the model reads, such as a webpage, email or RAG document.

### How to use STRIDE and ATLAS together

1. **STRIDE:** Identify the broad threat — *what could go wrong?*
2. **ATLAS:** Find the specific technique — *how could an attacker do it?*
3. **Mitigation:** Choose controls — *how will we reduce the risk?*

**MegaCorp example:** STRIDE identifies training-data tampering. ATLAS points to Data Poisoning (AML.T0020). The defender can now investigate data provenance, detect unusual training samples and monitor model performance for unexpected drift.

### Real-world case studies to remember

- **ShadowRay (AML.CS0023):** Attacks against exposed or vulnerable Ray infrastructure showed that AI training and compute environments can be compromised like other infrastructure.
- **Morris II Worm (AML.CS0024):** A research demonstration of self-propagating prompt injection through AI agents and RAG-style email workflows. It showed how malicious content could move between connected AI systems and cause data exposure.

**Recall:** ATLAS gives defenders a shared vocabulary and real attack examples — not just generic threat labels.

---

## 7. OWASP LLM Top 10 (2025) — connect risk to architecture

STRIDE tells us the **type** of threat. ATLAS describes the **technique**. OWASP LLM Top 10 helps identify **which LLM risks affect which components**.

### The ten risks

| ID | Risk | Quick meaning | Components to inspect |
|---|---|---|---|
| LLM01 | Prompt Injection | Malicious input or retrieved content changes model behaviour | Inference endpoint, RAG pipeline, vector database |
| LLM02 | Sensitive Information Disclosure | Personal, confidential or secret information is exposed | Training pipeline, inference endpoint, prompts |
| LLM03 | Supply Chain | Third-party models, datasets, tools or dependencies are compromised | Training pipeline, model registry, plugins/tools |
| LLM04 | Data and Model Poisoning | Training data, features or model files are corrupted | Training pipeline, feature store, model registry |
| LLM05 | Improper Output Handling | Model output is used without safe validation or sanitisation | Frontend, API gateway, downstream services |
| LLM06 | Excessive Agency | AI has more tools, permissions or autonomy than necessary | Agent/orchestration layer, tool integrations, API gateway |
| LLM07 | System Prompt Leakage | Hidden instructions or sensitive prompt content are revealed | Inference endpoint, prompt configuration |
| LLM08 | Vector and Embedding Weaknesses | Embeddings, retrieval or vector storage are manipulated or exposed | Embedding process, vector database, RAG pipeline |
| LLM09 | Misinformation | Convincing but false content influences users or systems | Inference endpoint, knowledge sources, output channels |
| LLM10 | Unbounded Consumption | Requests consume excessive tokens, compute, time or money | Inference endpoint, API gateway, training pipeline |

### Read the mapping in both directions

**Risk → Component:**  
“Where can prompt injection happen?” Check the inference endpoint and any content fed into the model, especially retrieved RAG content.

**Component → Risk:**  
“What can go wrong in a vector database?” Consider:
- LLM01 — malicious instructions in retrieved documents.
- LLM08 — embedding or vector-store weaknesses.
- LLM09 — stale or incorrect sources causing false answers.

This second way of thinking is useful when a new component is added to an architecture.

### Three important component profiles

**1. LLM inference endpoint — broadest exposure**

Common risks:
- LLM01 Prompt Injection
- LLM02 Sensitive Information Disclosure
- LLM05 Improper Output Handling
- LLM06 Excessive Agency
- LLM07 System Prompt Leakage
- LLM09 Misinformation
- LLM10 Unbounded Consumption

Why it matters: this is where prompts enter and model responses leave. It is often the main point attackers interact with.

**2. Vector database / RAG pipeline**

Common risks:
- LLM01 — indirect prompt injection from retrieved content.
- LLM08 — embedding/vector weaknesses.
- LLM09 — outdated or incorrect documents lead to misinformation.

Defensive focus: control who can add and retrieve documents, validate indexed content, protect the vector store and monitor source freshness.

**3. Training pipeline**

Common risks:
- LLM02 — sensitive data enters training.
- LLM03 — compromised third-party datasets or base models.
- LLM04 — training data or model poisoning.

Defensive focus: trusted sources, provenance, dataset checks, protected model artifacts and controlled deployment.

---

## 8. Put the frameworks together

| Framework | Its job | Ask this question |
|---|---|---|
| **STRIDE-AI** | Categorises threats | What type of thing could go wrong? |
| **MITRE ATLAS** | Describes attacker techniques | How could an attacker carry it out? |
| **OWASP LLM Top 10** | Maps common LLM risks to components | Which component is exposed to this risk? |

### A practical defender workflow

1. **Draw the architecture:** Include users, APIs, model, prompts, data stores, RAG, tools and training pipeline.
2. **Inventory assets:** Identify data, weights, embeddings, prompts, features and model artifacts.
3. **Trace data flow:** Record where data comes from, where it is stored, who can change it and where it is used.
4. **Apply STRIDE-AI:** Walk through each component and ask the six threat questions.
5. **Enrich with ATLAS:** Match findings to known AI attack techniques.
6. **Map OWASP risks:** Identify the LLM Top 10 risks relevant to each component.
7. **Choose controls:** Prioritise based on likelihood, impact and exposure.
8. **Keep evidence:** Log model versions, inputs, outputs, retrieval context and tool actions so incidents can be investigated.

---

## 9. Quick revision — don't mix these up

| If the question says… | Think… |
|---|---|
| “What did the model learn from?” | Training data |
| “What stores learned behaviour?” | Model weights |
| “What helps retrieve similar content?” | Embeddings / vector database |
| “Hidden instructions were revealed” | LLM07 — System Prompt Leakage |
| “Malicious instructions are in a retrieved document” | Indirect Prompt Injection — LLM01 / AML.T0051 |
| “The model learned from poisoned examples” | Data Poisoning — LLM04 / AML.T0020 |
| “The approved model was secretly replaced” | Model registry / supply chain / backdoor risk |
| “Nobody can explain an old model decision” | STRIDE Repudiation |
| “The AI can access tools it does not need” | STRIDE Elevation of Privilege / LLM06 |
| “The API bill explodes due to costly prompts” | Denial of Wallet / LLM10 |
| “A competitor copies model behaviour through API queries” | Model Extraction — AML.T0024 |

---

## 10. Active Recall — test yourself

Answer without looking back.

1. Name six AI-specific assets a threat modeller should inventory.
2. Why can training-data poisoning take weeks or months to become visible?
3. What are the five stages of the AI data supply chain?
4. What does non-deterministic behaviour make harder for defenders?
5. In STRIDE, which category covers the inability to explain an old AI decision?
6. A fake document is inserted into a RAG knowledge base. Which STRIDE category may apply?
7. What is the difference between a direct and an indirect prompt injection?
8. Which ATLAS ID is associated with Data Poisoning?
9. Which ATLAS technique describes copying a model through repeated queries?
10. What does STRIDE provide that ATLAS does not?
11. What does OWASP LLM Top 10 add to the assessment?
12. Name three OWASP risks relevant to a vector database/RAG pipeline.
13. Why is a model registry a high-value target?
14. Why is least privilege important for AI agents with tools?

### Answer key

1. Training data, model weights, embeddings, system prompts, feature stores, model registry/artifacts.
2. Poisoned data may only influence behaviour after the next training and deployment cycle.
3. Collection → cleaning/labelling → training → validation/packaging → inference.
4. Reproducible testing, auditing and incident investigation.
5. Repudiation.
6. Spoofing may apply if the fake source impersonates trusted content; Tampering may also apply because the knowledge source was altered.
7. Direct injection is placed in user input; indirect injection is hidden in content the model retrieves or processes.
8. AML.T0020.
9. Model Extraction — AML.T0024.
10. STRIDE categorises broad threat types; ATLAS documents specific adversary techniques.
11. It maps common LLM risks to the components where they can occur.
12. LLM01, LLM08 and LLM09.
13. An attacker may replace a trusted model with a malicious or backdoored artifact.
14. If the model is manipulated, narrow permissions limit what the attacker can make it do.

---

## Final memory line

**Inventory AI assets → trace the data supply chain → adapt STRIDE → enrich with ATLAS → map OWASP risks to components → apply controls and keep audit evidence.**
