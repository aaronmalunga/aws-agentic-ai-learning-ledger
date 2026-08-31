# Pytest Contract Refactor Failure

## Why This Lesson Exists

While expanding Reliora's deterministic evaluation system, we changed a shared result field from:

```text
invariant_id
```

to:

```text
evaluation_id
```

The change was conceptually correct.

Originally, Reliora's deterministic checks were all behavioural invariants such as:

```text
INV-004
INV-012
```

Later, the system introduced a factual evaluation:

```text
FACT-001
```

The name:

```text
invariant_id
```

was therefore no longer general enough.

We changed the shared field to:

```text
evaluation_id
```

After the refactor, pytest reported:

```text
2 failed, 8 passed
```

The failures were caused by tests that still referenced the old field name.

This lesson explains:

- why the refactor was necessary
- what a shared software contract is
- why changing a field can affect multiple files
- how pytest exposed the incomplete migration
- why the test failure was useful
- how we diagnosed the exact problem
- why we fixed consumers instead of reverting the design
- how to perform safer refactors in future

---

# 1. The Original Shared Model

Reliora's deterministic evaluators return a common result object.

Conceptually:

```python
EvaluationFinding(
    invariant_id="INV-004",
    passed=False,
    message="...",
    evidence={...},
)
```

The field:

```text
invariant_id
```

made sense when every evaluator represented a behavioural invariant.

---

# 2. What Is a Behavioural Invariant?

An invariant is a condition that should remain true when the system behaves correctly.

For example:

```text
INV-004
```

requires the bug-report workflow to ask for only one missing required field at a time.

Another example:

```text
INV-012
```

requires customer-visible output not to expose prohibited internal information.

Conceptually:

```text
Correct system behaviour
        ↓
certain conditions must remain true
        ↓
behavioural invariants
```

---

# 3. Why `invariant_id` Became Too Narrow

Reliora later added:

```text
FACT-001
```

This does not describe the same kind of check.

`FACT-001` evaluates whether required factual information is present in a response.

For example, a return-policy answer may need to preserve:

```text
30-day window
unused condition
original packaging condition
defective-item exception
```

This is a factual-completeness evaluation.

It is not naturally described as:

```text
an invariant ID
```

---

# 4. The Better Shared Name

We changed:

```text
invariant_id
```

to:

```text
evaluation_id
```

The new field can represent multiple evaluation families.

For example:

```text
INV-004
FACT-001
```

Both are:

```text
evaluation IDs
```

but only one family represents behavioural invariants.

---

# Why Naming Matters

Naming is not merely cosmetic.

A field name communicates the intended domain model.

Consider:

```python
finding.invariant_id
```

This implies:

> Every finding must represent an invariant.

Compare:

```python
finding.evaluation_id
```

This means:

> Every finding represents some identifiable evaluation.

The second contract is more general and more accurate.

---

# 5. The Updated Shared Model

The shared model became conceptually:

```python
class EvaluationFinding(BaseModel):
    evaluation_id: str
    passed: bool
    message: str
    evidence: dict[str, object]
```

The ID validation supports forms such as:

```text
INV-004
FACT-001
```

---

# 6. What Is a Shared Contract?

A shared contract is a structure or interface that multiple parts of a system depend on.

For example:

```text
EvaluationFinding
```

is used by:

- behavioural evaluators
- factual evaluators
- reproduction logic
- unit tests
- experiment runners
- potentially future reporting systems

A useful mental model is:

```text
                 ┌── bug evaluator
                 │
                 ├── leakage evaluator
Shared contract ─┼── factual evaluator
                 │
                 ├── reproduction logic
                 │
                 └── tests
```

Changing the contract can affect every consumer.

---

# 7. Why Shared-Contract Refactors Are High-Leverage

Suppose one local variable changes:

```python
response_text
```

to:

```python
text
```

inside a single function.

The effect may remain local.

But changing:

```text
EvaluationFinding.invariant_id
```

affects every place that reads or constructs that model.

This creates a larger blast radius.

---

# What Is Blast Radius?

In software engineering, blast radius describes how much of a system may be affected by a change or failure.

For this refactor:

```text
shared model change
        ↓
multiple evaluators
        ↓
tests
        ↓
reproduction pipeline
```

The blast radius was larger than a single-file rename.

---

# 8. The Refactor Was Applied

The model and relevant source code were updated to use:

```text
evaluation_id
```

However, not every consumer was updated immediately.

This created an inconsistent repository state.

Conceptually:

```text
New model contract
evaluation_id
        ↓
some code updated
        ↓
some tests still expect invariant_id
```

---

# 9. Running Pytest Exposed the Problem

We ran:

```powershell
uv run pytest -q
```

The result included:

```text
2 failed, 8 passed
```

This was valuable information.

Eight tests still succeeded.

Two tests had become incompatible with the new contract.

---

# Breaking Down the Command

## `uv run`

Runs the command using Reliora's managed project environment.

This helps ensure the correct:

- Python version
- dependencies
- project package

are used.

---

## `pytest`

Runs the Python test suite.

---

## `-q`

Means:

```text
quiet
```

It reduces normal pytest output while still displaying important failures and the final summary.

---

# 10. The Failure

The failing tests attempted to access:

```python
result.invariant_id
```

but the model now exposed:

```python
result.evaluation_id
```

The failure was similar to:

```text
AttributeError:
'EvaluationFinding' object has no attribute 'invariant_id'
```

---

# What Is an `AttributeError`?

In Python, an object can expose attributes.

For example:

```python
finding.evaluation_id
```

If code asks for an attribute that does not exist:

```python
finding.invariant_id
```

Python can raise:

```text
AttributeError
```

Conceptually:

```text
object exists
        ↓
requested attribute does not
        ↓
AttributeError
```

---

# 11. Why This Was Not a Pytest Problem

Pytest was functioning correctly.

It was reporting an inconsistency in our code.

This distinction matters.

The problem was not:

```text
pytest is broken
```

The problem was:

```text
tests still depend on the previous API contract
```

Pytest made that inconsistency visible.

---

# 12. Why This Was Not a Reason to Revert the Refactor

When tests fail immediately after a change, one possible reaction is:

> Undo the change.

That is not always correct.

The right question is:

> Is the new design wrong, or are some consumers still using the old design?

In this case, the new name:

```text
evaluation_id
```

was more accurate because the model now represented:

```text
INV-*
FACT-*
```

Therefore, reverting to:

```text
invariant_id
```

would preserve a misleading domain model merely to make stale tests pass.

---

# 13. The Correct Fix

We updated the stale tests.

Conceptually:

```python
assert result.invariant_id == "INV-004"
```

became:

```python
assert result.evaluation_id == "INV-004"
```

The assertions still tested the same semantic requirement.

Only the shared interface had changed.

---

# 14. Why Tests Are Consumers Too

It is easy to think:

```text
application code
→ real code

tests
→ separate
```

But tests are consumers of application interfaces.

If a public or shared contract changes, tests that call that contract must also be reviewed.

A useful model is:

```text
Production code
      ↓
Shared API/contract
      ↑
Tests
```

Tests are part of the dependency graph.

---

# 15. Rerunning Pytest

After correcting the stale references, we reran:

```powershell
uv run pytest -q
```

The affected suite then passed.

As additional evaluation components were completed, the full Reliora unit suite eventually reached:

```text
19 passed
```

---

# Why Rerunning Matters

Changing code because we believe it fixes a failure is not enough.

We need to verify the original failure condition is gone.

The loop is:

```text
test fails
    ↓
diagnose
    ↓
change code
    ↓
rerun test
    ↓
verify result
```

Without the final step, the fix is only an assumption.

---

# 16. Why the Failure Was Useful

The temporary:

```text
2 failed, 8 passed
```

result was not wasted work.

It proved the tests were capable of detecting an inconsistent refactor.

If all tests had passed while stale references existed, that would have suggested the affected code paths were not actually covered.

A failing test during a legitimate refactor can therefore provide useful evidence that tests are exercising the interface.

---

# 17. Tests as Refactoring Safety Nets

One major purpose of automated tests is enabling controlled change.

Without tests, we could rename:

```text
invariant_id
```

to:

```text
evaluation_id
```

and accidentally leave incompatible consumers unnoticed.

With tests:

```text
change contract
        ↓
run suite
        ↓
stale dependency detected
```

This reduces silent regression risk.

---

# 18. What Is a Regression?

A regression occurs when a change causes previously working behaviour to stop working.

For example:

```text
before change
tests can access result field

after change
tests crash on old field
```

The field migration caused a temporary compatibility regression in test consumers.

The test suite surfaced it before commit.

---

# 19. Why We Run Tests Before Committing

If we had committed immediately after editing the model, we could have preserved an internally inconsistent repository state.

Instead:

```text
modify contract
        ↓
run pytest
        ↓
detect stale consumers
        ↓
fix them
        ↓
rerun pytest
        ↓
commit only coherent state
```

This is one reason testing belongs before the Git commit boundary.

---

# 20. Refactoring vs Feature Development

This change was partly a refactor.

A refactor changes internal structure or representation while preserving intended system behaviour.

Here:

```text
old concept:
every finding is an invariant

new concept:
a finding can belong to multiple evaluation families
```

The rename also supported a broader domain model.

So it was not merely stylistic.

It enabled the factual evaluation extension cleanly.

---

# 21. Why We Did Not Keep Both Fields

One possible compatibility approach would have been to keep:

```text
invariant_id
```

and add:

```text
evaluation_id
```

But that could create ambiguity.

Questions would arise:

```text
Which field is authoritative?

Should FACT-001 populate invariant_id?

Can the values differ?

When can invariant_id be null?
```

For an early-stage internal API with controlled consumers, a clean migration was simpler.

---

# 22. When Backward Compatibility Would Matter More

In a public production API, changing a field name can be much more serious.

For example, suppose external customers receive:

```json
{
  "invariant_id": "INV-004"
}
```

and we suddenly change it to:

```json
{
  "evaluation_id": "INV-004"
}
```

Existing clients could break.

In that situation, we might need:

- API versioning
- migration periods
- aliases
- deprecation notices
- compatibility adapters

Reliora's evaluator model was still an internal project interface, so the controlled rename was acceptable.

---

# 23. Internal vs External Contracts

A useful distinction is:

```text
Internal contract
→ controlled by one codebase/team

External contract
→ consumed outside the immediate system
```

Breaking internal contracts can still be disruptive.

Breaking external contracts can impact unknown consumers.

The amount of migration planning should reflect the contract boundary.

---

# 24. Search Before Refactoring

A safer field rename can begin by searching the repository for:

```text
invariant_id
```

This gives an inventory of consumers.

For example, VS Code's repository search can show:

```text
models.py
bug_workflow.py
leakage.py
test_bug_workflow.py
test_leakage.py
reproduction.py
```

Then each occurrence can be classified.

---

# What Should Be Reviewed?

Not every text occurrence should automatically be replaced.

Some may be:

```text
code
tests
documentation
historical evidence
comments
dataset fields
```

A blind global replacement could incorrectly modify historical documentation where the original name is intentionally being discussed.

---

# 25. Semantic Refactor vs Text Replacement

This is an important distinction.

A text replacement asks:

```text
Where does this string appear?
```

A semantic refactor asks:

```text
Which concepts and consumers must change because the model changed?
```

The second is safer.

For example, documentation saying:

> The old field was `invariant_id`.

should not necessarily become:

> The old field was `evaluation_id`.

That would rewrite history incorrectly.

---

# 26. Why the New Regex Also Changed

The generalized model needed to accept both:

```text
INV-004
FACT-001
```

The validation pattern therefore became conceptually:

```regex
^(?:INV|FACT)-\d{3}$
```

---

# Breaking Down the Pattern

```text
^
```

means:

> Start of the string.

```text
(?:INV|FACT)
```

means:

> Match either `INV` or `FACT`.

```text
-
```

matches the literal hyphen.

```text
\d{3}
```

means:

> Exactly three digits.

```text
$
```

means:

> End of the string.

So valid examples include:

```text
INV-004
FACT-001
```

while malformed examples should be rejected.

---

# 27. Why Validation Belongs in the Shared Model

If every evaluator independently validates identifiers, behaviour can drift.

For example:

```text
Evaluator A accepts INV-4

Evaluator B requires INV-004

Evaluator C accepts arbitrary strings
```

Putting validation in the shared model creates one contract.

Conceptually:

```text
All evaluators
      ↓
EvaluationFinding
      ↓
one ID format
```

---

# 28. Contract Evolution Should Be Intentional

The migration can be summarized as:

```text
Original domain
behavioural invariants only
        ↓
new requirement
factual evaluation added
        ↓
shared model no longer general enough
        ↓
rename invariant_id → evaluation_id
        ↓
update ID validation
        ↓
update consumers
        ↓
run tests
        ↓
detect stale references
        ↓
fix and verify
```

This is controlled contract evolution.

---

# 29. How I Would Perform This Refactor Next Time

A stronger sequence would be:

```text
1. Identify why the domain model must change
2. Search all consumers of the old field
3. Update the shared model
4. Update evaluator constructors
5. Update reproduction logic
6. Update unit tests
7. Update active documentation
8. Run pytest
9. Run Ruff
10. Run mypy
11. inspect Git diff
12. commit only after all checks pass
```

This reduces the chance of temporarily forgetting consumers.

---

# 30. Why Mypy Can Also Help

Pytest catches problems when executed code paths run.

Static type checking can sometimes catch incompatible attribute access without executing the program.

For example, if mypy understands the type of:

```python
result
```

as:

```text
EvaluationFinding
```

then access to a removed attribute may also be detected statically.

This is one reason Reliora uses both:

```text
pytest
mypy
```

They catch different classes of defects.

---

# Pytest vs Mypy Here

```text
pytest
→ Does executed behaviour work?

mypy
→ Are typed interfaces used consistently?
```

Neither completely replaces the other.

---

# 31. Ruff's Role Is Different

Ruff would not normally tell us:

> `invariant_id` is the wrong domain field.

Its role is primarily linting and code-quality checks.

This incident demonstrates why the three tools are complementary:

```text
pytest
→ behaviour

Ruff
→ code-quality/linting

mypy
→ type/interface consistency
```

---

# 32. A Passing Test Suite Does Not Prove Everything

After the refactor, tests passing meant:

> The behaviours represented by those tests were currently consistent with the new contract.

It did not prove:

```text
every possible evaluator works
every AI response is correct
production reliability is guaranteed
```

Test claims should stay within the scope of the test suite.

---

# 33. Why We Preserve Failure History in the Learning Ledger

The final repository state contains:

```text
19 passed
```

Someone reading only the final Git state might never know that the intermediate refactor produced:

```text
2 failed, 8 passed
```

But that failure taught important lessons about:

- shared contracts
- incomplete migrations
- stale tests
- refactor blast radius
- verification loops

The Learning Ledger preserves that engineering reasoning separately from production code.

---

# 34. Failure Is Evidence

A failed test can provide evidence about the system.

In this case, it showed:

```text
the old interface was still actively referenced
```

That evidence helped locate the incomplete migration.

The goal is not to avoid seeing red test output.

The goal is to understand why it is red and return the system to a justified passing state.

---

# 35. Useful Questions When a Refactor Breaks Tests

When tests fail immediately after a structural change, ask:

1. What exact contract changed?
2. Is the new contract intentional?
3. Which consumers still use the old contract?
4. Is the failure behavioural or merely API compatibility?
5. Should compatibility be preserved?
6. Is this internal or externally consumed?
7. Are tests stale, or is the implementation wrong?
8. What other files may depend on the same field?
9. Can static analysis find additional consumers?
10. After fixing, does the full suite pass?

---

# 36. What Not to Do

Avoid responses such as:

```text
disable the failing tests
```

or:

```text
change assertions until they pass
```

without understanding why.

Tests are only useful if they remain tied to meaningful requirements.

---

# 37. Why We Did Not Change Expected Behaviour

The tests did not fail because Reliora had decided:

```text
INV-004 no longer matters
```

They failed because the property name changed.

Therefore, the semantic assertions remained.

For example:

```text
expected evaluation:
INV-004
```

stayed the same.

Only:

```text
how the ID is accessed
```

changed.

---

# 38. Repository Areas Affected by This Kind of Change

A shared evaluation contract can affect:

```text
src/reliora/evaluation/
tests/unit/evaluation/
evals/runners/
evals/datasets/
evidence/generated/
docs/
```

Not every change will touch all of them.

But each area should be considered when the schema evolves.

---

# 39. Source of Truth Matters

The shared model:

```text
EvaluationFinding
```

should be treated as the source of truth for evaluator result structure.

Tests should verify that contract.

They should not maintain a separate, contradictory idea of what an evaluation finding looks like.

---

# 40. Important Lessons

1. Shared contracts can have a larger blast radius than local code changes.
2. `invariant_id` became too narrow once factual evaluations were introduced.
3. `evaluation_id` more accurately represents multiple evaluation families.
4. Naming communicates domain meaning and is not merely cosmetic.
5. Tests are consumers of application interfaces.
6. `AttributeError` can indicate stale consumer code after an API refactor.
7. A failing test does not automatically mean the new design is wrong.
8. Reverting a correct design simply to satisfy stale tests would preserve the wrong abstraction.
9. Tests act as refactoring safety nets.
10. Repository-wide search should precede shared-contract changes.
11. Semantic refactoring is safer than blind string replacement.
12. Internal and external contracts require different compatibility strategies.
13. Pytest, mypy, and Ruff provide complementary forms of protection.
14. A passing test suite supports only the behaviours actually represented by the tests.
15. Intermediate failures are valuable engineering evidence when they are diagnosed and documented.

---

## Interview Explanation

> While extending Reliora from behavioural invariants to factual-completeness evaluation, I generalized the shared evaluator result field from `invariant_id` to `evaluation_id`. The refactor initially produced two pytest failures because some tests still referenced the old interface. Rather than reverting the broader domain model, I treated the failures as evidence of an incomplete contract migration, updated the remaining consumers, and reran the suite until the repository was consistent. It reinforced the importance of treating shared models as contracts, understanding refactor blast radius, and using automated tests as a safety net for architectural evolution.