# Prompt Injection — TryHackMe Notes

## 1. How LLMs Follow Instructions

An LLM receives tokens and predicts what token should come next. Its context may contain:

| Component | Meaning |
|---|---|
| System prompt | High-level instructions that define the assistant's role and boundaries |
| Developer prompt | Application-level instructions set by the developer |
| User prompt | The user's request |
| Retrieved context | Information brought in from documents, websites, or a knowledge base |
| Tool output | Results returned by tools the model can use |

The model must distinguish instructions from data. If untrusted data is treated as an instruction, it may influence the model's behaviour.

### Instruction-separation methods

- **ChatML:** A format that marks message boundaries and roles.
- **Harmony:** A structured format for representing messages and control flow.
- **System prompts and application rules:** Define expected behaviour.
- **Input filtering and metadata:** Can help identify or separate untrusted content.

These methods can improve structure, but they do not automatically eliminate prompt injection.

## 2. The Reality of the Token Stream

- Text is converted into tokens.
- The model processes the token sequence as context.
- It predicts the next token based on patterns learned during training.
- It may struggle to distinguish trusted instructions from malicious text included in the same context.

**Remember:** Clear boundaries and application-side controls are important because different parts of a conversation become input to the model.

## 3. What Is Prompt Injection?

**Prompt injection** is an attack in which someone crafts input that manipulates an LLM into ignoring or changing its intended instructions.

### Simple example

Imagine a translation chatbot whose task is to translate text into French. An attacker submits text that says to ignore the translation task and reveal hidden instructions. If the application does not separate instructions from untrusted content, the model may follow the attacker's text.

### Prompt injection vs. SQL injection

| Prompt injection | SQL injection |
|---|---|
| Targets an LLM's instruction-following behaviour | Targets database query handling |
| Uses crafted natural-language or structured content | Uses crafted SQL syntax |
| May manipulate responses or tool use | May manipulate database operations |
| Needs controls around instructions, data, and tool permissions | Needs parameterised queries and safe query handling |

## 4. Why Prompt Injection Matters

The impact depends on what the LLM-powered application can access or do.

A basic chatbot might produce an incorrect response. An AI agent with access to email, files, internal documents, APIs, or other tools could create a much bigger risk.

Potential impacts include:
- Revealing information included in the model's context.
- Producing unauthorised or misleading responses.
- Misusing connected tools.
- Accessing or exposing sensitive data available to the application.
- Triggering actions that the application allows the model to perform.

**Key lesson:** The more permissions and access an AI application has, the more important its security controls become.

## 5. Real-World Examples Mentioned in the Room

### Bing Chat / Sydney (2023)
The room discusses prompt manipulation involving Bing Chat, also known as Sydney, as an example of attempts to expose or influence hidden behaviour and instructions.

### Remoteli.io Twitter bot (2022)
The room describes a case where users were able to manipulate an AI-powered Twitter bot through crafted replies.

### Chevrolet dealership chatbot (2023)
The room mentions a chatbot that was manipulated into making an unusual offer involving a Chevrolet Tahoe for $1.

**Takeaway:** Public-facing AI systems can be manipulated through the content they process. Their responses and actions need appropriate limits.

## 6. Prompt Injection Techniques

### 6.1 Synonym and paraphrasing
An attacker rewrites an instruction with different wording rather than using an obvious phrase. Simple filters that look for exact words may miss differently phrased instructions.

### 6.2 Format-based injection
An attacker formats content so that it resembles a system message, configuration, or other trusted-looking structure. The application should clearly identify this content as untrusted data.

### 6.3 Simulated dialogue
An attacker includes a fake conversation or role-play sequence that attempts to steer the model into following a different set of instructions.

### 6.4 Multi-turn prompt shaping
An attacker uses multiple messages to gradually steer the model towards unintended behaviour. Earlier messages can influence later responses because conversation history is part of the context.

## 7. Indirect Prompt Injection

**Direct prompt injection** comes directly from the user's input.

**Indirect prompt injection** comes from external content that the AI system reads or processes, such as a webpage, email, document, or retrieved knowledge-base entry.

### Example
1. An AI assistant is asked to summarise a webpage.
2. The webpage contains ordinary information plus malicious instructions.
3. The assistant retrieves and processes the page.
4. If the application fails to treat the page as untrusted content, those instructions may influence the assistant.

### Possible sources
- Webpages
- Emails
- Uploaded documents
- Retrieved documents in a RAG system
- External content consumed by AI agents
- Tool outputs and connected services

### Direct vs. indirect prompt injection

| Direct | Indirect |
|---|---|
| Malicious instructions supplied directly by the user | Malicious instructions contained in external content |
| Enters through user input | Enters through an external data source |
| Example: a crafted chatbot message | Example: hidden instructions in a webpage being summarised |

### Zero-click prompt injection
A zero-click scenario is one where the victim does not need to deliberately submit the malicious instruction. The AI system may encounter it automatically while processing external content.

The room also references research examples such as **EchoLeak** and **Cursor** when discussing indirect prompt injection.

## 8. Important Terms — Quick Revision

| Term | Simple meaning |
|---|---|
| LLM | Large Language Model; an AI model trained to process and generate language |
| Token | A small unit of text processed by a language model |
| Context window | The information the model can consider during a request |
| System prompt | High-level instructions supplied by the application |
| Prompt injection | Crafted input that attempts to manipulate the model's instructions or behaviour |
| Direct injection | Malicious instructions supplied directly by the user |
| Indirect injection | Malicious instructions contained in external content the model processes |
| RAG | Retrieval-Augmented Generation; retrieving relevant information and giving it to an LLM |
| AI agent | An AI system that can use tools or take actions to complete tasks |
| Zero-click | An attack scenario that does not require the victim to manually submit the malicious input |

## 9. Security Testing Checklist

- [ ] Identify the system's intended role and instructions.
- [ ] Identify all input sources, including retrieved documents and tool outputs.
- [ ] Check whether untrusted content is clearly separated from trusted instructions.
- [ ] Test whether paraphrased or differently formatted malicious content changes the model's behaviour.
- [ ] Test whether multi-turn conversations can steer the model away from its intended task.
- [ ] Check whether external webpages, emails, or documents can influence the model.
- [ ] Review what tools and data the model can access.
- [ ] Verify that sensitive actions require appropriate authorisation outside the model.
- [ ] Check that tool permissions follow least privilege.
- [ ] Confirm that sensitive information is not unnecessarily included in the model's context.

Only perform testing on systems you own or have explicit permission to assess.

## 10. Memory Tricks

- **Direct = delivered by the user.**
- **Indirect = hidden in information the AI reads.**
- **Prompt injection = manipulating instructions through input.**
- **Context window = the model's working space.**
- **Least privilege = give the AI only the access it needs.**

### One-line summary

Prompt injection is a risk created when an LLM or AI application does not reliably distinguish trusted instructions from untrusted content. The risk becomes greater when the application can access sensitive information or use powerful tools.
