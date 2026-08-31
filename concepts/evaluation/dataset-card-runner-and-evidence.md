# Evaluation Datasets, Dataset Cards, Runners, and Evidence

## Why This Lesson Exists

While building Reliora's Stage-1 reproduction system, we created four related artifact types:

```text
evals/datasets/
evals/dataset_cards/
evals/runners/
evidence/generated/
```

At first, these can look like four different ways of storing evaluation information.

They are not interchangeable.

Each answers a different question:

```text
Dataset
→ What should be evaluated?

Dataset card
→ Where did these cases come from, and what do they mean?

Runner
→ How do we execute the evaluation?

Evidence
→ What actually happened when we ran it?
```

Understanding this separation is important because AI evaluation is not only about writing a test script.

A trustworthy evaluation system needs to preserve:

```text
inputs
provenance
execution
results
limitations
```

This document explains how those pieces fit together in Reliora.

---

# 1. The Four-Artifact Mental Model

A useful end-to-end model is:

```text
Evaluation Dataset
        ↓
defines cases to evaluate

Dataset Card
        ↓
documents provenance, scope, and limitations

Evaluation Runner
        ↓
executes cases against evaluator logic

Generated Evidence
        ↓
records what happened
```

Another way to think about it is:

```text
WHAT
WHY / WHERE FROM
HOW
WHAT HAPPENED
```

mapped to:

```text
Dataset
Dataset Card
Runner
Evidence
```

---

# 2. Reliora's Current Stage-1 Example

The current artifacts are:

```text
evals/
├── datasets/
│   └── stage1-reproduction-v1.json
│
├── dataset_cards/
│   └── stage1-reproduction-v1.md
│
└── runners/
    └── run_stage1_reproduction.py

evidence/
└── generated/
    └── stage1-reproduction-v1-report.json
```

These four files participate in one experiment.

However, they do not contain the same type of information.

---

# 3. The Evaluation Dataset

## File

```text
evals/datasets/stage1-reproduction-v1.json
```

## Core Question

The dataset answers:

> Which cases should be evaluated?

---

# What Is an Evaluation Dataset?

An evaluation dataset is a structured collection of test or evaluation cases.

A case can contain information such as:

```text
case identifier
input
candidate response
expected evaluator result
evaluation rule
annotations
evidence source
execution status
```

The exact schema depends on the evaluation system.

---

# Example Conceptual Case

A simplified case might look like:

```json
{
  "case_id": "REPRO-002",
  "evaluation_id": "INV-012",
  "response": "HUMAN HANDOFF\n\nFor further assistance...",
  "expected_passed": false
}
```

This says:

```text
Which case?
REPRO-002

Which rule?
INV-012

What response should be inspected?
the preserved Stage-1 response

What should the evaluator conclude?
the response should fail the rule
```

---

# 4. Why the Dataset Is Data, Not Code

The dataset does not implement the evaluation rule.

It describes cases.

For example:

```text
REPRO-004
→ factual omission case
```

The actual logic for determining factual completeness lives elsewhere.

This separation is important.

---

# Dataset vs Evaluator

```text
Dataset
→ provides examples

Evaluator
→ provides decision logic
```

A dataset should not contain Python code deciding whether itself passed.

Likewise, evaluator code should not hardcode one specific historical case.

---

# 5. Why the Dataset Lives Under `evals/datasets/`

The file is:

```text
evaluation input data
```

It is not:

```text
reusable application source
```

so it does not belong under:

```text
src/
```

It is not:

```text
a generated result
```

so it does not belong under:

```text
evidence/
```

Its responsibility is:

```text
define the evaluation cases
```

Therefore:

```text
evals/datasets/
```

is the appropriate location.

---

# 6. Why the Dataset Is Versioned

The filename contains:

```text
v1
```

because evaluation datasets can evolve.

For example, a later dataset might include:

```text
more positive cases
more negative cases
adversarial cases
new failure modes
different annotations
```

Changing the dataset can change the result.

Therefore, dataset evolution should be traceable.

---

# 7. Why Dataset Versioning Matters

Suppose one evaluation run reports:

```text
3 / 3 known defects detected
```

and another later reports:

```text
17 / 20 cases passed
```

Those results cannot be compared meaningfully unless we know:

```text
which dataset version each run used
```

The dataset is part of the experimental configuration.

---

# 8. The Dataset Card

## File

```text
evals/dataset_cards/stage1-reproduction-v1.md
```

## Core Question

The dataset card answers:

> What is this dataset, where did it come from, how was it created, and what are its limitations?

---

# 9. Why the Dataset Alone Is Not Enough

A JSON file can tell us:

```text
here are four cases
```

but it may not explain:

```text
Why were these cases selected?

Are they synthetic?

Are they historical?

Which repository did they come from?

Which commit was used?

Who reviewed the expectations?

Was raw evidence preserved?

What should this dataset not be used to claim?
```

That information belongs in human-readable documentation.

---

# 10. Machine-Readable vs Human-Readable

A useful distinction is:

```text
Dataset
→ optimized for software

Dataset card
→ optimized for humans
```

The dataset needs predictable structure.

The dataset card needs context.

Both are necessary.

---

# 11. What a Dataset Card Can Contain

A strong dataset card may document:

```text
dataset name
version
purpose
source system
source repository
source commit
evidence type
annotation policy
case descriptions
known limitations
intended use
prohibited interpretations
```

This turns the dataset from:

```text
a JSON file
```

into:

```text
an auditable evaluation artifact
```

---

# 12. Stage-1 Dataset Provenance

Reliora's Stage-1 reproduction dataset was derived from the frozen Stage-1 project state.

The frozen commit was:

```text
2da594b1126c566a593f31891f25a6f4ea4afa53
```

The dataset card preserves that provenance.

This matters because the experiment is specifically intended to reproduce known Stage-1 evaluation gaps.

---

# 13. Why Exact Source Version Matters

Without the frozen commit, someone might later modify Stage 1 and ask:

> Which version did Reliora actually reproduce?

A specific Git commit provides a stronger answer than:

```text
the old repository
```

It identifies a particular historical state.

---

# 14. Evidence Classification in the Dataset Card

Reliora distinguishes evidence types such as:

```text
OBSERVED
EXPERIMENTAL
MEASURED
TARGET
SYNTHETIC ASSUMPTION
```

For the Stage-1 reproduction cases, the historical inputs are primarily:

```text
OBSERVED
```

because they were preserved from Stage-1 repository evidence.

The new reproduction run produces:

```text
EXPERIMENTAL
```

evidence.

---

# 15. Why the Distinction Matters

Consider:

```text
Stage-1 response existed historically
```

That is:

```text
OBSERVED
```

Now consider:

```text
Reliora evaluator rejected that response during a controlled replay
```

That is:

```text
EXPERIMENTAL
```

They describe different events.

Combining them into one label would blur provenance.

---

# 16. Dataset Annotation Policy

Evaluation cases often contain reviewed expectations.

For example:

```text
REPRO-001
expected_passed = false
```

That expectation should not appear magically.

The dataset card can explain why it is justified.

---

# Example

The Stage-1 system contract explicitly required:

```text
ask only ONE missing field at a time
```

The preserved response requested:

```text
description
steps to reproduce
environment
```

Therefore the reviewed expectation is:

```text
INV-004 should fail
```

This expectation is grounded in the documented contract.

---

# 17. Why Annotation Policy Matters

Without annotation discipline, a developer could change labels simply to make the evaluator look better.

For example:

```text
Evaluator unexpectedly passes case
        ↓
change expected result to PASS
        ↓
metric improves
```

That would be evaluation gaming.

A dataset card helps make expectation changes accountable.

---

# 18. The Evaluation Runner

## File

```text
evals/runners/run_stage1_reproduction.py
```

## Core Question

The runner answers:

> How should this evaluation experiment be executed?

---

# 19. What Is an Evaluation Runner?

A runner is executable code that coordinates an experiment.

It may:

```text
load the dataset
validate input structure
calculate provenance metadata
dispatch cases to evaluators
collect results
aggregate summaries
write an output report
set an exit code
```

The runner is orchestration.

---

# 20. Runner vs Evaluator

This distinction is essential.

The evaluator decides:

> Does this case satisfy a particular rule?

The runner decides:

> How do we execute a collection of cases and preserve the resulting experiment?

---

# Mental Model

```text
Runner
    ↓
loads case
    ↓
calls evaluator
    ↓
collects result
    ↓
writes report
```

The runner should not unnecessarily duplicate evaluator logic.

---

# 21. Reusable Logic Belongs Under `src/`

Reliora keeps reusable evaluator logic under:

```text
src/reliora/evaluation/
```

For example:

```text
bug_workflow.py
leakage.py
factual.py
reproduction.py
```

These modules contain logic that can potentially be reused by:

```text
unit tests
offline experiments
CI/CD
future evaluation pipelines
```

---

# 22. Why the Specific Runner Lives Under `evals/runners/`

The file:

```text
run_stage1_reproduction.py
```

is tied specifically to the Stage-1 reproduction experiment.

It knows about:

```text
stage1-reproduction-v1.json
```

and:

```text
stage1-reproduction-v1-report.json
```

Therefore it is not generic application logic.

It is an experiment entry point.

---

# 23. Runner Responsibilities

The current runner conceptually performs:

```text
Locate project root
        ↓
Load dataset
        ↓
Normalize text for canonical hashing
        ↓
Calculate dataset SHA-256
        ↓
Evaluate each case
        ↓
Aggregate status counts
        ↓
Write generated evidence
        ↓
Print summary
        ↓
Exit non-zero if mismatches exist
```

Each step has a distinct reason.

---

# 24. Why the Runner Calculates a Dataset Hash

The generated report should identify exactly which dataset produced it.

Therefore the runner calculates:

```text
SHA-256
```

for the canonical dataset representation.

Conceptually:

```text
dataset bytes
      ↓
canonical normalization
      ↓
SHA-256
      ↓
report metadata
```

This strengthens provenance.

---

# 25. Why the Hash Basis Is Documented

Reliora encountered a Windows CRLF vs Git LF mismatch.

Because byte-level hashing was involved, we explicitly defined the hash basis as:

```text
UTF-8 text normalized to LF line endings before SHA-256
```

The runner now follows that rule.

The report records it.

---

# 26. Why the Runner Produces an Exit Code

A command-line program can communicate success or failure to the operating system using an exit code.

Conceptually:

```text
0
→ success

non-zero
→ failure / abnormal condition
```

The reproduction runner can return a failure exit code if:

```text
MISMATCH
```

results exist.

---

# 27. Why Exit Codes Matter for CI/CD

Later, GitHub Actions can run:

```text
reproduction runner
```

and inspect the exit status.

For example:

```text
no mismatches
→ exit 0
→ pipeline continues

mismatch exists
→ exit 1
→ pipeline fails
```

This makes an evaluation executable as a release control.

---

# 28. The Generated Evidence Report

## File

```text
evidence/generated/stage1-reproduction-v1-report.json
```

## Core Question

The evidence report answers:

> What happened when the experiment ran?

---

# 29. Why Evidence Is Different From the Dataset

The dataset says:

```text
run these cases
```

The evidence says:

```text
these were the results
```

Input and output should not be confused.

---

# Example

Dataset:

```text
REPRO-004
expected evaluator result:
false
```

Evidence:

```text
actual evaluator result:
false

status:
DETECTED

matched facts:
return_window
unused_condition
original_packaging

missing:
defective_item_exception
```

The evidence records execution outcome.

---

# 30. Why Evidence Is Generated

The report should be created by the runner rather than manually typed.

Why?

Because manually writing results creates a risk that:

```text
reported outcome
!=
actual evaluator outcome
```

Generated evidence ties the report directly to execution.

---

# 31. Why Generated Evidence Can Still Be Committed

Many generated files should not be committed.

Examples include:

```text
temporary caches
compiled files
local logs
```

However, selected experimental evidence can have lasting engineering value.

The Stage-1 reproduction report proves:

```text
what the evaluator did
against which dataset
at a preserved point in development
```

Therefore committing the evidence is justified.

---

# 32. Generated Does Not Mean Unimportant

The word:

```text
generated
```

only means the artifact was produced by tooling.

It does not mean:

```text
temporary
```

or:

```text
unimportant
```

Its value depends on purpose.

---

# 33. What the Current Report Records

The Stage-1 reproduction report includes summary information such as:

```text
Total cases: 4
Executable cases: 3
Documented non-executable cases: 1
Detected known defects: 3
Mismatches: 0
All executable expectations met: True
```

It also records case-level findings.

---

# 34. Interpreting the Summary Correctly

The result:

```text
All executable expectations met: True
```

does not mean:

```text
Reliora is perfectly reliable
```

It means:

> For this specific dataset, the three replayable known defects produced the reviewed expected evaluator outcomes.

The scope matters.

---

# 35. The Non-Executable Case

One Stage-1 case:

```text
REPRO-003
```

concerns documented exposure of:

```text
<thinking>...</thinking>
```

content.

The historical repository preserves evidence that this occurred.

However, the complete raw leaked output was not preserved.

Therefore the case is:

```text
DOCUMENTED
```

rather than replayed.

---

# 36. Why the Dataset Can Contain a Non-Executable Case

An evaluation dataset does not have to imply:

```text
every historical observation can be replayed
```

Sometimes provenance is incomplete.

The correct engineering response is to preserve the limitation.

---

# 37. Why We Did Not Reconstruct Missing Raw Evidence

We could theoretically invent a fake example containing:

```text
<thinking>
some reasoning
</thinking>
```

and run the evaluator against that.

But then the experiment would no longer be reproducing the actual historical evidence.

It would be a synthetic case.

That may be useful separately, but it must not be mislabeled as historical replay.

---

# 38. Dataset Card Documents This Limitation

The dataset card explains that:

```text
REPRO-003
```

is repository-preserved observed evidence but is not executable as an exact replay.

This prevents future readers from assuming:

```text
all four cases were rerun identically
```

---

# 39. Why the Report Preserves `DOCUMENTED`

The runner does not silently drop the case.

It records it as:

```text
DOCUMENTED
```

This preserves:

```text
known failure mode
+
provenance limitation
```

without fabricating evidence.

---

# 40. The Relationship Between All Four Artifacts

The complete flow is:

```text
stage1-reproduction-v1.json
        ↓
defines cases

stage1-reproduction-v1.md
        ↓
explains source and limitations

run_stage1_reproduction.py
        ↓
executes the experiment

stage1-reproduction-v1-report.json
        ↓
records results
```

Each artifact adds a different layer of trust.

---

# 41. What Happens If the Dataset Changes?

Suppose we modify:

```text
stage1-reproduction-v1.json
```

The previous generated evidence may no longer correspond to the current input.

Therefore we may need to:

```text
rerun experiment
regenerate report
verify hash
review result
restage evidence
```

This is why dataset and evidence cannot be edited independently without care.

---

# 42. What Happens If the Evaluator Changes?

Suppose:

```text
factual.py
```

changes.

Even if the dataset remains identical, the result may change.

Therefore a meaningful experiment is influenced by:

```text
dataset
+
evaluator code
+
runner code
+
configuration
```

Future provenance can include the Git commit to capture the code state.

---

# 43. What Happens If Only the Dataset Card Changes?

A documentation-only clarification may not require rerunning the experiment if:

```text
dataset content is unchanged
runner is unchanged
evaluator is unchanged
```

However, we should still inspect whether the documentation change alters the meaning of:

```text
scope
annotation
provenance
```

If it changes the interpretation materially, evaluation governance may require more review.

---

# 44. Source of Truth by Responsibility

A useful rule is:

```text
Evaluation cases
→ dataset is authoritative

Dataset provenance and limitations
→ dataset card is authoritative

Execution procedure
→ runner is authoritative

Observed experiment output
→ generated evidence is authoritative
```

No one artifact should try to replace all the others.

---

# 45. Why Not Put Expected Results Only in the Dataset Card?

Expected results need to be available to the runner.

If they exist only in Markdown:

```text
runner would need to parse human prose
```

That is fragile.

Machine-readable expectations belong in the dataset.

The dataset card explains why those expectations are justified.

---

# 46. Why Not Put Provenance Only in JSON?

We could place large blocks of human explanation inside the dataset.

But that would make the machine-readable artifact harder to maintain.

A dataset should contain enough structured provenance to identify itself, while the dataset card can contain deeper narrative explanation.

---

# 47. Why Not Put Everything Inside the Runner?

A single Python script could theoretically contain:

```text
cases
expected labels
source explanations
evaluation logic
execution
reporting
```

But this would tightly couple everything.

Problems would include:

- difficult review
- harder dataset versioning
- harder reuse
- hidden provenance
- difficult comparison between experiments
- less transparent evidence

Separation improves maintainability.

---

# 48. Why Not Put the Generated Report Under `evals/`

Because:

```text
evals/
```

primarily defines evaluation assets and execution.

The generated result is evidence of a run.

That is a different responsibility.

Therefore:

```text
evidence/generated/
```

makes the output role explicit.

---

# 49. Dataset vs Unit Test

A unit test might contain:

```python
def test_stage1_return_policy_omission_fails():
    ...
```

An evaluation dataset might contain dozens or thousands of cases.

The distinction is:

```text
Unit test
→ verifies evaluator/application code behaviour

Evaluation dataset
→ represents system behaviour cases to measure
```

They can overlap conceptually but serve different engineering purposes.

---

# 50. Why We Need Both

For example:

```text
test_factual.py
```

checks that:

```text
evaluate_required_fact_completeness(...)
```

works as intended.

Then:

```text
stage1-reproduction-v1.json
```

uses that evaluator against historical AI output.

So:

```text
Unit tests
→ test the evaluator

Evaluation dataset
→ lets the evaluator test the AI behaviour
```

This creates two levels of quality control.

---

# 51. Evaluator Tests vs Evaluation Cases

A useful model is:

```text
tests/
      ↓
Is our evaluator implementation correct?

evals/
      ↓
Does the AI behaviour satisfy the requirement?
```

This distinction is important because an evaluation system can itself contain bugs.

---

# 52. Why the Runner Has Tests Too

Reliora also tests reproduction logic.

For example, it includes a deliberate case where:

```text
actual evaluator result
!=
expected result
```

The runner should produce:

```text
MISMATCH
```

This proves it does not simply label every case successful.

---

# 53. Evidence Should Be Reproducible

A generated evidence file is stronger when another engineer can:

```text
check out code
identify dataset
run documented command
reproduce report
compare result
```

This is one goal of the current design.

---

# 54. Reproducibility Is More Than Rerunning Code

A meaningful reproduction also needs:

```text
same dataset
known evaluator version
known environment
known configuration
known model where applicable
```

Otherwise:

```text
I reran something
```

does not necessarily mean:

```text
I reproduced the same experiment
```

---

# 55. Why Deterministic Evaluation Helps Here

The current Stage-1 reproduction uses deterministic evaluators.

This reduces one source of variability.

Given the same:

```text
dataset
code
configuration
```

the expected evaluator results should remain stable.

Later generative evaluations may require additional model-version and sampling metadata.

---

# 56. Generative Evaluation Needs More Provenance

If an LLM judge is involved, experiment metadata may also need:

```text
judge model
model version
temperature
evaluation prompt
region
provider
timestamp
```

because those factors can influence the result.

The four-artifact model still applies.

The metadata becomes richer.

---

# 57. Evaluation Reports Should Not Rewrite History

Suppose a new evaluator version performs better.

We should not silently overwrite historical reports and pretend the previous experiment never existed.

A stronger system preserves:

```text
old run
new run
what changed
why results differ
```

This makes improvement traceable.

---

# 58. Versioning Strategy

A future experiment might use names such as:

```text
stage1-reproduction-v1.json
stage1-reproduction-v2.json
```

and reports such as:

```text
stage1-reproduction-v1-report.json
stage1-reproduction-v2-report.json
```

if the dataset changes materially.

Exact naming conventions can evolve, but the important principle is:

> Material experimental changes should be traceable.

---

# 59. Dataset Drift

Evaluation datasets themselves can drift.

For example, developers may keep adding cases that resemble current implementation behaviour.

Over time the dataset can become:

```text
too easy
too narrow
overfit to known bugs
unrepresentative
```

Dataset governance should therefore include periodic review.

---

# 60. Test-Set Gaming

A dangerous pattern is:

```text
evaluation case fails
        ↓
modify system specifically for exact wording
        ↓
score improves
        ↓
system does not generalize
```

This is analogous to overfitting.

A robust evaluation process includes:

```text
held-out cases
diverse phrasing
edge cases
adversarial cases
```

where appropriate.

---

# 61. Dataset Cards Help Detect Gaming

Because the dataset card records:

```text
purpose
collection method
limitations
annotation policy
```

it becomes easier to identify when the dataset no longer supports the claims being made.

---

# 62. Evidence Should Have a Defined Denominator

The current report says:

```text
3 executable known defects detected
```

out of:

```text
3 executable known defects
```

This denominator is explicit.

As datasets grow, evidence should continue to report:

```text
how many cases
which categories
which version
which exclusions
```

Without those details, percentages can become misleading.

---

# 63. Evidence Should Preserve Failures Too

Generated reports should not contain only successful cases.

A reliable evaluation system preserves:

```text
passes
failures
mismatches
documented non-executable cases
```

Otherwise the evidence becomes selection-biased.

---

# 64. Mismatches Are Valuable

A:

```text
MISMATCH
```

means:

> The evaluator's actual result disagreed with the reviewed expectation.

This is not something to hide.

It signals the need to investigate:

```text
evaluator logic
dataset annotation
requirement interpretation
data quality
```

---

# 65. Generated Evidence Should Not Be Manually "Cleaned"

Suppose the report contains an unexpected failure.

Editing the JSON manually to remove it would destroy the integrity of the experiment.

The correct response is:

```text
investigate
fix source cause if justified
rerun
generate new evidence
```

Generated evidence should reflect execution.

---

# 66. Human Review Still Matters

Automation does not eliminate the need for review.

Humans still need to inspect:

```text
dataset quality
annotations
requirements
evaluator validity
unexpected failures
claims derived from evidence
```

The system reduces ambiguity but does not replace engineering judgement.

---

# 67. Evidence Is Not the Same as a Metric Dashboard

A dashboard may aggregate current operational metrics.

The evidence directory preserves selected experimental artifacts.

These are related but distinct.

```text
Evidence artifact
→ durable record of a controlled run

Dashboard
→ changing operational view
```

Both can be useful.

---

# 68. Evidence Is Not the Same as Logs

Logs record execution events and diagnostics.

Generated evaluation evidence records interpreted experiment results.

For example:

```text
Log:
case REPRO-004 started

Evidence:
REPRO-004 → DETECTED
missing fact = defective_item_exception
```

Logs may help reproduce or debug evidence, but they are not identical.

---

# 69. Evidence and Provenance Chain

A stronger long-term chain looks like:

```text
Requirement
      ↓
Evaluation rule
      ↓
Dataset
      ↓
Dataset card
      ↓
Runner
      ↓
Code version
      ↓
Generated evidence
      ↓
Claim
```

Every portfolio statement should ideally trace back through this chain when practical.

---

# 70. Example Claim

A defensible claim is:

> In the `stage1-reproduction-v1` experiment, Reliora's deterministic evaluation layer detected all three executable known defects preserved from the frozen Stage-1 baseline, while keeping one non-replayable observed defect classified separately as documented evidence.

Why is this defensible?

Because we have:

```text
frozen source
dataset
dataset card
runner
evaluator code
tests
generated report
```

supporting the statement.

---

# 71. Weak Claim vs Strong Claim

Weak:

```text
Reliora fixed AI reliability.
```

Strong:

```text
The initial deterministic reproduction suite detected 3 of 3 executable known Stage-1 defects in the reviewed v1 dataset.
```

The second claim has:

```text
scope
denominator
experiment context
```

This is better engineering communication.

---

# 72. How This Supports Future CI/CD

The same structure can later support an automated pipeline.

Conceptually:

```text
Pull request
      ↓
load evaluation dataset
      ↓
run evaluator suite
      ↓
produce report
      ↓
compare thresholds / hard gates
      ↓
pass or block release
```

The current local experiment is therefore building toward LLMOps release safety.

---

# 73. How This Supports Model Experiments

Suppose Reliora later compares:

```text
Model A
Model B
```

We can hold constant:

```text
dataset
evaluation contracts
runner configuration
```

and change only:

```text
model
```

Then the generated evidence becomes more meaningful.

---

# 74. How This Supports Routing Experiments

The same principle applies to comparing:

```text
rule router
TF-IDF Logistic Regression
TF-IDF Linear SVM
LLM router
hybrid router
```

A versioned held-out dataset and clear runner are required if the results are to be trustworthy.

---

# 75. How This Supports Interview Explanations

A strong explanation is:

> I separate evaluation definition from evaluation execution and evidence. Datasets define machine-readable cases, dataset cards preserve provenance and limitations, runners orchestrate execution, and generated evidence records the actual result. This prevents the test data, experiment code, and reported outcome from being mixed together and gives me a clearer audit trail.

This shows understanding of experimental engineering rather than simply writing tests.

---

# 76. Important Lessons

1. Evaluation datasets, dataset cards, runners, and evidence serve different responsibilities.
2. A dataset answers what should be evaluated.
3. A dataset card explains provenance, purpose, annotation, and limitations.
4. A runner defines how the experiment is executed.
5. Generated evidence records what actually happened.
6. Machine-readable cases and human-readable provenance should remain separate.
7. Reusable evaluator logic belongs under `src`, not inside a one-off runner.
8. A specific experiment runner belongs under `evals/runners`.
9. Generated evidence belongs under `evidence`, not under application source.
10. Dataset versioning matters because changing the cases can change the result.
11. Dataset expectations should be grounded in requirements rather than adjusted to improve scores.
12. Exact source commits strengthen historical provenance.
13. Non-replayable historical evidence should be documented rather than fabricated.
14. SHA-256 can link generated evidence to a canonical dataset representation.
15. Exit codes allow evaluation runners to become future CI/CD release controls.
16. Unit tests verify evaluator implementation, while evaluation datasets measure AI-system behaviour.
17. Generated evidence should preserve failures and mismatches, not only successful cases.
18. Reports should be regenerated after material input or evaluator changes rather than manually edited.
19. Strong claims require scope, denominator, dataset identity, and provenance.
20. Separating input, documentation, execution, and evidence makes AI evaluation easier to audit, reproduce, maintain, and explain.

---

## Interview Explanation

> In Reliora, I treat evaluation as an experiment with separate artifacts. The versioned dataset defines the cases, the dataset card documents where those cases came from and what their limitations are, the runner executes the experiment using reusable evaluator logic, and the generated evidence report records the actual outcomes together with dataset provenance. This separation prevents evaluation inputs, execution logic, and reported results from becoming conflated, and it gives the project a reproducible audit trail that can later support CI/CD release gates and model comparisons.