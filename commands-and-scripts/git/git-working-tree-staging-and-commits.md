# Git Working Tree, Staging Area, and Commits

## Why This Lesson Exists

While building Reliora, I repeatedly used Git commands such as:

```powershell
git status --short
git add ...
git --no-pager diff --cached --stat
git --no-pager diff --cached --check
git commit -m "test: reproduce Stage-1 evaluation gaps"
git --no-pager log -1 --oneline
```

At first, these can look like unrelated commands.

They are actually different steps in one Git workflow.

Understanding this workflow is important because Git does not simply have two states:

```text
saved
not saved
```

Instead, Git separates:

```text
files on my computer
        ↓
changes selected for the next commit
        ↓
permanent local Git history
        ↓
optional remote repository such as GitHub
```

---

## The Core Git Mental Model

A useful mental model is:

```text
Working Directory
      │
      │ git add
      ▼
Staging Area
      │
      │ git commit
      ▼
Local Git History
      │
      │ git push
      ▼
Remote Repository
such as GitHub
```

These are four different things.

A file can therefore exist on my computer without being staged.

A file can be staged without being committed.

A commit can exist locally without being pushed to GitHub.

---

# 1. The Working Directory

## What Is the Working Directory?

The working directory is the project folder currently visible on my computer.

For Reliora, this was:

```text
C:\Users\Aaron\Projects\reliora-ai-support-platform
```

If I open a Python file in VS Code and change a line, the change initially exists in the working directory.

Git may detect the change, but it has not automatically become part of a commit.

---

## Example

Suppose I edit:

```text
src/reliora/evaluation/models.py
```

Git may report:

```text
 M src/reliora/evaluation/models.py
```

This means Git already knows about the file, but the working copy has been modified.

---

# 2. Untracked Files

## What Does `??` Mean?

During Reliora development, `git status --short` showed:

```text
?? evals/dataset_cards/
?? evals/datasets/
```

The two question marks mean:

> Git can see new files, but those files have never been added to Git tracking.

This is not an error.

It is a status.

---

## Why Git Calls Them Untracked

Git already knows the history of tracked files.

A brand-new file has no Git history yet.

For example:

```text
evals/datasets/stage1-reproduction-v1.json
```

was initially a normal file on the computer.

Until `git add` was used, Git treated it as:

```text
untracked
```

---

## `??` Is a Useful Warning Sign

`??` tells me:

> There is something new in this repository that is not yet part of Git history.

Before adding it, I should ask:

- Is this file supposed to be committed?
- Is it temporary?
- Does it contain credentials?
- Does it contain local-only data?
- Is it generated output?
- Is it covered by `.gitignore`?
- Does it belong in this repository?

This is one reason I should inspect Git status before using broad commands such as:

```powershell
git add .
```

---

# 3. Git Does Not Track Empty Directories

In the AWS Agentic AI Learning Ledger, directories such as:

```text
architecture/
concepts/
services/
interview/
certification-mapping/
```

existed on the computer.

However, some did not appear in Git status while they were empty.

Why?

Git tracks files, not empty folders.

A directory normally becomes represented in Git once it contains a tracked file.

This means:

```text
folder exists on disk
```

does not automatically mean:

```text
Git is tracking that folder
```

---

# 4. The Staging Area

## What Is the Staging Area?

The staging area is a preparation area for the next commit.

It allows me to decide exactly which version of which files should enter the next Git snapshot.

A useful analogy is packing a box.

```text
Working Directory
      │
      │ choose files
      ▼
┌─────────────────────┐
│    STAGING AREA     │
│                     │
│ files intended for  │
│ the next commit     │
└──────────┬──────────┘
           │
           │ git commit
           ▼
      Git History
```

---

## Why Git Has a Staging Area

Suppose I changed ten files, but only six belong to one logical piece of work.

The staging area lets me commit only those six.

This helps create commits that are:

- focused
- easier to review
- easier to understand
- easier to revert
- less likely to contain accidental files

---

# 5. `git add`

## Command

During Reliora development, we used commands such as:

```powershell
git add "src/reliora/evaluation/factual.py"
```

and later commands containing several explicit paths.

---

## What `git add` Does

`git add` places the current version of a file into the staging area.

In simple English:

> Prepare this version of this file for the next commit.

---

## What `git add` Does Not Do

`git add` does not:

- create a commit
- push anything to GitHub
- deploy anything to AWS
- permanently save the change in Git history
- upload the file anywhere

It only updates the staging area.

---

# Why We Used Explicit File Paths

Instead of immediately using:

```powershell
git add .
```

we often used explicit paths such as:

```powershell
git add "evals/datasets/stage1-reproduction-v1.json" "evidence/generated/stage1-reproduction-v1-report.json"
```

This was deliberate.

`git add .` can stage many changed files under the current directory.

Explicit paths reduce the chance of accidentally staging:

- unrelated work
- temporary files
- credentials
- local-only evidence
- unfinished changes

This is especially useful when creating a carefully scoped engineering commit.

---

# 6. A Staged File Is a Snapshot of That Moment

This was an important lesson during the Reliora evidence work.

Suppose I run:

```powershell
git add "dataset.json"
```

Git now has a staged copy of that file.

If I then edit `dataset.json` again, Git does not automatically replace the staged copy.

There can temporarily be:

```text
Working directory version
        ≠
Staged version
```

To update the staging area with the newest version, I must run:

```powershell
git add "dataset.json"
```

again.

---

## Why This Mattered in Reliora

We edited:

```text
evals/datasets/stage1-reproduction-v1.json
```

after it had already been staged.

Because the dataset changed, we also regenerated:

```text
evidence/generated/stage1-reproduction-v1-report.json
```

We then ran `git add` again for those files so the staging area contained their newest versions.

This was important because the commit should contain internally consistent evidence.

---

# 7. Understanding `git status --short`

## Command

```powershell
git status --short
```

This gives a compact view of repository changes.

---

## Why We Used `--short`

Normal:

```powershell
git status
```

can provide a longer explanation.

The `--short` version is useful when I already understand the Git states and want a compact inventory.

---

# The Two Status Columns

The short status format effectively has two status positions:

```text
XY path
```

where:

```text
X = staged/index status
Y = working-directory status
```

For example:

```text
A  file.py
```

means the file is staged as added.

```text
M  file.py
```

means a modification is staged.

```text
 M file.py
```

means the file is modified in the working directory but that modification is not staged.

A file can even have changes in both places.

---

# Important Status Symbols We Encountered

## `??` — Untracked

Example:

```text
?? evals/datasets/
```

Meaning:

> New file or files exist, but Git is not tracking them yet.

---

## `A` — Added

After staging new Reliora files, we saw:

```text
A  src/reliora/evaluation/factual.py
```

Meaning:

> This is a new file that has been staged and will become tracked in the next commit.

---

## `M` — Modified

We also saw:

```text
M  src/reliora/evaluation/models.py
```

Meaning:

> This existing tracked file has modifications staged for the next commit.

---

# Status Before and After Staging

Before staging, a new file may appear as:

```text
?? new-file.py
```

After:

```powershell
git add "new-file.py"
```

it may appear as:

```text
A  new-file.py
```

The file did not suddenly become committed.

It moved from:

```text
untracked
```

to:

```text
staged for addition
```

---

# 8. Inspecting What Is Staged

Staging files does not mean I should immediately commit them.

Before committing Reliora's evaluation work, we inspected the staging area.

This helps answer:

> What exactly is about to become permanent Git history?

---

## Staged Statistics

### Command

```powershell
git --no-pager diff --cached --stat
```

We received a result including:

```text
13 files changed, 884 insertions(+), 6 deletions(-)
```

---

## What Each Part Means

### `git diff`

Compare Git states.

### `--cached`

Inspect the staged version.

Another way to think about it is:

> Compare what is staged against the previous commit.

### `--stat`

Show a compact file-and-line-count summary rather than the complete code diff.

### `--no-pager`

Print the output directly in the terminal instead of opening Git's pager.

---

## Why `--stat` Was Useful

The statistics let us check whether the size and scope of the proposed commit made sense.

The Reliora commit contained:

- evaluation datasets
- dataset documentation
- evaluators
- tests
- experiment runner
- generated evidence

Therefore, a relatively large number of inserted lines was understandable.

The statistics helped us notice the scale without reading hundreds of lines in the terminal.

---

# 9. Checking Staged Whitespace

## Command

```powershell
git --no-pager diff --cached --check
```

This checks staged changes for certain whitespace problems.

---

## What Happened in Reliora

Git reported:

```text
evals/datasets/stage1-reproduction-v1.json:4: trailing whitespace.
```

The problem was an invisible space after a line.

The JSON was still syntactically valid, but the staged change contained unnecessary trailing whitespace.

---

## Why We Fixed It

In ordinary text, one trailing space may seem trivial.

In this project, the dataset was also being hashed using SHA-256.

Changing one byte changes a cryptographic hash.

Therefore, we removed the whitespace, regenerated the evidence report, restaged the changed files, and checked again.

The final command:

```powershell
git --no-pager diff --cached --check
```

returned no output.

---

## What No Output Meant

In this context:

```text
no output
```

meant:

> Git did not find the staged whitespace problems that this check looks for.

Silence from a command can therefore be a successful result.

The meaning depends on the command being executed.

---

# 10. Inspecting Staged File Status

Another useful command is:

```powershell
git --no-pager diff --cached --name-status
```

This can show a compact inventory such as:

```text
A    new-file.py
M    existing-file.py
```

It helps answer:

> Which files are entering this commit, and are they being added or modified?

---

# 11. Working-Copy `git diff`

## Command

```powershell
git --no-pager diff -- "path/to/file"
```

This is useful for inspecting changes to tracked files in the working directory.

---

## A Situation That Initially Looked Strange

After Ruff automatically fixed:

```text
tests/unit/evaluation/test_factual.py
```

we tried:

```powershell
git --no-pager diff -- "tests/unit/evaluation/test_factual.py"
```

but Git displayed nothing.

This did not mean Ruff had failed to modify the file.

The file was still untracked.

Git had no earlier committed version of that new file to compare against using ordinary `git diff`.

This reinforced an important lesson:

> Git command output must be interpreted together with the file's current Git state.

---

# 12. `git commit`

Once the Reliora bundle had passed:

```text
pytest
Ruff
mypy
staged whitespace checks
evidence hash verification
```

we created a commit.

## Command

```powershell
git commit -m "test: reproduce Stage-1 evaluation gaps"
```

---

## What `git commit` Does

It takes the currently staged changes and records them as a permanent snapshot in local Git history.

In simple English:

> Save the prepared staging-area snapshot into the repository's local history.

---

## What `-m` Means

The `-m` flag lets me provide the commit message directly.

Our message was:

```text
test: reproduce Stage-1 evaluation gaps
```

The message describes the purpose of the commit.

---

# What `git commit` Does Not Do

A local commit does not automatically:

- push to GitHub
- deploy to AWS
- create a pull request
- publish the repository
- modify cloud resources

The commit exists in local Git history until it is pushed to a remote repository.

---

# 13. The Reliora Commit Result

Git returned:

```text
[feat/evaluation-foundation d463784] test: reproduce Stage-1 evaluation gaps
```

This tells us:

```text
Branch:
feat/evaluation-foundation

Short commit ID:
d463784

Commit message:
test: reproduce Stage-1 evaluation gaps
```

Git also reported:

```text
13 files changed, 884 insertions(+), 6 deletions(-)
```

and listed the newly created files.

---

# 14. Commit IDs

Every Git commit has an identifier.

We saw the short form:

```text
d463784
```

This identifies the specific snapshot.

The full commit identifier is longer.

Commit IDs are useful for:

- referring to exact versions
- comparing changes
- reverting changes
- debugging
- provenance
- release tracking
- linking evidence to code

---

# 15. Verifying the Latest Commit

## Command

```powershell
git --no-pager log -1 --oneline
```

### What `log` Does

Shows Git commit history.

### What `-1` Means

Show only one commit.

### What `--oneline` Means

Display each commit in compact form.

### What `--no-pager` Means

Print directly to the terminal.

---

## Reliora Output

We received:

```text
d463784 (HEAD -> feat/evaluation-foundation) test: reproduce Stage-1 evaluation gaps
```

---

# 16. What Is `HEAD`?

`HEAD` represents the commit or branch position Git currently considers checked out.

The output:

```text
HEAD -> feat/evaluation-foundation
```

meant the current working position was on the:

```text
feat/evaluation-foundation
```

branch at commit:

```text
d463784
```

---

# 17. Verifying a Clean Working State

After committing, we ran:

```powershell
git status --short
```

It produced no output.

In that situation, it meant:

> Git had no uncommitted tracked or untracked changes to report.

This gave us a clean checkpoint before switching to the separate Learning Ledger repository.

---

# 18. Local Commit vs GitHub

A common beginner misconception is:

```text
git commit
=
uploaded to GitHub
```

That is incorrect.

The workflow is:

```text
Working Directory
        ↓
git add
        ↓
Staging Area
        ↓
git commit
        ↓
Local Git History
        ↓
git push
        ↓
Remote Git Repository
```

A commit can exist safely on my laptop without having been pushed.

---

# 19. Why We Did Not Immediately Commit Every Change

During the Reliora reproduction work, we repeatedly stopped before committing.

We first:

- validated JSON
- ran pytest
- ran Ruff
- ran mypy
- inspected Git status
- staged explicit files
- inspected staged statistics
- checked staged whitespace
- investigated CRLF/LF warnings
- verified evidence hashes
- reran quality checks

Only then did we commit.

This was deliberate.

The goal was not merely:

> Make Git accept the files.

The goal was:

> Make a coherent, evidence-backed, reviewable engineering snapshot.

---

# 20. Why Small, Coherent Commits Matter

A good commit should represent a meaningful unit of work.

Benefits include:

- easier code review
- easier debugging
- clearer project history
- easier rollback
- easier collaboration
- better provenance
- more understandable pull requests

The Reliora commit:

```text
test: reproduce Stage-1 evaluation gaps
```

contained one coherent theme:

> Add the machinery and evidence required to reproduce known Stage-1 evaluation weaknesses.

---

# 21. Git Warnings Are Not Automatically Errors

During `git add`, we saw warnings such as:

```text
CRLF will be replaced by LF the next time Git touches it
```

The staging command itself still succeeded.

A warning means:

> Something deserves attention.

It does not automatically mean:

> The operation failed.

However, in Reliora this particular warning eventually became important because our evidence process used byte-level SHA-256 hashes.

That issue is documented separately in the troubleshooting lesson about Git line endings and evidence hashes.

---

# 22. A Safer Git Workflow

A useful workflow from this project is:

```text
Create or modify files
        ↓
Run tests
        ↓
Run linting
        ↓
Run type checks
        ↓
git status --short
        ↓
Stage explicit files
        ↓
Inspect staged changes
        ↓
Check whitespace
        ↓
Verify evidence/provenance if relevant
        ↓
Commit
        ↓
Verify latest commit
        ↓
Verify clean repository state
```

This is much safer than blindly running:

```powershell
git add .
git commit -m "changes"
```

without inspection.

---

# 23. Commands From This Workflow

## Check Repository Status

```powershell
git status --short
```

**Purpose:** Show compact working-directory and staging-area status.

---

## Stage a Specific File

```powershell
git add "path/to/file"
```

**Purpose:** Put the current version of the selected file into the staging area.

---

## Inspect Staged Statistics

```powershell
git --no-pager diff --cached --stat
```

**Purpose:** Show a compact summary of the staged change set.

---

## Check Staged Whitespace

```powershell
git --no-pager diff --cached --check
```

**Purpose:** Detect certain whitespace problems before committing.

---

## Show Staged File Status

```powershell
git --no-pager diff --cached --name-status
```

**Purpose:** Show which staged files are added, modified, or otherwise changed.

---

## Create a Commit

```powershell
git commit -m "commit message"
```

**Purpose:** Save the current staging-area snapshot into local Git history.

---

## Show the Latest Commit

```powershell
git --no-pager log -1 --oneline
```

**Purpose:** Verify the most recent commit.

---

# 24. Warning Signs I Should Notice

When using Git, I should pay attention to:

### Unexpected `??`

Could indicate a file I forgot about or a file that should be ignored.

### Unexpected `A`

Could indicate I accidentally staged a file that should not enter the repository.

### Unexpected `M`

Could mean a tracked file changed unintentionally.

### Very Large Staged Diff

Could indicate unrelated changes were accidentally grouped together.

### Credentials or Secrets

These should never be committed merely because Git allows them to be staged.

### Line-Ending Warnings

Often normal, but can matter when byte-level file identity is important.

### Dirty Status After a Commit

Could mean some work was not included in the commit.

---

# 25. Important Lessons

1. The working directory, staging area, commit history, and GitHub are different states.
2. `git add` does not create a commit.
3. `git commit` does not automatically push to GitHub.
4. `??` is an untracked-file status, not an error.
5. `A` means a new file is staged for addition.
6. `M` means a tracked file has been modified.
7. A staged file does not automatically update if I edit it afterward.
8. Git does not normally track empty directories.
9. Inspecting staged changes before committing reduces mistakes.
10. No output can sometimes mean success, depending on the command.
11. Warnings should be interpreted in context rather than blindly ignored.
12. Good commits should represent coherent units of engineering work.

---

## Interview Explanation

> I treat Git's staging area as a deliberate review boundary between my working directory and permanent commit history. I inspect repository status, stage explicit files, review staged changes, run quality and whitespace checks, and then create focused commits. This reduces accidental changes, improves auditability, and produces a clearer engineering history.