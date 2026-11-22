# AI Models & Data --- Revision Notes

**TryHackMe AI Security Path \| Room 03**

> 🧠 **Main story:** Building AI is like hiring a chef. **Data** =
> ingredients; **training** = cooking; **base model** = a recipe
> inherited from another chef; **fine-tuning** = adapting the recipe;
> **model card** = food label. Bad ingredients or a secret recipe can
> create trouble even if the finished dish looks fine.

------------------------------------------------------------------------

## 1. Why Data Is a Security Issue

AI learns patterns from what it sees. Before asking "Is this model
safe?", ask: - **Where did its data come from?** - **Who cleaned and
labelled it?** - **Could someone have tampered with it?** - **Did it
include private information or secrets?**

**Real-life example:** A phishing detector learns from emails. If its
dataset contains incorrectly labelled phishing messages, it can learn to
trust dangerous mail.

**Recall:** `Bad data → Bad learning → Risky decisions.`

## 2. Four Training Data Sources

  ------------------------------------------------------------------------
  Source            Easy explanation  Example           Main security
                                                        concern
  ----------------- ----------------- ----------------- ------------------
  **Web scraping**  Automated         Blogs, forums,    Unknown quality,
                    collection from   articles          permissions,
                    websites                            hidden malicious
                                                        content

  **Licensed        Data acquired     A platform        Licence terms,
  datasets**        under an          licenses a text   consent, permitted
                    agreement         archive           uses

  **Synthetic       Artificially      An LLM creates    Repeated errors,
  data**            generated         sample customer   bias, unrealistic
                    examples          conversations     examples

  **Internal        Organisation's    Support chats,    PII,
  corpora**         own documents     knowledge base    confidentiality,
                                                        access controls
  ------------------------------------------------------------------------

**Fun analogy:** Web = public buffet; licensed = ingredients bought from
a supplier; synthetic = imitation ingredients made in a lab; internal =
food from your own kitchen. **None is automatically safe.**

**Common Crawl:** A large, publicly accessible web-crawl archive used in
many language-model training pipelines. Publicly available **does not
mean** verified, legally unrestricted, or free of secrets.

**Note on room statistics:** Figures about dataset sizes, percentages
and individual model training mixes are historical or source-dependent;
don't memorise them as universal rules.

## 3. Data Provenance --- "Ingredient Traceability"

**Definition:** A record of a dataset's origin and history.

Three questions: 1. **Where** did it come from? 2. **When** was it
collected? 3. **What changed** since collection?

**Example:** Your company downloads a "safe emails" dataset. Without
provenance, you cannot tell whether someone replaced samples or changed
labels last week.

**Security benefit:** Helps detect suspicious changes, assess legal
permissions, trace errors and respond to a compromised upstream source.

### SBOM vs ML-BOM

  -------------------------------------------------------------------------
  Term                    Think of it as          Contents
  ----------------------- ----------------------- -------------------------
  **SBOM** (Software Bill Software ingredient     Software
  of Materials)           list                    components/dependencies

  **ML-BOM** (Machine     AI ingredient list      Model/dataset components,
  Learning Bill of                                versions, sources,
  Materials)                                      licences and relevant
                                                  pipeline details
  -------------------------------------------------------------------------

**Memory hook:** No ingredient list = you don't know what you're
serving.

## 4. PII in Training --- "Secrets Mixed Into the Batter"

**PII** = Personally Identifiable Information, e.g. names combined with
contact details, personal records or identifiers.

**Problem:** Large datasets may contain personal messages, credentials
or other sensitive content. A model can sometimes **memorise and
reproduce** specific examples. This is a risk, not a guarantee that all
training records are stored verbatim.

**Real-life example:** A scraped webpage accidentally exposes an API
key. It enters a dataset; the credential may be copied into multiple
downstream systems and, in some circumstances, surfaced again.

**Controls:** - Minimise collection; don't train on unnecessary
sensitive data. - Scan and remove credentials and PII where
appropriate. - Track source, permission, retention and access. - Rotate
exposed credentials; don't rely on a model "forgetting" them. - Test
outputs for leakage and restrict retrieval.

**GDPR connection:** Data minimisation means collecting/processing only
personal data necessary for a specified purpose.

**Remember:** Once a secret enters the training pipeline, cleanup is
harder than preventing ingestion.

------------------------------------------------------------------------

## 5. Training Concepts: Epoch, Overfitting, Validation

### Epoch --- "One full revision of the textbook"

An **epoch** is one complete pass through the training dataset. Multiple
epochs let the model adjust its parameters repeatedly.

**Example:** If the dataset contains 1,000 examples, seeing all 1,000
once = one epoch (even if processed in smaller batches).

**Important:** More epochs ≠ automatically better performance.

### Overfitting --- "Memorising exam answers"

A model learns its training examples too closely and fails to generalise
to new cases.

**Example:** A student memorises last year's exam paper and gets 100%,
but struggles when the teacher changes the questions. A spam model
similarly recognises familiar spam but misses new wording.

**Security angle:** Poor generalisation can miss new attacks;
memorisation of specific training content may also contribute to privacy
risks. Overfitting and privacy leakage are related risks, **not the same
thing**.

### Validation --- "Mock test before the real exam"

Split data into separate roles:

  Dataset          Purpose                                 Student analogy
  ---------------- --------------------------------------- ------------------
  **Training**     Update model parameters                 Studying lessons
  **Validation**   Check progress and choose settings      Mock exam
  **Test**         Final evaluation on held-out examples   Final exam

**Overfitting warning sign:** Training performance improves while
validation performance stops improving or gets worse.

**Security check:** Use trustworthy, representative data; prevent data
leakage between splits; test against relevant unusual and adversarial
inputs. Validation reduces uncertainty but cannot prove a model is
secure.

**Mini flow:**
`Train → Check validation → Tune/stop → Final test → Deploy/monitor`.

------------------------------------------------------------------------

## 6. Post-Training Optimisation: Pruning vs Quantisation

A model may be compressed so it runs on cheaper or smaller hardware.

  ------------------------------------------------------------------------
  Technique          Simple meaning    Analogy           What to check
  ------------------ ----------------- ----------------- -----------------
  **Pruning**        Remove less       Trim branches     Accuracy, edge
                     useful weights,   from a tree       cases, security
                     connections or                      behaviour
                     structures                          

  **Quantisation**   Represent         Round precise     Accuracy, safety
                     numerical values  measurements to   tests, robustness
                     using lower       fewer digits      after conversion
                     precision                           
  ------------------------------------------------------------------------

**Example:** Reducing model weights from a high-precision representation
to a lower-bit representation can reduce memory needs. The exact format
varies; it is not always "32-bit float → 8-bit float."

**Security lesson:** Compression may change behaviour, including
responses to risky prompts. Test the **actual deployed compressed
version**, not only the original.

**Memory:**
`Pruning = remove pieces | Quantisation = use smaller numbers.`

## 7. Federated Learning --- "Group Project Without Sharing Notebooks"

Instead of sending everyone's raw data to one central location,
participants train locally and send model updates for aggregation.

``` text
Hospital A: local patient data → local training → update ─┐
Hospital B: local patient data → local training → update ─┼→ Aggregator → global model
Hospital C: local patient data → local training → update ─┘
```

**Benefit:** Raw records can stay with their owner, reducing the need to
centralise them.

**New risks:** - A malicious participant may submit **poisoned updates**
to distort the shared model. - Model updates themselves may leak
information in some settings. - The aggregator and participating clients
must be trusted or protected appropriately.

**Defences:** Participant authentication, robust aggregation, update
anomaly checks, secure aggregation/privacy techniques where suitable,
and monitoring.

**Fun analogy:** Everyone studies at home and submits their "learning
summary"; one student can deliberately submit wrong answers and spoil
the group's final notes.

**Remember:**
`Raw data stays local ≠ automatically private or poisoning-proof.`

------------------------------------------------------------------------

## 8. Pre-trained Models & Fine-tuning --- "Inherited Recipe"

### Pre-trained model

A model that has already learned broad patterns from a large general
dataset. You start with existing capability instead of training from
zero.

### Fine-tuning

Additional training on a smaller, more specific dataset to adapt
behaviour, style or domain performance.

**Example:** A company adapts a general language model to understand its
technical support terminology.

**Important technical distinction:** - **Full fine-tuning:** Updates
some or all original model weights. - **Parameter-efficient fine-tuning
(e.g. LoRA):** Typically freezes base weights and trains
additional/smaller parameters.

So it is **not universally true** that fine-tuning never changes base
weights. Either way, fine-tuning does **not guarantee removal** of
inherited risks.

### Inheritance Problem --- "A House With Hidden Plumbing"

When you build on someone else's model, you inherit dependencies on
its: - Original training data and provenance - Biases and limitations -
Safety behaviour - Model version/checkpoint - Potential hidden malicious
or unexpected behaviour

**Real-life example:** You customise a third-party model for an internal
helpdesk. It performs well on company FAQs, but the base model's
undocumented limitations and risks still need assessment.

### Three key inheritance risks

**A. Safety alignment erosion** - **Meaning:** Further training may
weaken previously learned safety behaviour, even when the new task seems
harmless. - **Analogy:** A driver learns road safety, then gets
repeatedly rewarded for speed; old habits may weaken. - **Defence:** Run
safety evaluations before and after fine-tuning.

**B. Specialisation can introduce new weaknesses** - **Meaning:** Domain
tuning changes how a model responds; some tuned models can become more
susceptible to particular attacks, including prompt injection. -
**Analogy:** An expert in finance might pay extra attention to messages
framed as financial instructions---even when the source is untrusted. -
**Defence:** Test attacks relevant to the exact deployment and its
domain.

**C. Version/checkpoint inheritance** - **Meaning:** Your tuned model
depends on a particular original model version. - **Example:** A
backdoor is discovered in base checkpoint v1; you must know whether your
model descended from v1. - **Defence:** Record exact checkpoint IDs,
hashes, model sources, licences, tuning data and transformations.

**Recall:** Fine-tuning = specialising an existing model, **not washing
it clean**.

------------------------------------------------------------------------

## 9. Black Box Problem --- "A Sealed Machine"

A model may contain billions of learned numerical values, but those
weights don't provide a straightforward human-readable explanation of
every decision.

**Example:** A fraud detector flags a transaction. We can test its
response and investigate which inputs influenced it, but cannot simply
open the weights and read a clear rule saying "I flagged this
because...".

### What security teams CAN do

-   Benchmark on representative data.
-   Red-team with adversarial prompts and unusual inputs.
-   Inspect the data pipeline, source, access and version history.
-   Monitor behaviour after deployment.
-   Use interpretability tools when suitable.

**Limit:** Passing tests proves behaviour on **tested cases**, not every
possible future input.

## 10. Model Cards --- "Nutrition Label for AI"

A **model card** documents what a model is for, how it was evaluated and
what users should know before deploying it.

  -----------------------------------------------------------------------
  Section                             Question to ask
  ----------------------------------- -----------------------------------
  Training data                       Where did the data come from? What
                                      is disclosed?

  Intended use                        What should this model be used for?
                                      What should it NOT do?

  Evaluation results                  How well did it perform, and on
                                      which benchmarks?

  Known limitations                   When does it struggle?

  Bias assessment                     Are some groups or conditions
                                      affected differently?

  Licence                             What use and distribution are
                                      permitted?
  -----------------------------------------------------------------------

**Analogy:** Before buying packaged food, read ingredients, allergens
and expiry date. Before deploying AI, read the model card.

### Documentation red flags

-   Missing or vague dataset sources
-   No clear evaluation method
-   Unreported limitations or licence
-   No version/checkpoint details
-   Tests only on the original model, not its tuned/compressed
    derivative

**Important nuance:** A model card is useful transparency documentation,
**not a security certificate**. Legal documentation requirements depend
on jurisdiction and use case; don't assume cards are always optional or
always mandatory.

------------------------------------------------------------------------

## 11. One-Page Security Checklist

Before deploying a third-party model, ask:

-   [ ] Do we know dataset origin, licences, sensitive-data risks and
    filtering?
-   [ ] Do we know the exact base model, checkpoint and version?
-   [ ] Was training/validation/test separation done correctly?
-   [ ] Was the final **fine-tuned or compressed** model retested?
-   [ ] Are privacy leakage and prompt-injection risks assessed?
-   [ ] Are federated participants/updates secured, if applicable?
-   [ ] Is there a useful model card or equivalent documentation?
-   [ ] Are access controls, logging, monitoring and incident response
    ready?

## 12. Fast Comparison Table

  Term                 One-line recall
  -------------------- ---------------------------------------------------------
  Data provenance      Origin + collection time + change history
  ML-BOM               Inventory of ML supply-chain components
  PII                  Information that can identify a person
  Epoch                One full pass over training data
  Overfitting          Memorises examples, fails on new cases
  Validation set       Mock test during development
  Pruning              Remove less useful model parts
  Quantisation         Lower numerical precision
  Federated learning   Train locally, combine updates
  Pre-trained model    Start with someone else's learned base
  Fine-tuning          Adapt a model for a narrower purpose
  Checkpoint           Specific saved model state/version
  Black box            Behaviour not straightforward to interpret from weights
  Model card           Model's documentation/nutrition label

## 13. 30-Second Memory Story

You hire a chef (AI model). Check the **ingredients' origin**
(provenance), don't mix in **customer secrets** (PII), let the chef
practise the whole recipe (**epochs**) but don't let them memorise only
one dish (**overfitting**). Give a **mock cooking test** (validation).
Trim unnecessary steps (**pruning**) or use simpler measurements
(**quantisation**). Multiple kitchens can train together without
exchanging raw recipes (**federated learning**), but one kitchen might
send bad updates. If you inherit another chef's recipe (**pre-trained
model**) and customise it (**fine-tuning**), you inherit hidden issues
too. The finished meal is hard to reverse-engineer (**black box**), so
demand the **nutrition label** (**model card**).

## 14. Active Recall --- Test Yourself

1.  Name four sources of training data.
2.  What three questions does provenance answer?
3.  Why is public web data not automatically safe?
4.  What's the difference between an epoch and overfitting?
5.  How do training, validation and test datasets differ?
6.  Pruning vs quantisation?
7.  Why can federated learning still be attacked?
8.  Does fine-tuning always leave base weights unchanged?
9.  Why is tracking a base checkpoint important?
10. What can and can't a model card prove?

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
1.  Web scraping, licensed, synthetic, internal corpora.
2.  Where from, when collected, whether/how modified.
3.  It can contain poor-quality content, PII, secrets or restricted
    material.
4.  Epoch = full dataset pass; overfitting = poor generalisation after
    learning examples too closely.
5.  Training updates weights; validation guides development; test gives
    final held-out evaluation.
6.  Remove less useful parts vs lower numerical precision.
7.  Malicious updates and possible information leakage.
8.  No: full fine-tuning may update them; parameter-efficient methods
    often freeze them.
9.  To trace inherited risks, versions and incidents.
10. It documents intended use/evidence/limitations; it cannot guarantee
    safety or complete transparency.

```{=html}
</details>
```

------------------------------------------------------------------------

**Final recall:**
`Data source → Provenance & privacy → Training & validation → Compression → Inherited base model → Deployment tests → Documentation & monitoring`.

**Further reading:** [Common Crawl](https://commoncrawl.org/) · [Data
Provenance Initiative](https://www.dataprovenance.org/) · [Model Cards
paper](https://arxiv.org/abs/1810.03993)
