# Commands and Scripts Learning Ledger

This section records the commands, CLI workflows, development tools, and troubleshooting patterns used while building AWS AI/ML and agentic AI projects.

The purpose is not to collect commands blindly.

Each entry should explain enough context that I can return later and understand:

```text
what the command does
why it was used
where it should be run
what important flags mean
what output to expect
what state it changes
what risks or side effects exist
what common failures look like
how to diagnose those failures
when I would use the command again
```

---

# Directory Map

```text
commands-and-scripts/
│
├── git/
├── powershell/
├── python/
├── troubleshooting/
├── uv/
└── README.md
```

Future sections may include:

```text
aws-cli/
terraform/
github-actions/
```

when those tools are actually used in project implementation.

Git does not track empty directories, so those future categories do not need placeholder files.

---

# 1. Git

Location:

```text
commands-and-scripts/git/
```

Current lessons:

```text
git-working-tree-staging-and-commits.md
pre-commit-quality-gate.md
```

Topics include:

```text
working directory
untracked files
staging
commits
HEAD
staged snapshots
status codes
diff inspection
local quality gates
```

Use this section when the question is:

> What state is Git in, and what exactly am I about to preserve in history?

---

# 2. PowerShell

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
relative and absolute paths
Get-Location
Set-Location
Get-ChildItem
Test-Path
environment variables
PowerShell providers
safe file operations
encoding
shell state
```

Use this section when the question is:

> What is the shell doing around the tools I am running?

---

# 3. Python Quality Tooling

Location:

```text
commands-and-scripts/python/
```

Current lesson:

```text
testing-linting-and-type-checking.md
```

This explains the different responsibilities of:

```text
pytest
Ruff
mypy
```

A useful mental model is:

```text
pytest
→ behavioural correctness represented by tests

Ruff
→ configured lint and code-quality checks

mypy
→ static type consistency
```

These tools complement one another.

A passing result from one does not imply that the properties checked by the others are also correct.

---

# 4. uv and Virtual Environments

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
Python virtual environments
dependency isolation
Python interpreter versions
uv
pyproject.toml
uv run
activation
deactivation
project-specific environments
```

Use this section when the question is:

> Which Python interpreter and dependency environment is the project actually using?

---

# 5. Troubleshooting

Location:

```text
commands-and-scripts/troubleshooting/
```

Current lessons:

```text
git-line-endings-and-evidence-hashes.md
git-pager-environment-error.md
powershell-utf8-file-encoding.md
pytest-contract-refactor-failure.md
ruff-import-order-i001.md
uv-module-name-mismatch.md
```

These entries preserve failures that actually occurred during project development.

They are not removed from the learning history simply because they were fixed.

---

## Troubleshooting Structure

A good troubleshooting entry should answer:

```text
What was the symptom?

What exact error or warning appeared?

Which layer actually failed?

What caused it?

How was the cause confirmed?

What was changed?

Why did that fix work?

What would happen if the issue were ignored?

How can I recognize the same problem again?
```

---

## Troubleshooting Principle

A central rule is:

> Fix the smallest layer that actually contains the fault.

Examples from Reliora include:

```text
invalid Git environment variable
→ fix the shell environment

invalid README encoding
→ fix the file encoding

uv module-name mismatch
→ fix project/package configuration

stale test consumer after model refactor
→ update the affected consumer

Ruff I001
→ fix the import block
```

We should not reinstall or redesign unrelated systems before identifying the failing layer.

---

# 6. Command Documentation Standard

When recording a terminal command, document the following.

## Command

Example:

```powershell
git status --short --untracked-files=all
```

## What it does

Explain the operation in plain language.

Example:

```text
Shows the current Git state in compact form and expands
untracked directories so individual untracked files are visible.
```

## Why it is being used now

Commands should have context.

For example:

```text
We used this before the Learning Ledger's first commit
to inspect every file Git could see before staging anything.
```

## Important arguments

For example:

```text
--short
→ compact status output

--untracked-files=all
→ show every individual untracked file
```

## Expected output

Explain what success looks like.

For example:

```text
?? file.md
```

means:

```text
the file exists but has not yet been staged
```

## Side effects

Classify whether the command is:

```text
read-only
changes shell state
changes filesystem state
changes Git state
changes cloud infrastructure
```

This should be understood before execution.

## Failure modes

Document meaningful errors or warnings and what they imply.

## Reuse

Explain when the command is useful again.

---

# 7. Read Commands vs Write Commands

A useful operational distinction is:

### Mostly read-only

Examples:

```text
Get-Location
Get-ChildItem
Get-Content
Test-Path
git status
git diff
```

These inspect state.

### State-changing

Examples:

```text
Set-Content
Rename-Item
Remove-Item
git add
git commit
terraform apply
aws ... create-*
```

These require more care because they modify something.

---

# 8. Shell State Matters

Commands execute inside an environment.

Important context can include:

```text
current directory
active Python environment
Git branch
environment variables
AWS credentials
AWS region
Terraform working directory
```

The same command may behave differently when this surrounding state changes.

---

# 9. Current Directory

Before running project-specific commands, know where the terminal is.

For example:

```text
C:\Users\Aaron\Projects\reliora-ai-support-platform
```

and:

```text
C:\Users\Aaron\Projects\aws-agentic-ai-learning-ledger
```

are separate repositories.

Running the correct command from the wrong repository can produce confusing results.

---

# 10. Environment State

Changing directories does not necessarily change:

```text
active virtual environment
environment variables
AWS authentication
```

These are separate forms of shell state.

For example:

```text
current directory
→ Learning Ledger

active Python environment
→ Reliora
```

can exist simultaneously unless the environment is explicitly deactivated.

---

# 11. Terminal Commands vs Source Code

PowerShell commands belong at the PowerShell prompt.

Example:

```powershell
uv run pytest -q
```

Python source code belongs in:

```text
.py files
```

Example:

```python
from pathlib import Path
```

Markdown belongs in:

```text
.md files
```

JSON belongs in:

```text
.json files
```

Understanding which environment owns a piece of text prevents many avoidable errors.

---

# 12. File Editing vs Terminal Execution

The terminal is generally best for:

```text
short commands
inspection
tests
Git operations
tool execution
```

VS Code is generally better for:

```text
Python source
Markdown
JSON
TOML
Terraform
large configuration files
```

Large terminal here-strings can become difficult to review and easy to paste incorrectly.

---

# 13. Quality-Gate Commands

Reliora currently uses local checks including:

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

Each checks a different property.

The relevant lesson is:

```text
commands-and-scripts/git/pre-commit-quality-gate.md
```

---

# 14. Command Results Are Evidence With Scope

For example:

```text
19 passed
```

means:

> All 19 collected tests passed in that particular run.

It does not mean:

```text
the entire system is defect-free
```

Likewise:

```text
All checks passed!
```

from Ruff means:

> Ruff found no configured lint problems within the checked scope.

It does not mean:

```text
the software is production-ready
```

Tool output must be interpreted according to what the tool actually measures.

---

# 15. Warnings Should Be Understood

A warning is not automatically:

```text
an error
```

but it should not automatically be ignored either.

For example, Git's CRLF/LF warnings became important in Reliora because exact text bytes were being used for:

```text
SHA-256 provenance hashes
```

Context determines whether a warning matters.

---

# 16. Error Categories

Different failures mean different things.

```text
Warning
→ operation may continue, but a condition deserves attention

Command error
→ command could not complete as intended

Test failure
→ executed behavioural expectation failed

Lint failure
→ configured static quality rule failed

Type failure
→ static type contract failed
```

The correct response depends on the failure category.

---

# 17. Do Not Hide Errors

Errors are useful engineering evidence.

For example:

```text
pytest failure
→ revealed stale contract consumers

Ruff I001
→ revealed import-order violation

Git hash mismatch
→ revealed working-copy vs canonical line-ending differences

Git pager failure
→ revealed shell-environment configuration problem
```

The objective is not to pretend development was error-free.

The objective is to understand why the error occurred and prevent repeated confusion.

---

# 18. Safe Command Workflow

Before a meaningful command:

```text
understand the command
        ↓
understand current context
        ↓
understand side effects
        ↓
run command
        ↓
read output
        ↓
interpret result
```

If the command fails:

```text
read exact error
        ↓
identify failing layer
        ↓
inspect actual state
        ↓
make targeted correction
        ↓
rerun original command
```

---

# 19. Future AWS CLI Material

When AWS CLI work begins, entries should capture topics such as:

```text
authentication context
region
resource identity
read vs write operations
expected AWS response
IAM permission failures
cost implications
resource cleanup
```

AWS commands deserve additional caution because they can modify real cloud resources and incur cost.

---

# 20. Future Terraform Material

When Terraform is introduced, this section should document commands such as:

```text
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply
terraform destroy
```

Each entry should explain:

```text
state implications
provider behavior
planned infrastructure changes
cost risk
destructive risk
```

especially before `apply` or `destroy`.

---

# 21. Future GitHub Actions Material

When CI/CD implementation begins, document:

```text
workflow triggers
jobs
steps
permissions
OIDC
secrets
quality gates
artifact handling
failure interpretation
```

CI should automate commands that are already understood locally rather than turn them into hidden magic.

---

# Learning Rule

A command is considered learned when I can explain:

> What it does, why I chose it, what state it depends on, what it changes, what success looks like, what failure looks like, and when I would use it again.
