# The Building Blocks of AI

> TryHackMe AI Security Path \| Revision Notes

## 1. Big Picture

-   **AI (Artificial Intelligence):** Broad field of systems performing
    tasks associated with human intelligence (reasoning, recognition,
    problem-solving, generation).
-   **ML (Machine Learning):** Subfield of AI that learns patterns from
    data rather than relying only on explicitly written rules.
-   **DL (Deep Learning):** Subfield of ML using multi-layer neural
    networks to learn representations.
-   **LLM (Large Language Model):** Usually transformer-based DL model
    trained on large amounts of text/token sequences to understand and
    generate language.

**Remember:** `AI → ML → DL → LLM` (nested categories, not four
competing technologies).

## 2. Why AI Matters in Cybersecurity

  -----------------------------------------------------------------------
  Defender use                        Attacker misuse
  ----------------------------------- -----------------------------------
  Spam/phishing detection             Convincing phishing content

  Network anomaly detection           Scaling social engineering

  Log triage and alert prioritisation Assisting malicious research and
                                      automation

  Threat-intelligence summarisation   Generating deceptive content
  -----------------------------------------------------------------------

**Security mindset:** AI outputs can be wrong or manipulated; validate
evidence and maintain human oversight for consequential decisions.

## 3. TryHackMe AI Agent Platform

-   Some tasks use an **AI agent chat panel**, not a terminal or VM.
-   Select **Open Agent**, type natural-language prompts, and inspect
    the response.
-   Each agent has a configured role, behaviours and sometimes protected
    information or a goal.
-   Tasks may require working **with** an agent or testing its
    resistance.
-   **Prompt wording matters:** different wording/context can produce
    different responses.

## 4. Machine Learning Lifecycle

`Define problem → Collect data → Clean/prepare → Train → Evaluate/tune → Deploy → Monitor → Retrain (when needed)`

**Example: spam detector** 1. Define spam vs legitimate email. 2. Gather
representative examples and prepare the data. 3. Train the classifier.
4. Evaluate on separate, unseen data; tune it. 5. Deploy to classify
incoming mail. 6. Monitor new spam tactics and retrain as appropriate.

**Overfitting:** model learns training examples too closely and performs
poorly on unseen data.\
**Recall cue:** *memorisation ≠ generalisation*.\
**Security link:** a detector that memorises known attacks may miss new
variations.

## 5. How an ML Algorithm Learns

  Component                     Function
  ----------------------------- -----------------------------------------
  Decision/prediction process   Produces an output from input
  Error/loss function           Measures how wrong the prediction is
  Optimisation                  Updates model parameters to reduce loss

**Loop:**
`Input → Prediction → Calculate loss → Update parameters → Repeat`

**Algorithm vs model:** algorithm = learning method; trained model =
learned result.

### Four Learning Types

  --------------------------------------------------------------------------------
  Type                  Training signal     Easy example      Security example
  --------------------- ------------------- ----------------- --------------------
  **Supervised**        Labelled examples   Spam/not spam     Classify known
                        with correct                          malicious vs benign
                        answers                               files

  **Unsupervised**      No target labels;   Customer clusters Group unusual
                        discovers structure                   network behaviour

  **Semi-supervised**   Small labelled +    Few labelled      Use a limited set of
                        large unlabelled    photos            analyst-labelled
                        dataset                               alerts

  **Reinforcement**     Actions plus        Game-playing      Train a simulated
                        rewards/penalties   agent             defensive-response
                                                              agent
  --------------------------------------------------------------------------------

**Memory trick:** Supervised = teacher gives answers; Unsupervised =
find patterns; Semi = a few answers; Reinforcement = trial, feedback,
reward.

## 6. Neural Networks and Deep Learning

An artificial neural network is **inspired by** biological neurons; it
does not literally replicate the brain.

`Input layer → Hidden layer(s) → Output layer`

-   **Input layer:** receives data/features (e.g., a 4×4 grayscale image
    may be represented by 16 pixel values).
-   **Hidden layers:** transform input into increasingly useful
    representations.
-   **Weights:** learned numbers controlling the influence of
    connections.
-   **Output layer:** produces scores, probabilities or other
    predictions.

**Digit recognition example:** pixel values → detect edges/curves →
combine shapes → score digits `0–9` → select predicted digit.

**Deep learning:** uses multiple learned layers to extract
representations. There is no universally fixed "more than three layers"
threshold.

**Important correction:** DL **can use labelled data** (e.g., supervised
image classification). It can also use self-supervised or unsupervised
approaches. Its advantage is learning useful features from raw or
complex data, often with less manual feature engineering---not that
labels are never required.

### NEURON-1 Exercise

1.  Give the agent an input, e.g., a suspicious firewall-log line.
2.  Identify input features (IP, port, event, frequency).
3.  Describe patterns the hidden layers might extract.
4.  Interpret the output classification.
5.  Compare prediction with the correct label and consider what should
    change during training.

**Recall:**
`Raw input → Feature extraction → Prediction → Feedback/learning`.

## 7. Large Language Models (LLMs)

LLMs are deep-learning models trained to model sequences of **tokens**
(words, word pieces or symbols). Many chat-oriented LLMs generate
responses by repeatedly predicting the next token.

### Training and response flow

`Text → Tokenisation → Pre-training → Further tuning/alignment → Prompt → Next-token generation → Response`

-   **Parameters:** learned numerical values (including weights) that
    influence model behaviour; not a simple database of facts.
-   **Pre-training:** learn from vast datasets, commonly via next-token
    prediction or related self-supervised objectives.
-   **Loss:** measures the mismatch between predicted and target tokens.
-   **Backpropagation:** computes how parameters contribute to loss; an
    optimiser then updates them.
-   **GPU parallelism:** makes large-scale model training practical.
-   **RLHF:** Reinforcement Learning from Human Feedback; human
    preferences help align outputs with desired behaviour. Not every LLM
    uses the exact same alignment process.

### Transformers and attention

-   Transformer architecture was introduced in the 2017 paper
    **Attention Is All You Need**.
-   **Attention:** helps a model weigh relevant tokens in context.
-   Example: "The bank approved the loan because it was financially
    stable." Context helps link **it** to **bank**.
-   Transformers allow substantial parallel computation during training;
    autoregressive response generation still proceeds token by token.

### Generative AI

-   **LLM:** language-focused model (some modern models are multimodal).
-   **Generative AI:** broader category producing text, images, audio,
    video, etc.
-   Examples discussed in the room: ChatGPT, LLaMA/Llama and DeepSeek.

## 8. AI Security Connections

  -----------------------------------------------------------------------
  Concept                             What a security analyst should
                                      consider
  ----------------------------------- -----------------------------------
  Training data                       Poisoned or unrepresentative data
                                      can degrade a model

  Overfitting                         Weak performance on unfamiliar
                                      threats

  Model output                        False positives, false negatives,
                                      hallucinations

  Agent instructions/tools            Prompt injection and excessive
                                      permissions

  LLM context                         Untrusted text may attempt to
                                      override intended instructions

  Deployment                          Monitor model quality, abuse and
                                      changing attack patterns
  -----------------------------------------------------------------------

## 9. Quick Revision: 30-Second Recall

-   **AI:** broad field; **ML:** learns from data; **DL:** multi-layer
    neural networks; **LLM:** language-focused DL.
-   **ML lifecycle:** problem → data → train → evaluate → deploy →
    monitor/retrain.
-   **Overfitting:** memorises training data, struggles on new examples.
-   **ML learning loop:** prediction → loss → optimisation.
-   **Learning types:** supervised / unsupervised / semi-supervised /
    reinforcement.
-   **Neural network:** input → weighted hidden layers → output.
-   **LLM:** token prediction, learned parameters, pre-training,
    possible RLHF.
-   **Transformer:** attention weighs context; GPUs help scale
    computation.
-   **Security:** protect data, models, prompts, tools and outputs.

## 10. Self-Test (Active Recall)

1.  What is the difference between AI, ML, DL and LLM?
2.  Why is ML deployment not the end of its lifecycle?
3.  Explain overfitting using a spam-detector example.
4.  Which learning type uses rewards and penalties?
5.  What do weights and hidden layers do?
6.  Does deep learning always require unlabelled data?
7.  What is the difference between a token and a parameter?
8.  What does attention help a transformer do?
9.  What are pre-training, backpropagation and RLHF?
10. Name one defender use and one attacker misuse of AI.

```{=html}
<details>
```
```{=html}
<summary>
```
Short answer key
```{=html}
</summary>
```
1.  Nested fields: AI → ML → DL → LLM.
2.  Data and threats change; monitor and retrain when necessary.
3.  Memorising old spam examples but failing on new spam.
4.  Reinforcement learning.
5.  Weights control influence; hidden layers learn representations.
6.  No; supervised DL also uses labels.
7.  Token = unit of sequence; parameter = learned numeric model value.
8.  Weight relevant context.
9.  Large-scale initial learning; gradient computation; preference-based
    alignment.
10. Example: anomaly detection; phishing generation.

```{=html}
</details>
```

------------------------------------------------------------------------

**One-line mental map:**
`Data → ML learns patterns → Deep networks learn representations → Transformers use attention → LLM generates tokens → AI security protects the entire pipeline.`
