# pytest, Ruff, and mypy

## Why This Lesson Exists

During Reliora development, I repeatedly used:

```powershell
uv run pytest -q
uv run ruff check src tests evals/runners
uv run mypy src evals/runners
```

These tools perform different jobs.

They are complementary engineering quality controls rather than substitutes for one another.

---

## The Basic Mental Model

```text
pytest
"Does the code behave the way we expect?"

Ruff
"Are there linting, import, formatting, or common code-quality problems?"

mypy
"Are Python type contracts being used consistently?"
```

Passing one does not guarantee that the others will pass.

---

# pytest

## What Is pytest?

`pytest` is an automated testing framework for Python.

It runs test functions that verify whether software behaves the way we expect.

A test usually represents a rule such as:

> Given this input, the function should produce this result.

or:

> Given this invalid behaviour, the evaluator should reject it.

---

## Command We Used

```powershell
uv run pytest -q
```

### What `uv run` Means

`uv run` executes the command using Reliora's controlled Python environment.

This helps ensure the tests use the correct Python version and installed dependencies.

### What `pytest` Means

Run the project's Python tests.

### What `-q` Means

`-q` means:

```text
quiet
```

It reduces unnecessary terminal output and gives a more compact summary.

---

## Example Successful Output

During Reliora development, we eventually received:

```text
...................                                     [100%]
19 passed in 0.35s
```

Each dot represents a test that passed.

The most important part is:

```text
19 passed
```

That means all 19 discovered tests completed successfully.

---

## What a Passing Test Suite Does Not Mean

A result such as:

```text
19 passed
```

does **not** mean:

> Reliora is completely bug-free.

It only means:

> Every currently defined test passed.

If an important behaviour has no test, pytest cannot verify it.

This is why test design matters as much as running the tests.

---

## What We Tested in Reliora

During the evaluation-foundation work, tests covered behaviours such as:

- allowing exactly one missing bug field to be requested
- rejecting multiple missing bug fields in one turn
- detecting internal route-name leakage
- detecting `<thinking>` markers
- detecting missing required FAQ facts
- reproducing known Stage-1 evaluation failures
- distinguishing reproduced defects from evaluator mismatches
- preserving non-replayable evidence as documented rather than inventing data

These tests helped turn requirements into executable checks.

---

# A Real pytest Failure We Encountered

## The Original Design

Reliora's shared evaluation result originally used:

```text
invariant_id
```

This made sense while every deterministic evaluator belonged to the behavioural invariant family:

```text
INV-004
INV-012
```

Later, we introduced a factual-completeness evaluator:

```text
FACT-001
```

At that point, the name:

```text
invariant_id
```

became too narrow.

Not every evaluation result was now a behavioural invariant.

---

## The Refactor

We changed the shared field to:

```text
evaluation_id
```

This allowed the same result contract to represent:

```text
INV-*    Behavioural evaluations
FACT-*   Factual evaluations
```

This was a design improvement because the shared model became more general without losing the distinction between evaluation families.

---

## What Broke

The application code had been changed to:

```python
result.evaluation_id
```

but two existing tests still contained:

```python
result.invariant_id
```

When pytest ran, we received:

```text
2 failed, 8 passed
```

with an error similar to:

```text
AttributeError:
'EvaluationFinding' object has no attribute 'invariant_id'
```

---

## What the Error Meant in Simple English

The test was asking:

> Give me the value stored in `invariant_id`.

But the object no longer had a field with that name.

The field had been renamed to:

```text
evaluation_id
```

The test code was stale.

---

## Why This Failure Was Useful

The failure showed that the refactor had not been applied consistently.

The source code and tests were temporarily using different versions of the same interface.

This is exactly the kind of problem automated tests are useful for detecting.

We changed the stale test assertions from:

```python
result.invariant_id
```

to:

```python
result.evaluation_id
```

After rerunning pytest:

```text
10 passed
```

Later, after additional factual and reproduction tests were added:

```text
19 passed
```

---

## Refactoring Lesson

Changing one shared interface can affect many parts of a repository.

Possible consumers include:

```text
source code
tests
datasets
experiment runners
schemas
documentation
other modules
```

A successful refactor is not complete until all relevant consumers have been updated.

---

# Ruff

## What Is Ruff?

Ruff is a fast Python linter and code-quality checker.

It analyzes Python files without needing to execute the application.

Ruff can detect problems such as:

- unused imports
- incorrect import ordering
- some formatting issues
- suspicious code patterns
- common Python quality problems

---

## Command We Used

```powershell
uv run ruff check src tests evals/runners
```

### What This Command Means

Run Ruff using Reliora's project environment and inspect Python code inside:

```text
src
tests
evals/runners
```

---

## Why We Checked These Directories

### `src`

Contains reusable Reliora application code.

Example:

```text
src/reliora/evaluation/factual.py
```

### `tests`

Contains automated test code.

Example:

```text
tests/unit/evaluation/test_factual.py
```

### `evals/runners`

Contains executable evaluation experiment scripts.

Example:

```text
evals/runners/run_stage1_reproduction.py
```

The runner was outside `src`, so we explicitly included that directory in the Ruff command.

---

# A Real Ruff Warning We Encountered

Ruff reported:

```text
I001 Import block is un-sorted or un-formatted
```

for:

```text
tests/unit/evaluation/test_factual.py
```

---

## What `I001` Meant

This was not a runtime failure.

It meant:

> The import section of the Python file does not follow the repository's expected import formatting or ordering.

The application logic could still work.

However, the file did not satisfy the project's code-quality rules.

---

## Ruff Told Us It Was Fixable

Ruff reported that the problem could be corrected using:

```text
--fix
```

We ran:

```powershell
uv run ruff check "tests\unit\evaluation\test_factual.py" --fix
```

---

## What `--fix` Means

Without `--fix`:

```powershell
ruff check
```

reports problems.

With:

```text
--fix
```

Ruff is allowed to modify the file automatically for issues it knows how to correct safely.

This means `--fix` has a side effect.

It can change source files.

---

## Why We Limited the Automatic Fix

Instead of immediately running a broad command such as:

```powershell
uv run ruff check . --fix
```

we targeted the exact file that Ruff identified.

This reduced the scope of the automated change.

That is a useful engineering habit:

> When allowing a tool to modify files automatically, keep the scope as narrow as practical and inspect the result.

---

## Result After the Fix

After correcting the import block, Ruff returned:

```text
All checks passed!
```

Later, once the reproduction runner had also been added, the final check was:

```powershell
uv run ruff check src tests evals/runners
```

with:

```text
All checks passed!
```

---

# A Git Detail We Learned During the Ruff Fix

After Ruff fixed the new test file, we tried:

```powershell
git --no-pager diff -- "tests/unit/evaluation/test_factual.py"
```

Git displayed nothing.

At first this could look like:

> Ruff did not change anything.

But that interpretation would have been wrong.

The reason was that the file was still:

```text
untracked
```

Git had no earlier committed version of that file to compare against.

Ordinary `git diff` works primarily by comparing tracked versions.

This demonstrated why understanding Git state is important when interpreting tool output.

---

# mypy

## What Is mypy?

`mypy` is a static type checker for Python.

Python is dynamically typed at runtime, but code can still declare expected types using type annotations.

For example:

```python
def normalize_text_bytes_for_hashing(raw_bytes: bytes) -> bytes:
    ...
```

This says:

```text
Input:
bytes

Output:
bytes
```

mypy checks whether the program uses those declared contracts consistently.

---

## Command We Used

```powershell
uv run mypy src evals/runners
```

The final output was:

```text
Success: no issues found in 8 source files
```

---

## What That Result Means

It means:

> mypy did not detect type inconsistencies in the files it checked.

It does not mean:

> The program is guaranteed to behave correctly.

Type correctness and behavioural correctness are different properties.

---

# Why mypy Is Different From pytest

Consider:

```python
def add(a: int, b: int) -> int:
    return a - b
```

The declared types are valid:

```text
int + int → int
```

The function accepts integers and returns an integer.

mypy may therefore find no type error.

But the implementation is behaviourally wrong if the function is supposed to perform addition.

A pytest test such as:

```python
assert add(2, 3) == 5
```

would fail.

This demonstrates:

```text
type correctness
!=
behavioural correctness
```

---

# Why Ruff Is Also Different

Ruff may inspect the same function and find:

- no import problem
- no formatting problem
- no lint problem

Yet the mathematical behaviour can still be wrong.

Therefore:

```text
clean linting
!=
correct business logic
```

---

# Why We Use All Three Tools

Each tool protects a different property.

```text
pytest
↓
Behaviour

Ruff
↓
Code quality and linting

mypy
↓
Type consistency
```

No single tool replaces the others.

---

## Reliora Quality-Gate Sequence

During the Stage-1 reproduction work, our final validation sequence included:

```text
Code changes
    ↓
pytest
    ↓
Ruff
    ↓
mypy
    ↓
Git staged checks
    ↓
commit
```

Before committing, the results were:

```text
pytest
→ 19 passed

Ruff
→ All checks passed!

mypy
→ Success: no issues found in 8 source files

git diff --cached --check
→ no output
```

Only after those checks were clean did we create the Git commit.

---

# What Is a Quality Gate?

A quality gate is a condition that must be satisfied before software is allowed to move to another stage.

For example:

```text
Developer changes code
        ↓
Tests pass?
        ↓
Lint clean?
        ↓
Types valid?
        ↓
Staged changes clean?
        ↓
Commit
```

Later, these checks can be automated through CI/CD.

For example, a GitHub pull request could be blocked if:

```text
pytest fails
or
Ruff fails
or
mypy fails
```

This turns local engineering discipline into automated release discipline.

---

## Why This Matters for AI Systems

An AI application has more failure surfaces than simply:

> Did the model produce an answer?

Reliora needs to consider:

```text
AI behaviour
software correctness
code quality
type safety
evaluation reliability
security
release safety
evidence
```

A model response can look good while software around it is broken.

Likewise, perfectly formatted Python code can still allow unsafe agent behaviour.

Multiple quality controls are therefore necessary.

---

## Warning, Error, or Failure?

It is useful to distinguish these terms.

### Warning

Something may require attention, but execution may continue.

Example:

```text
CRLF will be replaced by LF
```

### Lint Error

The code violates a configured static quality rule.

Example:

```text
I001 Import block is un-sorted or un-formatted
```

### Test Failure

Actual behaviour did not satisfy a test assertion.

Example:

```text
2 failed, 8 passed
```

### Type-Checking Failure

Declared type relationships are inconsistent.

mypy would report the specific type error.

These categories should not automatically be treated as equivalent.

The correct response depends on what the tool is reporting.

---

## Important Lessons

1. A passing test suite only proves what the tests actually cover.
2. Linting is not the same as behavioural testing.
3. Type checking is not the same as behavioural correctness.
4. Refactors can break tests even when the underlying behavioural logic still works.
5. Automated fixes should be scoped and inspected.
6. Quality checks are strongest when several independent controls are combined.

---

## Interview Explanation

> I use `pytest` for behavioural regression testing, Ruff for fast linting and common code-quality issues, and `mypy` for static type validation. I treat them as complementary controls because behavioural correctness, code cleanliness, and type consistency are different properties. In Reliora, I ran all three as part of the pre-commit quality gate before preserving experimental evaluation evidence.