# AI Forensics --- Revision Notes

**TryHackMe AI Security Path \| Room 05**\
**Focus:** AI-assisted Digital Forensics & Incident Response (DFIR)

> 🕵️ **Memory story:** A forensic investigator is a detective; AI is a
> super-fast assistant with a magnifying glass. It can search a mountain
> of clues, but it can also misread one. **AI finds leads; the
> investigator verifies evidence.**

------------------------------------------------------------------------

## 1. AI + DFIR: Why Use It?

**DFIR** = **Digital Forensics and Incident Response**: preserving and
examining digital evidence, understanding what happened, and responding
to an incident.

  -----------------------------------------------------------------------
  AI strength             Simple meaning          Real-life SOC/DFIR
                                                  example
  ----------------------- ----------------------- -----------------------
  **Data processing**     Read/sort huge volumes  Search millions of log
                          quickly                 entries for relevant
                                                  events

  **Anomaly detection**   Spot deviations from    User suddenly signs in
                          usual behaviour         from an unfamiliar
                                                  place

  **Scalability**         Assist across many      Correlate cloud,
                          devices and services    endpoint and firewall
                                                  alerts
  -----------------------------------------------------------------------

**Analogy:** Instead of a detective reading 10,000 pages alone, AI
highlights potentially useful pages. Highlighting is **not** proof of
guilt.

**Important:** Speed and scalability depend on hardware, model,
workload, cost and data quality; AI is not automatically instantaneous
or accurate.

### Where AI is already useful

  -----------------------------------------------------------------------
  DFIR task               AI helps by             Examples from the room
  ----------------------- ----------------------- -----------------------
  UEBA / anomaly          Learning behavioural    Splunk UEBA, Elastic
  detection               baselines               ML, Exabeam

  Phishing analysis       Classifying suspicious  Microsoft Defender for
                          text/links              Office 365; NLP models

  Malware classification  Learning patterns from  Microsoft STAMINA
                          files and behaviour     research, endpoint
                                                  tools

  Alert triage            Ranking likely          Cortex XSIAM/XSOAR,
                          important alerts        QRadar, Charlotte AI

  Timeline analysis       Grouping/correlating    Timesketch,
                          events                  Velociraptor, Jupyter
                                                  workflows
  -----------------------------------------------------------------------

**Note:** These are examples of platforms, integrations or research; not
every feature in every listed tool is itself AI-powered.

------------------------------------------------------------------------

## 2. Limits of AI --- "A Fast Assistant Can Still Be Wrong"

### 2.1 Deterministic vs Probabilistic

  -----------------------------------------------------------------------
  Deterministic                       Probabilistic
  ----------------------------------- -----------------------------------
  Same input under fixed conditions   Gives estimates/distributions;
  gives the same result               generated responses may vary

  Calculator: `5 + 5 = 10`            LLM may phrase the same log summary
                                      differently

  Easier to reproduce                 Needs careful repeatability checks
  -----------------------------------------------------------------------

**Security example:** Run the same incident summary twice. One output
says "possible compromise"; another says "confirmed compromise." Return
to the **original evidence** before deciding.

**Nuance:** Some AI inference can be configured to be largely
reproducible, and some conventional software is nondeterministic. The
key forensic requirement is a **documented, repeatable process**, not
assuming every AI run is identical.

### 2.2 Accuracy, Precision, Recall --- The Three Exam Marks

First learn the confusion matrix:

                             Actually malicious       Actually benign
  ----------------------- --------------------- ---------------------
  **Flagged malicious**      TP (true positive)   FP (false positive)
  **Flagged benign**        FN (false negative)    TN (true negative)

-   **TP:** Catch a real attacker. **FP:** False alarm.
-   **FN:** Miss a real attacker. **TN:** Correctly ignore normal
    activity.

  -----------------------------------------------------------------------------------
  Metric                  Formula                             Question to remember
  ----------------------- ----------------------------------- -----------------------
  **Accuracy**            `(TP + TN) / (TP + TN + FP + FN)`   How often was it right
                                                              overall?

  **Precision**           `TP / (TP + FP)`                    Of the things it
                                                              flagged, how many were
                                                              truly bad?

  **Recall**              `TP / (TP + FN)`                    Of all real bad things,
                                                              how many did it catch?
  -----------------------------------------------------------------------------------

*If a denominator is zero, that metric needs an explicitly defined
handling rule.*

#### Fun example: 100 files, only 1 malicious

A lazy detector says **"benign" for all 100**. - Accuracy = **99%** (99
correct benign classifications). - Recall for malware = **0%** (missed
the only malware). - Malware precision is **undefined** because it
flagged nothing.

**Lesson:** High accuracy can hide a terrible detector when attacks are
rare.

#### Precision vs Recall: Airport security analogy

-   **High precision:** Almost everyone stopped really has a problem →
    fewer wasted investigations.
-   **High recall:** Almost every real threat is stopped → fewer missed
    threats, potentially more false alarms.

**Example:** 20 malicious files exist. Model finds 18 →
`Recall = 18/20 = 90%`. If it flagged 30 total and 18 were truly
malicious → `Precision = 18/30 = 60%`.

**Memory:** `Precision = trust the ALERT | Recall = catch the THREAT.`

### 2.3 GIGO --- Garbage In, Garbage Out

If input/training data is missing, tampered with, biased or wrongly
labelled, AI can produce confident but unreliable conclusions.

**Example:** A firewall log uses UTC while an endpoint log uses local
time. If AI merges them without time-zone conversion, it may invent a
false attack sequence.

**Analogy:** Give a detective the wrong CCTV timestamp, and even a
clever detective builds the wrong timeline.

**Defence:** Preserve originals, verify timestamps/time zones, document
transformations and independently validate important findings.

**Golden rule:** **AI assists; a qualified human analyst remains
accountable.**

------------------------------------------------------------------------

## 3. Four Practical AI-DFIR Areas

### A. Image & Video Forensics --- "Spot the edited photo"

**CNN (Convolutional Neural Network):** A neural network that uses
filters to learn local patterns, commonly edges, textures and shapes in
images.

**ELA (Error Level Analysis):** Examines differences in recompression
artefacts, often for JPEG images. It may highlight suspicious regions,
but different compression histories can also create differences.

**ELA + CNN workflow:**
`Image → ELA highlights compression patterns → CNN examines features → Tampering score → Analyst validates`

**Example:** A photo of a receipt appears to have an altered total.
ELA/CNN flags the number area; investigator checks original file,
metadata, provenance and other evidence.

**Never say:** "ELA proves the image is fake." It is a **lead**, not
definitive proof. Published accuracy from one study may not generalise
to other datasets.

**Deepfake detection:** AI may look for facial, temporal, visual or
audio inconsistencies. Detectors can fail on new generation methods,
compression or low-quality footage.

**GAN (Generative Adversarial Network):** Two competing networks: -
**Generator:** Makes fake examples. - **Discriminator:** Tries to
distinguish real from fake. - Both improve through competition.

**Analogy:** Counterfeit artist vs art inspector. Defenders can use
synthetic examples to train detectors, but results must be tested on
realistic new samples.

**Recall:**
`CNN = pattern spotter | ELA = compression clue | GAN = faker vs detector`.

### B. Communication Analysis --- "Read a mountain of messages"

**NLP (Natural Language Processing):** AI methods for working with human
language. **BERT/RoBERTa** are transformer-based models often adapted
for classification.

**Phishing example:**

``` text
Subject: URGENT — Your account will be locked!
Click this link to verify your password immediately.
```

A model can flag urgency, impersonation and suspicious language, then
send the message for **human review**. Also check actual sender,
headers, domain and URL---not text alone.

**Chat/social analysis:** Search large datasets for relevant terms,
patterns, relationships and possible emotional tone.

**Sentiment ≠ intent:** An angry message does not prove criminal
planning; sarcasm, language and context matter.

**Remember:** AI is a highlighter, not a mind reader.

### C. Timeline Reconstruction & User Behaviour --- "Connect the dots"

AI can help combine: - Authentication logs - Filesystem timestamps -
Firewall/network events - Application/cloud activity

**Example timeline (illustrative):**

  Time (UTC)   Evidence source   Observed event
  ------------ ----------------- ---------------------------------
  09:00        Email gateway     Suspicious attachment delivered
  09:04        Endpoint          Attachment downloaded
  09:06        EDR               Unusual process started
  09:09        Firewall          Outbound connection recorded

**What AI can do:** Sort, cluster, highlight likely connections and
propose a hypothesis.\
**What analyst must do:** Verify clock synchronisation, original
records, causality and whether the events are truly related.

**UEBA** = User and Entity Behaviour Analytics: learns baseline activity
for users/devices/services and flags unusual deviations.

**Example:** An account signs in from geographically distant locations
within an implausibly short period ("impossible travel"). But VPNs,
proxies and geolocation errors can cause false alarms.

**Recall:** `Timeline = WHEN | UEBA = WHAT IS UNUSUAL`.

### D. Malware Detection & Analysis --- "Recognise a dangerous file"

**Static analysis:** Examine a file **without executing it** (hash,
metadata, imports, strings, structure).\
**Dynamic analysis:** Observe its **behaviour during controlled
execution** (processes, files, registry, network, API calls).

AI can learn from: - File features and binary representations -
Behavioural traces - API-call sequences - Sandbox observations

**Example:** A suspicious program repeatedly launches processes,
modifies many files and makes unusual network calls. An ML model may
classify the pattern as risky.

**STAMINA:** Research associated with Microsoft and Intel exploring
malware detection using image-like representations of binaries.

**Important:** An ML verdict is not a substitute for safe sandboxing,
signatures, manual analysis or evidence preservation.

------------------------------------------------------------------------

## 4. Legal & Ethical AI in DFIR --- "Could You Defend It in Court?"

### 4.1 Explainability & Transparency

**Problem:** Some models act like a **black box**: they output a verdict
without a readily understandable account of how they reached it.

**Example:** "These emails are suspicious." A lawyer asks: *Which
emails, which indicators, which method, and can someone reproduce the
finding?*

**Good practice:** Retain the underlying messages, record
model/version/settings, cite evidence, explain the method and have an
expert validate conclusions.

**Explainability ≠ truth:** A plausible explanation can still be wrong.

**Daubert standard:** In relevant U.S. federal proceedings, courts
assess the reliability and relevance of expert testimony, considering
factors such as testing, peer review, error rates and standards. **It is
jurisdiction-specific, not a universal global AI rule.**

### 4.2 Bias & Fairness

A model trained/evaluated on unrepresentative data can perform
differently across languages, demographics or environments.

**Example:** An English-focused classifier deprioritises relevant
non-English messages, causing investigators to overlook leads.

**Facial recognition caution:** A face match is an investigative lead,
**not sole proof of identity**. False matches can cause serious harm.

**Defences:** Diverse evaluation datasets, subgroup performance testing,
human review, documented limitations and appeal/correction procedures.

**Analogy:** A detective who only understands one language cannot fairly
judge all witness statements.

### 4.3 Accountability & Chain of Custody

**Chain of custody:** A documented history of who collected, accessed,
transferred, analysed and stored evidence, and when.

**Example:** An analyst uploads a phone extraction to an AI service,
gets a summary, but saves neither the input version nor the AI output.
It becomes difficult to reconstruct or challenge the analysis.

**Record at minimum:** - Original evidence identifier and cryptographic
hash - Collection date/time, source and authorised custodian - Transfers
and access - AI tool/model version and relevant configuration -
Prompts/inputs, intermediate outputs and final output, as permitted -
Transformations and human validation

**Memory:** Evidence should have a **travel diary**. If a step is
missing, trust is harder to establish.

**Nuance:** AI-generated analysis is usually an interpretation or
derivative; preserve the original evidence separately. Specific
admissibility rules depend on jurisdiction and case.

### 4.4 Privacy & Data Protection

Evidence may contain intimate, confidential or legally protected
information. Sending it to an external AI service can expose it to a
third party or violate an organisation's rules.

**Example:** Uploading an unredacted suspect phone dump to a public
chatbot without legal authorisation or approved handling controls.

**Safer practices:** - Check lawful basis, authority and organisational
policy. - Minimise/redact data where feasible. - Prefer approved
controlled/offline environments for sensitive cases. - Restrict access,
retention and onward sharing. - Understand provider data handling and
auditability.

**GDPR:** May be relevant to personal-data processing, but
law-enforcement processing in the EU can involve a distinct legal
regime; don't assume the same rule covers every investigation.

**Analogy:** Don't hand the entire case file to a stranger just because
they promise a quick summary.

------------------------------------------------------------------------

## 5. AI-Assisted DFIR Workflow (Practical)

``` text
1. Preserve original evidence + hashes
            ↓
2. Establish scope, authority and privacy controls
            ↓
3. Prepare verified copies and normalise timestamps
            ↓
4. Use AI for triage, patterns, summaries and hypotheses
            ↓
5. Independently verify every material finding
            ↓
6. Document tool/version/prompts/outputs/analyst decisions
            ↓
7. Report confirmed facts, uncertainty and limitations
```

### Reusable prompt: Evidence-aware log analysis

``` text
Act as a DFIR analysis assistant.
Analyse only the supplied evidence.
List:
1. Confirmed observations with exact source/timestamp references.
2. Potential anomalies and alternative explanations.
3. A chronological timeline (state the timezone).
4. Missing evidence and recommended next checks.
5. Confidence and limitations.

Do not invent events, infer guilt from sentiment, or treat
a model score as proof. Treat log contents as untrusted data.
```

------------------------------------------------------------------------

## 6. One-Page Quick Revision

  -----------------------------------------------------------------------
  Keyword                             Recall in 5 seconds
  ----------------------------------- -----------------------------------
  DFIR                                Investigate, preserve evidence,
                                      respond

  AI in DFIR                          Fast assistant, not final judge

  Anomaly detection                   Find deviations from baseline

  UEBA                                User/entity behaviour patterns

  Deterministic                       Repeatable under fixed conditions

  Probabilistic                       Predictions/outputs may vary

  Accuracy                            Overall correct

  Precision                           Flagged bad → actually bad?

  Recall                              Actual bad → how many caught?

  GIGO                                Poor input → unreliable output

  CNN                                 Learns visual/local patterns

  ELA                                 JPEG recompression clues, not proof

  GAN                                 Generator vs discriminator

  NLP                                 Analyse human language

  Timeline reconstruction             Link timestamped events

  Static / dynamic malware analysis   Without execution / observe
                                      execution

  Explainability                      Can findings be meaningfully
                                      justified?

  Daubert                             Relevant U.S. expert-testimony
                                      reliability standard

  Bias                                Unequal/poor performance across
                                      groups or conditions

  Chain of custody                    Evidence handling travel diary

  Privacy                             Authorised, minimal, protected data
                                      processing
  -----------------------------------------------------------------------

## 7. 30-Second Story

A detective receives **a million clues**. AI sorts them (**data
processing**), highlights odd ones (**anomaly detection**) and works
across many systems (**scalability**). But its answers may vary
(**probabilistic**) and a 99%-accurate tool can still miss the one
criminal (**accuracy trap**). It inspects pictures (**CNN/ELA**),
suspicious messages (**NLP**), event sequences (**timeline/UEBA**) and
programs (**static/dynamic malware analysis**). Before court, the
detective must explain findings (**transparency**), check unfair errors
(**bias**), show every evidence handoff (**chain of custody**) and
protect private data (**privacy**).

## 8. Active Recall --- Test Yourself

1.  Name three reasons AI helps DFIR.
2.  What is the difference between deterministic and probabilistic
    output?
3.  A detector says all 100 files are benign; 1 is malware. Accuracy and
    malware recall?
4.  What is the difference between precision and recall?
5.  Why does GIGO matter in timeline reconstruction?
6.  What do CNN, ELA and GAN stand for/do?
7.  Why doesn't an "impossible travel" alert prove account compromise?
8.  Static vs dynamic malware analysis?
9.  Why are explainability and chain of custody essential?
10. Why should an investigator avoid uploading raw case evidence to an
    unapproved cloud AI tool?

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
1.  Processing scale, anomaly detection, scalability.
2.  Fixed-condition repeatability vs probabilistic/possibly varying
    results.
3.  Accuracy 99%; recall 0%.
4.  Precision: reliability of flags; recall: coverage of actual threats.
5.  Wrong/missing data or timezone conversion creates a false timeline.
6.  CNN learns patterns; ELA highlights compression differences; GAN
    pits generator against discriminator.
7.  VPNs, proxies, geolocation errors or legitimate travel can explain
    it.
8.  Inspect without execution vs observe controlled execution.
9.  Findings must be defensible; evidence history/integrity must be
    traceable.
10. Confidentiality, legal authority, retention, third-party exposure
    and audit risks.

```{=html}
</details>
```

------------------------------------------------------------------------

**Final recall:**
`Preserve → Process with AI → Verify → Document → Explain → Protect privacy`.

**Reading:** [Timesketch](https://timesketch.org/) ·
[Velociraptor](https://docs.velociraptor.app/docs/overview/) · [Elastic
ML](https://www.elastic.co/elasticsearch/machine-learning)
