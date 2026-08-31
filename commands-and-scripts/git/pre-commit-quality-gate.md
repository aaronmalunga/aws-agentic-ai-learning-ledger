# Local Pre-Commit Quality Gate

## Why This Lesson Exists

Before committing important Reliora evaluation work, we did not immediately run:

```powershell
git add .
git commit ...
```

Instead, we verified several different properties of the work first.

The quality sequence included commands such as:

```powershell
uv run pytest -q
```

```powershell
uv run ruff check src tests evals/runners
```

```powershell
uv run mypy src evals/runners
```

and Git checks such as:

```powershell
git diff --cached --check
```

We also manually verified evaluation evidence and dataset hashes when provenance mattered.

The final evaluation foundation reached:

```text
pytest:
19 passed

Ruff:
All checks passed!

mypy:
Success: no issues found in 8 source files

git diff --cached --check:
no output
```

These results did not all mean the same thing.

Each tool answered a different engineering question.

The purpose of a local quality gate is to prevent known problems from being preserved in Git history unnecessarily.

---

# 1. What Is a Quality Gate?

A quality gate is a set of checks that must succeed before work is allowed to move to the next stage.

Conceptually:

```text
write/change code
      ↓
run quality checks
      ↓
checks pass?
    /       \
   no       yes
   ↓         ↓
fix       commit
```

Later, similar checks can be enforced automatically in CI/CD.

---

# 2. Why Run Checks Before Committing?

Git will happily commit code that:

```text
fails tests
contains lint violations
contains type inconsistencies
contains trailing whitespace
contains stale evidence
```

Git is a version-control system.

It does not know whether the software is good.

Therefore quality must be checked separately.

---

# 3. Git Commit Is Not a Quality Certification

A commit means:

> Git preserved this snapshot.

It does not mean:

```text
tests passed
code is correct
security is sound
evidence is valid
architecture is good
```

Those require separate controls.

---

# 4. Reliora's Local Quality Sequence

The evaluation work used a sequence similar to:

```text
1. Run behavioural/unit tests
2. Run linting
3. Run static type checking
4. Stage intended files
5. Inspect staged changes
6. Check staged whitespace/errors
7. Verify generated evidence/provenance
8. Commit
```

The order can vary slightly depending on the change.

The important principle is that the checks happen deliberately before preserving the snapshot.

---

# 5. Step 1 — Pytest

Command:

```powershell
uv run pytest -q
```

The final result for the evaluation foundation was:

```text
19 passed
```

---

# 6. What `uv run` Does

```text
uv run
```

executes the following command inside Reliora's managed project environment.

This helps ensure:

```text
correct Python version
correct project dependencies
correct pytest version
```

are used.

---

# 7. What `pytest` Does

Pytest executes automated Python tests.

These tests can verify behaviour such as:

```text
Does INV-004 detect multiple requested missing fields?

Does INV-012 detect prohibited route labels?

Does FACT-001 detect a missing required policy fact?

Does the reproduction runner detect expectation mismatches?
```

---

# 8. What `-q` Means

The:

```text
-q
```

flag means:

```text
quiet
```

It reduces the amount of routine output.

Instead of displaying extensive per-test information, pytest provides a more compact result.

---

# 9. Why We Used `-q`

During repeated local quality checks, we mainly needed to know:

```text
did the tests pass?
```

If a failure occurred, pytest would still provide relevant failure information.

The compact output makes repeated checks easier to read.

---

# 10. What `19 passed` Means

It means:

> All 19 tests collected during that run completed successfully.

This supports the claim:

```text
19/19 currently collected tests passed in that run
```

---

# 11. What `19 passed` Does Not Mean

It does not mean:

```text
Reliora has no bugs
Reliora is production-ready
all possible inputs work
AWS integration works
security is perfect
```

Tests provide evidence only for the behaviour they exercise.

---

# 12. Why Test Scope Matters

Suppose all tests concern:

```text
evaluation logic
```

They do not prove:

```text
Terraform deployment
AgentCore configuration
DynamoDB permissions
CloudWatch dashboards
```

Those components need their own verification later.

---

# 13. Tests Protect Refactors

The:

```text
invariant_id
→ evaluation_id
```

refactor demonstrated this.

After changing the shared contract, pytest reported:

```text
2 failed, 8 passed
```

The failures revealed stale test consumers.

That was useful.

---

# 14. A Failing Test Is Feedback

The correct interpretation was not:

```text
pytest is broken
```

The tests showed that part of the repository had not yet migrated to the new shared contract.

After correcting the stale references, the suite passed again.

---

# 15. Do Not Commit Just Because the Code "Looks Right"

Humans miss regressions.

Automated tests provide a repeatable independent check.

A useful workflow is:

```text
make change
    ↓
reason about it
    ↓
run tests
```

Both human reasoning and automated verification matter.

---

# 16. Step 2 — Ruff

Command:

```powershell
uv run ruff check src tests evals/runners
```

Final output:

```text
All checks passed!
```

---

# 17. What Ruff Checks

Ruff performs static Python linting.

It can detect configured issues such as:

```text
incorrect import ordering
unused imports
certain code-quality problems
style violations
some common programming mistakes
```

---

# 18. Why We Checked Multiple Paths

The command includes:

```text
src
tests
evals/runners
```

because Python code existed in each of those areas.

---

# 19. `src`

```text
src/
```

contains reusable application code.

For Reliora this includes evaluator implementation.

---

# 20. `tests`

```text
tests/
```

contains automated tests.

Tests are still Python code and should meet repository quality standards.

---

# 21. `evals/runners`

```text
evals/runners/
```

contains executable Python experiment runners.

These also need linting.

---

# 22. Why We Did Not Pass JSON and Markdown to Ruff

Ruff is a Python tool.

Files such as:

```text
.json
.md
.toml
```

need different validation mechanisms.

A quality tool should be applied to artifacts it understands.

---

# 23. The I001 Incident

Ruff previously found:

```text
I001 Import block is un-sorted or un-formatted
```

in newly created Python files.

The fix was targeted.

For example:

```powershell
uv run ruff check "tests/unit/evaluation/test_factual.py" --fix
```

---

# 24. Why Fix the Specific File First?

Because the known problem was in a known file.

Instead of automatically rewriting the entire repository:

```text
one known issue
→ one targeted fix
```

reduced the change scope.

---

# 25. What `All checks passed!` Means

It means:

> Ruff found no configured lint violations in the paths checked during that execution.

---

# 26. What It Does Not Mean

It does not mean:

```text
behaviour is correct
types are correct
security is correct
cloud deployment works
```

Ruff has a specific responsibility.

---

# 27. Step 3 — Mypy

Command:

```powershell
uv run mypy src evals/runners
```

Final output:

```text
Success: no issues found in 8 source files
```

---

# 28. What Mypy Does

Mypy performs static type checking.

It analyzes type information without needing to execute every possible application path.

For example, it can detect situations where code expects:

```text
str
```

but receives something typed as:

```text
int
```

or where an interface is used inconsistently.

---

# 29. Why Type Checking Matters

Reliora uses typed contracts such as:

```text
EvaluationFinding
BugField
evaluation result models
```

As the architecture grows, typed provider interfaces and structured outputs will become increasingly important.

Static checking can detect mismatches before runtime.

---

# 30. What `8 source files` Means

Mypy reported that it checked eight Python source files within the specified scope.

The number is part of the scope of the result.

---

# 31. What Mypy Success Does Not Mean

It does not prove:

```text
runtime behaviour is correct
all dynamic Python states are safe
external APIs return expected values
```

Static type checking provides one category of evidence.

---

# 32. Why Pytest and Mypy Are Both Needed

Consider:

```python
def add(a: int, b: int) -> int:
    return a - b
```

Mypy may accept the types.

But behaviour is wrong if the requirement is addition.

A behavioural test can detect that.

---

# 33. Reverse Example

A test may pass using one specific input while the type design remains inconsistent elsewhere.

Mypy can catch interface problems not exercised by that one runtime test.

Therefore:

```text
pytest
+
mypy
```

are complementary.

---

# 34. Why Ruff and Mypy Are Both Needed

Ruff may detect:

```text
unused import
```

while mypy does not care.

Mypy may detect:

```text
incompatible type
```

while Ruff does not care.

Different tools protect different properties.

---

# 35. Quality Is Multi-Dimensional

A useful mental model is:

```text
Behaviour
→ pytest

Code quality
→ Ruff

Type consistency
→ mypy

Repository/staging integrity
→ Git checks

Experiment integrity
→ evidence/provenance verification
```

No single tool replaces the others.

---

# 36. Step 4 — Inspect Git State

Before committing, inspect what Git sees.

Useful command:

```powershell
git status --short --untracked-files=all
```

This gives a compact view of:

```text
modified files
staged files
untracked files
```

---

# 37. Why `--untracked-files=all` Matters

Git can sometimes summarize untracked directories.

For the Learning Ledger especially, we want to see the actual individual files before the first commit.

Using:

```text
--untracked-files=all
```

asks Git to display all untracked files rather than only directory-level summaries.

---

# 38. Why Inspect Before `git add`

The goal is to answer:

```text
Which files actually exist?

Are any files missing?

Are unexpected temporary files present?

Are filenames correct?
```

before staging them.

---

# 39. Avoid Blind `git add .`

The command:

```powershell
git add .
```

can be convenient.

But it stages everything Git sees within the scope.

If unexpected files exist, they may be staged accidentally.

For important commits, explicit staging can be easier to review.

---

# 40. Explicit Staging

Conceptually:

```powershell
git add README.md
git add commands-and-scripts
git add concepts
git add architecture
```

or specific files.

The exact command should follow the inspected repository state.

---

# 41. Why We Should Inspect This Learning Repo First

The Learning Ledger has not yet made its first commit.

Several directories and documents have been created over time.

Before committing, we need an exact inventory.

Therefore we should not guess what should be staged.

---

# 42. Step 5 — Inspect Staged Changes

After staging, commands such as:

```powershell
git diff --cached --stat
```

can summarize the staged snapshot.

---

# 43. `--cached`

For Git diff:

```text
--cached
```

means compare:

```text
staging area
```

against:

```text
current HEAD
```

For a repository with no commits yet, the staged files represent the candidate initial snapshot.

---

# 44. `--stat`

```text
--stat
```

summarizes files and line changes rather than displaying every line.

It answers:

```text
Which files are included?
How large is the staged change?
```

---

# 45. Why Use a Summary First?

If a commit contains many documentation files, a full diff can be very large.

A staged summary helps detect obvious problems such as:

```text
unexpected file
unexpectedly huge change
missing folder
```

before deeper inspection.

---

# 46. `git diff --cached --name-status`

Another useful command is:

```powershell
git diff --cached --name-status
```

It can show statuses such as:

```text
A
M
D
```

for staged files.

---

# 47. Example

```text
A  README.md
A  concepts/evaluation/...
A  commands-and-scripts/git/...
```

means these are being added.

For the Learning Ledger's first commit, most files should naturally appear as:

```text
A
```

because they have never been committed before.

---

# 48. Step 6 — `git diff --cached --check`

Command:

```powershell
git diff --cached --check
```

This checks the staged diff for certain whitespace problems.

---

# 49. Expected Successful Output

Typically:

```text
no output
```

means no problems were found by this check.

---

# 50. Why "No Output" Can Be Success

Not every command prints:

```text
SUCCESS
```

Some Unix-style tools communicate success by:

```text
no diagnostic output
+
exit code 0
```

Understanding this prevents unnecessary troubleshooting.

---

# 51. What `--check` Can Catch

For example:

```text
trailing whitespace
```

in staged content.

Reliora's reproduction dataset previously triggered such an issue.

We corrected the whitespace and regenerated/rechecked the evidence.

---

# 52. Why Whitespace Can Matter

Often it is simply repository cleanliness.

But whitespace can also affect:

```text
hashes
generated evidence
diff noise
cross-platform consistency
```

when exact text representation matters.

---

# 53. Staging Is a Quality Boundary

The staging area is useful because it lets us distinguish:

```text
everything currently in the working directory
```

from:

```text
exactly what the next commit will contain
```

Quality checks can therefore target the intended snapshot.

---

# 54. Working Tree vs Staged Snapshot

Conceptually:

```text
working tree
→ everything currently edited

staging area
→ selected candidate commit

commit
→ preserved history
```

This is why Git staging is more than an administrative step.

It can be part of deliberate review.

---

# 55. Step 7 — Evidence Verification

For ordinary source changes, tests/lint/type checks may be enough.

For Reliora's evaluation experiments, we also had:

```text
dataset
runner
generated evidence
hash
```

that needed consistency.

---

# 56. The Hash Incident

The reproduction report initially contained a SHA-256 value that did not match the canonical Git representation of the dataset.

The cause was:

```text
Windows working copy:
CRLF

Git staged representation:
LF
```

The runner had hashed the working-copy bytes.

---

# 57. Why This Mattered

The report claimed to identify:

```text
the dataset that produced the experiment
```

If the recorded hash did not match the canonical artifact, provenance was weaker.

This was not merely cosmetic.

---

# 58. Canonical Hashing Fix

Reliora changed the hashing procedure to normalize text as:

```text
UTF-8
+
LF line endings
```

before calculating SHA-256.

The report documents the basis as:

```text
UTF-8 text normalized to LF line endings before SHA-256
```

---

# 59. Final Dataset Hash

The canonical dataset hash became:

```text
4f83b0ffa6a87b4f1dab68d27352b0f40d3556a87b4935e073bc59b0e38e1429
```

The report and canonical dataset representation then agreed.

---

# 60. Why Generated Evidence Must Be Verified

A generated file is only valuable if:

```text
it corresponds to the current input
it was produced by the current runner
its metadata is correct
its result is interpreted correctly
```

A stale report can be worse than no report because it creates false confidence.

---

# 61. Regenerate, Do Not Manually Repair Results

If evaluator input or execution logic changes:

```text
rerun the experiment
```

rather than manually editing:

```text
the generated result
```

This preserves the relationship between code and evidence.

---

# 62. Example Reproduction Command

The Stage-1 reproduction runner produced:

```text
Stage-1 reproduction complete
Total cases: 4
Executable cases: 3
Documented non-executable cases: 1
Detected known defects: 3
Mismatches: 0
All executable expectations met: True
```

This output became part of the experiment review before commit.

---

# 63. Why Re-Run After Changes

Suppose:

```text
factual evaluator changes
```

but the report remains from the previous evaluator.

Then:

```text
current code
```

and:

```text
evidence report
```

describe different states.

Regeneration keeps them synchronized.

---

# 64. Step 8 — Review What the Commit Means

Before committing, ask:

```text
What logical unit of work am I preserving?
```

A commit should ideally communicate one coherent engineering change.

---

# 65. Example Reliora Commit

One existing commit is:

```text
test: reproduce Stage-1 evaluation gaps
```

This is stronger than:

```text
stuff
```

because it communicates purpose.

---

# 66. Commit Message Structure

A useful pattern is:

```text
type: concise purpose
```

Examples already used include:

```text
chore: establish Reliora engineering foundation

docs: freeze Stage-1 baseline provenance

test: add deterministic behavioural evaluators

test: reproduce Stage-1 evaluation gaps
```

The exact convention can evolve, but consistency helps.

---

# 67. Why Commit Messages Matter

Git history should help answer:

```text
What changed?
Why did this change exist?
When was this capability introduced?
```

Good commit messages strengthen future debugging and project storytelling.

---

# 68. Commit Only After Reviewing the Snapshot

The desired flow is:

```text
working tree
    ↓
quality checks
    ↓
stage
    ↓
inspect staged snapshot
    ↓
commit
```

not:

```text
change files
    ↓
commit immediately
    ↓
discover issues later
```

---

# 69. Quality Gates Reduce, Not Eliminate, Defects

Even after:

```text
pytest passes
Ruff passes
mypy passes
Git checks pass
```

bugs can remain.

A quality gate improves confidence.

It does not create mathematical proof of whole-system correctness.

---

# 70. Local Gate vs CI Gate

A local gate runs on the developer's machine.

A CI gate runs in an automated environment after:

```text
push
pull request
```

or another trigger.

---

# 71. Why We Eventually Need Both

Local checks provide:

```text
fast feedback
```

CI provides:

```text
independent repeatable enforcement
```

A developer might accidentally forget a local step.

CI can ensure repository policy is still enforced.

---

# 72. Future GitHub Actions Flow

Conceptually:

```text
Pull request
      ↓
install environment
      ↓
pytest
      ↓
Ruff
      ↓
mypy
      ↓
evaluation regression suite
      ↓
security/IaC checks
      ↓
release decision
```

This will become part of Reliora's LLMOps work later.

---

# 73. CI Should Not Be Built Before We Understand Local Commands

It is valuable to understand:

```text
what each command does locally
```

before hiding it inside:

```text
.github/workflows/*.yml
```

CI automates engineering understanding.

It should not replace it.

---

# 74. A Passing CI Badge Is Also Narrow Evidence

A green CI pipeline means:

> The configured checks passed for that run.

It does not automatically mean:

```text
production is healthy
security is perfect
all requirements are met
```

The meaning depends on which checks the workflow actually contains.

---

# 75. Quality Gate Expansion

Reliora's local gate will eventually expand.

Potential later checks include:

```text
Gitleaks
Checkov
Terraform validation
Terraform formatting
security/adversarial evaluations
cloud integration tests
routing regression thresholds
structured-output validation
```

These should be introduced when the corresponding implementation exists.

---

# 76. Do Not Add Tools Merely to Increase Tool Count

A project does not become better because its CI contains:

```text
15 scanners
```

Each tool should protect a documented property.

The question is:

> Which failure are we trying to prevent or detect?

---

# 77. Example: Gitleaks

Future purpose:

```text
detect accidentally committed secrets
```

This directly supports the requirement:

```text
do not commit AWS credentials or other secrets
```

---

# 78. Example: Checkov

Future purpose:

```text
inspect infrastructure-as-code for security/configuration problems
```

This becomes relevant once Terraform exists.

Running it before there is infrastructure code would add little value.

---

# 79. Example: Terraform Validation

Future commands may include:

```text
terraform fmt
terraform validate
```

These protect Terraform syntax/configuration quality.

They do not replace:

```text
application tests
```

---

# 80. Example: Evaluation Regression Gate

Later:

```text
known critical invariant fails
→ CI fails
```

This is especially important for AI systems because code can remain syntactically correct while model behaviour regresses.

---

# 81. Traditional CI Is Not Enough for AI Systems

A normal software pipeline may verify:

```text
code compiles
tests pass
types pass
```

but AI behaviour can still change because of:

```text
prompt change
model change
retrieval change
routing threshold change
knowledge change
```

Therefore Reliora needs AI-specific evaluation gates too.

---

# 82. AI Release Safety

A mature release process can include:

```text
software checks
+
behavioural evaluations
+
factual evaluations
+
routing metrics
+
tool-safety checks
```

This is part of LLMOps.

---

# 83. Why Known Invariants Can Become Hard Gates

Suppose:

```text
INV-012
```

detects customer-visible internal leakage.

A release that reintroduces this may need to be blocked regardless of a high average generative score.

Critical properties should not disappear into an aggregate average.

---

# 84. Quality Gate Ordering

There is no universal perfect order.

A practical strategy is:

```text
cheap checks first
expensive checks later
```

For example:

```text
Ruff
→ fast

mypy
→ relatively fast

unit tests
→ fast/moderate

cloud integration tests
→ slower

LLM evaluations
→ slower/cost-bearing
```

This can fail early and reduce wasted time/cost.

---

# 85. Why We Sometimes Ran Pytest First

During active development, the immediate concern was behavioural correctness.

Therefore pytest was often run first.

The optimal CI ordering later can differ.

The workflow should fit the purpose.

---

# 86. Fast Feedback Loop

During implementation:

```text
edit
↓
targeted test
↓
targeted lint
↓
broader suite
```

is often efficient.

---

# 87. Example

If only:

```text
test_factual.py
```

has an import issue:

```powershell
uv run ruff check "tests/unit/evaluation/test_factual.py" --fix
```

is faster and safer than repeatedly auto-fixing the entire repository.

---

# 88. Broader Verification Before Commit

Before finalizing the logical change:

```powershell
uv run ruff check src tests evals/runners
```

checks the larger relevant scope.

This catches interactions outside the immediate file.

---

# 89. Targeted Development, Broad Final Verification

A useful pattern is:

```text
During debugging:
targeted commands

Before commit:
broader quality gate
```

This balances speed and confidence.

---

# 90. Warnings Need Interpretation

During Git staging, Windows line-ending warnings appeared.

For example, Git can warn that:

```text
CRLF will be replaced by LF
```

A warning does not automatically mean:

```text
command failed
```

But it should still be understood.

---

# 91. In Reliora, the Warning Became Important

Normally, line-ending normalization may be routine.

But because Reliora was calculating:

```text
SHA-256
```

over text evidence, byte differences affected provenance.

Therefore the warning pointed to a property that actually mattered.

---

# 92. Do Not Ignore Every Warning

A warning may be:

```text
harmless
informational
important under current requirements
```

Interpret it in context.

---

# 93. Error vs Failure vs Warning

Useful categories:

```text
Warning
→ operation may continue, but something deserves attention

Lint violation
→ static repository rule failed

Test failure
→ executed expectation failed

Type failure
→ static type contract failed

Command error
→ command could not complete as intended
```

These require different responses.

---

# 94. Quality Gate Should Be Reproducible

A future engineer should be able to run:

```text
the same documented commands
```

and understand:

```text
what success means
```

This is why the Learning Ledger documents each tool individually.

---

# 95. Quality Gate Should Be Scriptable Later

Once the workflow stabilizes, repeated commands may be wrapped in:

```text
task runner
script
Makefile equivalent
PowerShell script
CI workflow
```

But first understand each command.

---

# 96. Avoid "Magic Scripts"

A script such as:

```powershell
.\check-everything.ps1
```

can be convenient.

But if nobody knows what it does, debugging becomes difficult.

The Learning Ledger preserves the underlying commands and reasons.

---

# 97. Quality Gate Results Are Evidence

A successful local gate can support claims such as:

```text
19 tests passed
Ruff found no configured violations
mypy found no issues in the checked files
Git staged whitespace check passed
```

Each claim should retain its scope.

---

# 98. Do Not Turn the Gate Into "Production-Ready"

Even a perfect local quality gate does not prove:

```text
production readiness
```

Production readiness also includes:

```text
cloud integration
security
identity
observability
deployment
rollback
failure recovery
performance
cost
operational runbooks
```

Reliora will address those progressively.

---

# 99. Quality Gate Is One Layer of Defence

Conceptually:

```text
Developer reasoning
        +
unit tests
        +
lint
        +
type checks
        +
Git review
        +
evaluation
        +
CI
        +
cloud testing
        +
observability
```

Reliability comes from layers.

---

# 100. The Local Gate Is Also a Learning Tool

When a check fails, the failure teaches something.

Examples from Reliora:

```text
pytest failure
→ shared contract refactor affected tests

Ruff I001
→ import ordering rule violated

Git hash mismatch
→ CRLF/LF changed byte representation

Git pager failure
→ shell environment affected Git
```

Errors should become reusable engineering knowledge.

---

# 101. The Goal Is Not a Permanently Green Terminal

A system where no check ever fails may mean:

```text
the checks are too weak
```

Good checks should detect real mistakes.

The goal is:

```text
fail meaningfully
diagnose correctly
fix deliberately
return to green
```

---

# 102. Pre-Commit Checklist Mental Model

Before a meaningful Reliora commit:

```text
Do tests pass?

Does Ruff pass?

Does mypy pass?

Are only intended files staged?

Does staged diff look correct?

Does git diff --cached --check pass?

Are generated artifacts current?

Do evidence hashes/provenance agree?

Does the commit message describe one coherent change?
```

The exact checklist can evolve with the project.

---

# 103. Learning Ledger Commit Has Different Checks

The Learning Ledger currently consists primarily of Markdown documentation.

It does not need:

```text
Reliora's Python pytest suite
```

because it is a separate repository.

For its first commit, the most important checks are:

```text
exact file inventory
correct directory structure
correct filenames
no accidental files
Markdown readability
Git staged snapshot review
```

Quality gates should match repository content.

---

# 104. Do Not Copy Reliora's Gate Blindly to Every Repository

A documentation repository and a Python application have different risks.

The principle is:

> Design the quality gate around what can actually go wrong in that repository.

---

# 105. Learning Ledger First Commit

Before creating the first commit, we should:

```text
1. inspect all untracked files
2. inspect directory structure
3. confirm the expected documents exist
4. review root README state
5. stage intentionally
6. inspect staged snapshot
7. run Git staged checks
8. create the initial documentation commit
```

This should happen after this catch-up lesson.

---

# 106. README Work Comes After Inventory

The repository READMEs should eventually explain:

```text
what each directory contains
where to find different learning topics
how the ledger should be used
```

But accurate navigation depends on knowing the actual repository state.

Therefore directory inventory should come first.

---

# 107. After the First Ledger Commit

The Learning Ledger should stop being a separate documentation marathon.

Instead:

```text
build Reliora capability
        ↓
encounter concept/command/error
        ↓
understand it
        ↓
add or update relevant ledger entry
```

This keeps documentation grounded in actual engineering work.

---

# 108. Why Just-in-Time Documentation Is Better

If we document every future AWS concept now:

```text
we may teach architecture that changes before implementation
```

or:

```text
describe errors we never actually encounter
```

Just-in-time learning keeps the ledger connected to real decisions and evidence.

---

# 109. The Ledger Is Not a Textbook Copy

Its purpose is not to reproduce generic documentation.

It should answer:

```text
What did I use?

Why did I use it?

What happened?

What failed?

What did I learn?

How does this connect to architecture?

How would I explain it in an interview?
```

That makes it personally reusable.

---

# 110. Important Lessons

1. Git commits preserve snapshots; they do not certify software quality.
2. A quality gate checks defined properties before work moves to the next stage.
3. Reliora uses pytest, Ruff, mypy, Git inspection, and evidence verification for different purposes.
4. `uv run pytest -q` verifies behaviour represented by the current tests.
5. `19 passed` means all 19 collected tests passed in that run, not that the whole system is defect-free.
6. Ruff checks configured Python lint and code-quality rules.
7. `All checks passed!` applies only to Ruff's configured scope.
8. Mypy checks static type consistency and does not replace runtime testing.
9. Pytest, Ruff, and mypy are complementary rather than redundant.
10. `git status --short --untracked-files=all` is useful for reviewing an exact repository inventory.
11. The staging area represents the candidate commit rather than the entire working directory.
12. Explicit staging can reduce accidental inclusion of unrelated files.
13. `git diff --cached --stat` provides a high-level staged-change summary.
14. `git diff --cached --name-status` shows which staged files are added, modified, or deleted.
15. `git diff --cached --check` checks the staged diff for whitespace problems and may succeed with no output.
16. Generated experimental evidence requires additional provenance validation beyond ordinary source-code checks.
17. Line-ending differences mattered in Reliora because they affected SHA-256 artifact identity.
18. Generated reports should be regenerated after material evaluator or dataset changes rather than manually repaired.
19. Commit messages should describe a coherent unit of engineering work.
20. Local checks provide fast feedback; CI later provides independent automated enforcement.
21. AI systems need AI-specific regression/evaluation gates in addition to conventional software checks.
22. Critical behavioural invariants may become hard release gates rather than being averaged into broad scores.
23. Quality checks should be introduced because they protect documented properties, not to maximize tool count.
24. Targeted checks are useful during development; broader checks should run before a meaningful commit.
25. Warnings, lint failures, test failures, type failures, and command errors represent different classes of feedback.
26. A useful quality system is expected to fail when real problems are introduced.
27. The correct workflow is to understand the failure, fix the smallest responsible layer, and verify again.
28. Different repositories require different quality gates.
29. The Learning Ledger should document new concepts alongside actual engineering work after the initial catch-up.
30. The purpose of the gate is not merely to produce a green terminal; it is to preserve a repository state that we can explain and defend.

---

## Interview Explanation

> Before committing important Reliora changes, I use a layered local quality gate rather than treating a successful Git commit as proof of correctness. Pytest verifies behavioural expectations, Ruff checks configured Python quality rules, mypy validates static type contracts, and Git staged checks confirm that the intended snapshot is clean. For evaluation experiments I add another layer by verifying that generated evidence corresponds to the current versioned dataset and canonical hash. The tools are intentionally complementary: a passing linter does not prove behaviour, a passing unit suite does not prove type consistency or cloud integration, and none of them alone imply production readiness. The same checks will later move into CI/CD so release safety is independently enforced rather than depending only on developer discipline.