# Ruff Import-Order Error: I001

## Why This Lesson Exists

While adding Reliora's factual evaluation tests and later the Stage-1 reproduction runner, Ruff reported:

```text
I001 Import block is un-sorted or un-formatted
```

The Python code itself was not necessarily behaviourally incorrect.

Pytest could still execute the relevant logic.

The problem was that the import section did not follow the repository's configured formatting and ordering rules.

We fixed the affected file using a targeted command such as:

```powershell
uv run ruff check "tests/unit/evaluation/test_factual.py" --fix
```

and later verified the complete repository scope with:

```powershell
uv run ruff check src tests evals/runners
```

which eventually returned:

```text
All checks passed!
```

This lesson explains:

- what Ruff is
- what linting means
- what `I001` means
- why import ordering matters
- what `--fix` changes
- why we used targeted automatic fixes
- why tests and linting solve different problems
- why `git diff` initially showed nothing for an untracked file
- how to verify a lint fix safely

---

# 1. What Is Ruff?

Ruff is a Python linter and code-quality tool.

A linter analyzes source code for patterns that may indicate:

- style inconsistencies
- unused imports
- import-order problems
- suspicious code
- common programming mistakes
- violations of configured coding rules

A useful mental model is:

```text
Python source code
        ↓
Ruff
        ↓
static code-quality checks
        ↓
warnings/errors
```

Ruff does this without needing to run the application in the same way a behavioural test would.

---

# 2. What Is Linting?

Linting is static analysis of source code for predefined quality rules.

For example, a linter may detect:

```python
import os
import json
```

when repository conventions expect:

```python
import json
import os
```

It may also detect imports that are:

- unused
- duplicated
- incorrectly grouped
- incorrectly ordered

Linting improves consistency and can catch some defects before runtime.

---

# 3. What Does `I001` Mean?

Ruff reported:

```text
I001 Import block is un-sorted or un-formatted
```

The code:

```text
I001
```

identifies a specific lint rule.

The `I` family comes from import-order checking behaviour compatible with rules historically associated with tools such as `isort`.

In practical terms:

> The imports in this file are not arranged in the order Ruff expects.

---

# 4. What Is an Import Block?

An import block is the section of a Python file containing statements such as:

```python
import json
from pathlib import Path

import pytest

from reliora.evaluation.factual import evaluate_required_fact_completeness
```

Python projects commonly group imports into categories.

Conceptually:

```text
standard-library imports

third-party imports

local project imports
```

Each group is usually separated by a blank line.

---

# 5. Example of Import Grouping

A well-organized import block may look like:

```python
import json
from pathlib import Path

import pytest

from reliora.evaluation.reproduction import evaluate_reproduction_case
```

This communicates:

```text
json, pathlib
→ Python standard library

pytest
→ external dependency

reliora
→ local application package
```

The structure helps readers understand dependencies quickly.

---

# 6. Why Import Ordering Matters

Python itself may successfully execute many differently ordered import blocks.

So why care?

Consistent ordering helps with:

- readability
- code review
- merge conflicts
- dependency visibility
- automated formatting
- team consistency

Without a standard, developers may organize imports differently in every file.

---

# 7. The Reliora Failure

While creating:

```text
tests/unit/evaluation/test_factual.py
```

Ruff found that the import block was not organized according to the configured rules.

The diagnostic was:

```text
I001 Import block is un-sorted or un-formatted
```

The same kind of issue later appeared in:

```text
evals/runners/run_stage1_reproduction.py
```

during the evaluation-runner work.

---

# 8. This Was a Lint Failure, Not a Test Failure

This distinction is important.

A pytest failure might mean:

```text
expected behaviour
!=
actual behaviour
```

An `I001` Ruff failure means:

```text
source formatting / import organization
!=
repository lint rules
```

They are different categories of failure.

---

# Comparison

```text
pytest failure
→ behaviour or test expectation problem

Ruff I001
→ static code-quality/import-order problem

mypy failure
→ static typing/interface problem
```

The appropriate debugging response depends on which tool failed.

---

# 9. The Targeted Fix

We used:

```powershell
uv run ruff check "tests/unit/evaluation/test_factual.py" --fix
```

---

# Breaking Down the Command

## `uv run`

Runs the command inside Reliora's managed Python project environment.

This ensures Ruff comes from the dependency set associated with Reliora rather than an arbitrary globally installed version.

---

## `ruff`

Runs the Ruff executable.

---

## `check`

Tells Ruff to inspect source code for configured lint rules.

Conceptually:

```text
ruff
    ↓
check
    ↓
analyze selected code
```

---

## File Path

```text
tests/unit/evaluation/test_factual.py
```

limits the operation to one specific file.

This was deliberate.

---

## `--fix`

Tells Ruff:

> Automatically modify the file for violations that Ruff knows how to fix safely.

For `I001`, Ruff can normally reorganize the import block automatically.

---

# 10. Why `--fix` Has a Side Effect

Without:

```text
--fix
```

Ruff primarily reports detected problems.

With:

```text
--fix
```

Ruff may edit files on disk.

Therefore:

```powershell
uv run ruff check ... --fix
```

is not read-only.

This matters because we should inspect what an automatic tool changed.

---

# 11. Why We Used a Targeted File

We could have run something broad such as:

```powershell
uv run ruff check . --fix
```

That could potentially modify many files.

Instead, we knew exactly which file had:

```text
I001
```

so we used:

```powershell
uv run ruff check "tests/unit/evaluation/test_factual.py" --fix
```

This reduced the change scope.

---

# Smallest-Change Principle

A useful engineering rule is:

> When the cause and affected file are known, start with the smallest corrective scope.

Conceptually:

```text
one known lint issue
        ↓
one known file
        ↓
targeted automatic fix
```

rather than:

```text
one known lint issue
        ↓
automatically rewrite entire repository
```

---

# 12. What Ruff Changed

For an import-order issue, Ruff may transform something like:

```python
from reliora.evaluation.factual import evaluate_required_fact_completeness

from collections.abc import Sequence
```

into:

```python
from collections.abc import Sequence

from reliora.evaluation.factual import evaluate_required_fact_completeness
```

The exact change depends on the file and configuration.

The important point is that Ruff reorganizes imports according to known categories and sorting rules.

---

# 13. Automatic Fixes Still Need Verification

The fact that a tool can fix something automatically does not mean we should blindly trust every change.

A safe workflow is:

```text
Ruff reports violation
        ↓
understand violation
        ↓
run targeted --fix
        ↓
inspect file/diff
        ↓
rerun Ruff
        ↓
rerun relevant tests
```

Automation helps reduce manual work.

Verification remains our responsibility.

---

# 14. Why We Tried `git diff`

After Ruff fixed:

```text
test_factual.py
```

we wanted to inspect exactly what changed.

We ran a command conceptually like:

```powershell
git --no-pager diff -- "tests/unit/evaluation/test_factual.py"
```

Git displayed no diff.

Initially, that could look like:

> Ruff did not change anything.

But that interpretation was incorrect.

---

# 15. Why `git diff` Showed Nothing

The file was still:

```text
untracked
```

Git had no committed historical version of it.

Ordinary:

```powershell
git diff
```

primarily compares tracked content against Git's known versions.

For a brand-new untracked file:

```text
Git history
→ no earlier version

working copy
→ new file
```

there is no tracked baseline for ordinary `git diff` to compare.

---

# Mental Model

For a tracked file:

```text
committed version
        ↓
compare
        ↓
working version
```

Git can display the changes.

For a completely untracked file:

```text
committed version
→ does not exist

working version
→ exists
```

ordinary `git diff` may show nothing.

---

# 16. Why This Did Not Mean Ruff Failed

The correct diagnostic sequence was:

```text
Ruff reported I001
        ↓
run Ruff --fix
        ↓
file is still untracked
        ↓
ordinary git diff shows nothing
```

The absence of Git diff output was explained by Git state, not Ruff behaviour.

---

# 17. How to Check the Git State

A useful command is:

```powershell
git status --short
```

For a new file it may show:

```text
?? tests/unit/evaluation/test_factual.py
```

The:

```text
??
```

means:

> Git sees the file, but it is not yet tracked.

This context explains why ordinary `git diff` behaves differently.

---

# 18. Why Tool Output Must Be Interpreted Together

This incident involved two tools:

```text
Ruff
Git
```

Ruff answers:

> Does this Python file satisfy configured lint rules?

Git answers:

> How does this tracked repository content differ from Git's known states?

They do not answer the same question.

Therefore:

```text
git diff shows nothing
```

does not automatically mean:

```text
Ruff changed nothing
```

The file's Git state matters.

---

# 19. Verifying the Ruff Fix

After applying the targeted fix, the best validation is to rerun Ruff on that file:

```powershell
uv run ruff check "tests/unit/evaluation/test_factual.py"
```

If the relevant issue is resolved, Ruff should no longer report:

```text
I001
```

---

# 20. Full-Scope Verification

Once the immediate issue was fixed, we later ran:

```powershell
uv run ruff check src tests evals/runners
```

This expanded the check beyond a single file.

---

# What Those Paths Mean

```text
src
```

checks reusable application source code.

```text
tests
```

checks automated tests.

```text
evals/runners
```

checks the Python experiment runner code.

Together, these represented the Python code areas relevant to the current Reliora implementation.

---

# Successful Result

Eventually Ruff returned:

```text
All checks passed!
```

This meant:

> Ruff found no configured lint violations in the paths checked during that run.

It did not mean:

> The entire system is behaviourally correct.

Scope and tool purpose still matter.

---

# 21. Why We Did Not Run Ruff Against JSON and Markdown

Ruff is a Python linter.

Reliora also contains:

```text
.json
.md
.toml
```

files.

Those formats require different validation mechanisms.

For example:

```text
JSON
→ JSON parser/schema validation

Markdown
→ documentation review / Markdown tooling

TOML
→ TOML parser / project tooling
```

A quality tool should be used for the artifact type it understands.

---

# 22. Ruff vs Formatter

Linting and formatting overlap but are not identical concepts.

A formatter focuses primarily on consistent code layout.

A linter checks rules that can include:

- formatting-related issues
- unused imports
- questionable patterns
- possible errors

Ruff can also provide formatting functionality, but the command used here was:

```text
ruff check
```

with an automatic fix for a lint rule.

---

# 23. Why Import Sorting Is Usually Safe to Automate

Import sorting is generally well suited to automatic tooling because the transformation is structural and predictable.

However, imports can sometimes have side effects in Python.

Therefore, the safe assumption is not:

> Import order can never affect runtime behaviour.

Rather:

> For normal well-designed modules, repository-standard import sorting is an appropriate automated transformation, but tests should still be rerun afterward.

---

# 24. Import Side Effects

Python executes module-level code when modules are imported.

Poorly designed modules may therefore depend accidentally on import ordering.

For example:

```text
import module_a
→ mutates global state

import module_b
→ assumes that mutation happened
```

This creates fragile code.

Good software should avoid unnecessary import-time side effects.

---

# 25. Why the Reliora Fix Was Low Risk

The Reliora import-order correction involved normal evaluator/test/runner modules without intentional ordering-dependent side effects.

Therefore, using Ruff's standard import sorting was appropriate.

We still verified the project afterward with the quality suite.

---

# 26. Quality Gate Sequence

The completed Reliora evaluation work was checked with:

```text
pytest
Ruff
mypy
Git staged whitespace checks
evidence hash verification
```

Each step answered a different question.

---

## Pytest

```text
Does the tested software behaviour work?
```

Final result:

```text
19 passed
```

---

## Ruff

```text
Does the Python code satisfy configured lint rules?
```

Final result:

```text
All checks passed!
```

---

## Mypy

```text
Do the checked typed interfaces remain consistent?
```

Final result:

```text
Success: no issues found in 8 source files
```

---

# 27. Why One Passing Tool Does Not Cancel Another Failure

Suppose:

```text
pytest
→ pass

Ruff
→ fail
```

We should not say:

> Tests passed, so the Ruff failure does not matter.

Likewise:

```text
Ruff
→ pass

pytest
→ fail
```

does not mean the behaviour is correct.

The tools evaluate different properties.

---

# 28. Warning vs Lint Violation vs Test Failure

Reliora encountered several categories of development feedback.

### Warning

Example:

```text
CRLF will be replaced by LF
```

The command continued.

### Ruff Lint Violation

Example:

```text
I001 Import block is un-sorted or un-formatted
```

The code violated a configured static rule.

### Test Failure

Example:

```text
2 failed, 8 passed
```

Executed test expectations were not met.

### Type-Check Failure

Would indicate:

```text
typed interface inconsistency
```

These should not be treated as interchangeable.

---

# 29. Why We Keep Linting in the Development Workflow

Without linting, code-quality inconsistencies accumulate gradually.

For example:

```text
file A
→ one import style

file B
→ another import style

file C
→ unused imports

file D
→ suspicious pattern
```

Automated checks provide one consistent standard.

This becomes increasingly valuable as:

- repository size increases
- multiple engineers contribute
- CI/CD is introduced
- code review volume grows

---

# 30. Why This Matters for CI/CD

Later, Reliora's GitHub Actions pipeline can run commands such as:

```powershell
uv run ruff check src tests evals/runners
```

automatically.

Conceptually:

```text
developer proposes code
        ↓
CI runner
        ↓
Ruff
        ↓
pass / fail
```

This prevents code that violates agreed quality rules from silently entering protected branches.

---

# 31. Local Checks Before CI

Even when CI exists, local checks are useful.

A good sequence is:

```text
write code
        ↓
run local tests/lint/type checks
        ↓
commit
        ↓
push
        ↓
CI independently verifies again
```

Local checks provide fast feedback.

CI provides an independent standardized environment.

---

# 32. Why Ruff Is Useful in AI/ML Repositories

AI/ML projects sometimes focus heavily on model behaviour while neglecting ordinary software quality.

But production AI systems still contain substantial Python code for:

- orchestration
- evaluation
- preprocessing
- APIs
- tool calls
- data validation
- observability
- deployment logic

Those components benefit from the same software-engineering controls as other systems.

---

# 33. AI Quality Does Not Replace Code Quality

A model can produce high-quality outputs while its surrounding application code contains:

- unused imports
- inconsistent patterns
- brittle structure
- type problems
- insufficient tests

Production AI engineering needs both:

```text
AI evaluation
+
software engineering quality
```

Reliora deliberately treats these as separate concerns.

---

# 34. Why `--fix` Should Be Used Deliberately

Automatic fixing is useful when:

- the rule is understood
- the change is mechanically safe
- the scope is controlled
- results will be verified

It is less appropriate to blindly run broad fixes without inspection.

A useful pattern is:

```text
Understand
→ Fix
→ Inspect
→ Verify
```

not:

```text
Auto-fix everything
→ hope
```

---

# 35. Commands From This Incident

## Check a Specific File

```powershell
uv run ruff check "tests/unit/evaluation/test_factual.py"
```

### Purpose

Inspect one Python file for configured Ruff violations.

### Side Effect

None when used without `--fix`.

---

## Fix a Specific File

```powershell
uv run ruff check "tests/unit/evaluation/test_factual.py" --fix
```

### Purpose

Check the file and automatically correct supported violations such as import ordering.

### Side Effect

May modify the file on disk.

---

## Check the Relevant Python Repository Areas

```powershell
uv run ruff check src tests evals/runners
```

### Purpose

Run Ruff against Reliora's source code, tests, and Python evaluation runners.

### Successful Output

```text
All checks passed!
```

---

## Inspect Repository Status

```powershell
git status --short
```

### Purpose

Determine whether the affected file is tracked, modified, staged, or untracked.

### Example

```text
?? tests/unit/evaluation/test_factual.py
```

means the file is untracked.

---

# 36. A Better Troubleshooting Sequence

When Ruff reports a fixable violation:

```text
Read the rule ID and message
        ↓
understand the issue
        ↓
identify affected file
        ↓
decide whether automatic fixing is appropriate
        ↓
run targeted --fix
        ↓
inspect file / Git status
        ↓
rerun Ruff
        ↓
rerun relevant tests
        ↓
run broader quality gate
```

---

# 37. What Not to Do

Avoid automatically responding to Ruff failures by:

- disabling the rule without understanding it
- adding blanket `noqa` comments
- ignoring the failure because tests pass
- running repository-wide `--fix` without reviewing scope
- manually rearranging code randomly
- assuming no Git diff means no file changed

Each of those can hide the real lesson.

---

# 38. Suppressing Ruff Rules

Python linters often support ways to suppress a rule.

For example, code may sometimes use:

```text
# noqa
```

or configuration exclusions.

Suppression can be appropriate when a rule genuinely does not fit a justified case.

It should not be the default response to an ordinary correctable violation.

For Reliora's `I001` issue, there was no reason to suppress the rule.

The imports simply needed to be organized correctly.

---

# 39. Why Repository Standards Matter

The goal is not:

> Ruff personally prefers this import order.

The goal is:

> The repository has one automated standard that developers and CI can enforce consistently.

Automation removes subjective arguments over routine formatting issues and leaves code review focused on:

- correctness
- architecture
- security
- reliability
- maintainability

---

# 40. What This Incident Taught About Git

The Ruff fix unexpectedly reinforced another Git lesson:

```text
untracked file
→ no committed baseline
→ ordinary git diff may show nothing
```

This shows why troubleshooting frequently crosses tool boundaries.

Understanding Python tooling alone was not enough.

Understanding Git state helped explain the apparent inconsistency.

---

# 41. Important Lessons

1. Ruff performs static Python linting and code-quality checks.
2. `I001` means an import block is not sorted or formatted according to the configured rule.
3. Import groups commonly separate standard-library, third-party, and local imports.
4. A lint failure is different from a behavioural test failure.
5. `ruff check` is read-only unless a fixing option such as `--fix` is supplied.
6. `--fix` may modify files and should therefore be used deliberately.
7. Targeted fixes reduce the blast radius of automated changes.
8. Automatic fixes should still be inspected and verified.
9. Ordinary `git diff` may show nothing for a completely untracked file.
10. Git status provides necessary context for interpreting diff behaviour.
11. Pytest, Ruff, and mypy verify different properties.
12. Passing tests do not make lint failures irrelevant.
13. Import sorting is usually appropriate for automation, but tests should still be rerun.
14. Local quality gates prepare the project for future CI/CD enforcement.
15. Production AI systems require conventional software quality controls in addition to model evaluation.

---

## Interview Explanation

> During Reliora's evaluation implementation, Ruff flagged `I001` import-order violations in newly created test and runner code. I used targeted `ruff check --fix` commands rather than applying an uncontrolled repository-wide rewrite, then reran linting and the behavioural tests. One useful debugging detail was that `git diff` initially showed nothing because the corrected test file was still untracked, which reinforced the need to interpret tool output together with Git state. The final evaluation bundle passed pytest, Ruff, and mypy before it was committed.