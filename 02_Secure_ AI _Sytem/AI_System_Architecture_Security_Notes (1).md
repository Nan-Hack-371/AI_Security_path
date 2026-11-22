# AI System Architecture & Security --- Revision Notes

**TryHackMe AI Security Path \| Case study: TryTrainMe's TryAssist**

> **Story to remember:** TryTrainMe builds an AI code-review assistant.
> Developers see a chat box, but behind it the assistant reads
> documents, calls internal APIs, uses tools and stores conversations.
> Every connection is a possible entry point or data-leak path. The
> security architect must secure the whole journey---not just the chat
> window.

## 1. Introduction --- The Attack Surface Gets Bigger

Before TryAssist, TryTrainMe mainly secured its web application, APIs,
database and authentication layer. TryAssist adds natural-language
input, document retrieval, CI/CD and repository integrations, tool
calls, and conversation logs.

A developer might paste source code, credentials, private keys or
internal architecture into the chat. That information may be stored or
sent to an external model. **No hacker or software exploit is required
for accidental data leakage.**

**Samsung example (2023):** Engineers pasted proprietary source code and
internal meeting notes into ChatGPT. The lesson: ordinary use of an AI
tool can expose confidential information.

**Remember:** The chat box is only the front door. The real attack
surface includes every component and connection behind it.

## 2. Anatomy of an AI System

Traditional application flow:

`User → Web app → API → Database → Response`

An AI system adds components and new data flows. The big change is
**unstructured input**: users can type almost anything, rather than
selecting a valid date or number. Traditional fixed-field validation is
no longer enough.

### TryAssist's nine main components

1.  **User interface / chat box** --- developer submits a question.
2.  **API gateway** --- authenticates requests and applies rate limits.
3.  **Orchestration layer** --- manages conversation history and routes
    work.
4.  **Prompt construction** --- combines system instructions, user input
    and retrieved context.
5.  **LLM** --- interprets the prompt and generates a response.
6.  **Retrieval / vector store** --- supplies relevant internal
    documents or context.
7.  **Tool layer** --- allows requests such as fetching CI/CD or
    repository data.
8.  **Output processing** --- checks, filters and formats the response.
9.  **Logging system** --- records conversations and activity.

### Follow one request

A developer asks: **"Does this pull request handle authentication
correctly?"**

`Developer → Gateway → Orchestrator → Prompt builder → LLM`

The prompt builder combines the question, system rules and relevant
documents. The LLM analyses them and may request a tool call:

`LLM → Tool layer → Internal API → Tool result → LLM`

Finally:

`LLM → Output checks → Developer`

The conversation is also recorded in logs.

### Trust boundaries

A **trust boundary** is where data or control moves between components
with different trust levels.

Examples: - Developer/browser → API gateway - Gateway → internal
services - Retrieved documents → prompt - LLM → tool layer/internal
APIs - AI output → downstream application or logs

Every boundary needs suitable security controls. An unprotected boundary
may let an attacker reach something more sensitive.

## 3. Five System-Level Threats --- OWASP LLM Top 10 (2025)

This room focuses on five risks caused mainly by how an AI application
is **built and integrated**, rather than only by the model's internal
behaviour.

### 3.1 LLM10 --- Unbounded Consumption

**Story:** An attacker sends hundreds of requests per minute, each
asking TryAssist to analyse a huge codebase. The model consumes compute
and tokens; the bill can skyrocket and the service may become
unavailable.

-   **Meaning:** Excessive request volume or oversized inputs exhaust
    resources or increase costs.
-   **TryAssist risk:** No per-user quotas or input-size limits.
-   **Defence:** Rate limiting, input-length limits, per-user quotas,
    usage monitoring and cost ceilings at the API gateway.

**Recall:** Too many requests + too much text = resource/cost
exhaustion.

### 3.2 LLM07 --- System Prompt Leakage

**Story:** An attacker asks TryAssist to reveal its hidden instructions.
The prompt may disclose internal tool names, API addresses, database
details or behavioural rules.

-   **System prompt:** Developer-defined instructions guiding the
    model's role and behaviour.
-   **Risk:** Exposed instructions help an attacker understand the
    system and plan further attacks.
-   **TryAssist example:** The prompt contains internal CI/CD API
    addresses and database schema information.
-   **Defence:** Never put passwords, API keys or secrets in prompts.
    Assume prompts may be discovered. Protect actual resources with
    authentication and authorisation.

**Recall:** A system prompt is guidance---not a secret vault or security
boundary.

### 3.3 LLM05 --- Improper Output Handling

**Story:** TryAssist generates text containing SQL syntax. Another
component inserts that text directly into a database query. The database
executes it. The failure occurs because the application trusted and
mishandled the model's output.

-   **Meaning:** LLM output is passed to another component without safe
    validation or handling.
-   **TryAssist example:** A malicious string in a pull request appears
    in generated output and reaches a database query unsafely.
-   **Defences:** Parameterised SQL, schema validation, safe encoding,
    and never building SQL, shell commands or HTML by concatenating raw
    model output.

**Important distinction:** - Chevrolet chatbot agreeing to sell a car
for \$1 → prompt-injection-related behaviour, not LLM05 by itself. - Air
Canada chatbot inventing a refund policy → misinformation, not LLM05 by
itself. - **LLM05 requires unsafe downstream handling**, such as
generated content being executed or interpreted by another system.

**Recall:** Model output is data. Do not accidentally turn it into
executable instructions.

### 3.4 LLM06 --- Excessive Agency

**Story:** TryAssist only needs to review code, but it can update
databases, delete records and deploy software. A manipulated request or
response could trigger actions beyond its job.

Three forms:

  -----------------------------------------------------------------------
  Form                    Meaning                 Example
  ----------------------- ----------------------- -----------------------
  **Excessive             Too many tools/features Code reviewer can
  functionality**                                 deploy to production

  **Excessive             Tools have too much     Database tool has
  permissions**           access                  UPDATE/DELETE instead
                                                  of read-only

  **Excessive autonomy**  Acts without enough     Automatically approves
                          human oversight         and merges pull
                                                  requests
  -----------------------------------------------------------------------

-   **Defences:** Least privilege, read-only by default, scoped API
    tokens, allowlisted tools, and human approval for
    write/delete/deploy actions.

**Recall:** Don't give an assistant the keys to the whole building when
it only needs one room.

### 3.5 LLM02 --- Sensitive Information Disclosure

**Story:** A developer pastes a private SSH key into TryAssist. The full
conversation is stored in an unencrypted database that too many
employees can access. No exploit was needed.

-   **Meaning:** Confidential data is exposed through responses,
    prompts, retrieval, logs or connected services.
-   **TryAssist risk:** Credentials, source code and internal details
    are stored in accessible logs or sent to an external model API.
-   **Defences:** Minimise/redact sensitive data before logging, encrypt
    stored conversations, restrict access, set retention limits and
    review external provider data handling.

**Recall:** Users can accidentally leak data simply by using the AI
system.

### Threats at a glance

  -----------------------------------------------------------------------
  OWASP category          Memory phrase           Main defence
  ----------------------- ----------------------- -----------------------
  **LLM10 Unbounded       Too much usage          Rate limits, quotas,
  Consumption**                                   cost caps

  **LLM07 System Prompt   Hidden instructions     No secrets in prompts;
  Leakage**               exposed                 secure real resources

  **LLM05 Improper Output Unsafe output gets used Validate, parameterise,
  Handling**                                      encode

  **LLM06 Excessive       AI has too much power   Least privilege, tool
  Agency**                                        limits, human approval

  **LLM02 Sensitive       Private data escapes    Minimise, redact,
  Information                                     encrypt, restrict
  Disclosure**                                    access
  -----------------------------------------------------------------------

### CIA Triad connection

-   **Confidentiality:** Sensitive information disclosure and prompt
    leakage.
-   **Integrity:** Improper output handling or excessive agency can
    alter data or systems.
-   **Availability:** Unbounded consumption can exhaust resources or
    make the service unavailable.

AI security is not only about keeping information secret. It also means
preventing unauthorised changes and keeping services available.

## 4. Secure Design Patterns --- Build Defences Before Launch

**Story:** TryTrainMe does not rely on one filter. It adds several
checkpoints. If an attack passes one checkpoint, another should still
stop it.

This is **defence in depth**: multiple security layers placed across the
system and its trust boundaries.

### 4.1 Least Privilege

Give every component only the permissions it needs.

-   **Database:** Read-only by default. Grant write access only when
    justified.
-   **API tokens:** Scope tokens to the exact required endpoints; avoid
    admin/root tokens.
-   **Tool allowlisting:** The model can call only explicitly registered
    tools. Block and log unapproved calls.
-   **Human-in-the-loop:** Require approval before state-changing
    actions such as deploying code, deleting records or sending
    communications.

**Memory:** Minimum access + approved tools + human approval for risky
actions.

### 4.2 Input and Output Validation

AI accepts free-form text, but validation still matters.

**At the input boundary:** - Enforce maximum input length. - Apply rate
limits. - Flag suspicious or injection-like patterns. - Treat user input
and retrieved documents as untrusted content.

**At the output boundary:** - Never pass raw model output directly into
SQL, shell commands or HTML. - Extract only expected structured
fields. - Validate values against a defined schema. - Use safe APIs and
parameterised queries.

Pattern filters can miss new prompt-injection techniques. They are one
layer, not a complete solution.

### 4.3 Monitoring and MLSecOps

Security controls try to prevent attacks. **Monitoring** helps detect
suspicious behaviour that gets through.

Monitor: - Unusual request volume, input size and token/cost usage. -
Repeated attempts to reveal system prompts. - Unexpected tool calls or
denied permissions. - Sensitive data appearing in prompts or logs. -
Model behaviour changes, failures and unusual outputs.

**MLSecOps (Machine Learning Security Operations)** integrates security
throughout the ML lifecycle---from development and testing to deployment
and live operations.

It applies **shift left**: identify and address security risks early,
rather than bolting controls on after launch.

Traditional question: "Is the application secure?"\
MLSecOps also asks: "Is the model behaving as expected, and is the
system protected from misuse?"

## 5. Secure TryAssist --- End-to-End Flow

``` text
Developer submits request
          ↓
API gateway
Authentication + rate limits + input-size limits
          ↓
Orchestration
Access control + safe conversation handling
          ↓
Prompt construction / retrieval
Treat user input and retrieved documents as untrusted
          ↓
LLM
No secrets in system prompt
          ↓
Tool layer
Allowlisted tools + least privilege + scoped tokens
          ↓
Human approval for write / delete / deploy
          ↓
Output processing
Schema validation + safe encoding + no raw execution
          ↓
Developer receives response
          ↓
Secure logging
Redaction + encryption + restricted access + monitoring
```

Controls work together. For example, a malicious prompt might evade an
input filter, but the tool layer can still block an unauthorised action
because the model does not have permission to perform it.

## 6. Quick Revision

  -----------------------------------------------------------------------
  Term                                Recall
  ----------------------------------- -----------------------------------
  Attack surface                      Every component or entry point an
                                      attacker could target

  Trust boundary                      Where data/control crosses between
                                      trust zones

  Unstructured input                  Free-form text; fixed-field
                                      validation alone is insufficient

  API gateway                         Entry checkpoint: authentication,
                                      rate limits, quotas

  Orchestration                       Coordinates conversation and routes
                                      work

  Prompt construction                 Combines instructions, user request
                                      and retrieved context

  Tool layer                          Connects model to actions and
                                      internal services

  Defence in depth                    Multiple security checkpoints

  Least privilege                     Only the access needed---nothing
                                      extra

  Tool allowlisting                   Only approved functions can be
                                      called

  Human-in-the-loop                   Human approves sensitive actions

  MLSecOps                            Security throughout the ML
                                      lifecycle

  Shift left                          Address security early in
                                      design/development
  -----------------------------------------------------------------------

## 7. Active Recall

1.  Why does adding an AI assistant increase an application's attack
    surface?
2.  What is a trust boundary? Give two TryAssist examples.
3.  Which OWASP risk describes flooding the model with huge requests?
4.  Why should a system prompt never contain API keys or secrets?
5.  What makes a failure an example of improper output handling?
6.  What are the three forms of excessive agency?
7.  How can an ordinary developer conversation cause sensitive
    information disclosure?
8.  What is the difference between input and output validation?
9.  Explain defence in depth using TryAssist.
10. What does MLSecOps mean, and what is shift left?

```{=html}
<details>
```
```{=html}
<summary>
```
Answer key
```{=html}
</summary>
```
1.  It adds prompt construction, retrieval, LLM, tools, internal APIs
    and logging paths.
2.  A point where data/control crosses trust zones; e.g. browser to
    gateway, or LLM to tool layer.
3.  LLM10 --- Unbounded Consumption.
4.  Prompts may be extracted; secrets need real access controls, not
    hidden instructions.
5.  Generated content is passed to another system and handled unsafely,
    e.g. executed as SQL.
6.  Excessive functionality, excessive permissions and excessive
    autonomy.
7.  A developer may paste a private key or source code that is stored in
    logs or sent to an external API.
8.  Input validation checks incoming data; output validation checks
    generated data before another component uses it.
9.  Multiple checkpoints: gateway limits, safe prompts, restricted
    tools, human approval, output validation and secure logging.
10. MLSecOps integrates security throughout the ML lifecycle; shift left
    means addressing risks early.

```{=html}
</details>
```

------------------------------------------------------------------------

**Final memory line:**
`Map components → mark trust boundaries → identify threats → apply layered controls → monitor continuously.`
