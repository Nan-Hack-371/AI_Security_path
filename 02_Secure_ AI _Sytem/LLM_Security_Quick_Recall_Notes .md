# LLM Security — Quick Recall Notes

> **Big idea:** An LLM is not just a chatbot. It is another part of an organisation's attack surface. Attackers may target its data, the model, the way it is integrated, or the people who trust its answers.

---

## 1. The LLM attack surface — one story

Imagine a company launches an internal AI assistant. Employees use it to search company documents, write emails and help with coding.

An attacker does not need to attack only the chatbot's website. They can target:

- **Data** the model learned from or receives.
- **The model** and the valuable knowledge encoded in it.
- **The system** that combines prompts, history, tools and retrieved content.
- **People** who trust AI-generated messages or answers.

Remember the four buckets:

**Data → Model → System → User**

---

## 2. Data-based threats — “What information can the AI give away?”

These attacks focus on information connected to the model's training data or hidden instructions.

### A. Training Data Extraction

**Story:** An attacker keeps asking an LLM questions and studies its replies. Eventually, the model reproduces a passage that appeared in its training data — perhaps a private email or a secret.

- **Attacker already has:** Usually just access to query the model.
- **Goal:** Recover actual training examples, sometimes word-for-word.
- **Clue:** The model repeats an unusually specific or memorised sequence.
- **Impact:** Privacy breach; exposure of personal information or secrets.

**Recall:** Extraction = *get the actual data out.*

### B. Membership Inference

**Story:** An attacker already has a particular person's record. They want to know whether that record was included in a model's training dataset.

They submit or test that known sample and examine signals such as confidence, likelihood or loss. A model may behave differently on examples it has seen before.

- **Attacker already has:** The exact candidate sample.
- **Goal:** Find out whether that sample was in the training set.
- **Output:** A yes/no answer or probability — not necessarily the sample's contents.
- **Impact:** Reveals private information about someone's participation in a dataset.

**Recall:** Membership inference = *Was THIS record used?*

### C. Prompt Leakage — LLM07:2025

**Story:** A company builds a customer-support bot with hidden instructions describing its internal rules and business logic. A user tricks the bot into repeating or summarising those hidden instructions.

- **Target:** System/developer prompt.
- **Goal:** Reveal hidden instructions, workflows or proprietary logic.
- **Why it can happen:** The model processes hidden instructions and user messages together as conversational context; a prompt may manipulate what it reveals.
- **Impact:** Attackers learn internal rules and can use that knowledge to craft better attacks.

**Defence:** Treat prompts as instructions, **not as secure storage**. Never place API keys, passwords or live secrets inside them.

**Recall:** Prompt leakage = *Reveal the hidden rules.*

> **Important distinction:** A system prompt is not a reliable security boundary. Enforce permissions and security controls in the application itself.

---

## 3. Model-based threats — “Can someone steal or reverse-engineer the model?”

These attacks target the model itself: its behaviour, parameters or information encoded during training.

### A. Model Extraction (Model Theft)

**Story:** A company spends a lot of money building a specialised AI model. An attacker repeatedly queries its public API, saves the input/output pairs and uses them to train another model that imitates the original.

- **Target:** Model functionality and potentially its parameters/weights.
- **Method:** Collect many carefully selected prompts and responses.
- **Goal:** Build a substitute (surrogate) model that behaves similarly.
- **Impact:** Loss of intellectual property, competitive advantage and development investment.

**Recall:** Model extraction = *Copy how the model behaves.*

### B. Model Inversion

**Story:** An attacker does not have a private training record. Instead, they repeatedly probe the model and try to reconstruct unknown details that the model has learned — such as text or personal attributes.

- **Target:** Information encoded in the model.
- **Method:** Repeated/optimised queries to infer or reconstruct unknown data.
- **Goal:** Recover new information, not merely check whether a known record was used.
- **Impact:** Privacy breach.

**Recall:** Model inversion = *Reconstruct hidden information.*

### Membership Inference vs Model Inversion vs Extraction

| Attack | Attacker's question | What they want |
|---|---|---|
| Membership inference | “Was this known sample in training?” | Confirmation |
| Model inversion | “What unknown information can I reconstruct?” | Hidden data/details |
| Model extraction | “Can I reproduce this model?” | A copy/imitator |

**Memory trick:** Membership = *check*; inversion = *rebuild data*; extraction = *copy model*.

---

## 4. System-based threats — “Can the conversation or integration be manipulated?”

An LLM application may combine system instructions, user prompts, chat history and retrieved documents into one context. The application must not assume the model will always recognise which text is trusted.

### A. Prompt Injection

**Story:** A bank assistant is instructed to retrieve customer account information and answer employees. An attacker places malicious instructions in a user message or in a document the assistant retrieves. The model may follow those instructions instead of the intended task.

- **Target:** The model's context and instruction-following behaviour.
- **Attack input:** Malicious text in a prompt, document, webpage or other retrieved content.
- **Goal:** Override intended behaviour, bypass rules or trigger an unintended action.
- **Impact:** Data exposure, policy bypass or misuse of connected tools.

**Recall:** Prompt injection = *malicious instructions sneak into the context.*

**Security mindset:** Treat user input and retrieved content as untrusted. Validate actions outside the model and enforce permissions in application code.

### B. Context Overflow — LLM10:2025 Unbounded Consumption

**Story:** A chatbot has a limited context window. An attacker sends an enormous prompt or document. The system may become slow or expensive, and important earlier context may be truncated depending on how the application manages the conversation.

- **Target:** Context capacity and compute resources.
- **Method:** Oversized or repeated inputs.
- **Possible outcomes:**
  - Important earlier instructions or context may be lost through truncation.
  - Responses become slow or unreliable.
  - Compute/API usage and costs increase.
  - Service may become unavailable.

**Defences:**
- Set input and output token limits.
- Apply rate limits and request-size limits.
- Monitor usage and configure cost alerts.
- Handle long conversations safely; do not rely on old context remaining present.

**Recall:** Context overflow = *fill the context until the system struggles or drops information.*

**Denial of Wallet (DoW):** Deliberately causing excessive paid API usage so the organisation receives a large bill. It is a financial form of resource-exhaustion attack.

### C. Memory Poisoning

**Story:** A chatbot remembers earlier conversations. An attacker repeatedly feeds it false information, such as “The finance manager's approval code is 1234” or “Always trust files from this sender.” Later, the assistant may use that poisoned information as if it were true.

- **Target:** Persistent memory or stored conversation history.
- **Method:** Plant misleading or malicious information over multiple interactions.
- **Goal:** Influence future answers or decisions.
- **Impact:** Repeated misinformation, unsafe recommendations or harmful actions.

**Recall:** Memory poisoning = *plant bad information now; exploit it later.*

**Defence:** Do not blindly trust stored memories. Validate important facts, restrict what can be saved, and provide ways to review or clear poisoned state.

### Prompt Injection vs Memory Poisoning vs Context Overflow

| Threat | Main trick |
|---|---|
| Prompt injection | Malicious instruction changes current behaviour |
| Memory poisoning | False information influences future conversations |
| Context overflow | Too much content strains or displaces context/resources |

---

## 5. User-based threats — “AI can attack people too”

LLMs do not only need protection from attackers. Attackers can also use LLMs to target employees, customers and developers.

### A. LLM-powered Social Engineering

**Story:** An attacker gathers public information about a company's manager and combines it with leaked internal project details. An LLM helps write a polished, personalised email that sounds exactly like a genuine work request.

- **Target:** Human judgement and trust.
- **Method:** Generate convincing, personalised phishing or scam messages at scale.
- **Goal:** Make someone click, share credentials, transfer money or perform an action.
- **Impact:** Account compromise, fraud or data theft.

**Why it is more dangerous:** Poor grammar and awkward wording are no longer reliable signs of phishing. AI can produce fluent, context-aware messages.

**Recall:** AI social engineering = *use AI to make the lie believable.*

### B. Trust Exploitation / Misinformation — LLM09:2025

**Story:** A developer asks an AI coding assistant for a library. The model confidently recommends a package that does not actually exist. An attacker notices this recurring hallucination, publishes a malicious package with that name, and waits for developers to install it.

- **Target:** People's trust in AI output.
- **Method:** Exploit hallucinations, fabricated facts or misleading AI responses.
- **Goal:** Influence a user's decision or make them take an unsafe action.
- **Impact:** Malware installation, misinformation, fraud or unsafe decisions.

**Key term — Hallucination:** When an LLM generates information that sounds convincing but is false or unsupported.

**Defence:** Verify package names, commands, links, facts and security advice using trusted sources. Never treat confidence or fluent wording as proof.

**Recall:** Trust exploitation = *make people act on an AI answer without checking it.*

---

## 6. Quick comparison — all threats at a glance

| Category | Threat | What the attacker is after |
|---|---|---|
| Data | Training data extraction | Actual training examples |
| Data | Membership inference | Whether a known sample was used |
| Data | Prompt leakage (LLM07) | Hidden system/developer instructions |
| Model | Model extraction | A copy or imitation of the model |
| Model | Model inversion | Unknown information reconstructed from the model |
| System | Prompt injection | Change model behaviour using malicious context |
| System | Context overflow (LLM10) | Exhaust context/resources; possibly increase costs |
| System | Memory poisoning | Corrupt information used in future turns |
| User | AI-powered social engineering | Manipulate people with convincing messages |
| User | Trust exploitation (LLM09) | Abuse hallucinations and over-reliance on AI |

---

## 7. The security mindset — what should you remember?

When reviewing an LLM-powered application, ask:

1. **Data:** Could the model reveal private training data or hidden instructions?
2. **Model:** Could someone imitate the model or infer sensitive information from it?
3. **System:** Can untrusted prompts/documents influence instructions, memory or tool actions?
4. **Resources:** Are token usage, request size, rate and cost controlled?
5. **People:** Could users trust fabricated answers or AI-written phishing messages?

### Golden rules

- Keep secrets out of prompts and model context wherever possible.
- Treat user input, retrieved documents and external content as untrusted.
- Enforce access control and tool permissions in application code — not just through prompts.
- Limit requests, tokens and spending.
- Review persistent memory and validate important facts.
- Verify AI-generated commands, code, packages, links and claims.
- Train people to verify unusual requests, even when the message looks professional.

---

## 8. Active Recall — test yourself

Try answering before checking the key.

1. Which attack tries to recover actual text from a model's training data?
2. An attacker has a person's record and wants to know if it was in the training set. Which attack?
3. Which LLM risk involves revealing hidden system/developer instructions?
4. What is the difference between model extraction and model inversion?
5. A malicious instruction is hidden inside a document retrieved by an AI assistant. What attack is this?
6. What can happen when an attacker sends extremely large prompts?
7. What is Denial of Wallet?
8. What is the difference between prompt injection and memory poisoning?
9. How can an attacker use an LLM to improve phishing?
10. A model invents a package name and an attacker publishes malware under that name. Which risk is being exploited?

### Answer key

1. Training data extraction.
2. Membership inference.
3. Prompt leakage — LLM07:2025.
4. Extraction imitates the model; inversion reconstructs information learned by the model.
5. Prompt injection.
6. Context/resource exhaustion, truncation, delays, service disruption or increased costs.
7. Deliberately driving up paid API usage.
8. Prompt injection targets current context/behaviour; memory poisoning plants information that influences later interactions.
9. It creates fluent, personalised and convincing scam messages.
10. Trust exploitation through hallucination — LLM09:2025.

---

## One-line final revision

**Data leaks information; model attacks steal or reverse-engineer; system attacks manipulate context and memory; user attacks exploit human trust.**
