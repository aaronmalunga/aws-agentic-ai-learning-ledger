# AWS Agentic AI Learning Ledger

A reusable engineering, learning, and interview companion for hands-on AWS AI/ML and agentic AI projects.

This repository records the reasoning behind the systems I build: commands, architecture concepts, implementation decisions, failures, troubleshooting, evaluation methodology, reliability principles, AWS service knowledge, and interview explanations.

It is intentionally kept separate from production portfolio repositories such as **Reliora**.

The portfolio repository answers:

> What was built?

The Learning Ledger answers:

> Why was it built this way, how does it work, what failed, and what did I learn?

---

## Purpose

The goal of this repository is not to collect commands or definitions blindly.

A concept is considered learned when I can explain:

```text
what it is
why it was needed
where it appears in the project
how it works
what can fail
how I diagnosed failures
what trade-offs exist
how I would use it again
how I would explain it in an interview
```

The ledger is designed to grow alongside real engineering work.

New material should therefore come primarily from:

```text
actual implementations
actual commands
actual errors
actual architecture decisions
actual experiments
actual AWS deployments
```

rather than from pre-writing theoretical documentation for technologies that have not yet been used.

---

# Repository Relationship

The Learning Ledger is not the production application.

For example:

```text
Reliora repository
→ production-oriented implementation
→ application code
→ tests
→ evaluation datasets
→ infrastructure
→ generated evidence
→ architecture documentation

AWS Agentic AI Learning Ledger
→ engineering explanations
→ command knowledge
→ troubleshooting
→ architectural concepts
→ interview preparation
→ reusable lessons across projects
```

This keeps production repositories focused while preserving deeper learning separately.

---

# Repository Structure

```text
aws-agentic-ai-learning-ledger/
│
├── architecture/
│   └── repository-anatomy/
│
├── commands-and-scripts/
│   ├── git/
│   ├── powershell/
│   ├── python/
│   ├── troubleshooting/
│   └── uv/
│
├── concepts/
│   ├── evaluation/
│   ├── reliability/
│   └── software-engineering/
│
└── README.md
```

Additional sections will be added when real project work requires them.

Git does not track empty directories, so planned categories do not need placeholder files merely to make them visible.

---

# 1. Architecture

Location:

```text
architecture/
```

This section explains how project repositories and systems are organized.

Current material includes:

```text
architecture/repository-anatomy/reliora-repository-map.md
```

The repository map explains Reliora's artifact classes and why different files belong under areas such as:

```text
src/
tests/
evals/
evidence/
docs/
```

Use this section when the question is:

> Where does this component belong, and why?

---

# 2. Commands and Scripts

Location:

```text
commands-and-scripts/
```

This section documents commands and development workflows used during real project work.

A command entry should explain:

```text
what the command does
why it was used at that point
important flags and arguments
expected output
side effects
risks
common failures
how to diagnose them
when the command should be reused
```

Current categories include:

```text
git/
powershell/
python/
troubleshooting/
uv/
```

Future categories may include:

```text
aws-cli/
terraform/
github-actions/
```

when those tools are actually introduced into project implementation.

---

## Git

Location:

```text
commands-and-scripts/git/
```

Current lessons include:

```text
git-working-tree-staging-and-commits.md
pre-commit-quality-gate.md
```

Topics include:

```text
working tree
staging area
untracked files
commits
HEAD
staged snapshots
Git status codes
staged diff inspection
local quality gates
```

Use this section when the question is:

> What state is Git in, and what exactly will be committed?

---

## PowerShell

Location:

```text
commands-and-scripts/powershell/
```

Current lesson:

```text
powershell-paths-environment-and-safe-file-operations.md
```

Topics include:

```text
current working directory
relative vs absolute paths
environment variables
filesystem inspection
safe file operations
PowerShell providers
shell state
encoding
command context
```

Use this section when the question is:

> What is PowerShell doing around the development tools I am running?

---

## Python Quality Tooling

Location:

```text
commands-and-scripts/python/
```

Current lesson:

```text
testing-linting-and-type-checking.md
```

It explains the different responsibilities of:

```text
pytest
Ruff
mypy
```

and why they complement rather than replace one another.

Use this section when the question is:

> Which quality property is this Python tool actually checking?

---

## uv and Python Environments

Location:

```text
commands-and-scripts/uv/
```

Current lesson:

```text
uv-and-virtual-environments.md
```

Topics include:

```text
virtual environments
dependency isolation
Python versions
uv
pyproject.toml
uv run
environment activation/deactivation
```

Use this section when the question is:

> Which Python environment is this project actually using?

---

# 3. Troubleshooting

Location:

```text
commands-and-scripts/troubleshooting/
```

Troubleshooting entries preserve real failures encountered during project development.

Current examples include:

```text
git-line-endings-and-evidence-hashes.md
git-pager-environment-error.md
powershell-utf8-file-encoding.md
pytest-contract-refactor-failure.md
ruff-import-order-i001.md
uv-module-name-mismatch.md
```

Errors are intentionally preserved as learning material.

A troubleshooting entry should capture:

```text
symptom
error message
affected layer
root cause
diagnosis
fix
why the fix worked
what would happen if ignored
how to recognize the problem again
```

A guiding principle is:

> Fix the smallest layer that actually contains the fault.

For example:

```text
bad environment variable
→ fix shell environment

incorrect file encoding
→ fix file encoding

shared contract refactor
→ update affected consumers

module-name mismatch
→ fix package configuration
```

Do not reinstall or redesign unrelated components before identifying the failing layer.

---

# 4. Evaluation Concepts

Location:

```text
concepts/evaluation/
```

Current lessons include:

```text
deterministic-vs-llm-as-judge.md
behavioural-invariants-vs-factual-completeness.md
dataset-card-runner-and-evidence.md
evidence-classes-and-claim-discipline.md
regression-cases-and-reproduction-statuses.md
```

This section explains how AI behaviour is converted into explicit, reviewable evidence.

---

## Deterministic vs Generative Evaluation

Reliora distinguishes:

```text
hard contractual requirements
```

from:

```text
soft generative quality
```

Examples of hard requirements include:

```text
do not expose internal route labels
ask exactly one missing bug field at a time
do not create duplicate side effects
do not fabricate ticket IDs
```

These should be evaluated deterministically wherever practical.

Generative qualities such as:

```text
helpfulness
tone
clarity
semantic quality
```

may use human or LLM-based evaluation.

---

## Behavioural Invariants and Factual Completeness

Reliora distinguishes evaluation families such as:

```text
INV-*
→ behavioural invariants

FACT-*
→ factual completeness
```

For example:

```text
INV-004
→ one missing bug field at a time

INV-012
→ prohibited internal-information leakage

FACT-001
→ required policy facts must be preserved
```

This separation prevents every failure from being collapsed into one generic concept of "LLM correctness."

---

## Dataset, Card, Runner, Evidence

The evaluation artifact chain follows:

```text
Dataset
→ what cases are evaluated

Dataset card
→ why those cases exist and where they came from

Runner
→ how the evaluation is executed

Evidence
→ what actually happened
```

These artifacts serve different purposes and should remain separate.

---

# 5. Evidence and Claim Discipline

Reliora uses explicit evidence classes:

```text
OBSERVED
EXPERIMENTAL
MEASURED
TARGET
SYNTHETIC ASSUMPTION
```

These labels prevent architecture intentions, historical evidence, experiments, measurements, and assumptions from being presented as though they are the same thing.

Examples:

```text
Historical Stage-1 response
→ OBSERVED

Reliora reproduction run
→ EXPERIMENTAL

Future p95 latency test
→ MEASURED once actually collected

routing macro F1 >= 0.95
→ TARGET until evaluated

modelled traffic volume
→ SYNTHETIC ASSUMPTION unless based on real workload
```

A central rule is:

> Claims must not be stronger than the evidence supporting them.

---

## Denominators Matter

Instead of:

```text
100% reliable
```

prefer:

```text
3 of 3 executable known defects in stage1-reproduction-v1
were detected by the deterministic evaluator.
```

The second statement defines the scope.

---

## Evidence Chain

A defensible claim should be traceable through something like:

```text
source repository
        ↓
frozen Git commit
        ↓
source artifact
        ↓
canonical hash
        ↓
versioned dataset
        ↓
evaluator / runner
        ↓
generated evidence
        ↓
engineering claim
```

Evidence provenance is part of engineering quality.

---

# 6. Regression and Reproduction

Known failures should become reusable engineering controls when possible.

Reliora currently uses cases such as:

```text
REPRO-001
REPRO-002
REPRO-003
REPRO-004
```

A reproduction case identifies:

```text
a concrete historical example
```

while an evaluation ID such as:

```text
INV-004
FACT-001
```

identifies:

```text
the general rule used to evaluate that example
```

---

## Reproduction Statuses

Current conceptual statuses include:

```text
DETECTED
CONFIRMED
MISMATCH
DOCUMENTED
```

Meaning:

```text
DETECTED
→ known bad behaviour was correctly rejected

CONFIRMED
→ expected-valid behaviour was correctly accepted

MISMATCH
→ evaluator result disagreed with reviewed expectation

DOCUMENTED
→ historical evidence exists but cannot be honestly replayed
```

A known bad response may therefore correctly produce:

```text
passed = false
status = DETECTED
```

The response failed the behavioural rule, while the evaluator succeeded at detecting that failure.

---

# 7. Reliability Concepts

Location:

```text
concepts/reliability/
```

Current lessons include:

```text
llm-proposes-software-validates.md
idempotency-and-retry-safety.md
structured-outputs-and-validation-boundaries.md
human-handoff-and-autonomy-boundaries.md
intent-routing-vs-knowledge-coverage.md
```

These documents capture Reliora's main reliability architecture.

---

## Core Reliability Principle

> **The LLM may propose; deterministic software validates and authorizes.**

The LLM can contribute:

```text
language understanding
intent classification
information extraction
candidate decisions
response generation
```

But critical side effects should cross deterministic control boundaries.

Conceptually:

```text
User
  ↓
LLM proposal
  ↓
structured validation
  ↓
workflow validation
  ↓
authorization / policy
  ↓
idempotency
  ↓
tool execution
  ↓
validated result
```

---

# 8. Structured Outputs

Structured outputs create a clearer boundary between:

```text
probabilistic model output
```

and:

```text
deterministic application state
```

However:

```text
valid JSON
!=
valid schema

valid schema
!=
valid business action

valid business action
!=
authorized action

authorized action
!=
successful side effect
```

Each layer requires appropriate verification.

---

# 9. Idempotency and Retry Safety

Retries are normal in distributed systems.

A timeout does not prove:

```text
the operation failed
```

because the backend may have committed the side effect before the response was lost.

Reliora therefore treats operation identity as a reliability control.

Conceptually:

```text
logical operation
→ stable operation_id

attempt 1
→ same operation_id

retry
→ same operation_id
```

The backend should prevent repeated attempts from creating duplicate business effects.

For ticket creation:

```text
one customer-intended operation
→ one ticket
```

even if transport attempts occur multiple times.

---

# 10. Human Handoff and Autonomy

Reliora does not optimize for maximum autonomy.

It optimizes for:

```text
appropriate autonomy for the risk
```

A simplified risk model is:

```text
Risk 0
→ informational

Risk 1
→ controlled low-impact side effect

Risk 2
→ material business/customer action

Risk 3
→ financial, security-sensitive, legal,
   irreversible, or similarly high-impact action
```

Higher-risk actions require stronger control or human participation.

Handoff is treated as a structured system capability rather than as a sentence telling the user to "contact support."

---

# 11. Intent Routing vs Knowledge Coverage

Reliora separates:

```text
What is the user asking?
```

from:

```text
Do trusted sources support an answer?
```

For example:

```text
"Do you offer a student discount?"

intent:
PLATFORM

knowledge coverage:
UNSUPPORTED
```

A correct platform classification does not authorize the model to invent a business policy.

A conceptual flow is:

```text
intent
   ↓
knowledge coverage
   ↓
policy/action decision
   ↓
answer or handoff
```

---

# 12. Software Engineering Concepts

Location:

```text
concepts/software-engineering/
```

Current lesson:

```text
provider-boundaries-and-dependency-inversion.md
```

Reliora uses conceptual provider boundaries such as:

```text
Router
KnowledgeProvider
TicketProvider
HandoffProvider
TelemetryProvider
```

The goal is not artificial cloud agnosticism.

The goal is to keep:

```text
business/application rules
```

from becoming unnecessarily coupled to:

```text
AWS SDK calls
model-specific response formats
telemetry implementation details
external backend details
```

---

## Stable Core, Replaceable Edges

A useful mental model is:

```text
Stable core
→ business rules
→ workflow
→ validation
→ contracts

Replaceable edges
→ model implementation
→ knowledge backend
→ ticket backend
→ telemetry backend
```

Provider boundaries improve:

```text
testing
failure injection
model comparison
maintainability
observability
```

without automatically requiring microservices.

---

# 13. Quality-Gate Philosophy

Before preserving important Reliora code in Git, local checks include different controls for different properties.

Examples:

```text
pytest
→ behavioural regression testing

Ruff
→ lint/code-quality checks

mypy
→ static type checking

Git staged inspection
→ commit-snapshot review

evaluation runner
→ AI behavioural evidence

hash verification
→ artifact identity/provenance
```

No individual tool proves the entire system is correct.

---

# 14. Troubleshooting Philosophy

Errors should not be hidden from the engineering story.

They are useful because they reveal:

```text
which assumptions were wrong
which system boundary failed
which tools actually enforce a property
```

The preferred process is:

```text
read the error
      ↓
identify the failing layer
      ↓
inspect actual state
      ↓
form the smallest plausible explanation
      ↓
apply a targeted fix
      ↓
rerun the original check
```

This avoids changing unrelated systems blindly.

---

# 15. File Placement Rule

When creating a new Learning Ledger entry, first ask:

### Is this explaining system organization?

Place it under:

```text
architecture/
```

### Is this explaining how to operate a tool or command?

Place it under:

```text
commands-and-scripts/
```

### Is this preserving an actual error and its diagnosis?

Place it under:

```text
commands-and-scripts/troubleshooting/
```

### Is this explaining an engineering principle?

Place it under:

```text
concepts/
```

Then choose the relevant domain:

```text
evaluation/
reliability/
software-engineering/
```

Additional domains should be introduced only when real material justifies them.

---

# 16. Learning Ledger Workflow

The intended workflow is:

```text
Build something
      ↓
encounter a new command/concept/error
      ↓
understand it
      ↓
document the reusable lesson
      ↓
continue implementation
```

The ledger should follow project development rather than become a parallel theoretical textbook.

---

# 17. Documentation Standard

Entries should be written so that I can return months later and answer:

```text
What happened?

Why did we do this?

What did the command mean?

What did the output mean?

What could have gone wrong?

How did this connect to the architecture?

What evidence supported the decision?

How would I explain it in an interview?
```

The goal is durable understanding.

---

# 18. Interview Use

The repository also acts as an interview-preparation system.

Instead of memorizing isolated definitions, explanations can be grounded in real examples.

For example:

```text
Question:
Why use idempotency?

Answer:
Because Reliora's ticket workflow can experience an ambiguous timeout
after the backend commits a write. A stable operation ID allows retries
to recover the original result rather than creating another ticket.
```

This is stronger than reciting a generic definition because it connects the concept to an implemented system problem.

---

# 19. Evidence-Driven Portfolio Principle

Portfolio claims should distinguish:

```text
designed
implemented
tested
experimentally demonstrated
measured
operationally proven
```

These are different maturity levels.

The repository should never fabricate:

```text
traffic
incidents
ROI
production users
production scale
compliance outcomes
reliability percentages
```

to make a project appear more mature.

Strong engineering claims should be:

```text
specific
traceable
reproducible where practical
appropriately scoped
```

---

# 20. Current Project Focus

The current Learning Ledger material is primarily grounded in:

```text
Reliora — Production AI Support Reliability & Control Platform
```

Reliora extends lessons from an earlier AWS AgentCore customer-support project into a deeper study of:

```text
AI evaluation
behavioural contracts
side-effect safety
LLMOps
traceability
human escalation
unit economics
production-oriented agent architecture
```

Future AWS AI/ML projects can reuse and extend the same ledger rather than rebuilding the learning system from scratch.

---

## Core Principle

> Build the system, understand the system, preserve the evidence, document the failures, and be able to explain every important engineering decision.
