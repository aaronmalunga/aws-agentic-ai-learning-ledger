# uv and Python Virtual Environments

## Why This Lesson Exists

While building Reliora, the terminal prompt often looked like:

```text
(reliora-ai-support-platform) PS C:\Users\Aaron\Projects\reliora-ai-support-platform>
```

The text in parentheses indicated that Reliora's Python virtual environment was active.

Later, after changing directory into the AWS Agentic AI Learning Ledger repository, the prompt still initially showed:

```text
(reliora-ai-support-platform)
```

even though the current directory had changed.

This demonstrated an important distinction:

> Changing directories does not automatically change or deactivate the active Python environment.

---

## What Is a Python Virtual Environment?

A Python virtual environment is an isolated Python environment associated with a project.

It allows a project to have its own:

- Python runtime context
- installed libraries
- dependency versions
- development tools

This helps prevent one project's dependencies from interfering with another project's dependencies.

For example:

```text
Project A
└── Pydantic 2.x

Project B
└── Different dependency requirements
```

Separate environments allow both projects to coexist safely on the same computer.

---

## Why Virtual Environments Matter

Without project isolation, Python packages may be installed globally.

That can create problems such as:

- one project upgrading a package another project depends on
- code working on one machine but failing on another
- accidentally using the wrong Python version
- difficulty reproducing the development environment later

A project-specific environment reduces these risks.

---

## Reliora's Virtual Environment

Reliora uses Python 3.12.

Its local virtual environment lives under:

```text
reliora-ai-support-platform/.venv/
```

The `.venv` directory contains locally installed Python packages and environment-specific files.

It normally should not be committed to Git because:

- it can be recreated
- it may contain thousands of files
- it can be large
- its contents depend on the operating system
- installed packages are not source code

This is why `.venv` is normally included in `.gitignore`.

---

## What Is `uv`?

`uv` is a modern Python project, package, environment, and dependency-management tool.

In Reliora, it helps manage:

- the project's Python version
- runtime dependencies
- development dependencies
- virtual-environment execution
- project metadata
- repeatable Python commands

Reliora's Python project configuration is primarily stored in:

```text
pyproject.toml
```

---

## What Is `pyproject.toml`?

`pyproject.toml` is a standard Python project configuration file.

In Reliora, it describes things such as:

- project name
- project version
- supported Python version
- runtime dependencies
- development tooling
- package build configuration

Reliora intentionally requires Python 3.12 rather than simply accepting whichever Python version happens to be installed globally.

This makes the project's runtime requirements explicit.

---

## Why We Use `uv run`

A command such as:

```powershell
uv run pytest -q
```

means:

> Run `pytest` using the Python environment and dependencies associated with this project.

Likewise:

```powershell
uv run python script.py
```

means:

> Run Python using the project's controlled environment rather than blindly relying on whichever Python installation happens to be globally available.

This makes execution more predictable.

---

## Why This Matters on My Computer

My computer has more than one Python version installed.

Reliora is intentionally configured to use Python 3.12.

Without a controlled project environment, a command such as:

```powershell
python
```

could potentially use a different system-wide Python version.

Using the project environment reduces that ambiguity.

---

## How I Know a Virtual Environment Is Active

When a virtual environment is active, PowerShell may show its name at the beginning of the prompt.

For Reliora, I saw:

```text
(reliora-ai-support-platform)
```

before:

```text
PS C:\Users\Aaron\Projects\reliora-ai-support-platform>
```

This was a visual indicator that commands were being executed with Reliora's environment active.

---

## Changing Folder Does Not Change Environment

I used:

```powershell
Set-Location "$env:USERPROFILE\Projects\aws-agentic-ai-learning-ledger"
```

to move from the Reliora repository into the Learning Ledger repository.

The PowerShell directory changed, but the terminal still showed:

```text
(reliora-ai-support-platform)
```

This happened because two separate things were involved:

```text
Current directory
```

and:

```text
Active Python environment
```

Changing one does not automatically change the other.

---

## The `deactivate` Command

We then used:

```powershell
deactivate
```

to leave Reliora's virtual environment.

### What `deactivate` Does

It stops using the currently active virtual environment in the current PowerShell session.

### What `deactivate` Does Not Do

It does not:

- delete `.venv`
- uninstall packages
- delete the project
- remove Python
- modify Git
- affect AWS resources
- remove Reliora's dependencies permanently

It only changes the current shell environment.

---

## Why We Did Not Keep Using Reliora's Environment

The AWS Agentic AI Learning Ledger is a separate repository.

Its purpose is primarily documentation, learning notes, architecture explanations, troubleshooting records, AWS service notes, and interview preparation.

It should not silently rely on Reliora's Python packages.

Otherwise this could happen:

```text
Learning Ledger
      ↓
command appears to work
      ↓
because Reliora's environment is still active
      ↓
false impression that the Learning Ledger owns those dependencies
```

That would create hidden coupling between two repositories.

If the Learning Ledger later needs Python tooling of its own, it should receive its own environment.

---

## Mental Model

```text
Windows
│
├── Global Python installation(s)
│
├── Reliora repository
│   │
│   └── .venv
│       ├── Python 3.12 environment
│       ├── Pydantic
│       ├── pytest
│       ├── Ruff
│       └── mypy
│
└── AWS Agentic AI Learning Ledger
    │
    └── currently does not need Reliora's Python environment
```

---

## Commands Used During This Lesson

### Change Directory

```powershell
Set-Location "$env:USERPROFILE\Projects\aws-agentic-ai-learning-ledger"
```

**What it does:** Changes the current PowerShell working directory.

**Why we used it:** We had finished a clean Reliora development checkpoint and wanted to work in the separate Learning Ledger repository.

**Important side effect:** It changes the current directory, but it does not activate or deactivate Python environments.

---

### Deactivate the Virtual Environment

```powershell
deactivate
```

**What it does:** Stops using the currently active Python virtual environment.

**Why we used it:** The terminal had moved into the Learning Ledger but was still using Reliora's environment.

**Expected result:** The text:

```text
(reliora-ai-support-platform)
```

disappears from the PowerShell prompt.

---

## Important Lesson

A repository directory and a Python environment are related concepts, but they are not the same thing.

Changing the:

```text
current folder
```

does not automatically change the:

```text
active Python environment
```

I should pay attention to the environment name displayed in the terminal prompt before running project-specific Python commands.

---

## Why This Matters in Professional Engineering

Separate project environments improve:

- reproducibility
- dependency isolation
- debugging
- onboarding
- CI/CD consistency
- deployment reliability

An engineer should be able to explain which environment a command is running in and why.

---

## Interview Explanation

> I use project-specific Python environments so dependency versions and interpreter requirements remain isolated and reproducible. Reliora uses `uv` with Python 3.12, which prevents the project from accidentally depending on whichever Python installation happens to be globally available.