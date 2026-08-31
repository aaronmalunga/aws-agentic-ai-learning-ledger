# Regression Cases and Reproduction Statuses

## Why This Lesson Exists

Reliora's first evaluation work was not designed around randomly invented test prompts.

It began with failures already preserved in the frozen Stage-1 customer-support project.

Examples included:

```text
REPRO-001
REPRO-002
REPRO-003
REPRO-004
```

Three of those historical problems could be replayed using preserved input and response evidence.

One could only be documented because the complete raw output required for an exact replay had not been retained.

Reliora converted these known failures into a versioned reproduction dataset and then evaluated them using deterministic rules.

This introduced several result concepts that can initially be confusing:

```text
evaluator passed = false
```

can coexist with:

```text
reproduction status = DETECTED
```

because the response failed the rule exactly as expected.

This document explains:

- what a regression case is
- why known defects should become executable tests
- reproduction case IDs vs evaluation IDs
- expected result vs actual result
- `DETECTED`
- `CONFIRMED`
- `MISMATCH`
- `DOCUMENTED`
- why deliberate mismatch testing matters
- why historical failures should not simply remain in project notes

---

# 1. What Is a Regression?

A regression occurs when previously correct behaviour becomes incorrect after a system change.

For example:

```text
Version A:
asks for one missing field

Version B:
asks for three missing fields
```

If the system was required to ask for one field, Version B introduced a behavioural regression.

---

# 2. Regression Testing

Regression testing asks:

> Did a new change accidentally reintroduce behaviour that we already know is unacceptable?

A simple mental model is:

```text
known failure
      ↓
create permanent test
      ↓
future implementation changes
      ↓
run test again
      ↓
prevent recurrence
```

---

# 3. Why Known Failures Are Valuable

A discovered defect contains useful information.

It tells us:

```text
The system can fail in this specific way.
```

If that knowledge remains only in:

```text
a chat message
a reviewer comment
a screenshot
an issue
a developer's memory
```

then the same failure may return later.

A stronger engineering response is:

```text
known defect
      ↓
formalize requirement
      ↓
create evaluator
      ↓
create regression case
      ↓
automate verification
```

---

# 4. Stage 1 Gave Reliora Real Regression Material

Stage 1 produced several concrete examples.

These were more useful than inventing a reliability story from scratch.

They allowed Reliora to ask:

> Can the Stage-2 evaluation system detect actual weaknesses that existed in the previous implementation?

This created a grounded starting point.

---

# 5. The Current Reproduction Cases

The current dataset contains four cases:

```text
REPRO-001
REPRO-002
REPRO-003
REPRO-004
```

Each represents a particular historical issue.

---

# 6. `REPRO-001`

## Historical Failure

The Stage-1 bug workflow was required to ask for:

```text
only ONE missing field at a time
```

but the response requested:

```text
description
steps to reproduce
environment
```

in the same turn.

---

## Evaluation Rule

```text
INV-004
```

---

## Expected Evaluator Result

Because the historical response is known to violate the contract:

```text
expected_passed = false
```

---

# 7. `REPRO-002`

## Historical Failure

The response exposed:

```text
HUMAN HANDOFF
```

to the customer.

The Stage-1 system contract prohibited exposing internal route names.

---

## Evaluation Rule

```text
INV-012
```

---

## Expected Evaluator Result

```text
expected_passed = false
```

because the historical response is intentionally a known bad example.

---

# 8. `REPRO-003`

## Historical Failure

The Stage-1 repository recorded that:

```text
<thinking>...</thinking>
```

content had been visible in streamed development output.

---

## Evaluation Rule

The relevant rule family is:

```text
INV-012
```

because the issue concerns prohibited internal-information leakage.

---

## Important Limitation

The complete original leaked reasoning response was not preserved.

Therefore the exact historical output cannot be replayed honestly.

The case is retained as:

```text
DOCUMENTED
```

rather than fabricated.

---

# 9. `REPRO-004`

## Historical Failure

The return-policy response included:

```text
30-day return window
unused condition
original packaging
```

but omitted:

```text
defective-item exception
```

---

## Evaluation Rule

```text
FACT-001
```

---

## Expected Evaluator Result

```text
expected_passed = false
```

because the response is incomplete according to the reviewed policy requirements.

---

# 10. Reproduction Case ID vs Evaluation ID

These identifiers serve different purposes.

Example:

```text
REPRO-004
```

and:

```text
FACT-001
```

are not interchangeable.

---

## Reproduction ID

Answers:

> Which concrete example are we evaluating?

Example:

```text
REPRO-004
```

---

## Evaluation ID

Answers:

> Which rule determines whether that example is acceptable?

Example:

```text
FACT-001
```

---

# Mental Model

```text
REPRO-004
specific historical case
        ↓
FACT-001
general evaluation rule
```

One evaluation rule can eventually apply to many cases.

---

# 11. Specific Case vs General Rule

This distinction prevents the evaluator from becoming hardcoded to one historical response.

A weak evaluator might say:

```python
if case_id == "REPRO-004":
    return False
```

That does not test factual completeness.

It merely memorizes the expected answer.

A stronger architecture is:

```text
REPRO-004
      ↓
required facts
      ↓
FACT-001 evaluator
      ↓
independent result
```

The evaluator should work even if it has never heard of:

```text
REPRO-004
```

---

# 12. Expected Result

Each executable reproduction case contains a reviewed expectation.

For known defective cases, this is often:

```text
expected_passed = false
```

This means:

> Based on the documented requirement, the evaluator should reject this response.

---

# 13. Actual Result

The evaluator independently processes the case and returns something like:

```text
passed = false
```

or:

```text
passed = true
```

The runner should not determine the result merely from the expectation.

---

# 14. Why Expected and Actual Must Be Separate

If the runner simply did:

```text
expected = false
therefore actual = false
```

the evaluation would prove nothing.

The correct architecture is:

```text
Reviewed expectation
        │
        │
        ├────────────┐
        │            │
        ↓            ↓
stored in dataset   evaluator independently runs
                     │
                     ↓
                 actual result
                     │
                     ↓
              runner compares both
```

The comparison happens only after evaluation.

---

# 15. The Four Reproduction Statuses

Reliora currently uses statuses including:

```text
DETECTED
CONFIRMED
MISMATCH
DOCUMENTED
```

Each communicates something different.

---

# 16. `DETECTED`

`DETECTED` is used when:

```text
the historical case is known to be defective
```

and:

```text
the evaluator correctly rejects it
```

Conceptually:

```text
expected_passed = false
actual passed   = false
        ↓
DETECTED
```

---

# 17. Why `passed = false` Can Be Good Here

This is one of the most important distinctions.

The evaluator asks:

> Did the response satisfy the rule?

For a known bad response:

```text
No
```

so:

```text
passed = false
```

is correct.

The reproduction experiment asks:

> Did the evaluator recognize the known defect?

Because the evaluator rejected it:

```text
Yes
```

so:

```text
status = DETECTED
```

is correct.

---

# Two Different Levels

```text
Response evaluation:
FAIL

Evaluator reproduction:
SUCCESSFULLY DETECTED THE FAILURE
```

These are not contradictory.

---

# 18. Example: REPRO-001

Historical response:

```text
asks for three missing fields
```

Expected evaluator result:

```text
false
```

Actual INV-004 result:

```text
false
```

Therefore:

```text
REPRO-001 → DETECTED
```

---

# 19. Example: REPRO-004

Historical response:

```text
omits defective-item exception
```

Expected FACT-001 result:

```text
false
```

Actual result:

```text
false
```

Therefore:

```text
REPRO-004 → DETECTED
```

The evidence can additionally show:

```text
matched:
return_window
unused_condition
original_packaging

missing:
defective_item_exception
```

---

# 20. `CONFIRMED`

`CONFIRMED` applies when a case is expected to satisfy the evaluation rule and the evaluator agrees.

Conceptually:

```text
expected_passed = true
actual passed   = true
        ↓
CONFIRMED
```

---

# Why Positive Cases Matter

A reliable evaluator must not simply reject everything.

Suppose an evaluator were implemented as:

```python
return False
```

for every input.

It would appear to detect every known bad reproduction case.

But it would also reject all legitimate behaviour.

Therefore evaluation suites also need cases that should pass.

---

# 21. Positive Example

Suppose a bug workflow has:

```text
missing:
environment

requested:
environment
```

The rule:

```text
INV-004
```

should pass.

If:

```text
expected_passed = true
actual passed   = true
```

then:

```text
status = CONFIRMED
```

---

# 22. `MISMATCH`

`MISMATCH` means:

> The actual evaluator result disagreed with the reviewed expected result.

Conceptually:

```text
expected_passed != actual passed
        ↓
MISMATCH
```

---

# 23. Example Mismatch

Suppose:

```text
expected_passed = false
```

because the case is known to violate a requirement.

But the evaluator returns:

```text
passed = true
```

Then:

```text
MISMATCH
```

The evaluator may have missed the defect.

---

# 24. Reverse Mismatch

The opposite can also occur:

```text
expected_passed = true
actual passed   = false
```

This is also:

```text
MISMATCH
```

The evaluator may be incorrectly rejecting legitimate behaviour.

---

# 25. Why Mismatches Must Not Be Hidden

A mismatch is useful diagnostic evidence.

Possible causes include:

```text
evaluator bug
incorrect annotation
ambiguous requirement
bad dataset
unexpected application behaviour
overly narrow rule
overly permissive rule
```

The correct response is investigation.

---

# 26. Wrong Response to a Mismatch

A dangerous response would be:

```text
Evaluator disagrees with expected result
        ↓
change expectation until result turns green
```

without reviewing the requirement.

That would convert evaluation into score manipulation.

---

# 27. Correct Mismatch Investigation

A stronger process is:

```text
MISMATCH
    ↓
review requirement
    ↓
review dataset case
    ↓
review annotation
    ↓
review evaluator logic
    ↓
identify which component is wrong
    ↓
make justified correction
    ↓
rerun
```

---

# 28. Why Reliora Includes a Deliberate Mismatch Test

The reproduction logic itself is unit tested with a scenario designed to disagree with the expected outcome.

Why intentionally create a failing comparison?

To prove the runner can actually recognize disagreement.

---

# 29. Avoiding a Self-Congratulating Runner

Without a mismatch test, a poorly implemented runner might accidentally do something equivalent to:

```python
status = "DETECTED"
```

for every known bad case.

The report would always look successful.

That would be meaningless.

---

# 30. What the Mismatch Test Demonstrates

It demonstrates:

```text
expected result
```

and:

```text
actual result
```

are genuinely compared.

Therefore the runner can produce:

```text
MISMATCH
```

when the evaluator does something unexpected.

---

# 31. `DOCUMENTED`

`DOCUMENTED` means:

> The historical failure is preserved as valid evidence, but the current dataset does not contain enough authentic raw material for an exact executable replay.

---

# 32. Why `DOCUMENTED` Is Not a Failure of the Project

A temptation is to force every case into:

```text
PASS
FAIL
```

because quantitative results look cleaner.

But if raw evidence is missing, pretending otherwise weakens credibility.

`DOCUMENTED` explicitly preserves the limitation.

---

# 33. REPRO-003 Example

Repository evidence supports:

```text
thinking content was visible
```

but the complete raw reasoning output was not retained.

Therefore:

```text
we know the failure occurred
```

but:

```text
we cannot replay the exact historic response
```

So:

```text
REPRO-003 → DOCUMENTED
```

---

# 34. Why We Did Not Invent the Missing Response

We could create:

```text
<thinking>
synthetic reasoning
</thinking>
```

and verify that:

```text
INV-012
```

detects it.

That could be a useful synthetic test.

But it would not be:

```text
REPRO-003 historical replay
```

unless clearly labeled synthetic.

---

# 35. Historical Evidence vs Synthetic Test

These should be separated:

```text
Historical:
REPRO-003
→ DOCUMENTED

Synthetic leakage test:
new case
→ executable
```

This preserves provenance.

---

# 36. Status Truth Table

A simplified mapping is:

| Expected | Actual | Status |
|---|---|---|
| `false` | `false` | `DETECTED` |
| `true` | `true` | `CONFIRMED` |
| `false` | `true` | `MISMATCH` |
| `true` | `false` | `MISMATCH` |
| not executable | not run | `DOCUMENTED` |

This table is central to understanding reproduction reports.

---

# 37. Why `DETECTED` and `CONFIRMED` Are Different

Both mean the actual evaluator result matched the reviewed expectation.

But they communicate different semantic situations.

```text
DETECTED
→ known bad behaviour correctly rejected

CONFIRMED
→ acceptable behaviour correctly accepted
```

This makes reports easier to interpret.

---

# 38. Why Not Just Call Both `PASS`?

If both were labeled:

```text
PASS
```

we would lose important information.

For a bad reproduction case:

```text
PASS
```

could be misread as:

> The bad AI response passed the evaluator.

Using:

```text
DETECTED
```

makes the experiment meaning clearer.

---

# 39. Evaluator Result vs Case Status

These should remain separate fields.

For example:

```text
Evaluator:
passed = false

Reproduction case:
status = DETECTED
```

The first describes:

```text
the candidate response
```

The second describes:

```text
whether the reproduction behaved as expected
```

---

# 40. Three Levels of Meaning

A useful hierarchy is:

```text
Level 1:
Historical case
"Was this known as good or bad?"

Level 2:
Evaluator result
"Does the candidate satisfy the rule?"

Level 3:
Experiment comparison
"Did evaluator output match reviewed expectation?"
```

Confusing these levels leads to misleading reports.

---

# 41. Example Complete Flow

For REPRO-002:

```text
Historical evidence:
customer response exposes HUMAN HANDOFF
        ↓
Reviewed classification:
known defect
        ↓
Expected INV-012 result:
false
        ↓
Run evaluator
        ↓
Actual result:
false
        ↓
Compare
        ↓
DETECTED
```

---

# 42. Why Reproduction Comes Before Major Architecture Changes

Reliora deliberately created the known-defect reproduction layer before deploying new AWS architecture.

The reasoning was:

```text
We already know Stage 1 had weaknesses.
        ↓
Can the new evaluation layer see them?
        ↓
If yes, preserve these as regression controls.
        ↓
Now change the system.
        ↓
Use the controls to prevent recurrence.
```

This makes later improvements more measurable.

---

# 43. Evaluation-First Development

This approach can be called:

```text
evaluation-first development
```

in the context of the project.

It resembles a reliability-oriented version of test-driven thinking.

---

# 44. Test-Driven Analogy

Traditional software development might use:

```text
write failing test
      ↓
implement behaviour
      ↓
test passes
```

Reliora's initial reliability work used:

```text
known historical failure
      ↓
encode deterministic evaluator
      ↓
prove evaluator detects failure
      ↓
use as future regression gate
```

The exact workflow is different, but both start from an executable definition of desired behaviour.

---

# 45. Known Failure Becomes Institutional Memory

A useful principle is:

> A defect should not have to be rediscovered twice.

Once a failure has been understood, the system should preserve that learning.

Possible forms include:

```text
unit test
integration test
evaluation case
monitoring alert
architecture decision
runbook entry
```

Reliora uses reproduction cases for AI behavioural failures.

---

# 46. Regression Tests Convert Review Feedback Into Engineering Controls

Suppose a reviewer says:

```text
The model should not expose internal route names.
```

A weak response is:

```text
update prompt
```

A stronger response is:

```text
update prompt
+
create invariant
+
create test
+
create regression case
+
run in CI later
```

The feedback becomes executable.

---

# 47. Why Prompt Fixes Alone Are Weak

Prompts are probabilistic control mechanisms.

Adding:

```text
Never expose HUMAN HANDOFF.
```

can help.

But the system still needs a way to detect if the model later violates the instruction.

Therefore:

```text
instruction
+
verification
```

is stronger than instruction alone.

---

# 48. Regression Suites Protect Refactors

Reliora will later change:

```text
routing
structured outputs
tool validation
knowledge handling
model configuration
possibly prompts
```

Each change could accidentally reintroduce Stage-1 weaknesses.

A regression suite provides continuity across architectural change.

---

# 49. Regression Suites Protect Model Changes

Suppose the model changes from:

```text
Model A
```

to:

```text
Model B
```

The new model might improve general response quality but regress on:

```text
INV-004
```

or:

```text
INV-012
```

A known-defect suite helps detect that trade-off.

---

# 50. Regression Suites Protect Prompt Changes

Prompt optimization can also create unintended behaviour.

For example:

```text
prompt becomes shorter
        ↓
model becomes more concise
        ↓
but forgets one-field-at-a-time rule
```

A regression test can catch this even if overall judge score improves.

---

# 51. Regression Suites Protect Routing Changes

A new classical or hybrid router may improve aggregate F1 but accidentally alter handoff behaviour.

Known behavioural cases can remain independent release gates.

Aggregate improvements should not erase critical regressions.

---

# 52. Hard Gates vs Aggregate Scores

Suppose a future release has:

```text
routing macro F1 = 0.98
```

but:

```text
INV-012 leakage case fails
```

The high aggregate score should not necessarily compensate for a safety violation.

Critical regression cases can be configured as:

```text
hard gates
```

---

# 53. What Is a Hard Gate?

A hard gate means:

```text
if this requirement fails
→ release cannot proceed
```

rather than:

```text
average it with other scores
```

This is appropriate for selected non-negotiable requirements.

---

# 54. Example Future CI Behaviour

Conceptually:

```text
Pull request
    ↓
unit tests
    ↓
Ruff
    ↓
mypy
    ↓
reproduction suite
    ↓
MISMATCH count > 0?
   / \
 yes  no
 ↓     ↓
fail  continue
```

This turns historical lessons into automated release safety.

---

# 55. Why the Runner Returns Non-Zero on Mismatch

Command-line programs commonly use:

```text
exit code 0
→ successful execution

non-zero
→ failure condition
```

The reproduction runner can use:

```text
MISMATCH
```

as a reason to return a non-zero exit code.

This allows CI to detect the problem without parsing prose.

---

# 56. A Reproduction Suite Is Not a Full Evaluation Suite

The current dataset contains:

```text
4 historical cases
```

of which:

```text
3 are executable
1 is documented only
```

This is a regression foundation.

It is not intended to represent:

```text
all possible customer requests
all safety failures
all factual failures
all adversarial attacks
```

---

# 57. Why Scope Language Matters

A correct statement is:

> The reproduction suite covers currently preserved known Stage-1 evaluation gaps.

An incorrect statement would be:

> The reproduction suite comprehensively validates the entire support agent.

The dataset has not earned that broader claim.

---

# 58. Regression Set vs General Evaluation Set

These serve different purposes.

## Regression Set

Focuses on:

```text
failures we already know about
```

Goal:

```text
do not reintroduce them
```

---

## General Evaluation Set

Focuses on broader system behaviour.

May include:

```text
typical cases
edge cases
positive examples
negative examples
adversarial cases
distribution coverage
```

Goal:

```text
estimate broader system quality
```

Reliora will need both over time.

---

# 59. Regression Tests Can Become Too Narrow

If a system only tests known historical defects, it can still fail in new ways.

Therefore regression testing should be combined with:

```text
broader evaluation
adversarial testing
fault injection
operational monitoring
human review
```

Known failures are a foundation, not the entire reliability strategy.

---

# 60. New Failures Should Feed Back Into the Suite

A mature loop is:

```text
new defect discovered
       ↓
preserve evidence
       ↓
understand requirement
       ↓
create regression case
       ↓
create or strengthen evaluator
       ↓
run against current system
       ↓
include in future release checks
```

This makes the system learn operationally over time.

---

# 61. Incident-to-Regression Loop

For future production-style operation:

```text
incident
    ↓
root-cause analysis
    ↓
minimal reproduction
    ↓
regression test
    ↓
fix
    ↓
release gate
```

This is a common reliability-engineering pattern.

Reliora applies the same idea to AI behaviour.

---

# 62. Why Root Cause Matters

A regression case should target the underlying failure mode.

For example:

```text
specific sentence:
"Please provide description, steps, and environment."
```

is only one manifestation.

The root behavioural problem is:

```text
more than one missing field requested
```

The evaluator should encode the second concept.

---

# 63. Example Generalization

Weak rule:

```text
fail if response contains:
"description, steps, environment"
```

Stronger rule:

```text
if required fields are missing:
exactly one missing field may be requested
```

The stronger rule generalizes to new wording.

---

# 64. Regression Tests Should Not Freeze Bad Architecture

Another risk is preserving implementation details rather than requirements.

For example, a test should not require:

```text
one exact sentence
```

if the real requirement only demands:

```text
one missing field requested
```

Otherwise legitimate refactors become difficult.

Regression tests should preserve behaviour, not accidental implementation details.

---

# 65. Contract-Based Regression Testing

A strong regression case is tied to:

```text
requirement
```

rather than:

```text
old code
```

This allows architecture to change while the behavioural contract remains stable.

---

# 66. Positive Controls Matter

A good reproduction system also needs examples proving the evaluator can accept valid behaviour.

These are sometimes called positive controls.

For example:

```text
one missing environment field
one environment request
→ CONFIRMED
```

This helps verify the evaluator is not simply biased toward failure.

---

# 67. Negative Controls Matter

Known-defect cases function as negative examples.

For example:

```text
HUMAN HANDOFF exposed
→ expected failure
→ DETECTED
```

Together:

```text
positive cases
+
negative cases
```

provide a more meaningful evaluator test.

---

# 68. Edge Cases Matter

Future tests might include:

```text
no missing fields
duplicate requested-field annotations
different capitalization
punctuation variants
empty output
Unicode text
long responses
multiple policy exceptions
```

These test boundaries around the evaluator contract.

---

# 69. Adversarial Cases Matter

For leakage, future cases could intentionally try variants such as:

```text
human handoff
Human Handoff
<HINKING-like markers>
spacing changes
encoded labels
prompt injection attempts
```

Adversarial cases test whether controls remain robust under deliberate pressure.

---

# 70. Case Families Improve Organization

As datasets grow, cases may be grouped by:

```text
workflow
leakage
factual completeness
tool safety
handoff
routing
security
```

This allows more useful reporting than one overall score.

---

# 71. Per-Family Metrics

A future report could show:

```text
INV-004:
24 / 25 pass

INV-012:
40 / 40 pass

FACT-001:
31 / 35 pass
```

This helps identify where reliability is weakest.

---

# 72. Critical Failures Should Remain Visible

Suppose:

```text
99 total cases
98 pass
1 INV-012 leakage failure
```

The overall result:

```text
98.99%
```

could hide a serious defect.

Per-rule reporting keeps critical failures visible.

---

# 73. Regression Cases Need Provenance

Each historical case should answer:

```text
Where did this come from?

Which version produced it?

What requirement did it violate?

Is the evidence exact?

Is it replayable?

Was the expectation reviewed?
```

This is why Reliora uses a dataset card.

---

# 74. Regression Cases Need Stable IDs

Identifiers such as:

```text
REPRO-001
```

make it easier to refer to the same case across:

```text
dataset
dataset card
generated report
Git commits
issues
documentation
interviews
```

Stable IDs improve traceability.

---

# 75. Why Not Use the Prompt Text as the Identifier?

Prompts can change.

For example:

```text
"I found a technical bug on the website."
```

could later be rephrased.

A stable ID:

```text
REPRO-001
```

remains constant while the case metadata evolves deliberately.

---

# 76. Why IDs Should Not Encode Too Much Meaning

An identifier should help locate a case.

The detailed meaning belongs in metadata and documentation.

If IDs become overly descriptive, renaming becomes necessary whenever interpretation evolves.

Simple stable identifiers are easier to maintain.

---

# 77. Versioned Dataset + Stable Case IDs

A useful combination is:

```text
Dataset:
stage1-reproduction-v1

Case:
REPRO-001
```

The dataset version describes the collection state.

The case ID describes the individual example.

---

# 78. Evidence Should Record Actual Findings

For a detected case, evidence should provide more than:

```text
DETECTED
```

where practical.

Examples include:

```text
requested fields
missing fields
matched prohibited marker
matched required facts
missing required facts
```

This makes debugging much faster.

---

# 79. Diagnostic Evidence vs Raw Reasoning

Diagnostic evidence does not require exposing private model chain-of-thought.

For example:

```text
matched prohibited label:
HUMAN HANDOFF
```

is enough to explain the leakage finding.

Reliability tooling should capture necessary observable evidence without depending on hidden reasoning traces.

---

# 80. Reproduction Statuses Support Human Review

A reviewer can quickly interpret:

```text
DETECTED
→ known defect caught

CONFIRMED
→ valid case accepted

MISMATCH
→ investigate

DOCUMENTED
→ historical evidence retained but not replayed
```

This is much clearer than raw booleans alone.

---

# 81. Reproduction Statuses Support Automation

Automation can also use these values.

For example:

```text
MISMATCH
→ CI failure

DOCUMENTED
→ informational, not executable

DETECTED
→ expected known-negative result

CONFIRMED
→ expected known-positive result
```

The same vocabulary works for humans and machines.

---

# 82. Important Lessons

1. A known defect should become a regression case when it can be represented reliably.
2. Regression tests prevent previously understood failures from silently returning.
3. `REPRO-*` identifies a concrete reproduction case.
4. `INV-*` and `FACT-*` identify general evaluation rules.
5. Expected results and actual evaluator results must be stored and computed separately.
6. A known bad response should normally produce `passed = false`.
7. If that failure was expected, the reproduction status can correctly be `DETECTED`.
8. `CONFIRMED` means an expected-valid case was correctly accepted.
9. `MISMATCH` means actual evaluator behaviour disagreed with the reviewed expectation.
10. Mismatches should trigger investigation rather than label manipulation.
11. A deliberate mismatch unit test proves the runner does not simply report success automatically.
12. `DOCUMENTED` preserves legitimate historical evidence that cannot be exactly replayed.
13. Missing historical evidence should not be reconstructed and mislabeled as authentic.
14. Synthetic cases can supplement historical evidence if labeled accurately.
15. Known-defect regression suites and broad evaluation suites solve different problems.
16. Regression tests should encode underlying requirements rather than memorize exact historical text.
17. Positive and negative cases are both needed to validate evaluator behaviour.
18. Critical regression cases can later become CI/CD hard gates.
19. New incidents should feed back into regression coverage.
20. Stable IDs, dataset versioning, provenance, and diagnostic evidence make regression suites easier to maintain and audit.

---

## Interview Explanation

> I converted known Stage-1 failures into versioned regression cases instead of leaving them as reviewer notes or anecdotal observations. Each `REPRO-*` case identifies a concrete historical example, while an `INV-*` or `FACT-*` ID identifies the general rule used to evaluate it. The runner independently computes the evaluator result and then compares it with the reviewed expectation, producing `DETECTED`, `CONFIRMED`, `MISMATCH`, or `DOCUMENTED`. This matters because a known bad response correctly produces `passed = false`, but the reproduction experiment can still report `DETECTED` because the evaluator successfully identified the defect. I also test the mismatch path deliberately so the evaluation runner cannot simply congratulate itself on every case.