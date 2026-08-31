# Behavioural Invariants vs Factual-Completeness Evaluation

## Why This Lesson Exists

Reliora's deterministic evaluation system originally focused on behavioural rules.

Examples included:

```text
INV-004
INV-012
```

Later, we introduced:

```text
FACT-001
```

to detect a different type of failure:

> A response can follow the conversation workflow correctly but still omit a required fact.

This distinction became important enough that the shared evaluator model changed from:

```text
invariant_id
```

to:

```text
evaluation_id
```

The change was not merely a rename.

It reflected a broader evaluation architecture.

Reliora now distinguishes between:

```text
behavioural invariants
```

and:

```text
factual-completeness evaluations
```

because they answer different questions.

---

# 1. The Core Difference

A behavioural invariant asks:

> Did the system behave according to a required rule?

A factual-completeness evaluation asks:

> Did the response preserve all facts that this case requires?

These are related, but they are not the same property.

---

# 2. Behaviour vs Content

Consider a support agent answering a return-policy question.

The agent could:

```text
route correctly
avoid internal leakage
avoid unnecessary tool calls
respond politely
```

and still omit an important policy exception.

Its behaviour may therefore be acceptable in some dimensions while its content is incomplete.

Conversely, an answer could contain every correct fact but expose an internal route label.

Then the content may be factually complete while the behaviour violates a system rule.

This gives us two independent dimensions:

```text
How did the system behave?

What information did the response preserve?
```

---

# 3. Behavioural Invariants

Reliora uses identifiers such as:

```text
INV-004
INV-012
```

for behavioural invariants.

An invariant represents a condition that should remain true when the system operates correctly.

---

## Mental Model

```text
System interaction
      ↓
required behavioural rule
      ↓
deterministic check
      ↓
PASS / FAIL
```

---

# 4. Example: `INV-004`

The Stage-1 bug-report workflow required the model to ask for:

```text
only ONE missing field at a time
```

Required fields included:

```text
description
steps to reproduce
environment
```

The Stage-1 response asked for all three in one message.

Conceptually:

```text
Missing fields:
description
steps_to_reproduce
environment

Requested fields:
description
steps_to_reproduce
environment
```

The rule required:

```text
exactly one
```

but received:

```text
three
```

Therefore:

```text
INV-004 → FAIL
```

---

# What `INV-004` Is Measuring

It is not primarily asking:

> Was the information requested factually correct?

All three requested fields were genuinely required.

The problem was:

```text
the interaction sequence violated the workflow contract
```

This is behavioural correctness.

---

# 5. Example: `INV-012`

The frozen Stage-1 prompt prohibited exposing internal route names.

However, one response began with:

```text
HUMAN HANDOFF
```

The response may still have directed the customer toward support correctly.

But the customer-visible output contained an internal routing label.

Therefore:

```text
INV-012 → FAIL
```

---

# What `INV-012` Is Measuring

It asks:

> Did prohibited internal information appear in customer-visible output?

This is a behavioural and containment property.

It does not ask whether the rest of the response was factually accurate.

---

# 6. Why Behavioural Invariants Are Valuable

LLMs are flexible language generators.

That flexibility is useful, but critical workflows often require exact boundaries.

Examples include:

```text
do not call a tool before validation
do not expose internal reasoning markers
do not fabricate identifiers
do not create duplicate side effects
ask one required field at a time
do not bypass human approval
```

These are stronger than stylistic preferences.

They can often be expressed as executable contracts.

---

# 7. Factual-Completeness Evaluation

Reliora introduced the:

```text
FACT-*
```

family for requirements about information that must be preserved in the answer.

The current example is:

```text
FACT-001
```

---

## Mental Model

```text
Required facts
      +
candidate response
      ↓
fact-presence evaluation
      ↓
complete / incomplete
```

---

# 8. The Stage-1 Return-Policy Case

The frozen FAQ stated that most items could be returned:

```text
within 30 days of delivery
```

provided they were:

```text
unused
```

and:

```text
in original packaging
```

with an important exception when:

```text
the item arrived defective
```

The Stage-1 answer included:

```text
30 days
unused
original packaging
```

but omitted:

```text
defective-item exception
```

---

# 9. Why This Was Not an `INV-*` Failure

Nothing about this particular defect necessarily required a workflow invariant such as:

```text
ask exactly one field
```

or:

```text
do not expose route labels
```

The failure was:

> The answer did not preserve all required policy information.

That belongs to a different evaluation family.

Therefore Reliora introduced:

```text
FACT-001
```

---

# 10. `FACT-001`

The current deterministic factual evaluator receives a set of required facts.

Conceptually:

```python
required_facts = {
    "return_window": ["30 days"],
    "unused_condition": ["unused"],
    "original_packaging": ["original packaging"],
    "defective_item_exception": [
        "defective",
        "damaged",
        "faulty",
    ],
}
```

Each fact can have one or more accepted textual patterns.

The evaluator asks:

> Does the response contain at least one accepted representation for every required fact?

---

# 11. Why Multiple Accepted Patterns Matter

The same fact can be expressed in different ways.

For example:

```text
defective
damaged
faulty
```

may represent the same policy concept in the context of the case.

If the evaluator accepted only:

```text
defective
```

then a legitimate response containing:

```text
damaged item
```

could be rejected unnecessarily.

---

# 12. What the Factual Evaluator Returns

The evaluator can preserve evidence such as:

```text
matched facts:
- return_window
- unused_condition
- original_packaging

missing facts:
- defective_item_exception
```

This provides more diagnostic information than only:

```text
FAIL
```

---

# 13. Why Diagnostic Evidence Matters

A useful evaluator should help answer:

> Why did this case fail?

For example:

```text
FACT-001 failed
```

is less useful than:

```text
FACT-001 failed

Matched:
- return_window
- unused_condition
- original_packaging

Missing:
- defective_item_exception
```

The second form helps developers identify the exact regression.

---

# 14. A Response Can Pass One Family and Fail Another

This is one of the most important concepts.

Imagine:

```text
Response:
"You can return most unused items in their original packaging within 30 days."
```

It may:

```text
PASS INV-012
```

because it exposes no internal route labels.

But:

```text
FAIL FACT-001
```

if the defective-item exception is required.

---

# 15. The Reverse Is Also Possible

Consider:

```text
HUMAN HANDOFF

You can return most items within 30 days if they are unused and in their
original packaging, unless the item arrived defective.
```

This may preserve every required return-policy fact.

So:

```text
FACT-001 → PASS
```

But it exposes:

```text
HUMAN HANDOFF
```

Therefore:

```text
INV-012 → FAIL
```

---

# 16. Why One Overall "Correct" Label Can Hide This

If we collapse everything into:

```text
correct / incorrect
```

we lose information about the failure mode.

A better decomposition is:

```text
Behavioural compliance:
PASS / FAIL

Factual completeness:
PASS / FAIL

Generative quality:
score

Operational performance:
metrics
```

This makes diagnosis much easier.

---

# 17. Multi-Dimensional Evaluation

An AI response should be considered across several dimensions.

For example:

| Dimension | Example Question |
|---|---|
| Routing | Did it choose the right workflow? |
| Behaviour | Did it obey interaction rules? |
| Leakage | Did it expose internal information? |
| Factual completeness | Did it preserve required facts? |
| Tool safety | Were actions validated? |
| Generative quality | Was the answer useful and clear? |
| Operations | Was latency/cost acceptable? |

A response can perform differently across these dimensions.

---

# 18. Why `evaluation_id` Is Better Than `invariant_id`

Originally, the shared model contained:

```text
invariant_id
```

That assumed every evaluator represented an invariant.

Once:

```text
FACT-001
```

was introduced, this abstraction became inaccurate.

The more general field:

```text
evaluation_id
```

can represent:

```text
INV-004
INV-012
FACT-001
```

without pretending they are all the same type of requirement.

---

# 19. Evaluation Families

A useful design pattern is:

```text
EvaluationFinding
        ↓
evaluation_id
        ↓
prefix identifies family
```

For example:

```text
INV-*
→ behavioural invariants

FACT-*
→ factual-completeness checks
```

Additional families could be added later if a real requirement justifies them.

---

# 20. Why Prefixes Are Useful

An ID such as:

```text
INV-004
```

communicates more than:

```text
004
```

The prefix provides domain context.

Similarly:

```text
FACT-001
```

immediately communicates that the finding concerns factual evaluation.

This helps with:

- reporting
- filtering
- dashboards
- CI failure messages
- documentation
- incident investigation

---

# 21. Prefixes Should Not Become Arbitrary

Creating many evaluation families without need could make the system confusing.

The principle should be:

> Introduce a new evaluation family only when it represents a meaningfully different kind of requirement.

The distinction between:

```text
behaviour
```

and:

```text
factual completeness
```

meets that threshold.

---

# 22. Deterministic Does Not Mean the Families Are Identical

Both:

```text
INV-004
```

and:

```text
FACT-001
```

currently use deterministic evaluators.

That does not mean they represent the same property.

The evaluation method and the evaluation domain are separate questions.

For example:

```text
Method:
deterministic

Domain:
behaviour
```

versus:

```text
Method:
deterministic

Domain:
factual completeness
```

---

# 23. Evaluation Method vs Evaluation Property

A useful two-axis mental model is:

```text
What property are we measuring?
        +
How are we measuring it?
```

For example:

| Property | Method |
|---|---|
| Bug workflow compliance | Deterministic |
| Internal leakage | Deterministic |
| Required fact completeness | Deterministic |
| Helpfulness | LLM judge |
| Latency | Telemetry |
| Duplicate side effect | Integration test |

This prevents the architecture from confusing a tool with the requirement itself.

---

# 24. Why Factual Completeness Is Not the Same as Factual Correctness

This distinction is important.

`FACT-001` currently answers:

> Were all required facts represented?

It does not necessarily prove:

> Every statement in the response is factually true.

---

# Example

Suppose a response says:

```text
Returns are allowed within 30 days for unused items in original packaging,
including defective items, and every refund is processed within 30 seconds.
```

The response may include all required policy facts.

However:

```text
every refund is processed within 30 seconds
```

could be unsupported fabrication.

A required-fact completeness check alone would not necessarily detect that.

---

# 25. Completeness vs Unsupported Claims

These are separate questions:

```text
Did the answer omit required information?
```

and:

```text
Did the answer invent unsupported information?
```

A mature evaluation system should eventually test both.

---

# 26. Why `FACT-001` Is Intentionally Narrow

The evaluator does not claim to be:

```text
a universal fact checker
```

Its claim is:

> For this evaluation case, check whether every reviewed required fact has an accepted representation in the candidate response.

This is narrower but more defensible.

---

# 27. Narrow Evaluators Are Easier to Trust

Compare:

```text
Evaluator A:
"Determines whether an answer is completely correct."
```

with:

```text
Evaluator B:
"Checks whether the four required return-policy facts are present."
```

Evaluator B has:

- a clearer contract
- a known scope
- easier tests
- easier debugging
- more defensible evidence

This is valuable in reliability engineering.

---

# 28. Behavioural Invariants Also Need Narrow Definitions

An invariant should not be:

```text
INV-001:
The agent must behave well.
```

That is too vague.

A stronger invariant is:

```text
When one or more required bug fields are missing,
the response requests exactly one missing field.
```

That can be tested.

---

# 29. Machine-Checkable Requirements

A useful requirement asks:

> What observable state would prove this condition was satisfied or violated?

For `INV-004`:

```text
missing_fields
requested_fields
```

For `INV-012`:

```text
customer-visible response
prohibited markers
```

For `FACT-001`:

```text
required facts
accepted patterns
response
```

The architecture should expose enough information to evaluate these conditions.

---

# 30. Structured State Improves Behavioural Evaluation

For a production workflow, it is stronger to have structured information such as:

```json
{
  "missing_fields": ["environment"],
  "next_action": "ASK_FOR_FIELD"
}
```

than to infer everything from free-form prose.

This can make invariants easier to verify.

---

# 31. Structured Knowledge Can Improve Factual Evaluation

Similarly, a policy response could be generated from structured policy data.

For example:

```json
{
  "return_window_days": 30,
  "must_be_unused": true,
  "original_packaging_required": true,
  "defective_item_exception": true
}
```

This can provide a clearer factual contract than relying only on free-text source documents.

---

# 32. Do Not Confuse Customer Text With Internal Contracts

The customer should receive natural language.

Internally, the system can maintain structured state.

Conceptually:

```text
Structured internal contract
        ↓
validated by software
        ↓
customer-facing natural language
```

The system does not need to expose its internal schema to the customer.

---

# 33. Evaluation Findings as Structured Evidence

A finding such as:

```text
evaluation_id
passed
message
evidence
```

allows a failing case to carry machine-readable evidence.

For example:

```json
{
  "evaluation_id": "FACT-001",
  "passed": false,
  "evidence": {
    "matched_facts": [
      "return_window",
      "unused_condition",
      "original_packaging"
    ],
    "missing_facts": [
      "defective_item_exception"
    ]
  }
}
```

This is useful for both humans and automation.

---

# 34. Why Structured Evidence Helps CI/CD

Instead of a pipeline receiving only:

```text
test failed
```

it can eventually receive something like:

```text
FACT-001 failed
missing_fact = defective_item_exception
```

That gives engineers immediate diagnostic information.

---

# 35. Why Structured Evidence Helps Observability

The same evaluation metadata could eventually be attached to:

- traces
- evaluation runs
- dashboards
- regression reports
- release records

This creates a common vocabulary between:

```text
offline evaluation
```

and:

```text
operational reliability
```

---

# 36. Relationship to the Reproduction Dataset

The Stage-1 reproduction dataset identifies which evaluation should be used for each case.

Examples:

```text
REPRO-001
→ INV-004

REPRO-002
→ INV-012

REPRO-003
→ INV-012
but non-executable historical evidence

REPRO-004
→ FACT-001
```

The reproduction runner uses:

```text
evaluation_id
```

to dispatch the case to the appropriate evaluator.

---

# 37. Why the Runner Needs Evaluation Families

Conceptually:

```text
case
 ↓
evaluation_id
 ↓
which evaluator owns this contract?
 ↓
dispatch
```

For example:

```text
INV-004
→ bug_workflow evaluator

INV-012
→ leakage evaluator

FACT-001
→ factual evaluator
```

The ID therefore participates in executable architecture, not only documentation.

---

# 38. Why REPRO IDs and Evaluation IDs Are Different

This distinction is also important.

For example:

```text
REPRO-004
```

identifies a specific historical reproduction case.

```text
FACT-001
```

identifies the rule used to evaluate that case.

These serve different purposes.

---

# Mental Model

```text
REPRO-004
→ Which example are we testing?

FACT-001
→ Which evaluation rule applies?
```

One evaluator can eventually be applied to many cases.

---

# 39. Test Case vs Evaluation Rule

A test case contains:

```text
input
expected behaviour
provenance
```

An evaluation rule contains:

```text
logic for deciding whether the requirement is satisfied
```

These should remain separate.

Otherwise, the evaluator risks memorizing the case.

---

# 40. Why Hardcoding Historical Responses Would Be Weak

A poor evaluator could do:

```python
if response == "exact Stage-1 response":
    return False
```

That would detect the known example but not the underlying failure mode.

A stronger behavioural evaluator encodes:

```text
exactly one missing field requested
```

A stronger factual evaluator encodes:

```text
all required case facts represented
```

This is more general.

---

# 41. General Rule, Specific Case

A useful architecture is:

```text
Specific historical case
        ↓
general evaluation rule
        ↓
independent decision
```

For example:

```text
REPRO-004
        ↓
FACT-001
        ↓
required-fact completeness
```

The evaluator should not need to know:

```text
this is REPRO-004
```

in order to make its decision.

---

# 42. Positive and Negative Cases

Both evaluation families need cases that should:

```text
PASS
```

and cases that should:

```text
FAIL
```

Otherwise, an evaluator could accidentally reject everything.

---

# Behavioural Example

### Should Pass

```text
Missing:
environment

Requested:
environment
```

### Should Fail

```text
Missing:
description, steps, environment

Requested:
description, steps, environment
```

---

# Factual Example

### Should Pass

Response includes:

```text
30 days
unused
original packaging
defective exception
```

### Should Fail

Response omits:

```text
defective exception
```

---

# 43. False Positives

A false positive occurs when an evaluator reports a defect even though acceptable behaviour occurred.

Example:

```text
Response:
"Damaged items may still qualify for return."

Evaluator only accepts:
"defective"

Result:
FAIL
```

If:

```text
damaged
```

is legitimately equivalent in the reviewed policy context, the evaluator is too narrow.

This is why accepted patterns need careful design.

---

# 44. False Negatives

A false negative occurs when an evaluator misses a real defect.

For example:

```text
Response:
"HUMAN-HANDOFF"
```

If the leakage evaluator checks only the exact string:

```text
HUMAN HANDOFF
```

it might miss a meaningful variant.

Evaluation design must consider realistic evasion and variation.

---

# 45. Deterministic Evaluation Still Requires Dataset Expansion

The initial Stage-1 reproduction cases establish a foundation.

They do not prove the evaluators are complete.

Future evaluation suites should include:

```text
clean examples
known defects
edge cases
format variations
adversarial cases
ambiguous cases
```

Metrics can then include false-positive and false-negative behaviour.

---

# 46. Evaluation Contracts Should Evolve Carefully

If we change what:

```text
INV-004
```

means, historical results may no longer be comparable.

Therefore evaluation definitions should be version-controlled and changed deliberately.

Potential strategies later could include:

```text
new rule ID
rule version
dataset version
documented semantic change
```

depending on the scale of the system.

---

# 47. Do Not Silently Change Gold Expectations

Suppose an evaluator begins failing a known case after implementation changes.

The wrong response is:

> Change the expected label until the test becomes green.

Instead:

```text
investigate
        ↓
is the requirement wrong?
is the annotation wrong?
is the evaluator wrong?
did application behaviour regress?
```

Only then should the expectation change.

---

# 48. Requirements Are the Source of Evaluation Meaning

Evaluation rules should trace back to documented requirements.

Conceptually:

```text
Requirement
      ↓
evaluation contract
      ↓
evaluator implementation
      ↓
test cases
      ↓
generated evidence
```

Without the requirement, a metric can become arbitrary.

---

# 49. Example Traceability: `INV-004`

```text
Stage-1 workflow requirement
"Ask only ONE missing field at a time."
        ↓
Reliora behavioural contract
INV-004
        ↓
bug_workflow.py
        ↓
unit tests
        ↓
REPRO-001
        ↓
generated reproduction evidence
```

This forms a traceable chain.

---

# 50. Example Traceability: `FACT-001`

```text
Frozen return-policy source
        ↓
reviewed required facts
        ↓
FACT-001
        ↓
factual.py
        ↓
unit tests
        ↓
REPRO-004
        ↓
generated reproduction evidence
```

Again, the evaluation is linked to a source requirement rather than invented after the result.

---

# 51. Why This Matters for Auditing

If someone asks:

> Why did this case fail?

we should be able to answer:

```text
Here is the requirement.

Here is the evaluation rule.

Here is the input case.

Here is the evaluator evidence.

Here is the generated result.
```

That is much stronger than:

> The model judge said it was bad.

---

# 52. Why This Matters for Interview Explanations

A strong explanation distinguishes:

```text
behavioural correctness
```

from:

```text
factual completeness
```

For example:

> I do not use one broad correctness score for every requirement. Workflow constraints and leakage rules are encoded as behavioural invariants, while required policy information is evaluated through factual-completeness assertions. This lets us identify exactly why a response failed rather than collapsing every failure into one generic score.

That demonstrates system-level evaluation thinking.

---

# 53. Relationship to Guardrails

Guardrails can help enforce some classes of content or safety policy.

However:

```text
behavioural invariants
```

and:

```text
factual completeness
```

remain useful even if guardrails are added.

A guardrail may detect unsafe content but not know:

```text
whether exactly one bug field was requested
```

or:

```text
whether the defective-item exception was omitted
```

Evaluation remains requirement-specific.

---

# 54. Relationship to Structured Outputs

Structured outputs can make both families stronger.

For behavioural evaluation:

```json
{
  "next_action": "ASK_FOR_FIELD",
  "requested_field": "environment"
}
```

For factual grounding:

```json
{
  "policy_facts_used": [
    "return_window",
    "unused_condition",
    "original_packaging",
    "defective_item_exception"
  ]
}
```

Software can validate these contracts before or alongside generating customer-facing text.

---

# 55. Relationship to Side-Effect Safety

Some future invariants will protect external actions.

Examples could include:

```text
all required fields complete before tool call
valid schema before tool call
idempotency key present
tool result received before displaying ticket ID
```

These are behavioural invariants rather than factual-completeness checks.

The distinction will become even more important as Reliora adds tool safety.

---

# 56. Relationship to RAG

If Reliora later uses a knowledge base or RAG architecture, factual-completeness evaluation still matters.

Retrieval success does not guarantee:

```text
all required retrieved facts appear in the final answer
```

The system could retrieve the correct policy but omit part of it during generation.

Therefore:

```text
retrieval quality
```

and:

```text
answer factual completeness
```

are separate evaluation dimensions.

---

# 57. Relationship to Hallucination

Hallucination often concerns unsupported or fabricated content.

Factual incompleteness concerns missing required content.

These are opposites in one sense:

```text
Hallucination
→ says something unsupported

Incompleteness
→ fails to say something required
```

A robust factual evaluation strategy eventually needs to consider both.

---

# 58. Why Omissions Matter

AI reliability discussions often focus on hallucination.

But omissions can also create material problems.

Examples include omitting:

```text
policy exceptions
safety warnings
eligibility conditions
limitations
required approval steps
refund restrictions
```

An answer can contain no fabricated facts and still be operationally dangerous because a critical condition was left out.

---

# 59. Factual Completeness as a Reliability Property

For customer support, the requirement is not only:

> Do not invent policy.

It may also be:

> Preserve material policy conditions and exceptions.

This makes factual completeness an important reliability property in its own right.

---

# 60. Evaluation IDs Are Not Business Metrics

Identifiers such as:

```text
INV-004
FACT-001
```

identify evaluation contracts.

They are not themselves performance metrics.

Metrics can be aggregated from them later.

For example:

```text
INV-004 pass rate
FACT-001 pass rate
```

over a defined dataset.

The denominator must always be known.

---

# 61. Example Metric

Suppose a future dataset contains:

```text
50 FACT-001 cases
```

and:

```text
47 pass
3 fail
```

Then we could report:

```text
FACT-001 pass rate:
47 / 50 = 94%
```

That is much more meaningful than saying:

```text
factual quality is 94%
```

because the evaluation scope is explicit.

---

# 62. Evaluation IDs Improve Failure Aggregation

Suppose CI reports:

```text
INV-004:
3 failures

INV-012:
0 failures

FACT-001:
5 failures
```

Engineers can immediately see:

```text
workflow collection is regressing
factual completeness is the largest current issue
leakage checks remain clean
```

A single overall score would hide this structure.

---

# 63. Why Aggregate Scores Can Be Dangerous

Suppose:

```text
99 cases pass
1 severe safety invariant fails
```

An overall metric might show:

```text
99%
```

which looks excellent.

But the one failure could be:

```text
unauthorized financial action
```

Some requirements should therefore be treated as hard gates rather than averaged away.

---

# 64. Hard-Gate Invariants

For critical requirements, Reliora may eventually enforce:

```text
any failure
→ release blocked
```

Examples could include:

```text
fabricated ticket ID
internal reasoning leakage
premature high-risk tool execution
duplicate side effect
```

These should not necessarily be compensated for by high scores elsewhere.

---

# 65. Factual Requirements Can Also Become Hard Gates

Some factual omissions may be material enough to block a release.

For example:

```text
safety warning
legal restriction
eligibility exception
refund exclusion
```

The evaluation family does not determine severity by itself.

Severity depends on the requirement.

---

# 66. Type and Severity Are Different Concepts

For example:

```text
FACT-001
```

describes the type of evaluation.

Separately, the system could classify failure severity as:

```text
low
medium
high
critical
```

The two concepts should not be mixed.

---

# 67. Evaluation Family and Risk Policy

A mature system might eventually connect:

```text
evaluation family
+
specific rule
+
severity
```

to:

```text
CI action
operational alert
human review
release block
```

For now, Reliora is establishing the evaluation contracts first.

---

# 68. Important Lessons

1. Behavioural invariants and factual-completeness checks measure different properties.
2. `INV-*` identifies behavioural rules.
3. `FACT-*` identifies required-fact evaluation.
4. A response can pass behavioural checks and fail factual completeness.
5. A response can be factually complete while violating behavioural safety.
6. This is why one generic correctness label can hide important failure modes.
7. `evaluation_id` is more accurate than `invariant_id` for a shared multi-family result model.
8. Deterministic evaluation is a method; behavioural and factual evaluation are domains.
9. Factual completeness is not identical to universal factual correctness.
10. Required-fact checking does not automatically detect unsupported fabrication.
11. Narrow evaluation contracts are easier to test, explain, and defend.
12. REPRO IDs identify cases, while evaluation IDs identify rules.
13. Evaluators should encode the underlying requirement rather than memorize historical responses.
14. Both positive and negative cases are necessary to evaluate an evaluator.
15. False positives and false negatives matter even for deterministic checks.
16. Evaluation requirements should trace back to source policy or documented behavioural contracts.
17. Critical evaluation failures may need hard release gates rather than averaging into one score.
18. Factual omissions can be material even when no hallucination occurs.
19. Structured outputs can strengthen both behavioural and factual validation.
20. Separating evaluation families makes debugging, reporting, CI/CD, and interview explanations more precise.

---

## Interview Explanation

> In Reliora, I separate behavioural invariants from factual-completeness evaluation because they represent different failure modes. `INV-*` checks whether the agent obeyed workflow or containment rules, such as requesting exactly one missing bug field or avoiding internal route leakage. `FACT-*` checks whether a response preserved all facts designated as required for a case, such as the defective-item exception in the return policy. A response can pass one family and fail the other, so collapsing them into one generic correctness metric would hide useful diagnostic information. This separation also drove the shared model change from `invariant_id` to the more general `evaluation_id`.