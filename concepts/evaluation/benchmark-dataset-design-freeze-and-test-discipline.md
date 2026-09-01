# Benchmark Dataset Design, Freeze, and Test Discipline

## Why This Lesson Exists

A model benchmark is only as trustworthy as the process used to build,
review, freeze, and evaluate against its dataset.

It is easy to create a dataset that looks rigorous while accidentally
making model performance easier than it should be.

Examples include:

- putting paraphrases of the same scenario into both training and test
  sets;
- tuning prompts, rules, or models after repeatedly looking at test
  failures;
- changing difficult test examples because a model performs poorly on
  them;
- measuring duplicate strings while missing semantic leakage;
- treating synthetic class balance as if it represented production
  traffic;
- calculating metrics before the dataset's identity and provenance are
  fixed.

The important lesson is:

> Benchmark validity is an engineering property of the entire
> experimental process, not merely a property of the metric.

Reliora's routing benchmark provided a practical example of how to make
that process explicit.

---

# 1. The Core Mental Model

A trustworthy supervised benchmark should follow a controlled sequence:

```text
define the task
        ↓
define labels
        ↓
define semantic families
        ↓
generate or collect examples
        ↓
assign families to splits
        ↓
validate structure
        ↓
audit data quality and leakage
        ↓
perform human semantic review
        ↓
freeze benchmark content
        ↓
record canonical identity
        ↓
record Git provenance
        ↓
train and select models
        ↓
evaluate the selected configuration on held-out test data
```

The order matters.

Training a model before completing dataset review creates a risk that
the model's behaviour will influence the benchmark itself.

That weakens the independence of later evaluation evidence.

---

# 2. What Is a Semantic Family?

A semantic family is a group of examples that express the same
underlying intent, task, or scenario in different ways.

For example:

```text
family:
checkout-crash

variant A:
The checkout page crashes when I press Pay.

variant B:
Every time I try to pay, checkout closes unexpectedly.

variant C:
Can you help me understand why checkout crashes when I submit payment?

variant D:
payment page keeps crashing when i try to finish my order
```

The wording changes.

The underlying problem does not.

All four messages belong to the same semantic family.

---

# 3. Why Semantic Families Matter

Suppose the following examples are split independently:

```text
TRAIN:
The checkout page crashes when I press Pay.

TEST:
Every time I try to pay, checkout closes unexpectedly.
```

The strings are different.

But the model has effectively already seen the test concept during
training.

This is a form of semantic leakage.

A benchmark may therefore report excellent test performance while
measuring the model's ability to recognize paraphrases of previously
seen scenarios rather than its ability to generalize to new scenarios.

---

# 4. Example-Level Splitting vs Family-Level Splitting

## Example-Level Split

```text
family A variant 1 → train
family A variant 2 → train
family A variant 3 → validation
family A variant 4 → test
```

This creates leakage between splits.

---

## Family-Level Split

```text
family A → train
family B → train
family C → validation
family D → test
```

All variants belonging to one semantic family remain in the same split.

This provides a stronger test of generalization.

The split unit is therefore not:

```text
individual message
```

It is:

```text
semantic family
```

---

# 5. Reliora Routing Benchmark Example

Reliora's first routing benchmark contains:

```text
dataset:
routing-intent-v1

total records:
180

semantic families:
45

variants per family:
4

top-level classes:
BUG
PLATFORM
OTHER
```

The dataset is balanced by design:

```text
BUG       60
PLATFORM  60
OTHER     60
```

Its split distribution is:

```text
train       108 records
validation   36 records
test         36 records
```

At the family level:

```text
train       27 families
validation   9 families
test         9 families
```

Each class contributes:

```text
9 train families
3 validation families
3 test families
```

This means the held-out test set contains semantic families that do not
appear in training or validation.

---

# 6. Why Four Variants Per Family?

Multiple variants help measure whether a system is robust to ordinary
language variation.

Useful variation can include:

- formal wording;
- informal wording;
- direct requests;
- indirect requests;
- different sentence structures;
- capitalization differences;
- minor grammatical imperfections;
- different vocabulary expressing the same intent.

However, variants should not be used to inflate the apparent diversity
of a benchmark.

Four paraphrases of one semantic scenario are still one underlying
semantic family.

This is why both record count and family count should be documented.

---

# 7. Synthetic Data Does Not Represent Production Prevalence

Reliora's routing benchmark is synthetic.

Its three classes are deliberately balanced:

```text
BUG       33.3%
PLATFORM  33.3%
OTHER     33.3%
```

This is useful for controlled model comparison.

It does not mean real customer traffic will have the same distribution.

A balanced benchmark answers questions such as:

> Can the classifier distinguish the three intended routing classes
> under a controlled evaluation design?

It does not establish:

> One third of real Reliora traffic will be BUG requests.

This distinction is part of claim discipline.

---

# 8. Benchmark Distribution vs Production Distribution

These are different concepts.

## Benchmark Distribution

Designed to support controlled comparison.

May intentionally balance classes.

---

## Production Distribution

Observed from real system traffic.

May be highly imbalanced.

For example:

```text
PLATFORM  72%
BUG       18%
OTHER     10%
```

would be entirely possible in production.

The production distribution must eventually be measured rather than
assumed.

---

# 9. Structural Validation Is Necessary but Not Sufficient

A structural validator can verify rules such as:

- required fields exist;
- labels use allowed values;
- split names are valid;
- case IDs are unique;
- family IDs exist;
- family labels match record labels;
- family splits match record splits;
- expected counts are correct;
- required variants exist;
- exact duplicate messages do not exist.

These are deterministic properties.

They are excellent candidates for executable validation.

---

# 10. What Structural Validation Cannot Prove

A structural validator cannot reliably determine whether:

- a message is semantically ambiguous;
- two families overlap conceptually;
- one class uses an artificial writing style;
- labels make sense to a human reviewer;
- a difficult boundary case belongs in one class rather than another;
- test examples genuinely represent unseen semantic scenarios.

These require additional audit and review.

Therefore:

```text
structural validation
≠
dataset quality
```

---

# 11. Diagnostic Dataset Auditing

Reliora also used a diagnostic audit.

The audit examined characteristics such as:

- cross-split lexical similarity;
- possible near-duplicates;
- class-associated tokens;
- cross-family repeated tokens;
- message lengths;
- boundary-term distributions.

Unlike hard structural rules, these findings are diagnostic.

They tell the engineer where to investigate.

They do not automatically prove that a dataset is defective.

---

# 12. Example: Near-Duplicate Detection

Two useful similarity heuristics are:

```text
Jaccard token-set similarity
```

and:

```text
sequence similarity
```

They can identify suspicious pairs that deserve inspection.

For example:

```text
message A:
the checkout page crashes when i pay

message B:
checkout page crashes whenever i try to pay
```

These are likely semantically very close.

But lexical similarity has limitations.

Two messages may be semantically equivalent while sharing few words.

Likewise, two messages may share many words while expressing different
intent.

Therefore:

> Lexical similarity is a screening tool, not semantic proof.

---

# 13. Why Zero Near-Duplicate Findings Must Be Interpreted Carefully

If an audit reports:

```text
cross-split near duplicates: 0
```

the correct interpretation is:

> No pairs exceeded the documented similarity thresholds.

It does not prove:

> There is zero semantic overlap anywhere in the benchmark.

The strength of an evidence statement should match what the method
actually measured.

---

# 14. Class-Associated Tokens

Synthetic datasets can accidentally create class-specific writing
patterns.

For example:

```text
BUG messages:
always contain "broken"

PLATFORM messages:
always contain "policy"

OTHER messages:
always contain "help"
```

A classifier may learn these shortcuts instead of learning the intended
semantic distinction.

This is especially dangerous when examples are generated from templates
or by an LLM.

---

# 15. Why Token Audits Should Consider Semantic Families

Suppose one family contains four variants.

If the same word appears in all four variants, a raw message-frequency
audit may incorrectly interpret that as four independent examples of a
class-wide pattern.

But those four records are related.

A stronger analysis considers how many distinct semantic families use
the token.

This helps distinguish:

```text
one family repeated four times
```

from:

```text
a pattern occurring across many independent families
```

---

# 16. Do Not Artificially Equalize Vocabulary

The goal of auditing is not to make every class use identical language.

Some words are legitimately informative.

For example:

```text
crash
refund
policy
tracking
```

may naturally correlate with certain intents.

Removing all predictive language can make the benchmark unrealistic.

The goal is to reduce artificial shortcuts while preserving legitimate
semantic signal.

---

# 17. Human Semantic Review

Automated checks should be followed by human review before benchmark
freeze.

Human review asks questions such as:

- Is the expected label correct?
- Is the message understandable?
- Is the message ambiguous?
- Does the family represent one coherent semantic concept?
- Does another family represent essentially the same concept?
- Does the example depend on hidden context?
- Is the test example genuinely reasonable?
- Would an independent reviewer likely assign the same label?

This is especially important for synthetic datasets.

---

# 18. Why Human Review Happens Before Model Training

The safest sequence is:

```text
review benchmark
        ↓
freeze benchmark
        ↓
train model
```

not:

```text
train model
        ↓
inspect failures
        ↓
rewrite difficult examples
        ↓
train again
```

The second process creates test-set gaming.

Even when done unintentionally, the benchmark begins adapting to the
model.

---

# 19. Held-Out Test Discipline

The test set has a special role.

Training data is used to fit the model.

Validation data may be used to:

- compare candidate algorithms;
- choose hyperparameters;
- select thresholds;
- compare feature representations;
- select the final model configuration.

The test set should be used only after those decisions are fixed.

Mental model:

```text
TRAIN
→ learn

VALIDATION
→ choose

TEST
→ verify
```

---

# 20. Why the Test Set Must Be Closed to Model-Driven Editing

Once model experimentation begins, poor test performance should not
cause test cases to be rewritten merely to improve the score.

For example:

```text
model predicts PLATFORM
expected label is OTHER
```

The first question should be:

> What does this failure tell us about the model?

It should not automatically become:

> How should we rewrite the example so the model gets it right?

---

# 21. A Model Failure Is Evidence

After benchmark freeze:

```text
model failure
        ↓
experimental evidence
```

It may reveal:

- insufficient features;
- weak model capacity;
- poor class boundaries;
- vocabulary sensitivity;
- prompt weakness;
- distribution difficulty;
- legitimate ambiguity.

The failure itself is useful.

Deleting or rewriting failures simply because they are inconvenient can
destroy the value of the benchmark.

---

# 22. When Can a Frozen Test Case Change?

Frozen does not mean permanently infallible.

A genuine ground-truth defect may still be discovered.

Examples:

- the expected label is objectively incorrect;
- a message is corrupted;
- a family was assigned to the wrong split;
- required metadata is malformed;
- two supposedly different families are discovered to be duplicates.

But the correction must follow explicit change control.

---

# 23. Post-Freeze Change Control

A genuine correction should record:

- what changed;
- why it changed;
- who or what discovered the defect;
- whether model performance influenced the discovery;
- whether the dataset version changed;
- the new canonical hash;
- the new Git revision;
- whether previous benchmark results remain comparable.

This prevents silent benchmark mutation.

---

# 24. Why Dataset Versioning Matters

Suppose:

```text
routing-intent-v1
```

changes after benchmark results are published.

Without versioning, two experiments may claim to use the same dataset
while actually evaluating different content.

A meaningful benchmark identity therefore includes more than a file
name.

It may include:

```text
dataset ID
dataset version
canonical hash
Git revision
split definition
evaluation code version
```

---

# 25. What Does "Freeze" Mean?

A benchmark freeze means:

> The content used for the next model-comparison phase has been reviewed
> and intentionally fixed.

It does not mean:

- the dataset can never have a future version;
- the benchmark is perfect;
- the benchmark represents production traffic;
- no limitations exist;
- no future annotation defect can ever be discovered.

Freeze is an experimental-control mechanism.

---

# 26. Freeze-Before-Modeling

The most important experimental rule from this phase is:

> Freeze the benchmark before training candidate models.

This creates a clear boundary between:

```text
benchmark construction
```

and:

```text
model experimentation
```

That boundary makes later evidence easier to defend.

---

# 27. Why Git Is Useful for Benchmark Freeze

Git provides an immutable content snapshot.

A freeze commit establishes:

```text
this exact repository state
```

for the benchmark.

Reliora recorded a dedicated routing dataset freeze commit before model
benchmarking began.

This means later experiments can point to an exact historical benchmark
state rather than relying only on the current working tree.

---

# 28. Why a Hash Is Also Useful

A Git commit identifies a repository snapshot.

A cryptographic hash identifies exact file content.

Using both gives stronger provenance:

```text
Git revision
→ where the artifact existed

SHA-256
→ exactly which bytes represented the artifact
```

The hash basis must also be documented.

For Reliora, the canonical basis was:

```text
SHA-256 over file bytes with line endings normalized to LF
```

This avoids Windows CRLF vs repository LF differences creating
apparently different identities for logically identical text content.

---

# 29. Independent Verification From the Commit

A particularly strong provenance check is:

```text
record canonical hash
        ↓
commit frozen benchmark
        ↓
extract artifact directly from Git commit
        ↓
hash committed bytes
        ↓
compare with recorded identity
```

This proves that the commit being cited actually contains the benchmark
artifact represented by the documented hash.

---

# 30. Reliora Freeze Verification Example

Reliora's frozen routing artifacts were independently verified from the
freeze commit.

The result was:

```text
DATASET MATCH: True
MANIFEST MATCH: True
ALL COMMITTED HASHES MATCH: True
```

This established the chain:

```text
reviewed dataset
        ↓
canonical identity
        ↓
frozen Git commit
        ↓
committed artifact extraction
        ↓
independent hash verification
```

---

# 31. Data Content vs Evidence Packaging

A useful distinction is:

```text
dataset content freeze
```

versus:

```text
documentation/evidence finalization
```

The dataset can already be frozen while documentation is still being
synchronized with:

- final hashes;
- freeze commit IDs;
- validation results;
- review status;
- provenance details.

Documentation updates do not necessarily modify benchmark content.

This distinction prevents harmless evidence packaging from being
confused with benchmark mutation.

---

# 32. Validation vs Audit vs Review vs Freeze

These four concepts have different responsibilities.

## Validation

Checks deterministic rules.

```text
Is the structure valid?
```

---

## Audit

Surfaces suspicious statistical or lexical characteristics.

```text
Is there anything unusual worth investigating?
```

---

## Human Review

Evaluates semantic correctness and reasonableness.

```text
Does this example actually make sense?
```

---

## Freeze

Creates the controlled experimental boundary.

```text
Is benchmark content now fixed for model comparison?
```

None of these should be treated as interchangeable.

---

# 33. Why Benchmark Metrics Alone Are Not Enough

A statement such as:

```text
macro F1 = 0.97
```

is incomplete.

A stronger claim is:

```text
Candidate X achieved macro F1 = 0.97
on routing-intent-v1,
using the documented held-out split,
against the frozen dataset identity,
under the recorded experiment configuration.
```

Metrics require provenance to become defensible evidence.

---

# 34. Common Benchmark Failure Modes

## Random Paraphrase Leakage

Variants from the same semantic family appear in multiple splits.

---

## Test-Set Gaming

Test failures influence model tuning or benchmark wording.

---

## Silent Dataset Mutation

Examples change without version or provenance updates.

---

## Synthetic Shortcut Learning

Classes use artificial stylistic patterns.

---

## Metric-Only Reporting

Scores are reported without dataset or experiment identity.

---

## Production Overclaiming

Controlled synthetic benchmark results are presented as production
performance.

---

## Validation Overconfidence

A structurally valid dataset is assumed to be semantically correct.

---

# 35. Better Experimental Discipline

A stronger workflow is:

```text
1. define the routing contract
2. define class boundaries
3. define semantic families
4. create variants
5. split by family
6. run structural validation
7. run diagnostic audit
8. manually review examples
9. close the held-out test set
10. freeze dataset and manifest
11. calculate canonical identities
12. record Git provenance
13. independently verify frozen artifacts
14. begin model benchmarking
15. select using train/validation
16. evaluate the fixed configuration on test
17. preserve failures as evidence
```

---

# 36. Why This Matters for AI Engineering

AI systems are probabilistic.

That makes experimental discipline more important, not less important.

Without controlled evaluation:

- model improvements may be imaginary;
- prompt changes may overfit known examples;
- leakage may inflate metrics;
- regressions may be missed;
- different experiment runs may not be comparable;
- claims may become impossible to reproduce.

A trustworthy AI system therefore needs trustworthy evaluation
infrastructure.

---

# 37. Why This Matters for LLMOps

LLMOps is not only about deploying models.

It also involves:

- dataset lineage;
- experiment reproducibility;
- evaluation governance;
- regression detection;
- controlled promotion;
- evidence retention;
- traceable model or prompt changes.

Benchmark freeze and test discipline are part of that operational
foundation.

---

# 38. Why This Matters for Solutions Architecture

A Solutions Architect should be able to explain not only:

> Which model did you choose?

but also:

> Why should anyone trust the evidence used to choose it?

A strong answer discusses:

- evaluation contracts;
- data provenance;
- leakage prevention;
- validation;
- held-out testing;
- versioning;
- reproducibility;
- operational monitoring.

This moves the conversation from model experimentation to system
engineering.

---

# 39. Interview Explanation

A concise interview explanation could be:

> I designed the routing benchmark around semantic families rather than
> individual messages. All paraphrases of the same scenario remain in
> one split, so the held-out test set measures generalization to unseen
> scenarios instead of memorization of related wording. Before training
> any candidate model, I ran structural validation, diagnostic leakage
> audits, and human semantic review, then froze the dataset and recorded
> its canonical hashes and Git revision. After freeze, model failures are
> treated as experimental evidence rather than reasons to rewrite the
> test set. Genuine annotation defects require explicit versioned change
> control.

---

# 40. Questions I Should Be Able to Answer

After learning this topic, I should be able to explain:

1. What is a semantic family?
2. Why is family-level splitting stronger than random message splitting?
3. What is semantic leakage?
4. Why can synthetic balanced data still be useful?
5. Why does synthetic balance not represent production prevalence?
6. What does structural validation prove?
7. What does structural validation not prove?
8. Why use lexical similarity audits?
9. Why are lexical audits diagnostic rather than semantic proof?
10. Why should human review happen before model training?
11. What is test-set gaming?
12. Why should the test set be closed to model-driven editing?
13. What does benchmark freeze mean?
14. Why freeze before model selection?
15. When is a post-freeze benchmark change legitimate?
16. Why should such a change be versioned?
17. Why record both SHA-256 and a Git revision?
18. Why does the hash basis matter?
19. What is independent committed-artifact verification?
20. Why is benchmark provenance necessary for trustworthy metrics?

---

# Core Principle

> Do not design the benchmark around the model's behaviour.

Design, review, and freeze the benchmark first.

Then let the model's performance reveal what the model can and cannot
do.
