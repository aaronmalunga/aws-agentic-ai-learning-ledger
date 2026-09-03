# Pytest Collection, Import Paths, and `python -m pytest`

## Why This Lesson Exists

During Reliora development, the full regression command:

```powershell
uv run pytest -q
```

failed before any tests executed.

The failure was not a behavioural test failure.

Pytest failed during test collection because repository-local routing modules could not be imported.

The repeated error was:

```text
ModuleNotFoundError: No module named 'experiments'
```

This distinction mattered because changing working application code would have been the wrong response.

---

# 1. What Test Collection Means

Before pytest can run tests, it must:

```text
discover test files
-> import test modules
-> resolve their imports
-> collect test functions
-> execute tests
```

The Reliora failure occurred during:

```text
import test modules
```

not during:

```text
execute tests
```

Therefore:

```text
collection error
!=
test assertion failure
```

---

# 2. The Repository Layout

Reliora uses an installed application package under:

```text
src/reliora/
```

but routing experimentation code also exists at repository level under:

```text
experiments/
```

Tests import modules such as:

```python
from experiments.routing.benchmark.data import Intent
```

and:

```python
import experiments.routing.scripts.run_routing_benchmark
```

The `experiments` directory does not contain:

```text
experiments/__init__.py
```

and:

```text
experiments/routing/__init__.py
```

Those files were not required because Python can represent the directory as a namespace package when the repository root is on the module search path.

---

# 3. First Diagnostic

Instead of immediately changing package structure, we tested ordinary Python import behaviour:

```powershell
uv run python -c "import experiments; print(experiments)"
```

Observed result:

```text
<module 'experiments' (namespace) from ['C:\\Users\\Aaron\\Projects\\reliora-ai-support-platform\\experiments']>
```

This was important evidence.

It proved:

```text
the experiments namespace itself was valid
```

Therefore the problem was narrower than:

```text
experiments is broken
```

---

# 4. Why We Did Not Add `__init__.py`

A tempting reaction would have been:

```text
add __init__.py everywhere
```

But the direct Python import already demonstrated that the namespace package worked.

Changing package structure before understanding the failure could have:

```text
changed import semantics unnecessarily
hidden the real tooling issue
introduced unrelated repository changes
```

The correct sequence was:

```text
diagnose first
-> change structure only if required
```

---

# 5. Checking Existing Pytest Configuration

We searched for repository configuration that might already control pytest paths:

```powershell
git grep -n "pythonpath"
git grep -n "testpaths"
git grep -n "addopts"
```

No existing configuration explained the failure.

This reduced the likelihood that we were overriding or contradicting an established pytest policy.

---

# 6. The Successful Invocation

We then ran pytest through the active Python interpreter:

```powershell
uv run python -m pytest -q
```

Observed result:

```text
171 passed
```

The full verbose invocation also collected:

```text
171 items
```

and completed successfully.

---

# 7. What `python -m pytest` Means

In:

```powershell
python -m pytest
```

the `-m` flag means:

```text
run the installed pytest module as a program
```

Instead of starting the standalone pytest executable directly, Python starts first and executes pytest inside that interpreter context.

For this repository layout, that invocation allowed the repository-local `experiments` namespace to resolve correctly.

---

# 8. Why We Standardized the `uv` Form

This also worked in the currently activated environment:

```powershell
python -m pytest
```

However, the preferred documented Reliora command is:

```powershell
uv run python -m pytest
```

because it explicitly uses the project-managed environment.

That makes the command less dependent on:

```text
which shell is active
which Python is first on PATH
which packages happen to be globally installed
```

---

# 9. Failure Classification

The original command produced:

```text
9 errors during collection
```

That did not mean:

```text
9 Reliora behaviours are broken
```

It meant pytest could not successfully import nine collected modules.

A useful diagnostic classification is:

```text
collection error
-> runner/import/environment problem may exist

test failure
-> code executed but an assertion failed

runtime exception inside a test
-> code executed and raised unexpectedly

lint failure
-> static quality rule failed

type-check failure
-> declared type relationships are inconsistent
```

These categories require different responses.

---

# 10. Why We Preserved the AWS Adapter

Before the full-suite collection issue appeared, the new DynamoDB adapter had already passed:

```text
7 AWS-specific tests
```

and the core reliability contract had passed:

```text
23 tests
```

The collection error therefore did not provide evidence that the DynamoDB implementation was defective.

We deliberately did not rewrite the working AWS adapter merely because the full-suite command failed to collect unrelated routing modules.

---

# 11. Final Regression Evidence

Once pytest was invoked through Python:

```powershell
uv run python -m pytest -q
```

the complete project produced:

```text
171 passed
```

This established that:

```text
application tests
evaluation tests
routing tests
reliability tests
AWS adapter tests
```

could all execute successfully in the same regression run.

---

# 12. Engineering Consequence

The canonical full-suite Reliora command became:

```powershell
uv run python -m pytest
```

rather than:

```powershell
uv run pytest
```

for the current repository structure.

The important lesson is not that `python -m pytest` is universally superior.

The lesson is:

> Test-runner invocation can affect Python import resolution, so collection failures should be diagnosed as environment and module-loading problems before application code is changed.

---

# 13. Troubleshooting Sequence

A reusable sequence for similar failures is:

```text
pytest collection fails
        ->
read the exact import exception
        ->
test the failing import directly with Python
        ->
inspect package/namespace structure
        ->
inspect pytest configuration
        ->
compare pytest invocation methods
        ->
run the complete regression suite
        ->
change package structure only if evidence requires it
```

---

# 14. Important Lessons

1. Pytest collection happens before behavioural assertions execute.
2. A collection error is not the same as a failed test.
3. `ModuleNotFoundError` can be caused by invocation and import-path context rather than broken application logic.
4. Namespace packages do not always require `__init__.py`.
5. Direct Python import is a useful diagnostic before changing package structure.
6. `python -m pytest` executes pytest through a specific Python interpreter.
7. `uv run python -m pytest` explicitly uses the project-controlled environment.
8. Working code should not be rewritten merely because an unrelated test runner cannot collect modules.
9. Narrow diagnostics reduce unnecessary architectural changes.
10. A final full regression run is required after resolving the tooling problem.

---

## Interview Explanation

> While adding Reliora's DynamoDB execution adapter, the first full pytest command failed with nine collection errors saying that the repository-local `experiments` module could not be found. I did not treat that as an application regression because pytest had not executed the tests yet. I verified that ordinary Python could import `experiments` successfully as a namespace package, checked that no existing pytest path configuration explained the issue, and then ran pytest through the project interpreter using `uv run python -m pytest`. That successfully executed all 171 tests. The incident reinforced the importance of classifying collection, behavioural, lint, and type failures correctly before modifying working application code.