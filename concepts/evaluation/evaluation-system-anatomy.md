# Evaluation System Anatomy

## Purpose

This note explains the main components of a production-oriented AI evaluation system and how they work together.

The pattern emerged while building Reliora, but the concepts are reusable across AI agents, RAG systems, LLM applications, machine-learning systems, and other software where important behaviour must be evaluated repeatedly.

The core flow is:

```text
Requirement
    ↓
Evaluation Contract
    ↓
Evaluator
    ↓
Unit Tests
    ↓
Dataset
    ↓
Dataset Card
    ↓
Dispatcher
    ↓
Runner
    ↓
Generated Evidence
    ↓
Engineering / Release Decision
```

These components solve different problems.

They should not be collapsed into one large script simply because they participate in the same evaluation workflow.

---

# 1. Why an Evaluation System Exists

An AI application can appear successful while still violating important requirements.

For example, an agent may:

```text
sound helpful
answer fluently
receive a high LLM-as-a-judge score
```

while simultaneously:

```text
leaking internal labels
requesting incorrect information
omitting required policy details
calling a tool too early
creating duplicate side effects
fabricating a backend identifier
```

Evaluation therefore needs more structure than:

```text
prompt
→ model
→ score
```

Important behaviours should first be expressed as explicit engineering requirements.

Then the appropriate evaluation mechanism can be chosen.

---

# 2. Evaluation Contract

An evaluation contract defines:

> What behaviour must remain true?

Examples from Reliora include:

```text
INV-004
Missing information must be collected incrementally.

INV-012
Internal information must not appear in customer-visible output.

FACT-001
Every reviewed required fact must be represented.
```

The contract is the specification.

It does not necessarily contain executable code.

This distinction matters because a requirement may exist before the system component needed to test it exists.

For example:

```text
Requirement:
Retries must not create duplicate tickets.

Evaluator:
Cannot be fully implemented until persistence,
operation IDs, and retry behaviour exist.
```

Therefore:

```text
documented requirement
≠
implemented evaluator
≠
verified behaviour
```

These represent different maturity levels.

---

# 3. Evaluator

An evaluator is the reusable logic that checks one defined property.

It answers:

> Given this input or system state, does it satisfy this evaluation rule?

For example, a bug-workflow evaluator might receive:

```text
missing_fields:
- steps_to_reproduce
- environment

requested_fields:
- steps_to_reproduce
```

and determine:

```text
PASS
```

because exactly one currently missing field was requested.

An evaluator should normally have a narrow responsibility.

It should not need to know:

```text
which dataset file the input came from
where the final report will be written
whether GitHub Actions is running
how many other cases exist
```

Those responsibilities belong elsewhere.

---

# 4. Evaluation Finding

Different evaluators benefit from returning a common result structure.

For example:

```text
evaluation_id
passed
message
evidence
```

Conceptually:

```text
EvaluationFinding
├── evaluation_id
├── passed
├── message
└── evidence
```

This provides a consistent contract between:

```text
evaluators
dispatchers
runners
reports
```

even when the evaluators inspect very different behaviours.

---

# 5. Unit Tests

An evaluator can itself contain bugs.

Therefore the evaluation logic must also be tested.

Unit tests answer:

> Does the evaluator implementation behave according to its own contract?

For example:

```text
Evaluator:
INV-004

Test input:
two fields missing
two fields requested

Expected evaluator result:
FAIL
```

The unit test verifies that the evaluator actually produces that result.

This creates an important distinction:

```text
Unit test
→ verifies evaluator code

Evaluation dataset
→ supplies versioned behavioural cases
```

Both are useful.

They are not interchangeable.

---

# 6. Evaluation Dataset

An evaluation dataset is a versioned collection of cases that should be executed.

It answers:

> What cases are we evaluating?

A case may contain:

```text
case_id
evaluation_id
case_type
inputs
expected result
reviewed rationale
```

For example:

```text
case_id:
REG-INV004-003

evaluation_id:
INV-004

missing fields:
steps_to_reproduce
environment

requested fields:
steps_to_reproduce
environment

expected evaluator result:
FAIL
```

The dataset should normally be machine-readable so that evaluation can be automated.

JSON is one practical option.

---

# 7. Dataset vs Unit Test

A dataset and unit test may sometimes contain similar-looking inputs, but they serve different engineering roles.

```text
Unit test
→ protects implementation behaviour

Dataset
→ defines versioned evaluation cases
```

A dataset can later be reused for:

```text
manual experiments
CI
model comparisons
regression testing
release gates
evidence generation
```

without rewriting the underlying cases as Python tests.

---

# 8. Dataset Card

A dataset card is the human-readable documentation accompanying a dataset.

It answers:

> What does this dataset mean?

The dataset itself may say:

```json
{
  "case_id": "REG-INV004-003",
  "expected_evaluator_passed": false
}
```

The dataset card explains:

```text
why the case exists
where it came from
whether it is historical or designed
what the expected result means
what the dataset covers
what it does not cover
how results may be claimed
how the dataset should be versioned
```

The relationship is:

```text
Dataset
→ executable data

Dataset Card
→ interpretation and governance
```

The Markdown file is therefore not merely a written copy of the JSON.

---

# 9. Designed Regression Fixture

A designed regression fixture is a deliberately constructed case used to protect expected behaviour.

It is not necessarily something observed in production.

For example:

```text
HUMAN HANDOFF: Please contact support.
```

may be deliberately included because the leakage evaluator should reject that response.

The fixture does not imply:

```text
this exact response occurred in production
```

It means:

```text
if this behaviour appears,
the evaluator must detect it
```

---

# 10. Dispatcher

A dataset normally identifies which evaluation rule applies using an identifier.

For example:

```text
evaluation_id = INV-004
```

Something must translate that identifier into executable evaluator logic.

That component is the dispatcher.

Conceptually:

```text
dataset case
     ↓
evaluation_id
     ↓
dispatcher
     ↓
correct evaluator
```

Example:

```text
INV-004
→ incremental bug-collection evaluator

INV-012
→ leakage evaluator

FACT-001
→ required-fact evaluator
```

The dispatcher may also transform dataset input into evaluator-specific types.

For example:

```text
JSON string:
"environment"

        ↓

BugField.ENVIRONMENT
```

The dispatcher should normally reject unsupported evaluation IDs explicitly.

Silent failure would make evaluation evidence unreliable.

---

# 11. Runner

A runner orchestrates an entire evaluation experiment.

An evaluator asks:

> Does this one input satisfy this one rule?

A runner asks:

> What happened when this entire versioned dataset was executed?

A typical runner performs:

```text
load dataset
        ↓
validate / canonicalize input
        ↓
calculate dataset hash
        ↓
iterate through cases
        ↓
send each case to dispatcher/evaluator
        ↓
compare actual result with expected result
        ↓
calculate summary
        ↓
write evidence report
        ↓
return process exit code
```

The runner coordinates components.

It should not contain every piece of evaluator logic itself.

---

# 12. Why Runner and Evaluator Are Separate

Consider a leakage evaluator.

It may eventually be used by:

```text
unit tests
regression suite
security suite
manual investigation
CI
release gate
model comparison experiment
```

If the rule exists only inside one runner, it cannot be reused cleanly.

Instead:

```text
Evaluator
→ reusable rule

Runner
→ experiment orchestration
```

This is a separation-of-concerns principle.

---

# 13. Generated Evidence

The dataset defines expectations.

The generated report records what actually happened.

This distinction is fundamental.

```text
Dataset
→ expected result

Runner execution
→ actual result

Report
→ evidence of the execution
```

For example:

```text
expected:
false

actual:
false

expectation_met:
true
```

The report can then say that the evaluator behaved according to the reviewed expectation for that case.

---

# 14. Evidence Provenance

Generated evidence should be traceable to the exact dataset used during execution.

A useful chain is:

```text
dataset version
        ↓
canonical dataset bytes
        ↓
SHA-256
        ↓
runner execution
        ↓
generated report
        ↓
engineering claim
```

Reliora uses:

```text
UTF-8 text
        ↓
normalize line endings to LF
        ↓
SHA-256
```

for canonical text dataset hashing.

This prevents Windows CRLF conversion from changing the evidence identity of otherwise identical text.

---

# 15. Independent Hash Verification

A runner can calculate a dataset hash and write it into a report.

However, stronger verification independently recalculates the hash outside the runner.

Conceptually:

```text
Runner calculation
        ↓
report hash

Independent calculation
        ↓
calculated hash

compare
        ↓
match?
```

If:

```text
calculated hash
==
report hash
```

the report is independently tied to the intended canonical dataset.

This does not prove that every evaluator is correct.

It proves dataset identity.

---

# 16. Reproduction

Reproduction asks:

> Can we recreate or detect something known to have happened previously?

For example:

```text
historical Stage-1 response
        ↓
known behavioural defect
        ↓
current evaluator
        ↓
can the defect be detected?
```

Historical reproduction datasets should preserve provenance where possible.

Useful information can include:

```text
source repository
source commit
artifact hash
original prompt
original response
reviewed annotation
whether replay is executable
```

Missing historical evidence should not be fabricated merely to make a reproduction executable.

---

# 17. Regression

Regression testing asks:

> Did a later change break behaviour that was previously expected to work?

Suppose an evaluator currently detects:

```text
HUMAN HANDOFF
```

as prohibited customer-visible text.

A future refactor accidentally stops detecting it.

The regression case now produces:

```text
expected:
FAIL

actual:
PASS

result:
MISMATCH
```

That indicates a regression.

Regression therefore means:

> Previously established behaviour has gone backwards after a change.

---

# 18. Reproduction vs Regression

These concepts are related but different.

```text
REPRODUCTION
Past
→ something happened
→ can we recreate or detect it?

REGRESSION
Future protection
→ something works correctly now
→ did a later change break it?
```

An important engineering pattern is:

```text
historical defect
        ↓
understand defect
        ↓
fix / evaluator
        ↓
create regression case
        ↓
prevent silent recurrence
```

An important bug should ideally leave behind a test or regression control.

---

# 19. Expected Failure vs Evaluation Failure

A failing evaluator result does not automatically mean the evaluation experiment failed.

Suppose the fixture intentionally contains bad behaviour.

```text
fixture:
multiple missing fields requested

expected evaluator result:
FAIL

actual evaluator result:
FAIL
```

The evaluated behaviour failed the contract.

But the evaluator behaved correctly.

Therefore:

```text
evaluator_passed = false

expectation_met = true
```

This distinction prevents confusion between:

```text
the subject failed
```

and:

```text
the evaluation system failed
```

---

# 20. Regression Runner Status

A generic regression runner can use statuses such as:

```text
PASS
→ actual result matched expected result

MISMATCH
→ actual result differed from expected result
```

This differs from historical reproduction terminology such as:

```text
DETECTED
CONFIRMED
DOCUMENTED
MISMATCH
```

because reproduction needs additional historical semantics.

---

# 21. Process Exit Codes

An evaluation runner should communicate its result to the operating system.

For example:

```text
exit code 0
→ all reviewed expectations matched

exit code 1
→ one or more mismatches occurred
```

This becomes important when evaluation enters CI/CD.

A human no longer has to manually inspect every line before deciding whether a pipeline should continue.

---

# 22. CI and Release Gates

Once an evaluation runner produces meaningful exit codes, a CI system can execute it automatically.

Conceptually:

```text
developer change
      ↓
pull request
      ↓
unit tests
      ↓
lint
      ↓
type checking
      ↓
AI regression runner
      ↓
all expectations met?
   ┌───────┴───────┐
   │               │
  yes              no
   │               │
continue        block / investigate
```

This is how evaluation becomes an engineering control rather than a one-time notebook experiment.

---

# 23. Quality Gates Are Layered

No single check proves the entire system is correct.

For example:

```text
pytest
→ tested behavioural expectations

Ruff
→ configured lint/static code-quality rules

mypy
→ static type consistency

regression runner
→ versioned evaluator expectations

dataset hash
→ artifact identity

LLM-based evaluation
→ selected generative qualities

integration tests
→ component interaction

fault tests
→ behaviour under failures
```

Each provides evidence about a different property.

---

# 24. Hard vs Soft Evaluation

Deterministic evaluation is especially useful for requirements such as:

```text
exact schema
allowed enum
no duplicate action
no prohibited route label
required backend identifier
required workflow state
```

Generative evaluation may be more appropriate for qualities such as:

```text
clarity
tone
helpfulness
naturalness
semantic relevance
```

The architecture should not force every property into the same evaluator type.

---

# 25. Partial Evaluator Coverage

An evaluator may implement only part of a broader requirement.

For example:

```text
Contract:
Internal information must not leak.
```

A first deterministic implementation may detect only:

```text
HUMAN HANDOFF
<thinking> tags
```

A passing result therefore means:

> None of the currently implemented prohibited patterns were detected.

It does not mean:

> No possible internal information leakage occurred.

This is an example of claim discipline.

---

# 26. Lexical vs Semantic Evaluation

A lexical factual evaluator may determine whether required words or phrases are present.

For example:

```text
required fact:
return window

accepted pattern:
30 days
```

It can detect whether:

```text
"30 days"
```

appears.

However:

```text
"Returns are not allowed within 30 days."
```

also contains the same lexical pattern.

Therefore:

```text
lexical completeness
≠
semantic truth
```

The evaluator should be described according to what it actually proves.

---

# 27. Dataset Versioning

Once an evaluation dataset is used to generate evidence, material changes should normally produce a deliberate new version.

Changes that may require versioning include:

```text
case inputs
expected outcomes
coverage
interpretation
required assertions
```

This preserves the relationship:

```text
dataset version
→ dataset hash
→ evaluation report
→ engineering claim
```

Silently changing fixtures merely to make an evaluation pass destroys the reliability of the evidence.

---

# 28. Reliora Example

Reliora implemented the pattern using:

```text
Behavioural contract
→ evals/behavioural-evaluation-contract.md

Evaluators
→ src/reliora/evaluation/

Unit tests
→ tests/unit/evaluation/

Historical dataset
→ stage1-reproduction-v1.json

Regression dataset
→ deterministic-evaluator-regression-v1.json

Dataset cards
→ evals/dataset_cards/

Dispatchers
→ reproduction.py
→ regression.py

Runners
→ evals/runners/

Generated evidence
→ evidence/generated/
```

The first deterministic regression suite contained:

```text
15 reviewed fixtures

FACT-001    5
INV-004     5
INV-012     5
```

The experimental execution produced:

```text
15 matched expectations
0 mismatches
```

The correct claim is scoped to that dataset and execution.

It does not establish complete system reliability.

---

# 29. Mental Model

When encountering an evaluation repository, ask:

```text
What requirement are we testing?

Where is the evaluator?

How do we know the evaluator itself works?

Where are the cases stored?

How are those cases documented?

How does a case reach the correct evaluator?

What executes the full experiment?

Where are actual results preserved?

How is dataset identity verified?

What exactly does a passing result prove?

How will this detect future regressions?
```

If those questions can be answered clearly, the evaluation system is much easier to audit and maintain.

---

# 30. Interview Explanation

A concise explanation is:

> I separate evaluation requirements, evaluator logic, datasets, execution orchestration, and evidence. The evaluator checks one reusable rule, unit tests verify the evaluator itself, versioned datasets define reviewed cases, a dispatcher sends each case to the correct evaluator, and a runner executes the full dataset, compares actual results with expectations, hashes the dataset, and writes evidence. Regression suites then become CI release gates so later changes cannot silently reintroduce known behavioural failures.

---

# Core Principle

> Evaluation should produce traceable engineering evidence, not merely reassuring scores.