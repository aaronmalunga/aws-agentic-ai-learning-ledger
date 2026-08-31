# uv Build Backend Module-Name Mismatch

## Why This Lesson Exists

During the initial Reliora project setup, `uv` had difficulty building or recognizing the Python package because the project name and the actual Python module directory were different.

The project was named:

```text
reliora-ai-support-platform
```

but the importable Python package was:

```text
reliora
```

The source structure looked like:

```text
src/
└── reliora/
    └── __init__.py
```

We resolved the mismatch by explicitly configuring the `uv` build backend:

```toml
[tool.uv.build-backend]
module-name = "reliora"
```

This lesson explains why that was necessary and why a project name is not always the same thing as a Python import name.

---

## The Important Distinction

There were two names involved:

### Project or Distribution Name

```text
reliora-ai-support-platform
```

### Python Module or Package Name

```text
reliora
```

These names serve different purposes.

---

# What Is the Project Name?

Reliora's `pyproject.toml` contains project metadata similar to:

```toml
[project]
name = "reliora-ai-support-platform"
version = "0.1.0"
```

The value:

```text
reliora-ai-support-platform
```

is the project's package/distribution name.

It is useful for identifying the overall software project.

A descriptive project name can contain hyphens.

---

# What Is a Python Module Name?

Python imports use module/package identifiers.

Reliora code can be imported using syntax such as:

```python
from reliora.evaluation.factual import evaluate_required_fact_completeness
```

The first part:

```text
reliora
```

corresponds to the package directory:

```text
src/reliora/
```

This is the importable Python package name.

---

# Why We Cannot Import the Project Name Directly

The distribution name contains hyphens:

```text
reliora-ai-support-platform
```

Python import syntax does not work like:

```python
import reliora-ai-support-platform
```

because the hyphen:

```text
-
```

is interpreted as the subtraction operator in Python syntax.

The importable package therefore uses a valid Python identifier:

```python
import reliora
```

---

# Distribution Name vs Import Name

A useful mental model is:

```text
Project/distribution identity
reliora-ai-support-platform
        ↓
describes the overall Python project

Importable package
reliora
        ↓
used inside Python code
```

These can be related without being identical.

---

# Reliora's Source Layout

Reliora uses a `src` layout.

Conceptually:

```text
reliora-ai-support-platform/
│
├── pyproject.toml
│
└── src/
    └── reliora/
        ├── __init__.py
        └── evaluation/
```

The package Python needs to build and import is:

```text
src/reliora/
```

not:

```text
src/reliora-ai-support-platform/
```

---

# What Is the `src` Layout?

The `src` layout means the importable Python package is placed underneath:

```text
src/
```

rather than directly at the repository root.

For example:

```text
src/
└── reliora/
```

This layout helps make package boundaries explicit.

It can also reduce accidental imports from the repository working directory that would not behave the same way after packaging.

---

# What Is `pyproject.toml` Doing Here?

`pyproject.toml` describes how the Python project should be understood and built.

Reliora originally contained configuration similar to:

```toml
[project]
name = "reliora-ai-support-platform"
version = "0.1.0"
description = "A production-engineering case study for reliable, observable, and safely controlled AI-assisted customer support on AWS."
readme = "README.md"
authors = [{ name = "Aaron Malunga" }]
requires-python = ">=3.12,<3.13"
dependencies = []
```

and:

```toml
[build-system]
requires = ["uv_build>=0.12.7,<0.13.0"]
build-backend = "uv_build"
```

The build backend needed to know which Python package under `src/` represented the actual module.

---

# What Is a Build Backend?

A Python build backend is tooling responsible for converting project source into a Python package/distribution representation.

Conceptually:

```text
source files
    ↓
build backend
    ↓
Python package/distribution
```

Reliora uses:

```text
uv_build
```

as its build backend.

---

# The Mismatch

The project metadata said:

```text
Project name:
reliora-ai-support-platform
```

while the actual importable directory was:

```text
src/reliora/
```

Without an explicit mapping, the build system could not safely assume that:

```text
reliora-ai-support-platform
```

corresponded to:

```text
reliora
```

The names were intentionally different.

---

# The Fix

We added:

```toml
[tool.uv.build-backend]
module-name = "reliora"
```

to `pyproject.toml`.

---

## What This Configuration Means

It tells the `uv` build backend:

> The Python module/package that should be built for this project is named `reliora`.

This connects:

```text
project metadata
reliora-ai-support-platform
```

to:

```text
Python package
src/reliora/
```

---

# Why We Did Not Rename the Entire Project to `reliora`

We could theoretically have made the distribution name and module name identical.

However, they represent different concerns.

The repository/project name:

```text
reliora-ai-support-platform
```

communicates the product and portfolio scope.

The Python import name:

```text
reliora
```

is concise and appropriate for application imports.

For example:

```python
from reliora.evaluation.models import EvaluationFinding
```

is cleaner than a very long package identifier.

---

# Why Explicit Configuration Is Better Than Guessing

A build tool can sometimes infer package names from project metadata.

Inference becomes less reliable when:

```text
distribution name
!=
module directory name
```

Explicit configuration removes ambiguity.

A useful engineering principle is:

> When an important relationship cannot be inferred reliably, make the relationship explicit in configuration.

---

# What Is `__init__.py`?

Reliora contains:

```text
src/reliora/__init__.py
```

This file identifies package-level behaviour and metadata.

For example:

```python
"""Reliora AI Support Platform."""

__version__ = "0.1.0"
```

The existence of the package directory and its `__init__.py` reinforces that:

```text
reliora
```

is the Python package being developed.

---

# How Python Imports Follow the Package Structure

Consider:

```python
from reliora.evaluation.models import EvaluationFinding
```

This corresponds approximately to:

```text
src/
└── reliora/
    └── evaluation/
        └── models.py
```

Breaking the import down:

```text
reliora
→ package

evaluation
→ subpackage/domain

models
→ Python module

EvaluationFinding
→ class imported from that module
```

The directory organization and import syntax reinforce each other.

---

# Why This Matters Beyond Reliora

This distinction appears frequently in real Python projects.

A project might have a distribution name such as:

```text
my-company-data-platform
```

while the Python import is:

```python
import data_platform
```

The name users see when installing or identifying the project does not necessarily have to match the Python module name exactly.

---

# Why Hyphens and Underscores Can Be Confusing

Python packaging frequently exposes names in multiple forms.

For example:

```text
Distribution:
my-awesome-project
```

might correspond to:

```text
Python package:
my_awesome_project
```

or even a shorter module name.

Hyphens are common in project/distribution names.

Python identifiers generally use:

```text
letters
numbers
underscores
```

with normal identifier restrictions.

This is why understanding which kind of name a tool expects is important.

---

# The Debugging Lesson

When a Python packaging tool cannot locate or build a module, useful questions include:

1. What is the `[project].name` value?
2. What directory actually exists under `src/`?
3. What name is used in Python imports?
4. Which build backend is being used?
5. Does the build backend expect the module name to match the project name?
6. Is there configuration for explicitly mapping the module?

This is more useful than randomly renaming folders until the tool stops complaining.

---

# Why We Changed Configuration Instead of Application Imports

Reliora's code correctly used:

```python
import reliora
```

The package directory was correctly named:

```text
src/reliora/
```

The ambiguity existed at the build-configuration layer.

Therefore, the targeted fix belonged in:

```text
pyproject.toml
```

rather than rewriting the source package.

---

# Smallest-Layer Fix Principle

This incident illustrates a troubleshooting principle:

> Fix the problem at the layer where the mismatch actually exists.

The layers were:

```text
Product name
        ↓
Python project metadata
        ↓
Build backend
        ↓
Python package directory
        ↓
Application imports
```

The application package was correct.

The mapping between project metadata and build package needed clarification.

So we fixed the build configuration.

---

# Why `pyproject.toml` Is Important

This incident demonstrated that `pyproject.toml` is more than a list of dependencies.

It also communicates important project structure.

It can tell tooling:

```text
what this project is called
which Python it supports
which dependencies it needs
how it is built
which package/module it exposes
```

This makes it one of the most important root-level files in a modern Python repository.

---

# The Relevant Reliora Configuration

The important configuration became:

```toml
[project]
name = "reliora-ai-support-platform"
version = "0.1.0"

[build-system]
requires = ["uv_build>=0.12.7,<0.13.0"]
build-backend = "uv_build"

[tool.uv.build-backend]
module-name = "reliora"
```

The important relationship is:

```text
Project:
reliora-ai-support-platform

Build backend:
uv_build

Module:
reliora
```

---

# Why Python Version Was Also Explicit

Reliora's project metadata included:

```toml
requires-python = ">=3.12,<3.13"
```

This tells tooling that Reliora expects Python:

```text
3.12.x
```

rather than arbitrary Python versions.

This was important because the development laptop had multiple Python installations.

Project configuration makes the intended runtime explicit.

---

# How This Relates to `uv run`

Once the project structure and environment are correctly configured, commands such as:

```powershell
uv run python
```

or:

```powershell
uv run pytest -q
```

can operate against the correct Reliora environment and package structure.

The build/module configuration is therefore part of making the development workflow predictable.

---

# Package Name, Repository Name, and Product Name

Another useful distinction is that a project can have several identities.

For Reliora:

```text
Product name:
Reliora

Repository:
reliora-ai-support-platform

Python project/distribution:
reliora-ai-support-platform

Python module:
reliora
```

These do not all have to be identical.

They should, however, be intentionally related and documented.

---

# Why This Matters for Repository Design

A professional repository should make these boundaries understandable.

For example:

```text
reliora-ai-support-platform/
│
├── pyproject.toml
│      ↓
│   describes Python project/build configuration
│
└── src/
    └── reliora/
           ↓
        actual importable package
```

The root repository represents the entire engineering project.

The package under `src` represents the Python software namespace.

---

# What We Learned From the Error

The important lesson was not merely:

> Add `module-name = "reliora"`.

The deeper lesson was:

> Different tools may be referring to different kinds of names.

When debugging packaging, I should ask:

```text
Is this tool asking for:

repository name?
project/distribution name?
Python package name?
module name?
import path?
```

Using the wrong concept can create confusing errors even when each name looks reasonable on its own.

---

# Warning Signs for Similar Problems

I should investigate package/module configuration when:

- a build backend cannot find the expected module
- a project installs but imports fail
- the distribution name contains hyphens
- the directory under `src/` has a different name from the project
- tooling appears to infer the wrong package
- editable installs behave unexpectedly
- tests cannot import the project package

---

# What Not to Do

When encountering a packaging mismatch, avoid immediately:

- renaming many directories
- changing import statements everywhere
- deleting the virtual environment without evidence
- reinstalling Python
- changing the product name
- modifying unrelated source code

First identify which layer expects which name.

---

# Troubleshooting Mental Model

```text
Packaging/build error
        ↓
Inspect pyproject.toml
        ↓
Inspect src/ directory
        ↓
Compare project name and module name
        ↓
Inspect build backend
        ↓
Configure explicit mapping if needed
        ↓
rerun project command
```

---

# Relevant Reliora Files

```text
reliora-ai-support-platform/
│
├── pyproject.toml
│
└── src/
    └── reliora/
        └── __init__.py
```

---

# Why These Files Matter

## `pyproject.toml`

Defines the Python project's metadata and build configuration.

## `src/reliora/`

Contains the actual importable Reliora Python package.

## `src/reliora/__init__.py`

Provides package-level initialization and metadata.

---

# Important Lessons

1. A repository name, distribution name, and Python module name are not necessarily the same thing.
2. Hyphenated project names cannot be used directly as ordinary Python import identifiers.
3. `src/reliora/` defines the importable Reliora package.
4. `pyproject.toml` controls more than dependency installation.
5. The build backend must know which Python module it is building.
6. Explicit configuration can be better than relying on ambiguous inference.
7. Packaging problems should be fixed at the layer where the mismatch actually exists.
8. A concise Python namespace can coexist with a more descriptive repository/product name.
9. When debugging, identify which type of "name" each tool is asking for.
10. Repository structure and Python import structure are related but not identical concepts.

---

## Interview Explanation

> During Reliora's Python project setup, the distribution was named `reliora-ai-support-platform` while the actual importable package was `reliora` under the `src` layout. Rather than renaming the application package or changing imports, I made the relationship explicit in `pyproject.toml` using the `uv` build backend's `module-name` configuration. This reinforced the distinction between repository or distribution identity and the Python module namespace, and it kept the build configuration aligned with the intended package structure.