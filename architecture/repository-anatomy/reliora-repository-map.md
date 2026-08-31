# Reliora Repository Anatomy

## Why This Lesson Exists

When looking at a professional software repository, it can be difficult to understand why some files live under:

```text
src/
```

while others live under:

```text
tests/
evals/
evidence/
docs/
```

The folder structure is not arbitrary.

Each directory communicates something about the **responsibility of the files inside it**.

This document explains the current Reliora repository structure, what the important folders and files contain, why they exist, and why certain artifacts belong in one location rather than another.

The goal is to be able to look at a repository and reason about its architecture instead of seeing only a collection of folders.

---

# High-Level Mental Model

A simplified view of Reliora is:

```text
reliora-ai-support-platform/
│
├── src/
│   └── reusable application logic
│
├── tests/
│   └── automated verification of application logic
│
├── evals/
│   └── AI evaluation datasets and experiment runners
│
├── evidence/
│   └── generated proof of what experiments produced
│
├── docs/
│   └── human-readable engineering documentation
│
├── infra/
│   └── infrastructure-as-code, added later
│
├── pyproject.toml
├── .gitignore
├── .gitattributes
└── .editorconfig
```

A useful summary is:

```text
src/
→ What the software does

tests/
→ Whether the software logic behaves correctly

evals/
→ How AI behaviour is evaluated

evidence/
→ What actually happened when evaluations ran

docs/
→ Why the system is designed this way

infra/
→ How the cloud environment is defined
```

---

# 1. `src/` — Reusable Application Code

## What `src` Means

`src` is short for:

```text
source
```

This directory contains the Python code that forms part of the actual Reliora software package.

Current structure:

```text
src/
└── reliora/
    ├── __init__.py
    │
    └── evaluation/
        ├── __init__.py
        ├── models.py
        ├── bug_workflow.py
        ├── leakage.py
        ├── factual.py
        └── reproduction.py
```

---

## Why Application Code Lives Under `src/`

Reusable application logic should be separated from:

- tests
- documentation
- datasets
- generated output
- infrastructure definitions
- temporary scripts

This makes the code easier to:

- import
- test
- package
- reuse
- maintain
- review

If another part of Reliora needs an evaluator, it should be able to import that evaluator from the package rather than copy its logic.

---

## Why `src/reliora/` Exists

`reliora` is the Python package name.

A Python import can therefore look like:

```python
from reliora.evaluation.factual import evaluate_required_fact_completeness
```

This tells Python:

```text
reliora
└── evaluation
    └── factual
```

The folder structure becomes part of the logical Python package structure.

---

# `src/reliora/__init__.py`

## Purpose

This file identifies:

```text
reliora/
```

as the package root and currently contains basic package metadata.

For example, Reliora has used:

```python
__version__ = "0.1.0"
```

---

## Why It Lives Here

Package-level information belongs at the package root rather than inside a specific feature such as evaluation.

---

# `src/reliora/evaluation/`

## Purpose

This directory contains reusable deterministic evaluation logic.

These evaluators are part of Reliora's product and reliability architecture.

They are not tied to only one experiment.

---

# `models.py`

## Location

```text
src/reliora/evaluation/models.py
```

## Purpose

Defines shared typed data structures used by evaluators.

One important model is:

```text
EvaluationFinding
```

This represents the result produced by a deterministic evaluator.

Conceptually:

```text
EvaluationFinding
├── evaluation_id
├── passed
├── message
└── evidence
```

---

## Why a Shared Model Exists

Without a common structure, every evaluator could return a different result format.

For example:

```text
Evaluator A
→ boolean

Evaluator B
→ dictionary

Evaluator C
→ string
```

That would make aggregation and reporting difficult.

A shared result contract makes evaluator outputs predictable.

---

## Why `invariant_id` Became `evaluation_id`

Originally the result used:

```text
invariant_id
```

because the first deterministic checks were behavioural invariants such as:

```text
INV-004
INV-012
```

Later, Reliora added:

```text
FACT-001
```

which is a factual-completeness evaluation rather than a behavioural invariant.

The shared field therefore became:

```text
evaluation_id
```

This supports different evaluation families:

```text
INV-*   behavioural evaluation
FACT-*  factual evaluation
```

The general model belongs in `models.py` because several evaluator modules use it.

---

# `bug_workflow.py`

## Location

```text
src/reliora/evaluation/bug_workflow.py
```

## Purpose

Implements deterministic evaluation for the bug-report information-collection workflow.

The relevant behavioural rule is:

```text
INV-004
```

The rule checks whether the agent asks for only one missing required bug field at a time.

---

## Why This Exists

The frozen Stage-1 system prompt required:

> Ask for only one missing field at a time.

However, the actual Stage-1 response asked for:

```text
description
steps to reproduce
environment
```

in the same turn.

The generic LLM correctness evaluation did not surface this behavioural contract violation.

Reliora therefore encodes the requirement as deterministic software.

---

## Why It Lives Under `src/`

This is reusable evaluation logic.

It may later be used by:

- regression tests
- offline evaluation
- CI/CD release gates
- replay experiments
- production evaluation pipelines

Therefore it is product-domain logic rather than a one-off experiment script.

---

# `leakage.py`

## Location

```text
src/reliora/evaluation/leakage.py
```

## Purpose

Detects prohibited internal information in customer-visible responses.

The primary current behavioural rule is:

```text
INV-012
```

Examples of prohibited content include:

```text
HUMAN HANDOFF
<thinking>
</thinking>
```

---

## Why This Exists

Stage 1 contained a response that began with:

```text
HUMAN HANDOFF
```

even though the system prompt explicitly prohibited exposing internal route labels.

The frozen repository also documented visible streamed:

```text
<thinking>...</thinking>
```

content during development execution.

Reliora treats internal-information leakage as a deterministic safety/reliability property.

---

# `factual.py`

## Location

```text
src/reliora/evaluation/factual.py
```

## Purpose

Implements deterministic factual-completeness evaluation.

The current evaluation contract is:

```text
FACT-001
```

It checks whether all required facts are represented in a response.

---

## Why This Was Needed

The Stage-1 FAQ stated that most items could be returned:

```text
within 30 days
when unused
in original packaging
unless the item arrived defective
```

The Stage-1 model answered with:

```text
30 days
unused
original packaging
```

but omitted:

```text
defective-item exception
```

The generic evaluation still treated the answer as correct.

Reliora therefore introduced a deterministic factual completeness check.

---

## Behavioural vs Factual Evaluation

These are deliberately separate.

```text
INV-004
→ Did the agent follow the required interaction behaviour?

INV-012
→ Did the response expose prohibited internal information?

FACT-001
→ Did the response preserve all required facts?
```

This separation is important because different failure types require different evaluation methods.

---

# `reproduction.py`

## Location

```text
src/reliora/evaluation/reproduction.py
```

## Purpose

Contains reusable logic for executing a reproduction case against the appropriate evaluator.

Conceptually:

```text
Reproduction case
       ↓
read evaluation_id
       ↓
select evaluator
       ↓
run evaluator
       ↓
compare actual result with expected result
       ↓
normalized reproduction result
```

---

## Important Status Values

The reproduction logic can produce statuses such as:

```text
DETECTED
DOCUMENTED
MISMATCH
CONFIRMED
```

For the Stage-1 defect cases:

```text
DETECTED
```

means:

> The known bad behaviour was correctly rejected by the Reliora evaluator.

This is different from saying the bad response itself "passed."

---

## Why This Lives Under `src/`

The logic for evaluating and normalizing reproduction cases is reusable.

It could later support:

- additional datasets
- CI pipelines
- regression suites
- release gates

The logic is therefore part of the reusable evaluation system.

---

# 2. `tests/` — Automated Software Verification

Current relevant structure:

```text
tests/
└── unit/
    └── evaluation/
        ├── test_bug_workflow.py
        ├── test_leakage.py
        ├── test_factual.py
        └── test_reproduction.py
```

---

## Why Tests Are Separate From `src/`

The code under:

```text
src/
```

is the software being built.

The code under:

```text
tests/
```

verifies that software.

A useful relationship is:

```text
src/
→ implementation

tests/
→ verification
```

Keeping them separate helps distinguish runtime application code from development validation code.

---

# What Does `unit/` Mean?

A unit test checks a small component in isolation.

For example:

```text
test_factual.py
```

does not need to deploy AWS infrastructure or call a real model.

It directly tests the factual evaluation function.

That makes the tests:

- fast
- repeatable
- cheap
- easier to debug

---

# `test_bug_workflow.py`

Tests:

```text
src/reliora/evaluation/bug_workflow.py
```

Examples include:

- one missing field can be requested
- requesting multiple missing fields fails
- requesting a non-missing field fails
- no field should be requested when nothing is missing

---

# `test_leakage.py`

Tests:

```text
src/reliora/evaluation/leakage.py
```

Examples include:

- clean customer-facing output passes
- literal `HUMAN HANDOFF` is detected
- `<thinking>` markers are detected
- case-insensitive variants are detected

---

# `test_factual.py`

Tests:

```text
src/reliora/evaluation/factual.py
```

Examples include:

- a complete policy response passes
- the missing defective-item exception fails
- accepted alternative wording can pass
- matching is case-insensitive

---

# `test_reproduction.py`

Tests:

```text
src/reliora/evaluation/reproduction.py
```

Examples include:

- known Stage-1 bug-workflow defect is detected
- route leakage is detected
- non-replayable evidence is documented
- factual omission is detected
- an unexpected evaluator result becomes `MISMATCH`

---

# Why Tests Do Not Replace AI Evaluations

Unit tests verify deterministic software logic.

AI evaluation additionally deals with:

- prompts
- model outputs
- behavioural scenarios
- factual completeness
- generative quality
- probabilistic behaviour

Therefore:

```text
tests/
```

and:

```text
evals/
```

solve related but different problems.

---

# 3. `evals/` — AI Evaluation Assets

Current relevant structure:

```text
evals/
├── behavioural-evaluation-contract.md
│
├── datasets/
│   └── stage1-reproduction-v1.json
│
├── dataset_cards/
│   └── stage1-reproduction-v1.md
│
└── runners/
    └── run_stage1_reproduction.py
```

---

## What Does `evals` Mean?

`evals` is short for:

```text
evaluations
```

This area contains artifacts used to measure AI-system behaviour.

AI evaluations often require more than ordinary software tests.

They may need:

- curated scenarios
- prompts
- expected behaviour
- human-reviewed annotations
- model responses
- evaluator definitions
- experiment runners
- generated reports

---

# `behavioural-evaluation-contract.md`

## Purpose

Documents Reliora's behavioural invariants.

Examples include rules such as:

```text
one missing bug field at a time
no premature tool execution
no fabricated ticket ID
no internal route leakage
structured handoff requirements
```

---

## Why This Is Documentation Rather Than Python

The contract explains the intended rules in human-readable form.

Individual rules may later be implemented as executable evaluators.

The document defines:

```text
what must be true
```

while evaluator code implements:

```text
how we check whether it is true
```

---

# `evals/datasets/`

## Purpose

Contains versioned evaluation inputs.

Current example:

```text
stage1-reproduction-v1.json
```

---

# What Is an Evaluation Dataset?

An evaluation dataset contains cases that should be evaluated.

A case may include:

- case ID
- prompt
- model response
- expected evaluator result
- annotations
- source provenance
- evaluation ID

Conceptually:

```text
Evaluation dataset
→ what should be tested
```

---

# Why the Dataset Does Not Live Under `src/`

The JSON file is data.

It is not application logic.

Putting datasets under `src/` would mix:

```text
software implementation
```

with:

```text
experiment input data
```

Keeping them separate makes responsibilities clearer.

---

# Why the Dataset Is Versioned

The name:

```text
stage1-reproduction-v1.json
```

includes:

```text
v1
```

because evaluation datasets can evolve.

Changing an evaluation dataset can change measured results.

Therefore dataset changes should be deliberate and traceable.

---

# `evals/dataset_cards/`

## Purpose

Contains human-readable documentation for datasets.

Current example:

```text
stage1-reproduction-v1.md
```

---

# Dataset vs Dataset Card

The dataset is optimized for machines.

```text
stage1-reproduction-v1.json
```

The dataset card is optimized for humans.

```text
stage1-reproduction-v1.md
```

A useful distinction is:

```text
Dataset
→ What cases exist?

Dataset card
→ Where did they come from, what do they mean, and what are the limitations?
```

---

# What a Dataset Card Should Explain

A good dataset card can include:

- dataset purpose
- source repository
- source commit
- provenance
- evidence classification
- case descriptions
- annotation policy
- limitations
- intended use
- restrictions

---

# Why This Matters for AI Engineering

AI evaluation datasets can silently become unreliable if:

- cases are changed without documentation
- expected outputs are manipulated
- annotations are unclear
- historical evidence is reconstructed
- test cases are tuned simply to make metrics improve

Dataset cards make those assumptions visible.

---

# `evals/runners/`

## Purpose

Contains executable experiment entry points.

Current example:

```text
run_stage1_reproduction.py
```

---

# What the Stage-1 Runner Does

Conceptually:

```text
Load dataset
      ↓
calculate dataset provenance
      ↓
execute each case
      ↓
call reusable evaluators
      ↓
aggregate results
      ↓
write evidence report
      ↓
print summary
```

---

# Why the Runner Is Not Under `src/`

This distinction is important.

The reusable logic:

```text
evaluate_reproduction_case(...)
```

belongs under:

```text
src/reliora/evaluation/
```

because it can be reused.

But:

```text
run_stage1_reproduction.py
```

is tied specifically to:

```text
stage1-reproduction-v1.json
```

and:

```text
stage1-reproduction-v1-report.json
```

It is an experiment entry point.

Therefore:

```text
src/
→ reusable logic

evals/runners/
→ specific experiment execution
```

---

# 4. `evidence/` — Experimental Results

Current relevant structure:

```text
evidence/
└── generated/
    └── stage1-reproduction-v1-report.json
```

---

## Purpose

The `evidence/` directory records artifacts that demonstrate what actually happened.

This is different from an evaluation definition.

---

# Dataset vs Runner vs Evidence

This distinction is extremely important.

```text
Dataset
"What should we evaluate?"

Runner
"How do we run the evaluation?"

Evidence
"What happened when we ran it?"
```

For Reliora:

```text
evals/datasets/stage1-reproduction-v1.json
        ↓
input cases

evals/runners/run_stage1_reproduction.py
        ↓
experiment execution

evidence/generated/stage1-reproduction-v1-report.json
        ↓
recorded result
```

---

# Why Evidence Does Not Belong Under `src/`

Generated evidence is output.

It is not reusable product logic.

Mixing generated evidence with application code would make the repository harder to understand and audit.

---

# Why `generated/` Exists

The subfolder name:

```text
generated
```

communicates:

> These artifacts are created by running tooling rather than manually authored as source code.

That helps distinguish:

```text
human-authored definitions
```

from:

```text
machine-generated results
```

---

# Current Stage-1 Reproduction Result

The current reproduction evidence recorded:

```text
Total cases: 4
Executable cases: 3
Documented non-executable cases: 1
Detected known defects: 3
Mismatches: 0
All executable expectations met: True
```

This should not be interpreted as:

> Reliora is 100% correct.

It means:

> All three executable known Stage-1 defects in this specific reproduction dataset were detected by the current deterministic evaluation layer.

Scope matters when interpreting evidence.

---

# 5. `docs/` — Human Engineering Documentation

Reliora contains substantial human-readable documentation.

Important areas include:

```text
docs/
├── baseline/
├── business/
├── requirements/
├── scope/
├── governance/
├── security/
├── architecture/
└── operations/
```

---

# Why Documentation Is Part of the Repository

Professional software engineering is not only code.

Important knowledge includes:

- what the system is supposed to do
- what it must not do
- architecture decisions
- risk decisions
- security assumptions
- operating procedures
- known failure modes
- deployment constraints

If this information exists only in someone's memory, the system becomes harder to maintain and review.

---

# `docs/baseline/`

## Purpose

Preserves information about the system Reliora is improving from.

Current important example:

```text
stage1-baseline-manifest.md
```

It records the frozen Stage-1 baseline and provenance information.

---

# Why a Baseline Matters

Reliora is a Stage-2 production-hardening project.

To claim improvement, we need to know:

```text
what the original system was
```

before measuring:

```text
what changed
```

The baseline provides that reference point.

---

# `docs/business/`

## Purpose

Documents the business/product framing.

Current important example:

```text
project-charter.md
```

---

# Why Business Documentation Matters

Engineering decisions should connect to actual requirements.

The system should not become:

> A collection of interesting AWS services.

The project charter explains:

- why the system exists
- what problem it solves
- what outcome matters
- what the boundaries are

---

# `docs/requirements/`

Important files include:

```text
functional-requirements.md
non-functional-requirements.md
```

---

## Functional Requirements

Describe what the system must do.

Examples include:

- classify requests
- collect bug information
- validate tool execution
- support idempotency
- perform structured handoff
- preserve provenance

---

## Non-Functional Requirements

Describe qualities the system must exhibit.

Examples include:

- reliability
- security
- privacy
- observability
- maintainability
- cost control
- governance

---

# Why Separate Functional and Non-Functional Requirements

A system can technically perform a function while still being unacceptable.

For example:

```text
Function:
create ticket
```

may work.

But the implementation could still fail important non-functional requirements such as:

```text
security
latency
auditability
retry safety
privacy
```

Both categories matter.

---

# `docs/scope/`

Important example:

```text
moscow.md
```

---

## What MoSCoW Means

MoSCoW is a prioritization framework:

```text
Must Have
Should Have
Could Have
Won't Have
```

It helps prevent uncontrolled project expansion.

---

# Why Scope Control Matters

A portfolio project can easily become overloaded with technologies.

Reliora deliberately excludes technology that does not solve a documented requirement.

This keeps the project focused on:

```text
AI reliability
LLMOps
safe side effects
observability
release safety
unit economics
```

rather than adding tools for prestige.

---

# `docs/governance/`

Important example:

```text
autonomy-policy.md
```

---

## Purpose

Defines what the AI system is allowed to do automatically and what requires stronger controls or human involvement.

Conceptually:

```text
Low-risk information
→ automatic

Validated bug-ticket creation
→ automatic with controls

Higher-risk business actions
→ human approval / stronger authorization

Financial, legal, irreversible, or high-risk action
→ not autonomous in this project
```

---

# Why Autonomy Policy Matters

Agentic systems can take actions rather than only generate text.

Therefore an important question becomes:

> What is the AI allowed to do?

That question belongs in governance, not only in prompt engineering.

---

# `docs/security/`

Important example:

```text
threat-model.md
```

---

## Purpose

Documents known threat categories and system risks.

Examples include:

- prompt injection
- internal-information leakage
- malformed tool arguments
- unauthorized actions
- sensitive data exposure
- oversized inputs
- fabricated tool results

---

# Why Security Documentation Is Separate

Security is a cross-cutting engineering concern.

It should not depend on someone remembering:

> We should probably make this secure later.

A threat model makes risks explicit.

---

# `docs/architecture/`

Important content includes:

```text
logical-architecture.md
adr/
```

---

# `logical-architecture.md`

## Purpose

Explains the system's logical components and how requests flow through them.

A logical architecture focuses on responsibilities rather than decorative cloud icons.

---

# `adr/`

ADR means:

```text
Architecture Decision Record
```

An ADR documents an important design decision.

Examples of Reliora decisions include:

- use AgentCore Harness
- separate intent from knowledge coverage
- deterministic validation before side effects
- operation-level idempotency
- persistent memory disabled unless needed
- separate deterministic and generative evaluation
- structured handoff
- Terraform for infrastructure
- GitHub OIDC for CI/CD authentication

---

# Why ADRs Matter

An architecture diagram shows:

```text
what exists
```

An ADR explains:

```text
why it exists that way
```

Future engineers can understand the trade-offs instead of guessing.

---

# `docs/operations/`

Important examples:

```text
failure-mode-analysis.md
diagnostic-playbook.md
```

---

# Failure-Mode Analysis

Asks questions such as:

```text
What can fail?

How would we detect it?

What impact would it have?

How could we recover?

What might fail first as the system scales?
```

---

# Diagnostic Playbook

Focuses on practical investigation.

For example:

```text
symptom
    ↓
possible causes
    ↓
signals to inspect
    ↓
diagnosis
    ↓
recovery action
```

---

# Why Operations Documentation Matters

A system is not production-oriented merely because deployment succeeds.

Production engineering also requires understanding:

```text
failure
diagnosis
recovery
monitoring
operational ownership
```

---

# 6. `infra/` — Infrastructure as Code

Reliora plans to use Terraform for AWS infrastructure.

The `infra/` area will eventually contain infrastructure definitions.

Possible examples include:

- IAM roles
- Lambda
- DynamoDB
- AgentCore resources
- CloudWatch
- budgets
- deployment configuration
- remote Terraform state configuration

---

## Why Infrastructure Does Not Belong Under `src/`

Application logic and infrastructure definitions solve different problems.

```text
src/
→ application behaviour

infra/
→ cloud resource definition
```

They may evolve and deploy differently.

Separating them makes responsibilities clearer.

---

# Infrastructure as Code

Infrastructure as Code means defining cloud infrastructure in version-controlled configuration rather than manually creating everything in the AWS Console.

Benefits include:

- repeatability
- reviewability
- auditability
- easier teardown
- reduced configuration drift
- CI/CD integration

---

# 7. Important Root-Level Files

Some files live at the repository root because they affect the project as a whole.

---

# `pyproject.toml`

## Purpose

Defines the Python project configuration.

It can contain:

- project name
- version
- Python version requirement
- runtime dependencies
- development dependencies
- build-system configuration
- tool configuration

---

## Why It Is at the Root

It applies to the Python project as a whole rather than to one package module.

---

# `.gitignore`

## Purpose

Tells Git which files or directories should normally not enter version control.

Examples can include:

```text
.venv/
Python cache files
local secrets
temporary files
Terraform state
local-only evidence
```

---

## Why `.gitignore` Matters

Not every file created during development belongs in the repository.

Some files are:

- machine-specific
- reproducible
- temporary
- sensitive
- too large
- runtime-generated

`.gitignore` helps prevent accidental tracking.

---

# `.gitattributes`

## Purpose

Defines repository-level Git file-handling rules.

Reliora uses it to normalize text files to:

```text
LF
```

line endings.

---

## Why It Became Important

During evidence hashing, Windows CRLF working-copy bytes produced a different SHA-256 from Git's LF staged representation.

`.gitattributes` made the intended Git representation explicit.

---

# `.editorconfig`

## Purpose

Provides editor formatting conventions.

Examples include:

- UTF-8 encoding
- LF line endings
- indentation rules

---

# `.editorconfig` vs `.gitattributes`

A useful distinction is:

```text
.editorconfig
→ helps the editor create consistent text

.gitattributes
→ tells Git how repository files should be handled
```

They are related but operate at different layers.

---

# 8. File Type vs Folder Location

A useful way to decide where something belongs is to first ask:

> What kind of artifact is this?

---

## Reusable Python Logic

Example:

```text
evaluate_required_fact_completeness(...)
```

Location:

```text
src/
```

because it is reusable application logic.

---

## Automated Verification of a Python Function

Example:

```text
test_complete_response_passes()
```

Location:

```text
tests/
```

because it verifies application logic.

---

## AI Evaluation Input Cases

Example:

```text
stage1-reproduction-v1.json
```

Location:

```text
evals/datasets/
```

because it is evaluation input data.

---

## Human Explanation of an Evaluation Dataset

Example:

```text
stage1-reproduction-v1.md
```

Location:

```text
evals/dataset_cards/
```

because it documents the dataset.

---

## Script That Runs a Specific Evaluation Experiment

Example:

```text
run_stage1_reproduction.py
```

Location:

```text
evals/runners/
```

because it is an experiment entry point.

---

## Output Produced by an Experiment

Example:

```text
stage1-reproduction-v1-report.json
```

Location:

```text
evidence/generated/
```

because it records what happened.

---

## Requirements or Architecture Explanation

Location:

```text
docs/
```

because it is human-readable engineering documentation.

---

## Terraform Definition

Location:

```text
infra/
```

because it defines cloud infrastructure.

---

# 9. Why Not Put Everything in One Folder?

A small script may initially work with:

```text
project/
├── app.py
├── test.py
├── data.json
├── output.json
├── notes.md
└── terraform.tf
```

But as the project grows, responsibilities become mixed.

Questions become harder to answer:

```text
Which file is production code?

Which is generated?

Which is safe to delete?

Which is an input dataset?

Which is evidence?

Which belongs in deployment?

Which files should CI execute?
```

A structured repository makes those boundaries visible.

---

# 10. Repository Structure Communicates Architecture

Folder organization is not merely cosmetic.

Consider:

```text
src/reliora/evaluation/factual.py
```

The path itself tells us:

```text
This is source code
for the Reliora package
inside the evaluation domain
implementing factual evaluation.
```

Compare:

```text
evals/datasets/stage1-reproduction-v1.json
```

The path tells us:

```text
This is evaluation material
specifically an input dataset
for Stage-1 reproduction.
```

Good naming and structure reduce the amount of explanation required to understand a system.

---

# 11. Source Code vs Configuration vs Data vs Evidence

These are different artifact classes.

```text
Source code
→ instructions that implement software behaviour

Configuration
→ settings controlling how software or tools operate

Dataset
→ input data used for evaluation

Documentation
→ human explanation and decisions

Generated evidence
→ recorded output from an experiment

Infrastructure
→ definitions of cloud resources
```

Understanding the artifact class usually helps determine where it belongs.

---

# 12. Generated Files Need Special Attention

A generated file can still be committed if it serves a valid evidence or audit purpose.

However, generated files should be clearly identified.

Reliora uses:

```text
evidence/generated/
```

to communicate that these files are outputs rather than manually authored source code.

This matters because future automation may regenerate them.

---

# 13. Why Evidence Is Version Controlled Here

Some generated output is normally ignored.

For example:

```text
temporary logs
cache files
compiled files
```

But the Stage-1 reproduction report is valuable project evidence.

Keeping the selected report in Git allows the repository to preserve:

```text
which experiment was run
what dataset it referenced
what defects were detected
what code version preserved the result
```

Not every generated artifact should be committed.

The decision depends on whether the artifact has lasting engineering value.

---

# 14. How the Current Evaluation Components Connect

A useful end-to-end view is:

```text
docs/
behavioural requirements
        ↓

evals/datasets/
Stage-1 cases
        ↓

evals/runners/
reproduction experiment
        ↓

src/reliora/evaluation/
reusable evaluator logic
        ↓

evidence/generated/
experimental report
        ↓

tests/
verify evaluator and reproduction logic
```

Notice that each directory has a different responsibility.

---

# 15. Current Stage-1 Reproduction Flow

More specifically:

```text
stage1-reproduction-v1.json
        ↓
run_stage1_reproduction.py
        ↓
evaluate_reproduction_case(...)
        ↓
┌──────────────────────────────┐
│ INV-004 → bug_workflow.py    │
│ INV-012 → leakage.py         │
│ FACT-001 → factual.py        │
└──────────────────────────────┘
        ↓
stage1-reproduction-v1-report.json
```

This is a good example of why repository structure matters.

The files participate in the same workflow but do not all have the same responsibility.

---

# 16. Why REPRO-003 Is Different

The Stage-1 reproduction dataset contains four cases.

Three are executable.

One is:

```text
REPRO-003
```

which documents historical `<thinking>` leakage.

The repository preserves that the leak occurred, but not the full raw leaked reasoning text.

Therefore Reliora does not invent missing evidence.

It records the case as:

```text
DOCUMENTED
```

rather than pretending it was replayed.

This distinction is represented in the dataset and reproduction logic rather than hidden in prose.

---

# 17. Evidence Classification

Reliora uses evidence labels to avoid overstating results.

Examples include:

```text
OBSERVED
EXPERIMENTAL
TARGET
SYNTHETIC ASSUMPTION
MEASURED
```

For the Stage-1 reproduction dataset:

```text
OBSERVED
```

describes evidence preserved from the original system.

Running Reliora's evaluators against that dataset produces:

```text
EXPERIMENTAL
```

evidence.

This helps distinguish:

```text
what historically happened
```

from:

```text
what the current experiment demonstrated
```

---

# 18. What Will Be Added Later

Reliora is still early in implementation.

The repository will grow.

Possible future areas include:

```text
infra/
CI/CD workflows
routing experiments
integration tests
security tests
observability tooling
fault-injection tests
cost evidence
frontend application
internal support console
```

The repository map should evolve when those components actually exist.

It should not pretend that planned components have already been implemented.

---

# 19. How to Decide Where a New File Belongs

When creating a new file, ask these questions.

### Is it reusable application logic?

Use:

```text
src/
```

### Is it verifying application logic?

Use:

```text
tests/
```

### Is it input data for an AI evaluation?

Use:

```text
evals/datasets/
```

### Does it document an evaluation dataset?

Use:

```text
evals/dataset_cards/
```

### Does it execute a specific evaluation experiment?

Use:

```text
evals/runners/
```

### Is it a generated experiment result worth preserving?

Use:

```text
evidence/
```

### Is it human-readable system design, requirements, or operations documentation?

Use:

```text
docs/
```

### Does it define AWS infrastructure?

Use:

```text
infra/
```

This decision process is more useful than memorizing folder names.

---

# 20. A Repository Is an Engineering Interface

A repository is read by more than the original developer.

Possible readers include:

- future maintainers
- reviewers
- teammates
- security engineers
- platform engineers
- hiring managers
- interviewers
- auditors

Good repository organization helps those readers quickly understand:

```text
what the system does
where to find things
which files are authoritative
how to run tests
how evidence was produced
where architecture decisions are documented
```

---

# 21. Why This Matters for Portfolio Projects

A portfolio repository is itself part of the demonstration.

A hiring manager may not run every line of code.

They may first inspect:

```text
README
directory structure
architecture docs
tests
infrastructure
evidence
commit history
```

A well-structured repository signals that the project was approached as an engineering system rather than only as a notebook or demo.

---

# 22. Important Lessons

1. Folder structure communicates responsibility.
2. Reusable application logic belongs under `src/`.
3. Tests verify source code but are not runtime application code.
4. AI evaluation assets deserve their own area because they are more than ordinary unit tests.
5. Datasets and dataset cards serve different audiences.
6. Experiment runners and reusable evaluator logic should remain separate.
7. Generated evidence records what actually happened.
8. Human engineering decisions belong in documentation.
9. Infrastructure definitions have a different responsibility from application code.
10. Root configuration files apply to the project as a whole.
11. A file's location should help explain what kind of artifact it is.
12. Repository organization should evolve with the real implementation rather than pretending planned features already exist.

---

## Interview Explanation

> I organize Reliora so repository structure reflects engineering responsibility. Reusable product logic lives under `src`, automated software verification under `tests`, AI evaluation datasets and experiment runners under `evals`, generated experimental proof under `evidence`, architecture and operational decisions under `docs`, and cloud definitions under `infra`. This separation makes the system easier to test, audit, maintain, and explain, while also preserving a clear distinction between source code, evaluation inputs, experiment execution, and generated evidence.