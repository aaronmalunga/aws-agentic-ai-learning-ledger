# From a Frozen Dataset to Model Selection

## Building a defensible routing benchmark without turning validation into training data

This lesson documents a practical model-selection workflow developed while building an intent-routing component for an AI support system.

The application problem was simple to describe:

> Given a customer message, classify it as `BUG`, `PLATFORM`, or `OTHER`.

The engineering problem was substantially harder:

> How do we determine which routing architecture generalizes well enough to use, without contaminating the evidence used to make that decision?

The workflow progressed from deterministic rules to lightweight classical machine learning while preserving explicit train, validation, and held-out test boundaries.

The most important lesson was not that one model scored higher than another.

It was:

> **Training fit is not generalization evidence, and model selection is only credible when the evaluation boundaries are protected before results are observed.**

---

# 1. The experimental question

The initial routing options were intentionally simple.

| Candidate | Architecture | Purpose |
|---|---|---|
| `B0` | Majority-class baseline | Establish the lowest useful reference point |
| `B1` | Deterministic lexical rules | Determine whether simple rules are sufficient |
| `C1` | TF-IDF + Logistic Regression | Lightweight learned classifier |
| `C2` | TF-IDF + Linear SVM | Lightweight learned classifier |

The purpose of the experiment was not:

> Which technology sounds most advanced?

The actual question was:

> What is the least complex routing architecture that demonstrates adequate generalization and operational behavior?

This is an important architecture principle.

A more sophisticated technology should enter a system because evidence demonstrates that it solves a real limitation, not because it is fashionable.

---

# 2. Freeze the data before model development

The benchmark contained:

| Split | Records | Purpose |
|---|---:|---|
| Train | 108 | Model fitting and permitted development |
| Validation | 36 | Candidate/configuration selection |
| Held-out test | 36 | Final independent evaluation |
| **Total** | **180** | |

The three classes were intentionally balanced in this synthetic benchmark:

| Intent | Total records |
|---|---:|
| BUG | 60 |
| PLATFORM | 60 |
| OTHER | 60 |

The dataset was frozen before candidate evaluation.

Important provenance included:

| Property | Value |
|---|---|
| Dataset | `routing-intent-v1` |
| Version | `1.0.0` |
| Status | `FROZEN` |
| Canonical LF SHA-256 | `369600abb8711dcadc0acb96bd19edc33a9c06a83cf7c09208bccb6dbde85450` |

Freezing the dataset created an explicit boundary:

```text
DATASET DESIGN
      ↓
DATASET REVIEW
      ↓
DATASET FREEZE
      ↓
MODEL DEVELOPMENT
```

not:

```text
MODEL FAILS
      ↓
EDIT DATA
      ↓
MODEL FAILS AGAIN
      ↓
EDIT DATA AGAIN
```

The second workflow makes the benchmark progressively conform to the model.

---

# 3. Semantic-family splitting

Individual examples were not randomly distributed without structure.

Messages belonged to semantic families, with several wording variants belonging to the same underlying intent scenario.

The important constraint was:

> Variants from the same semantic family should not leak across development and evaluation boundaries.

For example, if four variants all describe the same password-reset failure, placing three in training and one in validation can make validation deceptively easy.

The model may merely recognize vocabulary associated with that family.

A stronger evaluation asks whether the system can generalize to **new semantic families**, not just new surface wording from familiar families.

This distinction matters in many ML systems:

| Weak evaluation | Stronger evaluation |
|---|---|
| Random row split | Group/family-aware split |
| Familiar topic, new wording | New topic/family |
| Measures interpolation | Better test of generalization |
| Greater leakage risk | Lower leakage risk |

---

# 4. Model input versus evaluation metadata

Each routing record contained useful metadata such as:

- case ID;
- semantic family ID;
- variant ID;
- split;
- provenance;
- expected intent.

However, only the customer message was permitted as predictive input.

The boundary was:

```text
MODEL INPUT
    message
       ↓
   classifier
       ↓
 predicted intent
```

Evaluation could then reattach metadata:

```text
prediction
    +
case_id
family_id
variant_id
provenance
expected_intent
    ↓
failure analysis
```

This matters because metadata can be useful for diagnosis while still being inappropriate as a predictive feature.

The principle is:

> **Keep diagnostic metadata rich, but keep the predictive boundary narrow.**

---

# 5. Establish a trivial baseline first

`B0` predicted the majority class from the training data.

Because the frozen training split was balanced:

| Intent | Train count |
|---|---:|
| BUG | 36 |
| PLATFORM | 36 |
| OTHER | 36 |

there was a three-way tie.

A deterministic canonical tie order was therefore defined:

```text
BUG → PLATFORM → OTHER
```

This did not mean BUG was considered operationally more important.

It simply made the baseline reproducible.

The result provided the expected weak reference:

| Candidate | Validation accuracy | Validation macro F1 |
|---|---:|---:|
| B0 | 0.3333 | 0.1667 |

A trivial baseline is valuable because it answers:

> Is the proposed system actually learning anything beyond an obvious default strategy?

---

# 6. Build the deterministic baseline using TRAIN only

`B1` used lexical rules and explicit precedence.

The important experimental constraint was:

> Rules could be developed using training failures, but validation and test examples could not be used to expand the rules.

This distinction is essential.

Inspecting training errors is model development.

Inspecting validation errors and repeatedly modifying the model from them eventually turns validation into another training set.

---

# 7. A real debugging example: the word `return`

An early deterministic rule used the word:

```text
return
```

as a platform-related signal.

That caused an unexpected classification error for SQL-related text involving:

```text
returns duplicate rows
```

The router interpreted the lexical token `return` as evidence of a commerce return-policy question.

The fix was not:

> Add another arbitrary exception.

Instead, the rule was made more contextual.

This demonstrated a common weakness of keyword systems:

> Lexical overlap is not semantic equivalence.

The word `return` can describe:

- returning merchandise;
- returning a value from a function;
- SQL returning rows;
- returning home;
- returning a response from an API.

As deterministic rule dictionaries grow, these collisions become increasingly difficult to manage.

---

# 8. Another debugging example: `login` versus `log in`

A training failure appeared to involve login behavior.

A manually typed diagnostic sentence containing:

```text
login
```

classified correctly.

However, the actual frozen dataset example still failed.

The difference turned out to be:

```text
login
```

versus:

```text
log in
```

The important debugging move was inspecting the **exact source representation** rather than relying on a manually retyped approximation.

Using exact source text revealed the lexical mismatch.

The lesson is broader than routing:

> When debugging text-processing systems, reproduce the exact input before changing the implementation.

A manually recreated example can accidentally remove the bug.

---

# 9. The deterministic baseline reached 108/108 training accuracy

After permitted train-only development:

| B1 training result | Value |
|---|---:|
| Correct | 108 |
| Total | 108 |
| Accuracy | 1.0000 |

At first glance, this looks excellent.

It was not evidence of generalization.

It only demonstrated:

> The rule system could represent the examples used to develop it.

This distinction became obvious after validation.

---

# 10. Freeze metrics before looking at validation

The benchmark measured:

- accuracy;
- macro precision;
- macro recall;
- macro F1;
- per-class precision;
- per-class recall;
- per-class F1;
- confusion matrix;
- preserved failure cases.

The primary metric was:

> **Macro F1**

Macro F1 gives each class equal weight.

That was appropriate because the benchmark itself intentionally contained balanced classes and because performance on BUG, PLATFORM, and OTHER all mattered.

No arbitrary numerical failure-severity score was invented.

Operational severity could be considered qualitatively during architecture selection, but there was no preregistered mathematical severity scale.

This avoided creating a metric after seeing results.

---

# 11. Preregister the classical search space

The benchmark protocol allowed validation to be used for configuration selection.

However, the exact search space was frozen before validation exposure.

The classical candidates used TF-IDF word features.

Shared TF-IDF choices included:

| Parameter | Value |
|---|---|
| Analyzer | Word |
| Lowercase | Yes |
| `min_df` | 1 |
| `max_df` | 1.0 |
| Norm | L2 |
| IDF | Enabled |
| Smooth IDF | Enabled |
| Sublinear TF | Disabled |
| Stop-word removal | None |

Two n-gram configurations were allowed:

| ID | Range |
|---|---|
| N1 | `(1, 1)` — unigrams |
| N2 | `(1, 2)` — unigrams + bigrams |

Three regularization values were allowed:

```text
C = 0.5
C = 1.0
C = 2.0
```

For each classifier family:

| Family | Configurations |
|---|---:|
| Logistic Regression | 6 |
| Linear SVM | 6 |
| **Total classical configurations** | **12** |

Together with B0 and B1:

```text
14 total validation candidates
```

The important methodological rule was:

> The search space was defined before validation results were known.

---

# 12. Why a bounded search matters

Suppose validation results are observed and then the workflow becomes:

```text
try C=2
      ↓
validation improves
      ↓
try C=3
      ↓
try C=4
      ↓
add trigrams
      ↓
add character n-grams
      ↓
change tokenization
      ↓
validation improves again
```

Validation has effectively become a development target.

The reported validation score becomes increasingly optimistic.

A bounded preregistered search says:

> These are the configurations we will compare, and we will not expand the search simply because the results tempt us to.

This does not mean validation cannot be used for selection.

That is exactly what validation is for.

The problem is **uncontrolled adaptive optimization against validation**.

---

# 13. A provenance failure caught before validation

The initial classical search-space document was committed and later inspected using Git.

The committed artifact unexpectedly contained only 38 lines rather than the complete intended document.

Instead of silently rewriting history or pretending the original artifact was complete, a corrective commit was made before validation exposure.

The sequence was preserved:

| Commit | Meaning |
|---|---|
| `12cabcf` | Initial preregistration artifact — discovered incomplete |
| `24cebec` | Search-space document completed before validation |

This provided another lesson:

> Do not assume the file you intended to commit is the file Git actually contains.

Useful verification includes:

```text
git status
git diff
git diff --cached
git show
```

Version-control history is part of experiment provenance.

---

# 14. What TF-IDF actually contributes

TF-IDF is not itself a classifier.

It converts text into numeric features.

Conceptually:

```text
customer message
      ↓
tokenization
      ↓
term frequency
      ×
inverse document frequency
      ↓
sparse numeric vector
      ↓
classifier
```

Term frequency increases with how often a term occurs in a document.

Inverse document frequency reduces the influence of terms that occur in many documents.

This tends to give more weight to words that are informative for distinguishing messages.

---

# 15. Logistic Regression versus Linear SVM

Both candidates operated over the same TF-IDF representation.

The comparison was therefore primarily between linear decision mechanisms.

## Logistic Regression

Useful characteristics include:

- linear decision boundary;
- probabilistic formulation;
- commonly interpretable coefficients;
- inexpensive training and inference.

## Linear SVM

Useful characteristics include:

- linear separating boundary;
- margin-based optimization;
- well suited to sparse high-dimensional text features;
- inexpensive inference.

Neither architecture required:

- embeddings;
- transformer fine-tuning;
- GPU inference;
- an LLM call;
- external network access.

This was important because the goal was to identify the **smallest architecture that solved the demonstrated problem**.

---

# 16. Validation was opened only after the experiment machinery was frozen

Before the first formal validation execution, the following had already been versioned:

| Component | Frozen before validation? |
|---|---|
| Dataset | Yes |
| Data boundary | Yes |
| B0 | Yes |
| B1 | Yes |
| Metrics | Yes |
| Search space | Yes |
| Classical implementations | Yes |
| Evidence layer | Yes |
| Validation runner | Yes |

The validation runner freeze commit was:

```text
5d055b4
```

Only after that commit was the validation runner executed.

This creates a useful provenance relationship:

```text
5d055b4
    │
    │ experiment machinery already fixed
    ▼
VALIDATION EXPOSED
```

---

# 17. Validation results

The results were:

| Candidate | Accuracy | Macro F1 | Failures |
|---|---:|---:|---:|
| **C2-N1-C2.0** | **0.9722** | **0.9722** | **1** |
| C2-N1-C0.5 | 0.9444 | 0.9444 | 2 |
| C1-N1-C2.0 | 0.9444 | 0.9444 | 2 |
| C2-N1-C1.0 | 0.9444 | 0.9444 | 2 |
| C1-N2-C2.0 | 0.9167 | 0.9165 | 3 |
| C2-N2-C0.5 | 0.9167 | 0.9165 | 3 |
| C2-N2-C1.0 | 0.9167 | 0.9165 | 3 |
| C1-N2-C1.0 | 0.9167 | 0.9165 | 3 |
| C1-N2-C0.5 | 0.9167 | 0.9165 | 3 |
| C2-N2-C2.0 | 0.9167 | 0.9165 | 3 |
| C1-N1-C0.5 | 0.8889 | 0.8889 | 4 |
| C1-N1-C1.0 | 0.8889 | 0.8889 | 4 |
| B1 | 0.3889 | 0.2767 | 22 |
| B0 | 0.3333 | 0.1667 | 24 |

The strongest candidate was:

```text
C2-N1-C2.0
```

meaning:

| Component | Choice |
|---|---|
| Feature representation | TF-IDF |
| N-grams | Unigrams |
| Classifier | Linear SVM |
| C | 2.0 |

---

# 18. The strongest lesson: 108/108 training became 14/36 validation

The deterministic rule baseline produced:

| Stage | Correct | Total | Accuracy |
|---|---:|---:|---:|
| Training after permitted development | 108 | 108 | 1.0000 |
| Validation | 14 | 36 | 0.3889 |

Its validation macro F1 was:

```text
0.2767
```

Per-class recall made the failure even clearer:

| Intent | B1 validation recall |
|---|---:|
| BUG | 0.0833 |
| PLATFORM | 0.0833 |
| OTHER | 1.0000 |

The fallback class dominated.

This was not a minor performance difference.

The deterministic router generalized poorly to unseen semantic families.

---

# 19. What B1 failed on

Examples included:

| Expected class | New semantic concept |
|---|---|
| BUG | Password-reset links opening error pages |
| BUG | Wishlist synchronization failures |
| BUG | Notification links producing 404/error pages |
| PLATFORM | Gift-card policy |
| PLATFORM | Restocking questions |
| PLATFORM | Refund processing time |

The rules had achieved perfect development-set coverage but had not learned a general representation of support intent.

This demonstrated the maintenance trajectory of a growing lexical router:

```text
new wording
    ↓
missed rule
    ↓
add keyword
    ↓
keyword collides elsewhere
    ↓
add exception
    ↓
more branches
    ↓
growing maintenance burden
```

---

# 20. Why we did not repair B1 after validation

After observing B1's validation failures, it would have been easy to add rules for:

- gift cards;
- restocking;
- refund timing;
- password-reset links;
- wishlist synchronization.

That would probably increase validation performance.

It would also compromise the evaluation.

The validation set had already influenced development.

The correct response was:

> Record the failure as evidence.

not:

> Rewrite the model until validation becomes perfect.

---

# 21. C2's remaining validation failure

The selected-looking C2 configuration made one validation error:

| Case | Expected | Predicted |
|---|---|---|
| `ROUTE-PLATFORM-012-C` | PLATFORM | OTHER |

Message:

> What is the normal refund processing time?

The failure is operationally relevant because a legitimate platform-support request could enter a fallback or escalation path rather than self-service.

However:

| Class | C2 validation recall |
|---|---:|
| BUG | 1.0000 |
| PLATFORM | 0.9167 |
| OTHER | 1.0000 |

No BUG validation cases were missed.

No unrelated OTHER validation cases were incorrectly pulled into PLATFORM.

---

# 22. Why failure cases matter in addition to metrics

Two candidates can have similar aggregate scores but operationally different mistakes.

For example:

```text
PLATFORM → OTHER
```

may cause:

- unnecessary human escalation;
- reduced self-service;
- increased support load.

Whereas:

```text
OTHER → PLATFORM
```

may cause:

- irrelevant platform responses;
- unnecessary downstream retrieval;
- confusing user experience.

And:

```text
BUG → OTHER
```

could suppress a genuine defect-reporting workflow.

Therefore model selection should inspect:

```text
aggregate metrics
      ↓
per-class metrics
      ↓
confusion matrix
      ↓
actual preserved failures
      ↓
operational interpretation
```

not only:

```text
highest score wins
```

---

# 23. Unigrams beat the unigram-plus-bigram configurations

An interesting result was that every tested N2 configuration reached approximately:

```text
macro F1 ≈ 0.9165
```

while the best unigram model reached:

```text
macro F1 = 0.9722
```

This does not prove that bigrams are generally inferior.

It shows that:

> More features did not improve this benchmark.

One plausible hypothesis is increased feature sparsity relative to the small training dataset.

However, that remains a hypothesis rather than a proven causal explanation.

The evidence itself is simply:

| Feature representation | Best observed validation macro F1 |
|---|---:|
| Unigrams | **0.9722** |
| Unigrams + bigrams | ~0.9165 |

This is another reason to benchmark rather than assume increasing feature complexity is automatically beneficial.

---

# 24. When a larger model becomes justified

Dataset size alone should not determine architecture complexity.

A useful progression is:

```text
rules
  ↓
TF-IDF + linear classifier
  ↓
embeddings + classifier
  ↓
fine-tuned transformer
  ↓
LLM
```

but only when the previous level fails a demonstrated requirement.

A larger dataset does not automatically require an LLM.

More important questions include:

- Are intents lexically separable?
- Do messages require semantic reasoning?
- Is context across turns required?
- Are class boundaries subtle?
- Is multilingual generalization required?
- Is labeled data sufficient?
- What latency budget exists?
- What is the acceptable cost per request?

For the current routing problem, classical ML demonstrated strong enough performance that embeddings, transformers, and LLM routing were not yet justified.

---

# 25. Operational evidence after validation

Classification quality was not the only decision factor.

The shortlisted candidates were measured for:

- local inference latency;
- local training latency;
- repeated prediction determinism.

The operational comparison was:

| Candidate | Validation macro F1 | Failures | Deterministic | Local p50/message | Local p95/message |
|---|---:|---:|---|---:|---:|
| B1 | 0.2767 | 22 | Yes | 0.0167 ms | 0.0307 ms |
| C1-N1-C2.0 | 0.9444 | 2 | Yes | 0.1079 ms | 0.1563 ms |
| **C2-N1-C2.0** | **0.9722** | **1** | **Yes** | **0.1071 ms** | **0.1627 ms** |

Classical training latency:

| Candidate | Local training p50 |
|---|---:|
| C1-N1-C2.0 | 21.0384 ms |
| C2-N1-C2.0 | 6.7938 ms |

The training measurements are secondary because:

- the dataset is small;
- only a small number of repetitions were performed;
- training is outside the request-serving path.

---

# 26. Relative latency versus absolute latency

B1 was approximately five times faster than C2 in local inference.

That sounds important until absolute magnitude is considered.

Approximate local p95:

| Candidate | p95 |
|---|---:|
| B1 | 0.0307 ms/message |
| C2 | 0.1627 ms/message |

Absolute difference:

```text
≈ 0.132 ms/message
```

Saving roughly a tenth of a millisecond would not justify accepting the severe classification regression observed with B1.

This is an important performance-engineering lesson:

> Relative performance can sound dramatic while the absolute difference is operationally irrelevant.

---

# 27. Latency evidence must be scoped correctly

The operational benchmark measured:

> local in-process routing inference.

It did **not** measure end-to-end production latency.

Excluded components included:

- HTTP networking;
- AWS service latency;
- Lambda/container cold starts;
- serialization/deserialization;
- queueing;
- telemetry transport;
- Bedrock invocation;
- AgentCore invocation;
- external tool calls;
- downstream APIs.

The per-message measurements were also derived from batch runs over a fixed synthetic probe set.

Therefore:

```text
0.1627 ms/message
```

must not be represented as:

> Reliora production request latency is 0.1627 ms.

The correct statement is:

> Local batch-normalized routing inference p95 was approximately 0.1627 ms/message in the measured environment.

This distinction prevents benchmark numbers from turning into misleading production claims.

---

# 28. Prediction determinism is not experiment reproducibility

All shortlisted candidates returned identical predictions across repeated calls in the operational benchmark.

That demonstrates:

> prediction determinism for a fixed fitted model and fixed inputs.

It does not necessarily prove:

> retraining will always produce byte-for-byte identical model artifacts.

Nor does it prove:

> another machine will reproduce identical latency measurements.

Three concepts should remain separate:

| Concept | Meaning |
|---|---|
| Prediction determinism | Same fitted model + same input → same prediction |
| Training reproducibility | Repeating training produces equivalent model behavior |
| Experiment reproducibility | Another engineer can recreate the experiment and interpret the evidence |

Conflating these concepts leads to overclaiming.

---

# 29. Why C2 was selected

Before opening the held-out test, the candidate-selection decision was:

```text
C2-N1-C2.0
```

The decision matrix was:

| Dimension | B1 | C1-N1-C2.0 | C2-N1-C2.0 |
|---|---:|---:|---:|
| Validation macro F1 | 0.2767 | 0.9444 | **0.9722** |
| Validation failures | 22 | 2 | **1** |
| BUG recall | 0.0833 | 1.0000 | **1.0000** |
| Deterministic | Yes | Yes | Yes |
| Local p95/message | **0.0307 ms** | 0.1563 ms | 0.1627 ms |
| Requires ML dependency | No | Yes | Yes |
| Rule-maintenance burden | High as vocabulary expands | Low | Low |
| Overall decision | Reject | Strong alternative | **Select** |

The C1/C2 latency difference was approximately:

```text
0.0064 ms at p95
```

which was not meaningful enough to compensate for C1's weaker validation performance.

---

# 30. Architecture decision

The architecture decision became:

> Use TF-IDF word unigrams with a Linear SVM (`C=2.0`) as the routing classifier, subject to one untouched held-out test evaluation.

The decision does not mean classical ML is permanently sufficient.

It means:

> Classical ML is currently the least complex architecture supported by the available evidence.

Future evidence could justify a different architecture.

Examples might include:

- multilingual routing requirements;
- semantic ambiguity not captured by lexical features;
- significantly larger intent taxonomy;
- conversational-context requirements;
- unacceptable production error rates;
- new classes with weak lexical separability.

---

# 31. Why an LLM was not selected

An LLM router would introduce:

- network latency;
- inference cost;
- model/provider dependency;
- prompt-version management;
- potentially nondeterministic behavior;
- additional observability requirements;
- more complex failure handling.

The current classical model already demonstrated:

- strong validation performance;
- perfect validation BUG recall;
- deterministic predictions;
- negligible local computational cost.

Therefore an LLM router currently has no demonstrated requirement.

The principle is:

> **Do not use an LLM where a simpler model already satisfies the measured requirement.**

This is not anti-LLM architecture.

It is evidence-based architecture.

---

# 32. Scripts are part of the experiment

The experiment gradually moved from one-off terminal inspection toward reproducible scripts.

This transition matters.

Early exploration can reasonably use terminal commands.

Once results become evidence, the procedure should become executable and version-controlled.

Examples included:

| Script type | Purpose |
|---|---|
| Dataset validator | Verify structural invariants |
| Dataset audit | Inspect dataset properties |
| Validation runner | Execute all preregistered candidates consistently |
| Operational measurement runner | Measure shortlisted candidate characteristics |
| Held-out runner | Final single-candidate evaluation |

A script is not merely convenience.

It encodes:

- experiment procedure;
- assumptions;
- boundaries;
- output format;
- failure conditions.

---

# 33. Tests can enforce experimental discipline

Tests were not only verifying ordinary program correctness.

Some tests acted as **experimental controls**.

Examples included:

| Control | What the test protected |
|---|---|
| Model input boundary | Metadata cannot become predictive input |
| Confusion-matrix orientation | Rows/columns cannot silently be misinterpreted |
| Frozen candidate registry | Off-registry configurations fail |
| Validation runner | Synthetic fixtures test orchestration without opening real validation |
| Operational runner | Only TRAIN requested for fitting |
| Held-out runner | TRAIN for fitting, TEST for final evaluation, no VALIDATION |
| Selection identity | Held-out runner points only to frozen selected candidate |

This is an important engineering pattern:

> Experimental methodology can sometimes be enforced in software rather than left as a documentation promise.

---

# 34. Why the validation runner did not expose TEST

The validation runner deliberately had no test mode.

This is a useful design principle:

> If an operation should not happen yet, removing the capability can be stronger than documenting “please do not use it.”

The code architecture therefore created an experimental firewall.

Later, a separate held-out runner was created only after the selected candidate had been frozen.

---

# 35. Git commits became experiment boundaries

Several commits had methodological meaning, not merely source-control meaning.

| Commit | Experimental meaning |
|---|---|
| `4c86dae` | Routing dataset frozen |
| `6a237c6` | Benchmark protocol frozen |
| `312c0d6` | Deterministic routing baseline frozen |
| `4faafe7` | Metrics frozen |
| `24cebec` | Classical search space completed before validation |
| `f59fe55` | Classical candidates frozen |
| `db875e5` | Evidence layer frozen |
| `5d055b4` | Validation runner frozen before validation exposure |
| `5e19899` | First validation evidence preserved |
| `5a33421` | Operational measurement machinery frozen |
| `c03c39f` | Operational evidence preserved |
| `fd47277` | `C2-N1-C2.0` selected before TEST |

This history establishes chronology:

```text
DECISION
   ↓
COMMIT
   ↓
RESULT EXPOSURE
```

rather than:

```text
RESULT EXPOSURE
   ↓
CHANGE DECISION
   ↓
PRETEND DECISION WAS PREDEFINED
```

---

# 36. Hashes provide artifact identity

Git identifies repository history.

Cryptographic hashes identify exact artifacts.

Examples:

| Artifact | SHA-256 |
|---|---|
| Frozen routing dataset | `369600abb8711dcadc0acb96bd19edc33a9c06a83cf7c09208bccb6dbde85450` |
| Validation evidence | `615CC62703F466FFE9C75F016E3F8B8C6E743BDE661B0B70470DF0A0219D4B9F` |
| Operational evidence | `5D3AA410BE54BB15BDBF4D2D23FD8C170488E633891CC6E52F76CCFDE070F96F` |

Hashes are useful when a claim needs to point to the exact evidence artifact used to support it.

---

# 37. Canonical line endings matter for hashes

Windows commonly writes CRLF line endings.

Repositories may normalize text to LF.

Without normalization:

```text
same visible text
+
different line endings
=
different SHA-256
```

For dataset identity, canonical LF hashing was therefore used.

This separates:

> logical dataset identity

from:

> operating-system-specific newline encoding.

Git warnings such as:

```text
CRLF will be replaced by LF
```

were not treated as experiment failures.

They indicated line-ending normalization.

---

# 38. PowerShell rendering is not source truth

Another practical lesson came from terminal display.

PowerShell wrapping sometimes made text appear:

- joined;
- truncated;
- split strangely across lines.

The correct debugging rule became:

> Never modify source solely because terminal rendering looks suspicious.

Verify the exact source first using tools such as:

```text
Get-Content
Select-String
repr()
git diff
git show
```

A visually odd line is not necessarily a source defect.

---

# 39. `python -m pytest` versus the direct pytest entry point

The project encountered namespace/import behavior where:

```text
pytest
```

and:

```text
python -m pytest
```

did not behave identically in the environment.

The reliable project convention became:

```text
uv run python -m pytest
```

This ensures pytest runs through the interpreter belonging to the controlled uv environment.

The lesson is:

> Tool entry points are part of environment reproducibility.

---

# 40. Ruff and mypy solve different problems

Both static-analysis gates proved useful.

Examples:

| Tool | Problems caught |
|---|---|
| Ruff | Unused imports |
| Ruff | Import ordering |
| Ruff | Unnecessarily complex constructs |
| Ruff | Modern Python API recommendations |
| mypy | Incorrect domain types |
| mypy | Unsafe assumptions about broad `object` values |
| mypy | Third-party typing boundaries |
| pytest | Runtime behavior and contracts |

A passing type checker does not make a linter redundant.

A passing linter does not make tests redundant.

Each gate addresses a different failure class.

---

# 41. Third-party typing boundaries should be narrow

scikit-learn did not provide the typing metadata expected by mypy in this environment.

The response was not to globally disable type checking.

A targeted suppression was used only at the boundary:

```python
import sklearn  # type: ignore[import-untyped]
```

This preserves type checking throughout the rest of the project.

The principle is:

> Suppress uncertainty at the narrowest possible boundary.

---

# 42. Scripts and tests are learning material

A useful way to document a technical step is:

| Field | Question |
|---|---|
| Action | What did I do? |
| Purpose | Why did I do it now? |
| Mechanism | How does it work? |
| Expected evidence | What result would indicate success? |
| Observed result | What actually happened? |
| Engineering consequence | What decision does the result enable? |

For example:

| Field | Operational benchmark |
|---|---|
| Action | Measure B1, C1, and C2 locally |
| Purpose | Determine whether C2's accuracy advantage has unacceptable runtime cost |
| Mechanism | Warm runs + repeated `perf_counter_ns()` timing + determinism checks |
| Expected evidence | Latency and deterministic-prediction records |
| Observed result | C2 p95 ~0.1627 ms/message locally; deterministic |
| Engineering consequence | C2's runtime cost did not outweigh its validation advantage |

This produces much stronger learning documentation than recording only the command that was executed.

---

# 43. Current experimental state

At the time this lesson was written:

| Stage | Status |
|---|---|
| Dataset | Frozen |
| Validation | Observed |
| Validation evidence | Preserved |
| Operational evidence | Preserved |
| Selected candidate | `C2-N1-C2.0` |
| Selection decision | Frozen |
| Held-out test | **Not yet opened** |

This is intentional.

The final test result is not included here because it had not yet been observed.

---

# 44. What happens next

The selected candidate will be evaluated once on the untouched held-out test split.

The final runner is deliberately restricted to:

```text
TRAIN
  ↓
fit C2-N1-C2.0
  ↓
TEST
  ↓
final metrics
```

It does not perform:

- validation evaluation;
- configuration search;
- model comparison;
- hyperparameter tuning;
- candidate replacement.

If the held-out result is disappointing:

> The result becomes evidence.

It does not authorize tuning the model against the test set.

A future improvement would require a new, explicitly versioned evaluation cycle.

---

# 45. Interview explanation

A concise explanation of the architecture decision could be:

> I started with deterministic routing rules because they were the least complex solution. After train-only development they classified all 108 training cases correctly, but on semantic-family-held-out validation they collapsed to roughly 39% accuracy and 0.28 macro F1. I had preregistered a bounded classical search space before seeing validation, so I compared TF-IDF with Logistic Regression and Linear SVM without expanding the search reactively. A unigram Linear SVM with C=2.0 achieved 0.972 macro F1 and 35 of 36 validation cases correct, including perfect BUG recall. I then measured local latency and determinism; the SVM remained deterministic and its roughly 0.16 ms local p95 per-message processing cost was negligible relative to the future cloud request path. I therefore selected the classical model before opening the final held-out test. I deliberately did not jump to an LLM because the evidence did not justify the additional cost and complexity.

---

# 46. Reusable engineering principles

The experiment produced several reusable principles.

| Principle | Meaning |
|---|---|
| Training accuracy is not generalization evidence | Fit and generalization answer different questions |
| Freeze before exposure | Dataset, metrics, candidates, and procedures should precede results |
| Validation is for selection | But uncontrolled adaptive tuning eventually contaminates it |
| Test is not another validation set | Open it only after the candidate is fixed |
| Inspect failures, not only scores | Operationally different errors can have identical metrics |
| Use the simplest adequate architecture | Complexity must solve a demonstrated problem |
| Preserve provenance | Git commits and hashes establish chronology and artifact identity |
| Separate model input from diagnostic metadata | Rich evaluation need not create feature leakage |
| Scripts encode methodology | Reproducibility requires more than terminal history |
| Tests can enforce experimental boundaries | Methodology can sometimes become executable policy |
| Relative latency can mislead | Always consider absolute magnitude and measurement scope |
| Do not overclaim benchmarks | Local model timing is not production SLA evidence |
| Exact source beats recreated examples | Especially in text-processing debugging |
| Keep uncertainty local | Narrow type ignores and explicit runtime guards are preferable |
| More features are not automatically better | Measure instead of assuming |
| An LLM is not automatically the right router | Use it only when simpler models fail demonstrated requirements |

---

# 47. Final takeaway

The most important result was not:

```text
Linear SVM = 0.9722 macro F1
```

The most important result was the process that made that number meaningful:

```text
define the problem
      ↓
design the benchmark
      ↓
freeze the data
      ↓
separate train / validation / test
      ↓
establish simple baselines
      ↓
develop using TRAIN only
      ↓
freeze metrics
      ↓
preregister candidate search
      ↓
freeze implementations
      ↓
open VALIDATION
      ↓
inspect metrics and failures
      ↓
measure operational trade-offs
      ↓
freeze the selected candidate
      ↓
open TEST once
```

That workflow transforms model choice from:

> “I tried some algorithms and this one scored highest.”

into:

> **“I can show why this architecture was selected, what evidence supports it, which evidence was unavailable at decision time, and what would justify changing it later.”**

That distinction is central to reliable ML engineering and defensible AI architecture.