# RAG Security Fundamentals — TryHackMe Notes

> **Big idea:** In a Retrieval-Augmented Generation (RAG) system, the LLM uses documents retrieved at query time. If untrusted or misleading content enters the knowledge base and gets retrieved, it can influence the model's answer without changing the model or the user's prompt.

---

## 1. What Is RAG?

**RAG** stands for **Retrieval-Augmented Generation**.

A RAG application searches an external knowledge base for information related to a user's question. It then provides that retrieved content to an LLM as context, and the LLM generates an answer using it.

### Simple analogy

Imagine an open-book exam:

- You = user asking a question.
- Library = vector store / knowledge base.
- Librarian = retriever.
- Notes found in the library = retrieved documents.
- Student writing the answer = LLM.

If someone sneaks incorrect or malicious notes into the library, the student may use them to answer the question. The student does not automatically know that the notes are fake.

**Important:** RAG can introduce new information at inference time. That means an attacker may influence answers by poisoning retrieved documents, without retraining or changing the LLM itself.

---

## 2. Core Components of a RAG System

| Component | What it does | Easy way to remember |
|---|---|---|
| **Embedding model** | Converts text into numerical vectors | Turns meaning into numbers |
| **Vector store** | Stores document embeddings | The searchable library |
| **Retriever** | Finds documents similar to the user's query | The librarian |
| **LLM** | Generates the final response using retrieved context | The answer writer |

### What are embeddings?

An **embedding** is a numerical representation of text that captures aspects of its meaning.

The system can compare the embedding of a user's query with document embeddings to find semantically similar content.

Similarity is not the same as trustworthiness:
- A document can be highly relevant to a query.
- That does not mean the document is correct, safe, current, or approved.

### How data flows through RAG

```text
User submits a query
        |
        v
Embedding model converts query into a vector
        |
        v
Vector store searches for similar document vectors
        |
        v
Retriever selects top matching documents
        |
        v
Retrieved text is added to the LLM's context
        |
        v
LLM generates the response
```

**Security gap:** The model does not independently verify that retrieved content is correct or safe.

---

## 3. Where RAG Security Risks Concentrate

The room highlights three important risk areas:

1. **Ingestion** — malicious, outdated, or untrusted documents enter the knowledge base.
2. **Retrieval** — poisoned documents are selected because they appear semantically relevant.
3. **Context injection** — retrieved text is added to the LLM's context and influences its response.

The broader RAG attack surface includes four stages:

| Stage | What happens | Security concern |
|---|---|---|
| Document ingestion | Documents enter the knowledge base | Untrusted content may be accepted |
| Embedding generation | Text is converted into vectors | Meaning is represented numerically; trust and authorship are not automatically established |
| Similarity-based retrieval | Relevant documents are selected | Relevance does not prove correctness or safety |
| Context injection | Retrieved content is added to the prompt/context | Malicious text can influence generation |

### Why each stage matters

**1. Document ingestion**

RAG systems may ingest content from:
- Shared drives.
- Internal wikis.
- Automated feeds.
- Other connected document sources.

If source validation and approval are weak, malicious or misleading content can enter the knowledge base.

**2. Embedding generation**

Text is converted into vectors. The embedding process does not automatically preserve or enforce trust decisions such as:
- Who authored the document?
- Was it approved?
- Is it still current?
- Is it safe to use?

**3. Similarity-based retrieval**

The retriever looks for semantic similarity. It does not inherently know whether a document is trustworthy or whether its intent is malicious.

An attacker may craft content that sounds relevant to common or sensitive questions.

**4. Context injection**

Retrieved documents are inserted into the LLM's context. If the content includes misleading information or instruction-like text, it may influence the generated answer.

---

## 4. Retrieval Abuse and Context Manipulation

**Retrieval abuse** is a RAG-specific attack technique where unintended or malicious documents influence the LLM's output through retrieval.

Unlike a direct prompt injection, the attacker may not need to interact with the model's prompt. Instead, they influence which documents the retriever selects.

### Passive poisoning vs active manipulation

| Type | Meaning | Example |
|---|---|---|
| **Passive poisoning** | Malicious content is added once and left in the knowledge base | An attacker gets a misleading document indexed and waits for normal questions to retrieve it |
| **Active manipulation** | Content is deliberately crafted to rank highly for common or sensitive queries | A document is written with wording likely to match many relevant searches |

In both cases, the attacker may not need continuous access after the content has entered the knowledge base.

### How context manipulation works

A retrieved document may:
- Contain false or misleading information.
- Include instructions disguised as ordinary documentation.
- Frame information in a way that changes how the LLM responds.

The retriever selects based on semantic relevance, not safety or intent. If the document ranks highly, it may be included in the model's context.

### Why the model may not defend itself

From the model's perspective:
- It may not know where the retrieved document came from.
- It cannot independently verify the document's intent.
- It may not have access to the retriever's ranking or trust decisions.
- It may struggle to distinguish instructions from ordinary data in the retrieved context.

The room describes this as a design limitation: content can gain influence because it is placed in the context, not because it has been verified.

### Why retrieval abuse is hard to detect

- Responses can look logical and well-written.
- There may be no visible prompt injection in the user's message.
- Logs may simply say that relevant documents were retrieved.
- The system may appear to be functioning normally.

**Key lesson:** Retrieval is a security boundary, not merely a performance feature.

---

## 5. Real-World RAG Failure Scenarios

The TryHackMe room presents three case studies. These are summaries of the examples as described in the supplied room text.

### Case study 1: Microsoft Copilot — email-based retrieval abuse (2026)

The room describes a scenario involving Microsoft 365 Copilot and content embedded in emails.

**How the failure occurred:**
1. Emails were accepted as ingestion sources.
2. Email content was retrieved during normal queries.
3. Retrieved text was injected into the model's context.
4. The model could not reliably distinguish legitimate information from embedded instructions or misleading guidance.

**What went wrong:**
- Trust was implicitly granted at ingestion.
- Retrieval surfaced unverified content.
- Context injection increased the influence of that content.

**Impact described in the room:**
- Sensitive enterprise information was exposed.
- Organisations restricted Copilot access.
- Microsoft issued security guidance and mitigations.

**Lesson:** Internal data sources can also become attack vectors. Internal does not automatically mean trustworthy.

### Case study 2: ChatGPT plugins — untrusted external content (2023)

The room describes plugin-based retrieval from external services, webpages, and third-party APIs.

**How the failure occurred:**
1. The assistant retrieved external content.
2. Retrieved content contained instruction-like text.
3. That text entered the model's context.
4. It influenced model behaviour without changing the system prompt or model parameters.

This is an example of **indirect prompt injection via retrieval**.

**What went wrong:**
- Retrieved sources were trusted by default.
- Content intent was not adequately validated.
- Data and instructions were not sufficiently separated.

**Impact described in the room:**
- Unsafe or manipulated outputs.
- Some plugin features were temporarily disabled.
- Retrieval and plugin security approaches were redesigned.

**Lesson:** External content can influence an assistant even when the user did not directly submit that content.

### Case study 3: Web-connected AI assistants — stale and incorrect retrieval

This scenario is about outdated information rather than necessarily a malicious attacker.

**How the failure occurred:**
1. A source document was updated.
2. An old version remained indexed.
3. Retrieval selected the stale content.
4. The assistant presented outdated information as if it were current.

**What went wrong:**
- No effective document lifecycle management.
- No freshness or version validation.
- Semantic relevance was prioritised over freshness.

**Impact described in the room:**
- Users could follow incorrect guidance.
- Operational and compliance risks increased.
- Trust in the AI system decreased.

**Lesson:** RAG can fail even without an attacker. Poor document governance is enough to create risk.

### Why these failures are dangerous

The output may look:
- Logical.
- Confident.
- Well-structured.
- Authoritative.

But a polished answer can still be wrong or unsafe. Users may trust the answer because they cannot see which retrieved documents influenced it.

---

## 6. Defensive Considerations for RAG Systems

There is no single control that reliably prevents every form of retrieval abuse. The room recommends layered defence.

### A. Guardrails on retrieved content

Guardrails attempt to limit how retrieved content influences the model.

Common approaches:
- Limit how much retrieved text is inserted into prompts.
- Separate retrieved data from system instructions.
- Use heuristics to flag instruction-like patterns.

**Limitation:** Instruction-like text can be ambiguous. Attackers may rephrase or obfuscate it, so simple pattern checks can be bypassed.

Guardrails reduce risk; they do not guarantee safety.

### B. Validation during ingestion

Preventing malicious or untrusted documents from entering the knowledge base is important because detection becomes harder after the content is embedded and indexed.

Useful controls include:
- Review document sources.
- Enforce approval workflows.
- Track document ownership.
- Record update history.
- Validate whether documents are current and approved.

**Remember:** It is easier to stop an untrusted document before ingestion than to find its influence after retrieval.

### C. Monitoring and output review

Even with validation and guardrails, failures can still happen. Monitor behaviour over time.

The room highlights these signals:

| Signal | What it might indicate |
|---|---|
| Unusual retrieval patterns | Unexpected documents are being selected |
| Repeated retrieval of the same documents | A particular document may be disproportionately influencing answers |
| Gradual changes in response tone or behaviour | Possible output drift |

### What is output drift?

**Output drift** is a gradual change in how a system responds over time.

For example:
- Earlier, an assistant gives balanced answers.
- Over time, it increasingly repeats claims from one questionable document.
- Its responses slowly become biased toward that document's content.

The room identifies output drift as a possible warning sign of poisoning or persistent influence from misleading data.

Monitoring and regular review can reveal subtle changes that a one-time scan may miss.

### D. Defence in depth

No single control fully protects a RAG system.

Layered defence aims to:
1. Reduce the likelihood of successful abuse.
2. Limit the impact if malicious content is retrieved.
3. Detect problems early.

A useful mental model:

```text
Validate sources before ingestion
             |
             v
Control and review retrieval
             |
             v
Separate data from instructions
             |
             v
Apply guardrails to retrieved context
             |
             v
Monitor retrieval and generated outputs
             |
             v
Review and improve controls regularly
```

---

## 7. Quick Revision Sheet

| Term | Simple meaning |
|---|---|
| RAG | Retrieves external information and gives it to an LLM as context |
| Embedding | Numerical representation of text meaning |
| Vector store | Stores embeddings for similarity search |
| Retriever | Selects documents relevant to a query |
| Ingestion | Adding documents to the knowledge base |
| Context injection | Placing retrieved content into the LLM's context |
| Inference-time poisoning | Influencing output through retrieved content without retraining the model |
| Retrieval abuse | Malicious or unintended documents influence output through retrieval |
| Passive poisoning | Poisoned content is added and left for normal queries to retrieve |
| Active manipulation | Content is crafted to rank highly for targeted queries |
| Indirect prompt injection | Instructions reach the model through external or retrieved content |
| Output drift | Gradual change in system responses over time |
| Document lifecycle management | Managing document ownership, updates, versions, and removal |
| Defence in depth | Multiple overlapping security controls |

### The RAG flow mnemonic

**E → V → R → C → G**

- **E**mbed the query.
- **V**ector search.
- **R**etrieve relevant documents.
- **C**ontext injection.
- **G**enerate the answer.

### The three key risk areas

**Ingestion → Retrieval → Context injection**

- Ingestion: Can untrusted content get in?
- Retrieval: Can poisoned content rank highly?
- Context injection: Can retrieved content influence the LLM?

### The defensive memory hook

**Check before it enters → Control what gets retrieved → Constrain what reaches the model → Monitor what comes out.**

---

## 8. Self-Test

Try answering without looking at the notes.

1. What does RAG stand for?
2. What does an embedding model do?
3. What is the role of a vector store?
4. Does similarity-based retrieval prove that a document is trustworthy?
5. What are the three main RAG risk areas highlighted in the room?
6. What is inference-time data poisoning?
7. What is the difference between passive poisoning and active manipulation?
8. Why can retrieved instruction-like text influence an LLM?
9. Why is retrieval abuse difficult to detect?
10. How can stale documents cause RAG failures without an attacker?
11. What is output drift?
12. Name three ingestion validation controls.
13. Why are guardrails on retrieved content imperfect?
14. Why does RAG security require defence in depth?

### Answers

1. Retrieval-Augmented Generation.
2. It converts text into numerical vectors.
3. It stores document embeddings and supports similarity comparisons.
4. No. Similarity measures relevance, not trust, correctness, or safety.
5. Ingestion, retrieval, and context injection.
6. Malicious or misleading retrieved content influences the model during inference, without retraining it.
7. Passive poisoning leaves malicious content in the knowledge base; active manipulation crafts content to rank highly for particular queries.
8. Retrieved content is added to the model's context, where the model may not reliably distinguish instructions from data.
9. Responses can look normal, there may be no visible prompt injection, and logs may show only successful retrieval.
10. An outdated version can remain indexed and be retrieved as if it were current.
11. A gradual change in the system's response behaviour over time.
12. Source review, approval workflows, and ownership/update-history tracking.
13. Instruction-like language is ambiguous and can be rephrased or obfuscated.
14. No single control prevents every attack or failure; multiple controls reduce risk, limit impact, and improve detection.

---

## 9. Practical Security Review Checklist

When assessing a RAG application, use the concepts from this room to ask:

- [ ] Are document sources known and approved?
- [ ] Is there an ingestion review or approval workflow?
- [ ] Are document owners and update histories tracked?
- [ ] Can outdated or revoked documents be removed from the index?
- [ ] Does retrieval consider more than semantic similarity?
- [ ] Is retrieved content clearly separated from system instructions?
- [ ] Are instruction-like patterns in retrieved content reviewed?
- [ ] Are unusual or repeated retrieval patterns monitored?
- [ ] Are generated outputs reviewed for gradual behaviour changes?
- [ ] Are multiple defensive layers in place?

> **Final takeaway:** A RAG system does not just depend on the LLM. It also depends on the quality, trustworthiness, and freshness of the documents it retrieves. Secure the entire path from ingestion to the final generated answer.
