# Evidence Classes and Claim Discipline

## Why This Lesson Exists

One of the easiest ways to weaken an AI engineering project is to make claims that are broader than the evidence supporting them.

For example, these statements are very different:

```text
Reliora detected 3 of 3 executable known Stage-1 defects
in the stage1-reproduction-v1 experiment.
```

and:

```text
Reliora is 100% reliable.
```

The first statement is supported by a defined experiment.

The second statement is not.

Reliora therefore uses explicit evidence classes:

```text
MEASURED
OBSERVED
EXPERIMENTAL
TARGET
SYNTHETIC ASSUMPTION
```

These labels help answer:

> What kind of evidence supports this statement?

This document explains what each evidence class means, why the distinction matters, and how to translate technical results into defensible engineering claims.

---

# 1. Evidence Is Not All the Same

During a project, we may encounter information from many sources.

For example:

```text
a historical log
a manually observed failure
a controlled local experiment
a future reliability objective
a synthetic workload assumption
a measured latency result
```

All of these can be useful.

But they should not be presented as though they have the same evidentiary strength.

---

# 2. The Core Evidence Classes

Reliora currently distinguishes:

```text
OBSERVED
EXPERIMENTAL
MEASURED
TARGET
SYNTHETIC ASSUMPTION
```

A useful mental model is:

```text
What happened historically?
→ OBSERVED

What happened in a controlled experiment?
→ EXPERIMENTAL

What value did we actually measure using a defined method?
→ MEASURED

What value do we want to achieve?
→ TARGET

What value did we invent deliberately for modelling/testing?
→ SYNTHETIC ASSUMPTION
```

---

# 3. `OBSERVED`

## Meaning

`OBSERVED` means:

> Evidence preserves that something occurred, but it was not necessarily produced by the current controlled experiment.

This is often historical evidence.

---

## Reliora Example

The frozen Stage-1 repository preserved evidence that streamed:

```text
<thinking>...</thinking>
```

content had been visible during development execution.

The repository contained references such as:

```text
[streamed <thinking> content visible]
```

and documentation describing the observation.

Therefore the historical failure can be classified as:

```text
OBSERVED
```

---

# 4. What `OBSERVED` Does Not Mean

It does not automatically mean:

```text
we can reproduce the exact original event
```

or:

```text
we retained every original byte of output
```

This distinction became important for:

```text
REPRO-003
```

because the repository preserved evidence that the leakage occurred, but not the complete original leaked reasoning text.

---

# 5. Why REPRO-003 Remains `DOCUMENTED`

Because the exact raw historical response was not preserved, we did not reconstruct one and pretend it was original evidence.

Instead:

```text
historical failure
→ OBSERVED

reproduction status
→ DOCUMENTED
```

This preserves the known failure without fabricating replay evidence.

---

# 6. `EXPERIMENTAL`

## Meaning

`EXPERIMENTAL` means:

> A result was produced through a controlled experiment conducted during the current project.

The experiment should ideally define:

```text
input
procedure
code
expected result
actual result
scope
```

---

# 7. Reliora Example

The Stage-1 reproduction experiment used:

```text
evals/datasets/stage1-reproduction-v1.json
```

and:

```text
evals/runners/run_stage1_reproduction.py
```

to execute Reliora's deterministic evaluators.

The result was:

```text
Total cases: 4
Executable cases: 3
Documented non-executable cases: 1
Detected known defects: 3
Mismatches: 0
All executable expectations met: True
```

This result is:

```text
EXPERIMENTAL
```

---

# 8. A Defensible Experimental Claim

A defensible statement is:

> In `stage1-reproduction-v1`, Reliora's deterministic evaluation layer detected all 3 of the 3 executable known Stage-1 defects represented in the reviewed dataset.

This tells the reader:

```text
which experiment
which evaluator class
which cases
numerator
denominator
scope
```

---

# 9. An Unsupported Version

The following would be unsupported:

```text
Reliora detects 100% of AI failures.
```

The experiment did not contain:

```text
every possible AI failure
```

It contained:

```text
three executable known defects
```

A result must not be generalized beyond its dataset without additional evidence.

---

# 10. `MEASURED`

## Meaning

`MEASURED` means:

> A value was actually collected using a defined measurement method and environment.

Examples may eventually include:

```text
latency
token usage
cost
error rate
tool success rate
routing accuracy
memory consumption
```

---

# 11. Measurement Needs a Method

A number alone is not enough.

For example:

```text
latency = 850 ms
```

raises questions:

```text
mean or p95?

local or AWS?

how many requests?

which model?

which region?

warm or cold execution?

what request size?

what time period?
```

A meaningful measured claim needs context.

---

# 12. Example of a Better Measurement Claim

Instead of:

```text
Latency is 850 ms.
```

a stronger statement would resemble:

```text
MEASURED:
On 100 requests against configuration X in us-east-1,
p95 end-to-end latency was 850 ms.
```

The exact methodology would be documented alongside the result.

---

# 13. Why `MEASURED` and `EXPERIMENTAL` Can Overlap

A controlled experiment can produce measured values.

For example:

```text
router experiment
→ EXPERIMENTAL setup

macro F1 = 0.96
→ MEASURED result within that experiment
```

The categories describe different aspects of the evidence.

---

# 14. `TARGET`

## Meaning

`TARGET` means:

> A value or condition the system aims to achieve.

A target is not evidence that the condition has already been achieved.

---

# 15. Reliora Target Examples

Current design targets include values such as:

```text
schema validity:
100% accepted or safely rejected

premature tool execution:
0

fabricated ticket IDs:
0

duplicate retry side effects:
0

routing macro F1:
>= 0.95

routing per-class F1:
>= 0.90
```

These are:

```text
TARGETS
```

until they are actually evaluated.

---

# 16. Why the Label Matters

Compare:

```text
Target routing macro F1 >= 0.95
```

with:

```text
Routing macro F1 = 0.95
```

The first describes an engineering goal.

The second claims a measurement.

Confusing the two would misrepresent project maturity.

---

# 17. A Target Can Later Become a Measured Result

The lifecycle may look like:

```text
Define target:
macro F1 >= 0.95
        ↓
run routing benchmark
        ↓
measure:
macro F1 = 0.962
        ↓
compare measurement with target
        ↓
target satisfied
```

The original target and the measured outcome should still remain conceptually distinct.

---

# 18. `SYNTHETIC ASSUMPTION`

## Meaning

`SYNTHETIC ASSUMPTION` means:

> A value was deliberately invented or simulated for design, testing, or modelling because real production data was unavailable.

Synthetic assumptions can be useful.

They simply must not be presented as observed business reality.

---

# 19. Example

Suppose we later model cost under:

```text
10,000 support conversations per day
```

but Reliora has never handled real production traffic.

That workload would be:

```text
SYNTHETIC ASSUMPTION
```

not:

```text
MEASURED production volume
```

---

# 20. Why Synthetic Assumptions Are Legitimate

Architecture often requires planning before real production data exists.

We may need assumptions to evaluate:

```text
cost
scaling
capacity
queue depth
storage
load
```

The problem is not using assumptions.

The problem is hiding that they are assumptions.

---

# 21. Example of Honest Cost Modelling

A defensible statement might be:

> Using a synthetic workload assumption of 10,000 support interactions per day, the estimated monthly service cost is approximately X under configuration Y.

That is very different from:

> Reliora serves 10,000 customers per day.

Only the first statement matches the evidence.

---

# 22. Claims Should Trace to Evidence

A useful principle is:

```text
Claim
   ↓
What evidence supports it?
   ↓
Which artifact contains that evidence?
   ↓
What evidence class is it?
```

If that chain cannot be established, the claim should be weakened or removed.

---

# 23. Evidence Chain Example

Consider:

> Reliora detected the known Stage-1 one-field-at-a-time workflow defect.

The evidence chain is:

```text
Stage-1 contract
        ↓
OBSERVED Stage-1 response
        ↓
REPRO-001 dataset case
        ↓
INV-004 evaluator
        ↓
EXPERIMENTAL reproduction run
        ↓
generated report
        ↓
DETECTED
```

This is traceable.

---

# 24. Claims Need Denominators

Percentages are particularly dangerous when the denominator is hidden.

Suppose we say:

```text
100% defect detection
```

That sounds broad.

The actual result was:

```text
3 / 3 executable known reproduction defects
```

Those statements are mathematically related but communicate very different scope.

---

# 25. Better Percentage Reporting

Instead of:

```text
100% success
```

prefer:

```text
3 of 3 executable cases produced the reviewed expected result.
```

or:

```text
100% (3/3) of executable known defects in stage1-reproduction-v1 were detected.
```

The denominator makes the statement auditable.

---

# 26. Why Small Samples Need Context

Suppose a model passes:

```text
6 / 6
```

cases.

That does not mean:

```text
the model has 100% real-world accuracy
```

It means:

```text
it passed 6 of 6 cases in that specific evaluation
```

The Stage-1 project itself demonstrated why this distinction matters.

---

# 27. Perfect Scores Can Be Misleading

A score of:

```text
1.00
```

looks impressive.

But its usefulness depends on:

```text
what was measured
how it was measured
which cases were included
which failures the evaluator can detect
```

The metric should never be interpreted independently from the evaluation design.

---

# 28. Evidence Strength Is Not Only About Numbers

A number is not automatically stronger than qualitative evidence.

For example:

```text
OBSERVED:
repository preserved a prohibited route label
```

may provide extremely strong evidence for that specific defect.

Meanwhile:

```text
0.95 score
```

may be weak if the metric's definition is unclear.

Evidence quality depends on traceability and relevance to the claim.

---

# 29. Direct Evidence vs Inference

Another useful distinction is:

```text
direct evidence
```

versus:

```text
inference
```

For example:

```text
Stage-1 output literally contains:
HUMAN HANDOFF
```

is direct evidence of route-label exposure.

Saying:

```text
This likely indicates the internal router chose a handoff route.
```

is an inference.

Both can be useful, but they should not be confused.

---

# 30. Do Not Reconstruct Missing Evidence

Suppose a historical record says:

```text
thinking content was visible
```

but the full content is unavailable.

We should not create:

```text
<thinking>
invented historical reasoning
</thinking>
```

and present it as the original output.

That converts:

```text
unknown historical content
```

into:

```text
fabricated evidence
```

---

# 31. Synthetic Cases Must Be Labeled as Synthetic

It may still be useful to create a new adversarial case containing:

```text
<thinking>
synthetic internal content
</thinking>
```

to test the leakage evaluator.

That is legitimate if it is labeled:

```text
SYNTHETIC
```

It simply cannot be presented as:

```text
the original Stage-1 response
```

---

# 32. Evidence Should Preserve Uncertainty

Engineering documentation should sometimes say:

```text
unknown
not measured
not preserved
not yet evaluated
```

These are valid outcomes.

Forcing every field to contain a confident answer creates false precision.

---

# 33. `TBD` Can Be Better Than an Invented Number

Reliora currently leaves some operational objectives such as:

```text
p95 latency
cost per interaction
```

as:

```text
TBD until measured
```

This is better than inventing attractive numbers simply to make the architecture look complete.

---

# 34. Targets Should Be Justifiable

Even targets should not be random.

For example:

```text
routing macro F1 >= 0.95
```

should eventually be justified based on:

```text
business risk
routing consequences
class distribution
fallback policy
human handoff behaviour
```

Targets are design decisions, not decorative numbers.

---

# 35. Measurement Without Environment Is Weak

Suppose we measure:

```text
300 ms
```

locally.

It would be misleading to label that:

```text
production AWS latency
```

The environment matters.

A measurement should indicate whether it came from:

```text
local laptop
local mock
AWS sandbox
personal AWS account
CI runner
production-like environment
```

---

# 36. Local Experimental Evidence Is Still Valuable

A result does not need to come from production to be useful.

For example:

```text
local deterministic reproduction
```

can strongly demonstrate that evaluator logic detects known defects.

The correct claim is simply:

```text
local experimental evidence
```

rather than:

```text
production reliability evidence
```

---

# 37. Evidence Maturity Can Increase Over Time

A project may evolve through:

```text
TARGET
        ↓
local EXPERIMENTAL evidence
        ↓
cloud MEASURED evidence
        ↓
shadow/replay evidence
        ↓
production operational evidence
```

Each stage strengthens what can legitimately be claimed.

---

# 38. Portfolio Claims Should Reflect Current Maturity

Early project stage:

```text
Implemented deterministic reproduction suite for known Stage-1 failures.
```

Later:

```text
Measured routing accuracy across a held-out evaluation set.
```

Later:

```text
Validated retry safety through fault-injection experiments.
```

Each statement should correspond to actual completed work.

---

# 39. Do Not Turn Architecture Intent Into Implementation Claims

An architecture document may say:

```text
Reliora will use idempotency controls.
```

That does not mean:

```text
idempotency has already been implemented and tested.
```

Planning and implementation are different evidence states.

---

# 40. Planned vs Implemented vs Verified

A useful maturity ladder is:

```text
Planned
→ documented design exists

Implemented
→ code/configuration exists

Verified
→ tests or experiments demonstrate behaviour

Measured
→ quantitative data collected under defined conditions

Operationally proven
→ evidence exists from sustained real execution
```

These levels should not be collapsed.

---

# 41. Architecture Diagrams Are Not Runtime Evidence

A diagram showing:

```text
CloudWatch
```

does not prove:

```text
CloudWatch dashboards exist
```

Similarly, a diagram showing:

```text
idempotency layer
```

does not prove:

```text
duplicate retries have been tested successfully
```

Diagrams describe architecture.

Evidence demonstrates execution.

---

# 42. Tests Are Evidence, but Their Scope Matters

If:

```text
19 unit tests pass
```

we can say:

> The current unit suite contains 19 passing tests.

We cannot automatically say:

> The entire system has no defects.

Tests only cover the cases they exercise.

---

# 43. Linting Is Evidence of a Different Property

If Ruff reports:

```text
All checks passed!
```

we can say:

> Ruff found no configured lint violations in the checked paths.

We should not say:

> The Python application is correct.

Ruff is not a behavioural correctness system.

---

# 44. Type Checking Is Also Narrow Evidence

If mypy reports:

```text
Success: no issues found in 8 source files
```

that supports:

```text
typed interface consistency within the checked scope
```

It does not prove:

```text
runtime behaviour is correct
```

Again, evaluator purpose limits claim scope.

---

# 45. Tool Output Should Be Translated Carefully

A strong engineering habit is:

```text
Tool output
        ↓
What exactly does this tool measure?
        ↓
What claim does the result support?
```

Not:

```text
Tool passed
        ↓
everything is good
```

---

# 46. Example: Pytest

Output:

```text
19 passed
```

Supported claim:

> All 19 currently collected unit tests passed in this run.

Unsupported claim:

> Reliora has zero bugs.

---

# 47. Example: Ruff

Output:

```text
All checks passed!
```

Supported claim:

> The selected Python paths passed the configured Ruff lint checks.

Unsupported claim:

> The implementation is production-ready.

---

# 48. Example: Mypy

Output:

```text
Success: no issues found in 8 source files
```

Supported claim:

> Mypy found no type-checking issues in the eight checked source files.

Unsupported claim:

> No runtime type errors are possible.

---

# 49. Example: Reproduction Experiment

Output:

```text
Detected known defects: 3
Mismatches: 0
```

Supported claim:

> The deterministic evaluator detected the three executable known defects represented in the reviewed v1 reproduction dataset.

Unsupported claim:

> All behavioural and factual defects are now detectable.

---

# 50. Why This Matters for Hiring Managers

Experienced engineers often notice exaggerated claims quickly.

Statements such as:

```text
production-grade
enterprise-scale
100% accurate
zero hallucinations
millions of users
```

are weak if the repository contains no supporting evidence.

A smaller, defensible claim can be more impressive because it shows engineering judgment.

---

# 51. "Production-Oriented" vs "Production"

Reliora can legitimately be described as:

```text
production-oriented
```

or:

```text
production-engineering case study
```

while it is still being validated.

That communicates:

```text
architecture and controls are designed with production concerns in mind
```

without claiming:

```text
the system has already operated as a real production service
```

---

# 52. "Reference Application" vs "Production Application"

Similarly, the planned Reliora interface can be called:

```text
a product-grade reference application
```

until it has undergone the testing and operational validation required for a stronger claim.

Language should reflect evidence maturity.

---

# 53. Avoid Fabricated Business Impact

Suppose we have not deployed Reliora to a real support organization.

We should not claim:

```text
reduced support costs by 37%
```

or:

```text
improved customer satisfaction by 24%
```

unless those outcomes were actually measured.

---

# 54. Business Metrics Can Be Modelled as Assumptions

We can still model hypothetical impact.

For example:

```text
SYNTHETIC ASSUMPTION:
Average human handoff costs $X.
```

Then calculate:

```text
estimated cost under scenario Y
```

That is analysis, not observed ROI.

---

# 55. Avoid Fabricated Traffic

Likewise, do not claim:

```text
handles 1 million requests per day
```

because infrastructure could theoretically scale.

A better statement is:

> The architecture is designed to support horizontal scaling, and load capacity will be validated through explicit performance testing.

Architecture potential is not measured throughput.

---

# 56. Avoid Fabricated Incidents

Failure-mode documents may describe hypothetical incidents such as:

```text
tool timeout
duplicate retry
model provider degradation
DynamoDB throttling
```

These should not be described as:

```text
incidents that happened
```

unless they actually occurred.

They are:

```text
failure scenarios
```

until fault injection or operational evidence exists.

---

# 57. Fault Injection Changes the Evidence Class

Suppose we deliberately simulate:

```text
Lambda timeout
```

and observe recovery behaviour.

Then we can say:

```text
EXPERIMENTAL:
During fault injection, Reliora recovered according to X behaviour.
```

This is much stronger than simply saying:

```text
The architecture can recover from failures.
```

---

# 58. Evidence Can Upgrade Claims

Before test:

```text
TARGET:
duplicate retries should not create duplicate tickets
```

After deterministic integration test:

```text
EXPERIMENTAL:
0 duplicate tickets occurred across N controlled retry cases
```

After larger measured exercise:

```text
MEASURED:
0 duplicate side effects across N tested retries under configuration X
```

Evidence allows claims to mature.

---

# 59. Evidence Should Be Preserved Near the Code

Reliora keeps selected evidence under:

```text
evidence/
```

because the code and the claims should remain traceable.

This is stronger than placing unverifiable screenshots or numbers only in a résumé.

---

# 60. Screenshots Are Supporting Evidence, Not Always Primary Evidence

Screenshots can be useful for:

```text
AWS deployment confirmation
dashboard output
console state
UI demonstration
```

But whenever possible, machine-readable artifacts are stronger for technical verification.

For example:

```text
JSON report
test output
Git commit
hash
structured metrics
```

can often be reproduced more easily than an image.

---

# 61. Evidence Should Be Reproducible When Practical

A strong evidence artifact allows another engineer to answer:

```text
Which input produced this?

Which code generated it?

Which command can reproduce it?

What version was used?

What does the result mean?
```

This is why Reliora connects:

```text
dataset
runner
hash
Git
generated evidence
```

---

# 62. Evidence Should Be Reviewable

Machine-readable evidence alone can still be difficult to interpret.

Human-readable documentation should explain:

```text
method
scope
limitations
interpretation
```

This is where dataset cards and engineering notes complement generated output.

---

# 63. Claim Discipline Applies to README Files

The project README will eventually contain summary results.

Those results should be written carefully.

For example:

```text
Stage-1 reproduction:
3/3 executable known defects detected
1 historical case preserved as non-replayable documented evidence
```

is appropriate.

A badge saying:

```text
100% AI Reliability
```

would not be.

---

# 64. Claim Discipline Applies to LinkedIn

A portfolio post should also distinguish:

```text
what I built
what I measured
what I learned
```

For example:

> A perfect-looking LLM judge score hid exact behavioural violations, so I built a deterministic regression layer and reproduced three known failures.

This is stronger than:

> Built a flawless production AI support system.

---

# 65. Claim Discipline Applies to Résumés

A résumé bullet could eventually say:

> Designed a deterministic AI evaluation layer that detected 3/3 executable known failures in a frozen Stage-1 regression dataset and preserved dataset provenance through SHA-256-linked evidence.

That is specific and auditable.

---

# 66. Claim Discipline Applies to Interviews

If asked:

> How reliable is the system?

A strong answer is not automatically a percentage.

A stronger answer might be:

> Reliability is decomposed by requirement. The current deterministic reproduction suite detects all three replayable known Stage-1 defects, but broader routing, tool-safety, latency, and cloud reliability targets still require later evaluation phases.

This demonstrates maturity.

---

# 67. Confidence Should Match Evidence

A useful principle is:

```text
Weak evidence
→ cautious claim

Strong evidence
→ stronger claim
```

Confidence should increase because evidence improved, not because the wording became more enthusiastic.

---

# 68. Unknown Is Better Than False Precision

If the current cost is unknown:

```text
cost per interaction:
TBD
```

is better than:

```text
$0.0027
```

unless that value was genuinely calculated and documented.

False precision makes a project look less credible.

---

# 69. Estimated Values Need Labels

Sometimes estimation is necessary.

Then label it clearly:

```text
ESTIMATE
ASSUMPTION
SCENARIO
TARGET
```

and document the formula.

For example:

```text
Estimated monthly cost
=
synthetic requests/day
× measured or published unit cost
× days/month
```

This makes the calculation inspectable.

---

# 70. Published Vendor Data Is Not Your Measurement

Suppose AWS documentation states a service can support a particular throughput.

That is:

```text
vendor documentation
```

It is not:

```text
Reliora measured throughput
```

Both can be cited, but the source must remain clear.

---

# 71. External Benchmarks Are Not Local Results

Similarly, if a model card reports:

```text
benchmark accuracy = X
```

we should not say:

```text
Reliora achieved X
```

unless we independently measured the same metric in Reliora.

---

# 72. Evidence Taxonomy Helps Architecture Reviews

During architecture review, each statement can be challenged:

```text
Is this measured?

Is it only a target?

Is it based on an assumption?

Did we observe it historically?

Was it produced experimentally?
```

This helps separate decisions from facts.

---

# 73. Example Architecture Statement

Weak:

```text
DynamoDB will easily handle our production traffic.
```

Better:

```text
TARGET:
The data layer should tolerate the expected workload without throttling.

SYNTHETIC ASSUMPTION:
Initial load tests will use X requests per second.

MEASURED:
To be recorded during the scaling phase.
```

The second version is much more rigorous.

---

# 74. Evidence Taxonomy Helps Cost Analysis

Weak:

```text
Reliora is cheap to operate.
```

Better:

```text
TARGET:
Keep cost per resolved interaction within the defined budget.

MEASURED:
To be calculated from actual Bedrock, AgentCore, Lambda, and supporting service usage during cloud experiments.
```

Again, the evidence state is explicit.

---

# 75. Evidence Taxonomy Helps Security Claims

Weak:

```text
Reliora is secure.
```

Better:

```text
Implemented:
specific security controls.

EXPERIMENTAL:
security test suite result.

TARGET:
zero sensitive values in application logs.

MEASURED:
results from defined scanning/inspection procedure.
```

Security should also be decomposed into testable claims.

---

# 76. Evidence Taxonomy Helps Reliability Claims

Weak:

```text
The agent is reliable.
```

Better:

```text
Behavioural contract:
specific pass/fail evaluations

Side-effect safety:
specific retry/idempotency tests

Routing:
held-out classification metrics

Generative quality:
judge/human evaluation

Operations:
latency/error/cost measurements
```

Reliability becomes an evidence portfolio rather than one vague label.

---

# 77. Evidence Classes Can Coexist in One Document

A document might contain:

```text
OBSERVED:
Stage-1 route leakage occurred.

EXPERIMENTAL:
Reliora's current leakage evaluator detects the frozen response.

TARGET:
zero route-label leakage across the future adversarial suite.

MEASURED:
to be reported once the expanded evaluation suite is run.
```

This is useful because it shows progression.

---

# 78. Do Not Retroactively Relabel Evidence

If something was originally a synthetic assumption, later real measurement does not make the old assumption retroactively measured.

Instead preserve:

```text
Assumption:
X

Later measurement:
Y
```

Then compare them.

This allows us to learn whether our architecture assumptions were reasonable.

---

# 79. Assumption Tracking Improves Engineering

Suppose we assume:

```text
support interactions average 1,500 tokens
```

Later we measure:

```text
2,900 tokens
```

That difference could materially affect cost.

If the original assumption was documented, the architecture review can explain why estimates changed.

---

# 80. Evidence Is Part of Decision-Making

Evidence is not only for proving success.

It may tell us:

```text
an architecture choice was wrong
a target was unrealistic
an evaluator is too weak
a model is too expensive
a router underperforms
a fallback path is necessary
```

Negative results are valuable.

---

# 81. Do Not Delete Unfavourable Results

If an experiment fails:

```text
preserve the result
understand why
change the system if justified
rerun
compare evidence
```

Deleting the first result hides the engineering process.

---

# 82. Git History Adds Another Evidence Layer

Git commits can show:

```text
when evaluator code changed
when datasets changed
when evidence was regenerated
why the change occurred
```

This helps connect:

```text
engineering decision
```

to:

```text
historical implementation state
```

---

# 83. Hashes Add Artifact Identity

SHA-256 adds another level:

```text
This exact artifact
→ this fingerprint
```

This is especially useful for:

```text
datasets
generated reports
frozen source artifacts
```

when byte-level identity matters.

---

# 84. Hashes Do Not Prove Semantic Correctness

A hash can prove:

```text
file identity
```

It does not prove:

```text
the file's content is correct
```

For example:

```text
a badly annotated dataset
```

can still have a perfectly valid SHA-256 hash.

Provenance and validity are different concerns.

---

# 85. Evidence Quality Has Multiple Dimensions

Useful questions include:

```text
Is it authentic?
Is it traceable?
Is it reproducible?
Is it relevant?
Is the method valid?
Is the scope clear?
Is the denominator known?
Are limitations documented?
```

No single mechanism answers all of them.

---

# 86. A Practical Claim Checklist

Before writing a technical claim, ask:

```text
1. What exactly am I claiming?
2. Which artifact supports it?
3. What evidence class is it?
4. What was the environment?
5. What is the denominator?
6. What limitations apply?
7. Is this historical, experimental, measured, targeted, or assumed?
8. Am I generalizing beyond the experiment?
9. Could another engineer reproduce or inspect it?
10. Is the wording stronger than the evidence?
```

---

# 87. Example: Good Reliora Statement

```text
EXPERIMENTAL:
Using the versioned stage1-reproduction-v1 dataset,
Reliora detected all 3 of 3 executable known Stage-1 defects,
with 0 mismatches. One additional repository-preserved historical
defect remained documented but non-replayable because the complete
raw output was not retained.
```

This statement includes:

```text
evidence class
dataset
numerator
denominator
mismatch count
limitation
```

---

# 88. Example: Poor Reliora Statement

```text
Reliora achieved perfect AI reliability and eliminated hallucinations.
```

Problems include:

```text
no defined metric
no denominator
no dataset
no environment
no evidence class
no hallucination experiment
unsupported universal claim
```

---

# 89. Evidence Discipline Is Part of Responsible AI

Responsible AI includes not only how the model treats users.

It also includes how engineers communicate system capability.

Overstating reliability can cause people to trust automation beyond what has been validated.

Therefore accurate claims are themselves a safety practice.

---

# 90. Evidence Discipline Is Part of Architecture

Architecture decisions often depend on:

```text
expected traffic
latency
failure rate
security assumptions
cost
risk tolerance
```

If those inputs are mislabeled or invented, architecture decisions become less grounded.

Evidence discipline therefore affects system design itself.

---

# 91. Evidence Discipline Is Part of LLMOps

LLMOps involves comparing:

```text
models
prompts
routers
datasets
evaluators
releases
```

Those comparisons are only meaningful if experiment metadata and evidence classification remain consistent.

Otherwise teams may optimize against incomparable results.

---

# 92. Evidence Discipline Is Part of FinOps

Cost optimization requires distinguishing:

```text
AWS list price
estimated cost
synthetic cost model
measured cost
real workload cost
```

A FinOps claim without this distinction can be misleading.

---

# 93. Evidence Discipline Is Part of SRE

Reliability engineering similarly distinguishes:

```text
availability target
availability measurement
synthetic test result
incident observation
load-test evidence
```

Reliora should apply the same rigor to AI reliability.

---

# 94. Evidence Discipline Prevents Resume Inflation

A portfolio should demonstrate:

```text
what I actually designed
what I actually implemented
what I actually tested
what I actually measured
```

rather than what the architecture theoretically could do.

Precise claims create stronger interview material.

---

# 95. Evidence Discipline Makes Future Improvement Easier

If current limitations are recorded honestly, later progress becomes visible.

For example:

```text
Phase 3:
3/3 historical defects reproduced

Phase 5:
router macro F1 measured

Phase 8:
idempotency fault tests completed

Phase 10:
operational latency measured

Phase 12:
CI regression gate enforced
```

The project gains a real engineering story.

---

# 96. Important Lessons

1. Evidence types should not be treated as interchangeable.
2. `OBSERVED` preserves something that occurred historically.
3. `EXPERIMENTAL` describes a controlled project experiment.
4. `MEASURED` describes a value actually collected using a defined method.
5. `TARGET` describes a desired future condition, not a result.
6. `SYNTHETIC ASSUMPTION` describes deliberately invented modelling or testing input.
7. Percentages require visible denominators.
8. Small evaluation suites should not be generalized to universal reliability claims.
9. Historical evidence should not be reconstructed when the raw artifact is missing.
10. Synthetic cases are legitimate when labeled accurately.
11. Unknown or TBD is preferable to fabricated precision.
12. Planned architecture is not the same as implemented architecture.
13. Implemented architecture is not automatically verified architecture.
14. Tests, linting, type checks, evaluation runs, and telemetry support different kinds of claims.
15. Tool success should be translated into only the property the tool actually measured.
16. Local experimental evidence is useful without being misrepresented as production evidence.
17. Business impact, production traffic, security outcomes, and cost savings should not be invented.
18. Architecture diagrams describe systems but do not prove runtime behaviour.
19. SHA-256 proves artifact identity, not semantic correctness.
20. Strong technical claims should trace back to evidence artifacts, methods, scope, and limitations.
21. Negative results and mismatches are valuable evidence and should not be hidden.
22. Evidence maturity can progress from target to experiment to measurement to operational validation.
23. Precise claims are stronger portfolio and interview material than exaggerated claims.
24. Evidence discipline is part of responsible AI, LLMOps, SRE, FinOps, and trustworthy architecture.

---

## Interview Explanation

> I use an explicit evidence taxonomy in Reliora so architecture intentions, historical observations, experimental results, measured values, and synthetic assumptions are not mixed together. For example, the Stage-1 defects are historical `OBSERVED` evidence, while the deterministic replay result is `EXPERIMENTAL`: 3 of 3 executable known defects in the versioned reproduction dataset were detected with zero mismatches. I do not translate that into a claim of universal reliability. Targets such as routing F1 or zero duplicate side effects remain targets until they are evaluated, and future latency or cost figures will be labeled measured only when the environment and method are defined. This keeps technical, portfolio, and interview claims aligned with what the evidence actually demonstrates.