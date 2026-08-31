# Deterministic Evaluation vs LLM-as-a-Judge

## Why This Lesson Exists

The central reliability lesson behind Reliora came from the Stage-1 customer-support agent.

Stage 1 used a Bedrock generative evaluator:

```text
Builtin.Correctness
```

with Amazon Nova Pro.

The evaluation dataset contained:

```text
6 prompts
```

and the recorded result was:

```text
6 / 6
Correctness score: 1.00
```

At first glance, this looked excellent.

However, closer inspection of the actual responses and the frozen system contract revealed several violations.

Examples included:

```text
REPRO-001
```

The bug-report workflow was required to ask for:

```text
only ONE missing field at a time
```

but the model requested:

```text
description
steps to reproduce
environment
```

in one response.

---

Another case:

```text
REPRO-002
```

The model exposed:

```text
HUMAN HANDOFF
```

even though the system prompt explicitly prohibited exposing internal route names.

---

Another case:

```text
REPRO-004
```

The return policy required preservation of a defective-item exception.

The Stage-1 answer correctly included:

```text
30-day return window
unused condition
original packaging
```

but omitted:

```text
the defective-item exception
```

The overall generative correctness evaluation still gave the dataset a perfect-looking result.

This did not mean the generative evaluator was useless.

It meant that:

> A broad generative correctness evaluator was being asked to measure requirements that were more precise than the metric itself.

Reliora therefore separates:

```text
deterministic evaluation
```

from:

```text
generative evaluation
```

instead of treating either one as sufficient for every requirement.

---

# 1. What Is Evaluation?

Evaluation asks:

> Does the AI system behave acceptably according to defined requirements?

That sounds simple, but AI systems have many different properties.

For example, we may need to evaluate:

```text
Was the answer relevant?

Was it factually complete?

Did the agent obey a workflow rule?

Did it call a tool too early?

Did it expose internal information?

Was the response helpful?

Was the tone appropriate?

Did it fabricate a ticket ID?

Did it preserve required policy facts?

Was latency acceptable?

Was the operation retry-safe?
```

These are not all the same type of question.

Therefore, one evaluator should not automatically be expected to measure all of them equally well.

---

# 2. The Core Problem: Different Requirements Need Different Evaluators

Consider two requirements.

### Requirement A

```text
The response should be clear and helpful.
```

This is partly qualitative.

Different valid responses may express the same idea in many ways.

---

### Requirement B

```text
The system must never call the bug-report tool until all required fields are complete.
```

This is much more exact.

The state is essentially:

```text
tool called too early
```

or:

```text
tool not called too early
```

Trying to evaluate both requirements using exactly the same mechanism can create blind spots.

---

# 3. What Is Deterministic Evaluation?

A deterministic evaluator uses explicit software rules to decide whether a defined condition was satisfied.

Given the same:

```text
input
+
evaluator code
```

it should produce the same result.

Conceptually:

```text
Input
  ↓
explicit rule
  ↓
PASS / FAIL
```

---

## Example

Suppose the required rule is:

> Ask for only one missing bug field at a time.

The evaluator can inspect:

```text
missing_fields
requested_fields
```

and enforce:

```text
if required fields are missing:
    exactly one requested field must be present
    and that requested field must actually be missing
```

This does not require another LLM to interpret whether the response "seems correct."

---

# 4. Reliora's Deterministic Evaluation Examples

Reliora currently implements deterministic checks including:

```text
INV-004
INV-012
FACT-001
```

---

## `INV-004`

Checks the bug-report collection contract.

The requirement is:

```text
Ask for only one missing required field at a time.
```

The evaluator can inspect structured annotations such as:

```text
missing fields:
- description
- steps_to_reproduce
- environment

requested fields:
- description
- steps_to_reproduce
- environment
```

Because three fields were requested when only one was permitted:

```text
INV-004 → FAIL
```

---

## `INV-012`

Checks for prohibited internal-information leakage.

Current examples include detection of:

```text
HUMAN HANDOFF
<thinking>
</thinking>
```

If one of those prohibited markers is found in customer-visible output:

```text
INV-012 → FAIL
```

---

## `FACT-001`

Checks required factual completeness.

For the Stage-1 return-policy case, required facts were defined as:

```text
return_window
unused_condition
original_packaging
defective_item_exception
```

The Stage-1 answer contained the first three but not the final exception.

Therefore:

```text
FACT-001 → FAIL
```

---

# 5. Why These Rules Are Good Deterministic Candidates

These requirements have explicit contracts.

They are not primarily asking:

> Is this generally a good answer?

They are asking:

```text
Was exactly one missing field requested?

Was a forbidden marker present?

Were all required facts represented?
```

Those properties can be represented as software checks.

---

# 6. What Is LLM-as-a-Judge?

LLM-as-a-judge uses a language model to evaluate another AI output.

Conceptually:

```text
Prompt
+
candidate response
+
evaluation instructions
        ↓
Judge model
        ↓
score / label / explanation
```

The judge can evaluate qualities that are difficult to encode using simple deterministic rules.

---

# 7. Where LLM-as-a-Judge Is Useful

Generative evaluation can be valuable for properties such as:

```text
helpfulness
relevance
clarity
coherence
tone
semantic faithfulness
answer quality
instruction interpretation
```

For example, consider two responses:

```text
Response A:
Most orders arrive within three business days.

Response B:
You should normally receive the order in about three working days.
```

A literal string comparison would treat them as different.

A language model can understand that they express approximately the same meaning.

This semantic flexibility is useful.

---

# 8. Why Generative Evaluation Is Attractive

LLM judges can evaluate outputs that would otherwise require substantial human review.

They can:

- interpret language variations
- reason about meaning
- compare responses with references
- produce explanations
- handle open-ended text
- scale evaluation beyond purely exact-match checks

This makes them useful components in AI evaluation systems.

---

# 9. Why LLM-as-a-Judge Is Not a Universal Oracle

An LLM judge is itself a model.

It is not an infallible mathematical verifier.

Its result depends on factors such as:

```text
evaluation prompt
judge model
scoring rubric
reference information
sampling configuration
context provided
interpretation of requirements
```

A judge can therefore assign a high score even when a narrow contractual requirement was violated.

That is what made the Stage-1 result important.

---

# 10. The Stage-1 Lesson

The Stage-1 system received:

```text
6 / 6
Builtin.Correctness = 1.00
```

but deterministic inspection later found known defects in the evaluated behaviour.

The lesson is not:

> LLM judges are bad.

The lesson is:

> A broad correctness metric should not be treated as proof of compliance with every detailed system invariant.

---

# 11. Metric-Requirement Alignment

An evaluator should match the property being measured.

Conceptually:

```text
Requirement
     ↓
What kind of property is it?
     ↓
Choose evaluator
```

For example:

| Requirement | Better Evaluation Approach |
|---|---|
| Exactly one missing bug field requested | Deterministic |
| No internal route label exposed | Deterministic |
| No fabricated ticket ID | Deterministic |
| Required policy facts preserved | Deterministic or structured factual evaluation |
| Response is clear and helpful | LLM judge / human review |
| Response tone is appropriate | LLM judge / human review |
| Semantic answer relevance | LLM judge / retrieval evaluation |
| p95 latency below target | Measured operational metric |
| Duplicate retry causes no duplicate ticket | Integration/deterministic test |

No single evaluator covers all of these well.

---

# 12. Hard Constraints vs Soft Qualities

A useful distinction is:

```text
Hard constraints
```

versus:

```text
Soft qualities
```

---

## Hard Constraint

A requirement where violation is generally unacceptable.

Examples:

```text
do not expose internal route names
do not fabricate ticket IDs
do not execute a side effect before validation
do not create duplicate tickets on retry
```

These are strong candidates for deterministic validation.

---

## Soft Quality

A desirable property with legitimate interpretation.

Examples:

```text
friendly
concise
helpful
well-written
natural
professionally phrased
```

These are more suitable for generative or human evaluation.

---

# 13. Some Requirements Are Hybrid

Not every property fits neatly into one category.

For example:

```text
factual correctness
```

can require multiple layers.

Suppose an answer says:

> Returns are accepted within 30 days.

A deterministic check might verify that:

```text
30 days
```

appears.

But whether a complex answer accurately represents an entire policy may require deeper semantic evaluation.

Therefore a layered system can use:

```text
deterministic required-fact checks
+
semantic generative evaluation
```

rather than choosing only one.

---

# 14. Reliora's Layered Evaluation Philosophy

Reliora treats evaluation as several complementary layers.

Conceptually:

```text
1. Routing evaluation

2. Behavioural contract evaluation

3. Side-effect / tool safety evaluation

4. Generative quality evaluation

5. Operational evaluation
```

Each layer measures a different property.

---

# 15. Layer 1 — Routing Evaluation

Routing asks:

> Did the system send the request to the correct logical path?

Example classes may include:

```text
BUG
PLATFORM
OTHER
```

A router can be evaluated using classification metrics such as:

```text
precision
recall
F1
confusion matrix
```

This is different from judging whether the eventual answer was well written.

---

# 16. Layer 2 — Behavioural Contract Evaluation

This asks:

> Did the agent obey exact workflow and safety requirements?

Examples:

```text
INV-004
INV-012
```

These are primarily deterministic checks.

---

# 17. Layer 3 — Side-Effect Evaluation

This asks:

> Did actions involving external systems occur safely?

Examples include:

```text
Was the tool called only when fields were complete?

Were tool arguments valid?

Was the same operation retried safely?

Was a ticket ID returned only from a successful tool result?
```

These often require:

```text
deterministic tests
integration tests
idempotency tests
tool-trajectory inspection
```

---

# 18. Layer 4 — Generative Quality Evaluation

This asks questions such as:

```text
Was the response helpful?

Was it relevant?

Did it appropriately answer the customer's question?

Was it coherent?

Did it remain grounded?
```

LLM judges can be useful here.

---

# 19. Layer 5 — Operational Evaluation

An AI support system also has ordinary production metrics.

Examples include:

```text
latency
error rate
cost per interaction
tool failure rate
handoff rate
token usage
availability
```

These are measured from telemetry rather than language-model judgment.

---

# 20. Why "Correctness" Is Too Broad Without Definition

The word:

```text
correct
```

can hide many separate properties.

A response could be:

```text
factually correct
```

but:

```text
behaviourally non-compliant
```

or:

```text
helpful
```

but:

```text
unsafe
```

or:

```text
accurate
```

but:

```text
operationally too expensive
```

Therefore Reliora tries to decompose broad claims into measurable properties.

---

# 21. Example: Stage-1 Bug Response

Stage-1 response:

```text
Please provide a detailed description of the bug, including the steps you took
to encounter it and the environment (browser, operating system, device) in
which it occurred.
```

At a conversational level, this seems reasonable.

A generic judge may interpret it as helpful and correct.

But the contract stated:

```text
ask only ONE missing field at a time
```

So behavioural correctness was:

```text
FAIL
```

This demonstrates why natural-language quality and workflow compliance are different.

---

# 22. Example: `HUMAN HANDOFF`

Stage-1 response:

```text
HUMAN HANDOFF

For further assistance, please contact our customer support team...
```

The response may still successfully communicate that the user should contact support.

However, the system contract explicitly prohibited exposing:

```text
HUMAN HANDOFF
```

Therefore:

```text
customer-helpfulness
```

and:

```text
internal-information containment
```

must be evaluated separately.

---

# 23. Example: Return Policy

Stage-1 answer preserved:

```text
30 days
unused
original packaging
```

but omitted:

```text
defective-item exception
```

The answer was not entirely fabricated.

It was partially correct but incomplete.

This is another reason a single binary concept such as:

```text
correct
```

can be misleading.

A more useful decomposition is:

```text
major policy facts preserved?
yes, mostly

all required facts preserved?
no
```

---

# 24. Why We Introduced `FACT-001`

`FACT-001` encodes the narrower question:

> Did this response include all facts designated as required for this evaluation case?

That is easier to defend than:

> Is the entire response universally factually correct?

The evaluator's claim is deliberately narrow.

---

# 25. Narrow Claims Are Stronger Claims

A production engineering principle used in Reliora is:

> Claim only what the evidence actually demonstrates.

For example, after the Stage-1 reproduction experiment we can say:

> Reliora's deterministic evaluation layer detected all three executable known defects represented in `stage1-reproduction-v1`.

We should not say:

> Reliora detects every possible AI failure.

The first claim has a known denominator.

The second does not.

---

# 26. The Current Reproduction Result

The Stage-1 reproduction runner produced:

```text
Total cases: 4
Executable cases: 3
Documented non-executable cases: 1
Detected known defects: 3
Mismatches: 0
All executable expectations met: True
```

This means:

```text
3 known executable defects
        ↓
3 detected
```

within that specific dataset.

---

# 27. Why REPRO-003 Was Not Forced Into a Deterministic Replay

`REPRO-003` concerns historical exposure of streamed:

```text
<thinking>...</thinking>
```

content.

The Stage-1 repository preserved evidence that the event occurred.

However, the full original leaked reasoning output was not preserved.

Reliora therefore classified the case as:

```text
DOCUMENTED
```

instead of inventing a response and pretending it had been replayed.

This is part of trustworthy evaluation design.

---

# 28. Evaluation Must Respect Evidence Quality

A useful hierarchy is:

```text
We have exact replayable evidence
→ execute evaluator

We have preserved historical observation but incomplete raw data
→ document observation

We have only an assumption
→ label assumption

We have not measured something
→ do not claim a measurement
```

Evaluation rigor includes knowing when not to produce a score.

---

# 29. Deterministic Does Not Mean Automatically Correct

A deterministic evaluator can also be badly designed.

For example:

```python
if "30 days" in response:
    return PASS
```

would be too weak for a policy requiring multiple facts.

Deterministic evaluation provides consistency.

It does not automatically provide validity.

The rule itself must still represent the requirement correctly.

---

# 30. The Evaluator Can Have Bugs

An evaluator is software.

Therefore it needs:

```text
unit tests
type checking
linting
review
version control
```

Reliora's deterministic evaluators are tested under:

```text
tests/unit/evaluation/
```

This is why the evaluation system itself is treated as production code.

---

# 31. Testing the Evaluator, Not Only the Agent

This creates two levels:

```text
Agent
→ system being evaluated

Evaluator
→ system deciding whether behaviour satisfies a requirement
```

If the evaluator is wrong, the resulting metrics can also be wrong.

Therefore:

```text
evaluation code
```

also requires engineering quality controls.

---

# 32. The Deliberate Mismatch Test

Reliora's reproduction tests include a case designed to produce:

```text
MISMATCH
```

This is important.

Without such a test, an evaluation runner could accidentally be implemented to always declare:

```text
expected result achieved
```

The mismatch test demonstrates that the runner can distinguish:

```text
expected evaluator result
```

from:

```text
unexpected evaluator result
```

---

# 33. Avoid Self-Congratulating Evaluations

A poor evaluation pipeline might effectively do:

```text
expected = failure
result = failure
therefore everything is good
```

without proving that the evaluator actually made the decision independently.

A stronger design ensures:

```text
input
        ↓
evaluator independently produces result
        ↓
runner compares actual vs expected
        ↓
DETECTED / CONFIRMED / MISMATCH
```

The comparison occurs after evaluation.

---

# 34. Evaluation Labels Must Have Clear Meaning

Reliora uses result statuses such as:

```text
DETECTED
CONFIRMED
MISMATCH
DOCUMENTED
```

Their meanings must remain distinct.

---

## `DETECTED`

For a known defective case:

> The evaluator independently rejected the known bad behaviour as expected.

---

## `CONFIRMED`

For a case expected to pass:

> The evaluator accepted the behaviour as expected.

---

## `MISMATCH`

> The evaluator result differed from the reviewed expectation.

This requires investigation.

---

## `DOCUMENTED`

> The historical case is preserved but cannot be validly replayed with available evidence.

---

# 35. Why `passed=False` Can Produce `DETECTED`

This can initially be confusing.

Suppose a known bad response is evaluated.

The evaluator produces:

```text
passed = False
```

This means:

> The response violated the evaluation rule.

But the reproduction experiment expected that known defect to fail.

Therefore the experiment status becomes:

```text
DETECTED
```

Conceptually:

```text
bad response
     ↓
evaluator says FAIL
     ↓
expected FAIL
     ↓
known defect DETECTED
```

The response did not pass.

The evaluator successfully detected the defect.

---

# 36. Evaluation of the AI vs Evaluation of the Evaluator

There are two different questions:

### Question 1

Did the Stage-1 response satisfy the contract?

For known defects:

```text
No.
```

### Question 2

Did Reliora's evaluator correctly recognize that violation?

For the three executable reproduction cases:

```text
Yes.
```

Keeping these levels separate prevents confusing terminology.

---

# 37. Determinism Helps CI/CD

Deterministic checks are especially useful in release gates.

For example:

```text
developer changes prompt/router
        ↓
CI runs behavioural suite
        ↓
INV-004 fails
        ↓
release blocked
```

Because the rule is explicit, the failure is easier to diagnose.

---

# 38. Example CI Safety Rule

Suppose a regression causes a response to expose:

```text
HUMAN HANDOFF
```

A deterministic leakage evaluator can detect it.

The pipeline could then enforce:

```text
INV-012 failures > 0
        ↓
CI failure
        ↓
do not promote release
```

This turns a prompt instruction into an executable release control.

---

# 39. Prompt Rule vs Executable Rule

Stage 1 contained instructions inside the prompt.

For example:

```text
Do not expose route names.
```

That is useful.

But a prompt instruction is not enforcement.

Reliora adds:

```text
prompt instruction
+
deterministic evaluator
+
tests
+
release gating
```

This creates multiple layers of control.

---

# 40. The LLM May Propose; Software Validates

Reliora follows the principle:

> The LLM may propose; deterministic software validates and authorizes.

For example:

```text
LLM proposes:
BUG route
+
tool arguments

deterministic software:
validates schema
checks required fields
checks policy
checks idempotency
authorizes side effect
```

This prevents every critical decision from depending only on model judgement.

---

# 41. Why This Matters More for Tool-Using Agents

A chatbot that only produces text has one class of risk.

A tool-using agent may also:

```text
create records
send requests
change system state
trigger workflows
```

A wrong answer is undesirable.

A wrong side effect can be materially worse.

Therefore exact preconditions should often be enforced outside the LLM.

---

# 42. Example: Ticket Creation

Suppose the model believes it has enough information to create a bug ticket.

Instead of trusting that belief directly:

```text
LLM:
"I think all fields are present"
        ↓
tool call
```

Reliora's direction is:

```text
LLM proposes tool call
        ↓
Pydantic / schema validation
        ↓
required-field validation
        ↓
policy validation
        ↓
idempotency validation
        ↓
authorized tool call
```

This creates a deterministic control boundary.

---

# 43. Generative Evaluation Still Remains Important

Adding deterministic checks does not remove the need for generative evaluation.

Imagine a response that:

- contains all required facts
- exposes no forbidden labels
- follows workflow rules

but is:

```text
confusing
irrelevant
hostile
poorly written
```

A deterministic contract suite may not detect those qualities well.

Generative evaluation remains valuable.

---

# 44. Layering Is Stronger Than Replacement

The architecture is not:

```text
LLM judge
        ↓
replace with deterministic checks
```

It is:

```text
deterministic checks
        +
generative evaluation
        +
operational metrics
        +
human review where appropriate
```

Each layer covers different failure modes.

---

# 45. Human Evaluation Still Matters

Some cases remain difficult even for an LLM judge.

Human review can be especially useful for:

- ambiguous policies
- evaluation-set creation
- disagreement resolution
- gold-label validation
- novel failure modes
- high-risk decisions

Automated evaluation should reduce unnecessary manual work, not pretend human judgement is never needed.

---

# 46. Gold Labels

A gold label is a reviewed expected result used as a reference.

For example:

```text
REPRO-001
expected evaluator result:
False
```

because the reviewed Stage-1 response violated the one-field-at-a-time requirement.

Gold labels should be created carefully.

If the expected label is wrong, evaluator metrics become misleading.

---

# 47. Avoid Tuning to the Test Set

Another risk is changing the evaluator until it perfectly recognizes the known cases without generalizing.

For example:

```python
if response == "exact known bad sentence":
    return FAIL
```

would detect the reproduction example but provide little value beyond memorization.

A better evaluator encodes the underlying rule.

For example:

```text
count requested missing fields
```

rather than:

```text
recognize one exact historic sentence
```

---

# 48. Test Cases Should Represent the Requirement

The goal is:

```text
example
        ↓
derive underlying failure mode
        ↓
encode general rule
```

not:

```text
example
        ↓
hardcode answer
```

This distinction separates regression engineering from test-set gaming.

---

# 49. Why Evaluation Dataset Design Matters

The quality of an evaluator cannot be judged from one example.

Datasets should eventually include:

```text
positive cases
negative cases
edge cases
ambiguous cases
adversarial cases
```

For example, a leakage evaluator should not only test:

```text
HUMAN HANDOFF
```

but also clean responses that contain no prohibited markers.

Otherwise we could accidentally build an evaluator that fails everything.

---

# 50. False Positives and False Negatives

Evaluation systems also make mistakes.

### False Positive

The evaluator reports a defect where acceptable behaviour occurred.

### False Negative

The evaluator misses an actual defect.

Both matter.

For high-risk safety requirements, false negatives can be particularly important.

But an evaluator that rejects every response is also unusable.

---

# 51. Why We Need Positive and Negative Tests

Suppose the leakage evaluator simply returns:

```python
False
```

for every response.

It would "detect" every known bad leakage case.

But it would also reject every clean response.

Therefore tests should include:

```text
bad input → fail
good input → pass
```

This checks both sides of the decision boundary.

---

# 52. Deterministic Factual Checks Also Have Limits

Reliora's current factual evaluator uses reviewed accepted patterns for required facts.

This is intentionally narrow.

For example:

```text
defective
damaged
faulty
```

may be accepted patterns for the defective-item exception.

This does not make it a general semantic fact-checking system.

It is a deterministic required-fact assertion.

---

# 53. Why We Document That Limitation

It would be misleading to call this:

```text
universal factual verification
```

It actually answers:

> Does the response contain one accepted representation for every required fact in this evaluation case?

That is still useful.

But the claim is narrower and more defensible.

---

# 54. When Structured Outputs Improve Deterministic Evaluation

A model can sometimes return structured fields rather than only prose.

For example:

```json
{
  "intent": "BUG",
  "missing_fields": ["environment"],
  "next_action": "ASK_FOR_FIELD"
}
```

This can make deterministic evaluation easier than trying to infer every internal decision from free-form text.

Structured outputs can create clearer contracts between:

```text
model reasoning layer
```

and:

```text
application control layer
```

---

# 55. Free-Form Text Should Not Be the Only Control Surface

If critical information exists only inside prose:

```text
"I think this is probably a bug and maybe we can create a ticket..."
```

software has to infer intent from text again.

A better system can use:

```text
typed structured decision
+
separate customer-facing response
```

This is one direction of Reliora's architecture.

---

# 56. Evaluation and Observability Are Related

Offline evaluation answers:

> How does the system behave on controlled cases?

Observability answers:

> What is happening during actual execution?

Eventually, the two should connect.

For example:

```text
evaluation invariant:
no premature tool execution
```

could correspond operationally to telemetry showing:

```text
validation state
tool-call attempt
authorization result
```

This makes failure diagnosis easier.

---

# 57. Evaluation and Incident Response

Suppose an operational incident reveals a new failure mode.

A mature cycle is:

```text
incident
   ↓
root-cause analysis
   ↓
new regression case
   ↓
new/updated evaluator
   ↓
CI release gate
```

This converts operational learning into automated prevention.

---

# 58. Known Deviations Should Become Tests

A particularly important lesson from Stage 1 was that a perfect-looking score can coexist with known deviations.

Once a defect is known, it should not remain merely a note.

Where possible:

```text
known defect
      ↓
reproduction case
      ↓
automated evaluator
      ↓
regression test
```

This is exactly what the Stage-1 reproduction dataset begins to do.

---

# 59. Evaluation Should Influence Architecture

Evaluation is not something that should be added only after development is complete.

If we know that a critical requirement must be deterministically measurable, architecture should expose the necessary state.

For example:

```text
missing_fields
requested_fields
tool authorization
operation_id
route
knowledge coverage
```

Structured state makes evaluation easier and more reliable.

---

# 60. Evaluation-First Development

Reliora deliberately introduced evaluation before deploying new AWS infrastructure.

The reasoning is:

```text
Known Stage-1 defects exist
        ↓
encode them first
        ↓
prove the new evaluator can detect them
        ↓
then change architecture
        ↓
rerun same controls
```

This gives the project a measurable baseline.

---

# 61. Why We Did Not Deploy AWS First

It would have been easy to immediately add:

```text
more Bedrock services
more AgentCore features
more infrastructure
```

But doing that before defining evaluation would make it harder to answer:

> Did the system actually become more reliable?

Evaluation-first development creates a measurement system before optimization.

---

# 62. The Reliora Reliability Loop

A useful long-term loop is:

```text
Requirement
    ↓
Evaluation
    ↓
Implementation
    ↓
Experiment
    ↓
Evidence
    ↓
Failure analysis
    ↓
Architecture improvement
    ↓
Regression evaluation
```

Evaluation is integrated throughout the lifecycle.

---

# 63. Metrics Need Denominators

A statement such as:

```text
100% detection
```

is incomplete without context.

For the current experiment:

```text
3 / 3 executable known defects detected
```

is clearer.

The denominator matters.

Later, larger evaluation suites may produce different results.

---

# 64. Avoiding Fake Production Claims

Reliora currently has experimental evidence.

Therefore we should say:

```text
EXPERIMENTAL:
3 of 3 executable known Stage-1 reproduction defects were detected.
```

We should not say:

```text
Production reliability is 100%.
```

No evidence currently supports that claim.

---

# 65. Evaluation Evidence Categories

Reliora uses evidence labels such as:

```text
MEASURED
OBSERVED
EXPERIMENTAL
TARGET
SYNTHETIC ASSUMPTION
```

These help prevent different evidence types from being mixed together.

---

## `OBSERVED`

Something was preserved as having occurred.

Example:

```text
Stage-1 reasoning-tag leakage documented in repository evidence.
```

---

## `EXPERIMENTAL`

Something was produced by a controlled experiment.

Example:

```text
Reliora deterministic reproduction run detected three executable known defects.
```

---

## `TARGET`

A desired future value.

Example:

```text
routing macro F1 >= 0.95
```

A target is not a measured result.

---

## `SYNTHETIC ASSUMPTION`

A value introduced for modelling or testing, not derived from real operations.

---

## `MEASURED`

A value actually measured under a defined environment and method.

Keeping these distinct strengthens project credibility.

---

# 66. Why Evaluations Need Versioning

If we change:

```text
dataset
evaluation code
prompt
model
routing logic
```

the result can change.

Therefore evaluation assets should be version controlled.

For example:

```text
stage1-reproduction-v1.json
```

has an explicit version.

Later, if the dataset materially changes, we should not silently overwrite history.

---

# 67. Why Dataset Provenance Matters

A score without knowing what dataset produced it is weak evidence.

Reliora's generated report records a SHA-256 fingerprint of the evaluation dataset.

Conceptually:

```text
evaluation report
        ↓
references exact dataset fingerprint
```

This strengthens reproducibility.

---

# 68. Why Code Provenance Will Matter Too

Eventually, strong experiment provenance can include:

```text
dataset version
dataset hash
Git commit
model identifier
prompt version
configuration
environment
timestamp
result
```

This makes comparison between runs more meaningful.

---

# 69. Deterministic Evaluation Is Not Deterministic Generation

Another important distinction:

```text
deterministic evaluator
```

does not mean:

```text
the LLM itself must always generate identical text
```

The model may remain probabilistic.

The evaluator can still deterministically inspect defined properties of the resulting behaviour.

---

# 70. Example

Two model responses might differ:

```text
Could you tell me which browser and operating system you were using?
```

and:

```text
What environment were you using when the bug occurred?
```

Both may satisfy:

```text
request exactly one missing environment field
```

The evaluator should focus on the contract rather than exact phrasing where possible.

---

# 71. Exact Match vs Contract Match

An exact-match evaluator might incorrectly require one exact sentence.

A behavioural evaluator asks:

> Did the response satisfy the rule?

This is a stronger abstraction.

For structured workflows, annotations or structured model output can make that decision more reliable.

---

# 72. When a Deterministic Rule Should Not Be Forced

Some requirements are too semantic for a simple rule.

For example:

```text
Does this answer appropriately acknowledge the user's concern without being unnecessarily verbose?
```

Trying to encode this as:

```python
word_count < 80
```

would be a poor substitute for the real requirement.

Not everything should be reduced to a simplistic deterministic check.

---

# 73. Evaluator Selection Is an Architecture Decision

A useful decision process is:

```text
Can the requirement be expressed as an explicit machine-checkable contract?
        ↓
Yes
→ deterministic evaluation preferred

No / highly semantic
→ generative or human evaluation

Mixed
→ layered evaluation
```

This is a better approach than automatically choosing an LLM judge for every AI property.

---

# 74. Why This Matters in Interviews

A strong AI engineering answer is not:

> We used an LLM judge and got a high score.

A stronger answer is:

> We decomposed system quality into measurable properties. Hard workflow and safety contracts were evaluated deterministically, semantic response quality used generative evaluation, and operational requirements used telemetry. We discovered this need after a broad correctness judge scored the original six-case Stage-1 dataset 1.00 despite reproducible behavioural and factual defects.

That demonstrates understanding of evaluation design rather than tool usage alone.

---

# 75. Important Lessons

1. AI-system quality is multidimensional.
2. One metric should not be assumed to measure every requirement.
3. Stage 1 received a 6/6 generative correctness result while known contract violations remained.
4. This does not make LLM-as-a-judge useless; it shows the need for metric-requirement alignment.
5. Deterministic evaluation is appropriate for explicit machine-checkable contracts.
6. Generative evaluation is useful for semantic and qualitative properties.
7. Some requirements need layered evaluation.
8. Hard constraints and soft qualities should not be treated identically.
9. Deterministic evaluators themselves require tests and engineering controls.
10. A known bad response failing an evaluator can mean the defect was successfully detected.
11. Evaluator outcome and experiment outcome must be kept conceptually separate.
12. Historical evidence that cannot be validly replayed should be documented rather than invented.
13. Evaluation datasets need positive, negative, edge, and eventually adversarial cases.
14. Narrow evidence-backed claims are more credible than broad unsupported claims.
15. Evaluation should influence architecture rather than being added only after implementation.
16. Known production or test deviations should become regression cases where possible.
17. Deterministic checks are particularly valuable as CI/CD release gates.
18. Tool-using agents require stronger deterministic boundaries because their failures can create side effects.
19. Structured outputs can make critical AI decisions easier to validate.
20. Reliable AI engineering combines deterministic checks, generative evaluation, operational metrics, and human review where appropriate.

---

## Interview Explanation

> Reliora was motivated by an evaluation gap in the original support agent: a Bedrock LLM-as-a-judge correctness evaluation scored all six Stage-1 cases successfully, yet repository-backed review exposed behavioural and factual contract violations, including requesting three missing bug fields at once, exposing an internal `HUMAN HANDOFF` route label, and omitting a required return-policy exception. I therefore separated hard machine-checkable requirements from semantic quality. Reliora uses deterministic evaluators for workflow, leakage, side-effect, and required-fact contracts, while retaining generative evaluation for qualities such as relevance and helpfulness. The principle is not to replace LLM judges, but to align each requirement with an evaluator that can actually measure it.