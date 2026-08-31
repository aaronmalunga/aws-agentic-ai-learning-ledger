# PowerShell Paths, Environment Variables, and Safe File Operations

## Why This Lesson Exists

Reliora and the AWS Agentic AI Learning Ledger are being developed on Windows using:

```text
PowerShell
VS Code
Git
Python
uv
```

During the project, several PowerShell concepts repeatedly became important:

```text
changing directories
checking current location
listing files
working with relative and absolute paths
quoting paths correctly
reading environment variables
removing problematic environment variables
writing text files with explicit encoding
checking whether files exist
distinguishing shell commands from Python code
```

Several troubleshooting incidents were not actually Python or Git problems.

They involved the shell environment around those tools.

Examples included:

```text
moving between the Reliora repository and the Learning Ledger

checking whether a Markdown file really existed

removing the invalid GIT_PAGER_IN_USE environment variable

writing README.md explicitly as UTF-8

understanding why changing directory did not deactivate a Python virtual environment
```

Understanding PowerShell therefore matters even though PowerShell is not part of the Reliora application itself.

---

# 1. What Is PowerShell?

PowerShell is a command shell and scripting environment.

It allows us to:

```text
navigate the file system
launch programs
inspect files
manipulate environment variables
automate administrative tasks
run Git
run uv
run Python
run AWS CLI
run Terraform
```

When the terminal shows something such as:

```text
PS C:\Users\Aaron\Projects\reliora-ai-support-platform>
```

the:

```text
PS
```

indicates PowerShell.

---

# 2. PowerShell Is Not Python

This distinction caused confusion earlier in development.

PowerShell accepts commands such as:

```powershell
git status
```

```powershell
uv run pytest -q
```

```powershell
Get-ChildItem
```

Python syntax belongs inside:

```text
.py files
```

or an explicitly opened Python interpreter.

For example:

```python
from pathlib import Path
```

is Python code.

It should not normally be pasted directly into an ordinary PowerShell prompt.

---

# 3. Shell Command vs Source Code

A useful rule is:

```text
PowerShell prompt
→ commands that launch or control programs

.py file
→ Python application code

.md file
→ documentation

.json file
→ structured data

.toml file
→ project configuration
```

Knowing which environment owns the text prevents many avoidable errors.

---

# 4. Current Working Directory

Every shell has a current working directory.

This is the directory from which relative paths are interpreted.

For example:

```text
C:\Users\Aaron\Projects\reliora-ai-support-platform
```

might be the current directory.

---

# 5. `Get-Location`

PowerShell can show the current directory with:

```powershell
Get-Location
```

Conceptually:

```text
Where am I?
        ↓
Get-Location
        ↓
current directory
```

This command is read-only.

It does not modify files.

---

# 6. Why Current Location Matters

Suppose we run:

```powershell
git status
```

Git searches for the repository associated with the current location.

Likewise:

```powershell
uv run pytest -q
```

depends on running from the appropriate Python project.

Being in the wrong directory can therefore produce errors that appear to come from Git, Python, or uv.

---

# 7. `Set-Location`

PowerShell changes the current directory using:

```powershell
Set-Location
```

Example:

```powershell
Set-Location "C:\Users\Aaron\Projects\reliora-ai-support-platform"
```

This means:

> Make this directory the shell's current working location.

---

# 8. Short Alias: `cd`

PowerShell also supports:

```powershell
cd "C:\Users\Aaron\Projects\reliora-ai-support-platform"
```

`cd` is commonly used because it is shorter.

For learning documentation, the full:

```text
Set-Location
```

name can sometimes make the command's meaning clearer.

---

# 9. Changing Directory Does Not Change the Environment Automatically

Earlier, we moved from the Reliora repository into the Learning Ledger while a Python virtual environment was active.

Changing directories with:

```powershell
Set-Location
```

does not automatically deactivate that environment.

These are separate concepts.

---

# Mental Model

```text
Current directory
→ where the shell is looking

Virtual environment
→ which Python environment is active
```

Changing one does not necessarily change the other.

---

# 10. Example

Suppose the prompt shows:

```text
(reliora-ai-support-platform) PS C:\Users\Aaron\Projects\aws-agentic-ai-learning-ledger>
```

This means:

```text
Current directory:
aws-agentic-ai-learning-ledger

Active Python environment:
Reliora environment
```

That state is possible.

---

# 11. `deactivate`

When a Python virtual environment is active, the command:

```powershell
deactivate
```

removes that virtual environment from the current shell session.

It does not:

```text
delete the .venv
delete packages
delete source code
change repository files
```

It only stops that environment from being active in the current terminal.

---

# 12. Absolute Paths

An absolute path identifies a location from the filesystem root.

Example:

```text
C:\Users\Aaron\Projects\reliora-ai-support-platform
```

This location does not depend on the current working directory.

---

# 13. Relative Paths

A relative path is interpreted from the current working directory.

Example:

```text
commands-and-scripts\git
```

If the current directory is:

```text
C:\Users\Aaron\Projects\aws-agentic-ai-learning-ledger
```

then PowerShell interprets:

```text
commands-and-scripts\git
```

as:

```text
C:\Users\Aaron\Projects\aws-agentic-ai-learning-ledger\commands-and-scripts\git
```

---

# 14. Why Relative Paths Are Useful

Inside a repository, relative paths are often shorter and more portable.

For example:

```powershell
Get-ChildItem "commands-and-scripts\git"
```

is easier to read than repeating the complete absolute path.

---

# 15. Why Absolute Paths Are Useful

Absolute paths are helpful when:

```text
the current directory is uncertain
switching between repositories
running commands from another location
debugging path problems
```

They remove ambiguity about which file is being addressed.

---

# 16. `.` Means Current Directory

A single dot commonly represents:

```text
the current directory
```

For example:

```powershell
Get-ChildItem .
```

means:

> List the contents of the current directory.

---

# 17. `..` Means Parent Directory

Two dots represent:

```text
the parent directory
```

For example:

```powershell
Set-Location ..
```

moves one directory upward.

---

# 18. Example

From:

```text
C:\Users\Aaron\Projects\reliora-ai-support-platform
```

running:

```powershell
Set-Location ..
```

would move to:

```text
C:\Users\Aaron\Projects
```

---

# 19. Path Separators

Windows commonly uses:

```text
\
```

as its path separator.

Example:

```text
commands-and-scripts\troubleshooting
```

Python's `pathlib` can handle platform-aware path construction, which becomes useful inside Python code.

---

# 20. Why We Quote Paths

We often write:

```powershell
Set-Location "C:\Users\Aaron\Projects\Some Project"
```

The quotation marks ensure a path containing spaces is interpreted as one argument.

Without quoting:

```text
Some Project
```

could be parsed as separate pieces.

---

# 21. Quoting Even When There Are No Spaces

We sometimes also quote paths such as:

```powershell
Get-ChildItem "commands-and-scripts\git"
```

even when no spaces exist.

This is harmless and can make the command boundary visually clear.

---

# 22. `Get-ChildItem`

PowerShell lists directory contents using:

```powershell
Get-ChildItem
```

Example used during the Learning Ledger work:

```powershell
Get-ChildItem "commands-and-scripts\git"
```

This allowed us to verify that:

```text
git-working-tree-staging-and-commits.md
```

actually existed.

---

# 23. Why We Used It

There had been confusion because the terminal visually wrapped the filename.

Rather than guessing whether the filename was wrong, we asked the filesystem directly.

This is a good troubleshooting principle:

> Inspect actual state before modifying it.

---

# 24. Terminal Wrapping Is Not Filename Modification

A narrow terminal window may visually display:

```text
git-working-tree-staging-
and-commits.md
```

across two screen lines.

That does not necessarily mean a newline exists inside the filename.

The terminal may simply be wrapping the display.

`Get-ChildItem` helped verify the real filesystem entry.

---

# 25. `Test-Path`

PowerShell can check whether a path exists with:

```powershell
Test-Path "commands-and-scripts\git\git-working-tree-staging-and-commits.md"
```

Possible output:

```text
True
```

or:

```text
False
```

---

# 26. Why `Test-Path` Is Useful

Before:

```text
reading
overwriting
deleting
moving
```

a file, it can be useful to verify that the path points to what we expect.

This is especially useful in scripts.

---

# 27. Read-Only Inspection Before Modification

A safe workflow is often:

```text
inspect
    ↓
confirm
    ↓
modify
```

rather than:

```text
guess
    ↓
modify
```

Useful inspection commands include:

```text
Get-Location
Get-ChildItem
Test-Path
git status
```

---

# 28. Environment Variables

An environment variable stores information associated with the operating-system process environment.

Examples can include:

```text
PATH
USERPROFILE
AWS_REGION
AWS_PROFILE
GIT_PAGER_IN_USE
```

Programs can read these values while they execute.

---

# 29. PowerShell Environment Variable Syntax

PowerShell accesses an environment variable using:

```text
$env:
```

For example:

```powershell
$env:USERPROFILE
```

may produce something like:

```text
C:\Users\Aaron
```

---

# 30. Why `$env:USERPROFILE` Is Useful

Instead of hardcoding:

```text
C:\Users\Aaron
```

a script can derive the current Windows user's home directory.

Conceptually:

```powershell
Set-Location "$env:USERPROFILE\Projects"
```

This is more portable between user accounts.

---

# 31. Environment Variables Affect Child Processes

Suppose the current PowerShell environment contains:

```text
AWS_REGION
```

When we launch:

```text
aws
python
terraform
```

those programs may read the variable.

This is why shell environment problems can appear as application problems.

---

# 32. The `GIT_PAGER_IN_USE` Incident

During Reliora work, Git produced:

```text
fatal: bad boolean environment value '' for 'GIT_PAGER_IN_USE'
```

The problem was not the repository itself.

Git was reading an invalid environment-variable value.

---

# 33. Removing the Problematic Variable

We used:

```powershell
Remove-Item Env:GIT_PAGER_IN_USE -ErrorAction SilentlyContinue
```

This removed the variable from the current PowerShell process environment.

---

# 34. `Env:` Is a PowerShell Provider

PowerShell exposes environment variables through a provider named:

```text
Env:
```

Therefore:

```text
Env:GIT_PAGER_IN_USE
```

refers to that environment variable.

---

# 35. Breaking Down `Remove-Item`

```powershell
Remove-Item Env:GIT_PAGER_IN_USE
```

means:

> Remove this item from the environment-variable provider.

It does not delete a project file named:

```text
GIT_PAGER_IN_USE
```

---

# 36. `-ErrorAction SilentlyContinue`

The command included:

```powershell
-ErrorAction SilentlyContinue
```

This tells PowerShell:

> If this removal produces a normal PowerShell error, suppress its display and continue.

This was appropriate because:

```text
if the variable does not exist
```

there is nothing that needs fixing.

---

# 37. Why This Flag Must Be Used Deliberately

Suppressing errors broadly can hide important failures.

Therefore:

```text
SilentlyContinue
```

should not be added everywhere.

In this specific case, absence of the variable was an acceptable state.

---

# 38. Current Shell Scope

Removing the variable this way primarily affects:

```text
the current PowerShell process
```

and processes subsequently launched from it.

It does not automatically mean:

```text
the variable has been permanently removed from every Windows configuration location
```

Scope matters.

---

# 39. Shell State Can Be Temporary

Examples of temporary shell state include:

```text
active virtual environment
environment variables
current directory
some aliases/functions
```

Opening a new terminal can produce a different state.

This is useful when diagnosing problems.

---

# 40. Persistent Configuration vs Session Configuration

A useful distinction is:

```text
Session configuration
→ affects current terminal

Persistent configuration
→ survives new terminals/reboots depending on scope
```

Before making permanent changes, understand which one is required.

---

# 41. Writing Files From PowerShell

PowerShell provides commands such as:

```text
Set-Content
Add-Content
Out-File
```

that can write files.

These commands have side effects.

They should therefore be used carefully.

---

# 42. `Set-Content`

Conceptually:

```powershell
Set-Content -Path "README.md" -Value $content
```

writes content to the specified file.

If the file already exists, its previous contents can be replaced.

This is a meaningful side effect.

---

# 43. Why `Set-Content` Requires Care

Running it against the wrong path could overwrite an important file.

Before using it, verify:

```text
current directory
target path
intended content
encoding
```

---

# 44. The README Encoding Incident

During Reliora setup, `uv` reported that:

```text
README.md
```

was not valid UTF-8.

The issue came from how the file had been written through PowerShell.

We rewrote the file explicitly with:

```text
UTF-8
```

encoding.

---

# 45. `-Encoding utf8`

A command can specify:

```powershell
-Encoding utf8
```

to make the encoding explicit.

For example:

```powershell
Set-Content -Path "README.md" -Value $content -Encoding utf8
```

This reduces ambiguity about how characters are represented.

---

# 46. Why Encoding Matters

Tools such as:

```text
uv
Python
GitHub
package builders
```

may expect text files to use UTF-8.

An encoding problem can therefore break tooling even when the visible text looks normal in the editor.

---

# 47. Encoding vs Line Endings

These are different concepts.

```text
Encoding
→ how characters become bytes

Line endings
→ which bytes represent the end of each line
```

For example:

```text
UTF-8 + LF
```

and:

```text
UTF-8 + CRLF
```

use the same character encoding but different line-ending conventions.

---

# 48. `.editorconfig`

Reliora uses `.editorconfig` to communicate editor-level text conventions such as:

```text
UTF-8
LF
```

This reduces accidental inconsistency between files.

---

# 49. `.gitattributes`

Reliora also uses `.gitattributes`.

Git can use this to manage repository line-ending normalization.

This operates at a different layer from `.editorconfig`.

---

# 50. PowerShell Version Differences Matter

Different PowerShell versions have historically had different default behaviours around text encoding.

Therefore explicit encoding is safer when exact byte representation matters.

---

# 51. Large Here-Strings

PowerShell supports multi-line strings called here-strings.

Conceptually:

```powershell
$content = @"
multiple
lines
of
text
"@
```

These can be useful for short scripts.

---

# 52. Why Large Here-Strings Became Problematic

During repository setup, very large text blocks were awkward to paste into the terminal.

Problems can include:

```text
paste errors
delimiter mistakes
quoting mistakes
hard-to-review content
terminal confusion
```

For large source or documentation files, VS Code is usually easier and safer.

---

# 53. Terminal Is Best for Commands

Examples:

```powershell
uv run pytest -q
```

```powershell
git status --short
```

```powershell
Get-ChildItem
```

These are appropriate terminal operations.

---

# 54. Editor Is Better for Large Files

Examples:

```text
Python modules
Markdown documentation
JSON datasets
TOML configuration
Terraform files
```

Editing these in VS Code provides:

```text
syntax highlighting
review before save
undo
search
formatting
```

which is safer than giant terminal paste operations.

---

# 55. This Is Why We Introduced Strict Boundaries

For Learning Ledger work, instructions now distinguish:

```text
===== START FILE =====
...
===== END FILE =====
```

from:

```text
===== TERMINAL COMMAND =====
...
===== END TERMINAL COMMAND =====
```

The purpose is to distinguish:

```text
content to save
```

from:

```text
commands to execute
```

---

# 56. File Contents Are Not Terminal Commands

For example:

```markdown
# Reliora
```

belongs in:

```text
README.md
```

It is not something to execute at the PowerShell prompt.

---

# 57. Terminal Commands Are Not File Contents

Likewise:

```powershell
git status --short
```

should not normally be pasted into:

```text
README.md
```

unless the documentation intentionally presents the command as an example.

Context matters.

---

# 58. `Get-Content`

PowerShell can read a text file with:

```powershell
Get-Content "README.md"
```

This is useful for inspecting smaller text files from the terminal.

It is read-only.

---

# 59. Why VS Code May Still Be Better for Long Files

Large Markdown or source files are generally easier to inspect with:

```text
editor tabs
search
Markdown preview
line numbers
```

Terminal output is better suited to targeted inspection.

---

# 60. Wildcards

PowerShell supports wildcards such as:

```text
*
```

For example:

```powershell
Get-ChildItem "*.md"
```

can list Markdown files.

---

# 61. Wildcards Need Care With Destructive Commands

A command such as:

```powershell
Remove-Item "*.md"
```

could delete multiple files.

Therefore wildcard usage should be especially deliberate when the command modifies the filesystem.

---

# 62. Read Commands vs Write Commands

A useful risk distinction is:

### Mostly read-only

```text
Get-Location
Get-ChildItem
Get-Content
Test-Path
git status
git diff
```

### Potentially modifying

```text
Set-Content
Remove-Item
Move-Item
Copy-Item
git add
git commit
```

Knowing which category a command belongs to is important before execution.

---

# 63. `Remove-Item` Can Be Dangerous

The same command used safely for:

```text
Env:GIT_PAGER_IN_USE
```

can also delete filesystem items.

For example:

```powershell
Remove-Item "some-file.txt"
```

deletes the target file.

Therefore always inspect:

```text
which provider/path
```

the command is addressing.

---

# 64. PowerShell Providers

PowerShell can expose several systems using path-like syntax.

Examples can include:

```text
filesystem
environment variables
registry
```

Therefore:

```text
Env:NAME
```

has different semantics from:

```text
C:\path\file
```

even though both can be manipulated using some of the same cmdlets.

---

# 65. Why Command Names Are Verb-Noun

PowerShell commands commonly follow:

```text
Verb-Noun
```

examples:

```text
Get-Location
Set-Location
Get-ChildItem
Remove-Item
Set-Content
Test-Path
```

The verb often communicates whether the operation reads or changes something.

---

# 66. Useful Mental Mapping

```text
Get
→ retrieve/read

Set
→ modify/assign

Remove
→ delete/remove

Test
→ check

New
→ create
```

This helps infer unfamiliar command behaviour.

---

# 67. PowerShell Pipeline

PowerShell can pass output from one command to another using:

```text
|
```

For example:

```powershell
Get-ChildItem | Where-Object { $_.Extension -eq ".md" }
```

Conceptually:

```text
Get files
   ↓
pass objects
   ↓
filter them
```

---

# 68. PowerShell Pipelines Pass Objects

Unlike traditional shells that often primarily pass text, PowerShell commonly passes structured .NET objects through the pipeline.

This allows access to properties such as:

```text
Name
Extension
Length
LastWriteTime
```

---

# 69. Why This Matters

A listing from:

```powershell
Get-ChildItem
```

is not merely text printed on the screen.

PowerShell can continue operating on the returned file objects.

This makes scripting powerful.

---

# 70. But We Do Not Need Complex PowerShell Yet

Reliora does not currently require sophisticated PowerShell automation.

The immediate goal is to understand the commands we actually use.

We should learn complexity when requirements justify it.

---

# 71. Command History

PowerShell keeps command history within the session.

The up-arrow key can often retrieve previous commands.

This can save typing.

However, inspect a recalled command before pressing Enter.

---

# 72. Why Reusing Commands Requires Attention

A previous command may contain:

```text
old path
old filename
destructive flag
old branch
```

Never assume a recalled command is safe for the current context.

---

# 73. Tab Completion

PowerShell supports tab completion for many:

```text
paths
commands
arguments
```

This can reduce path-typing mistakes.

---

# 74. Verify Before Running Destructive Commands

Before commands that:

```text
delete
overwrite
move
deploy
destroy
```

verify:

```text
current location
target
scope
side effect
```

This principle will become particularly important with:

```text
Terraform
AWS CLI
```

later.

---

# 75. PowerShell and Git

Commands such as:

```powershell
git status
```

launch the external Git executable from PowerShell.

PowerShell is not performing Git operations itself.

---

# 76. PowerShell and `uv`

Likewise:

```powershell
uv run pytest -q
```

launches:

```text
uv
```

which then launches:

```text
pytest
```

inside the managed project environment.

---

# 77. PowerShell and AWS CLI

Later:

```powershell
aws sts get-caller-identity
```

will launch the AWS CLI.

The shell may provide:

```text
environment variables
current path
credentials context
```

that influence the AWS command.

---

# 78. PowerShell and Terraform

Similarly:

```powershell
terraform plan
```

launches Terraform in the current directory.

Terraform then looks for configuration and state according to its own rules.

Again, current location matters.

---

# 79. Shell Context Is Part of Reproducibility

When documenting a command, useful context may include:

```text
repository
directory
active environment
branch
required credentials
```

The same command can behave differently in another context.

---

# 80. Why We Explain Commands Before Running Them

For this project, commands are not treated as magic instructions.

Before running significant commands, understand:

```text
what it does
why we need it now
what the arguments mean
expected output
side effects
failure modes
```

This transforms terminal use from copying into engineering knowledge.

---

# 81. Example: Read-Only Command

```powershell
Get-ChildItem "commands-and-scripts\git"
```

### What

Lists objects inside the directory.

### Why

Verify whether a file exists and inspect its actual name.

### Risk

Minimal because the command is read-only.

---

# 82. Example: Environment Modification

```powershell
Remove-Item Env:GIT_PAGER_IN_USE -ErrorAction SilentlyContinue
```

### What

Removes the specified environment variable from the current PowerShell environment.

### Why

Git rejected the empty value as an invalid boolean environment configuration.

### Risk

Changes shell state, but does not delete repository files.

---

# 83. Example: File Modification

```powershell
Set-Content -Path "README.md" -Value $content -Encoding utf8
```

### What

Writes `$content` into `README.md` using UTF-8.

### Why

Ensure the project README has a valid encoding.

### Risk

Replaces existing file content.

This deserves more caution than a read command.

---

# 84. Diagnosing "Command Not Found"

If PowerShell reports that a command is not recognized, possible causes include:

```text
program not installed
program not on PATH
misspelled command
virtual environment not active
wrong shell/session
```

Do not immediately reinstall software.

First identify the layer that failed.

---

# 85. Diagnosing "Path Not Found"

Possible causes include:

```text
wrong current directory
misspelled path
file has not been created
relative path resolved somewhere unexpected
```

Useful diagnostics:

```text
Get-Location
Get-ChildItem
Test-Path
```

---

# 86. Diagnosing Environment Problems

If an external program behaves strangely, inspect relevant environment variables.

The Git pager incident demonstrated:

```text
application error
can originate from shell configuration
```

Debugging should consider the entire execution environment.

---

# 87. Smallest-Layer Fix

A useful troubleshooting principle is:

> Fix the smallest layer that actually contains the fault.

For the Git pager incident:

```text
bad environment variable
```

was the fault.

We did not need to:

```text
reinstall Git
reclone repository
recreate virtual environment
```

We removed the invalid variable.

---

# 88. Another Smallest-Layer Example

For the README encoding issue:

```text
README file encoding
```

was the fault.

We did not need to:

```text
reinstall uv
reinstall Python
recreate repository
```

We corrected the file encoding.

---

# 89. Safe Troubleshooting Sequence

A useful sequence is:

```text
Read error
    ↓
identify which layer produced it
    ↓
inspect current state
    ↓
form smallest plausible cause
    ↓
apply targeted fix
    ↓
rerun original command
```

This reduces unnecessary changes.

---

# 90. Important Lessons

1. PowerShell is the shell environment from which Reliora's development tools are being launched.
2. PowerShell commands and Python source code belong in different execution contexts.
3. The current working directory affects Git, uv, Terraform, AWS CLI, and relative paths.
4. `Get-Location` shows the current directory.
5. `Set-Location` changes the current directory.
6. Changing directories does not automatically deactivate a Python virtual environment.
7. Absolute paths identify complete filesystem locations.
8. Relative paths are interpreted from the current directory.
9. Quotes protect paths containing spaces and make argument boundaries clear.
10. `Get-ChildItem` lists filesystem contents and helped verify Learning Ledger filenames.
11. Terminal line wrapping does not necessarily alter a filename.
12. `Test-Path` can verify whether a target exists before acting on it.
13. Environment variables are accessed in PowerShell through `$env:` or the `Env:` provider.
14. Shell environment state can change the behaviour of programs such as Git.
15. `Remove-Item Env:...` modifies environment state rather than deleting a project file.
16. `-ErrorAction SilentlyContinue` should only suppress errors that are genuinely acceptable.
17. `Set-Content` modifies files and can overwrite existing content.
18. Explicit UTF-8 encoding helps avoid cross-tool text-encoding problems.
19. Encoding and line-ending conventions are different concerns.
20. Large source/document files are usually safer to edit in VS Code than through huge terminal here-strings.
21. Read-only inspection should generally happen before destructive modification.
22. Commands recalled from history should be reviewed before execution.
23. Current directory, environment, credentials, and repository state are part of command context.
24. Troubleshooting should target the smallest layer that actually contains the fault.
25. Terminal commands should be understood in terms of purpose, flags, expected output, side effects, and failure modes rather than copied blindly.

---

## Interview Explanation

> I use PowerShell as the local orchestration shell for Git, uv, Python, AWS CLI, and Terraform, so I treat shell state as part of the development environment rather than as invisible plumbing. During Reliora, that mattered when an invalid `GIT_PAGER_IN_USE` environment variable caused a Git failure and when a PowerShell-written README had an encoding issue that prevented `uv` from building the project. In both cases I diagnosed the layer that actually failed and applied a targeted fix instead of reinstalling unrelated tooling. I also distinguish current directory, active virtual environment, environment variables, and repository state because each can independently change how a command behaves.