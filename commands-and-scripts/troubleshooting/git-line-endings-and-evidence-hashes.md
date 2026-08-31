# Git Line Endings and Evidence Hash Mismatch

## Why This Lesson Exists

While building Reliora's Stage-1 reproduction evidence, Git repeatedly displayed warnings such as:

```text
warning: in the working copy of 'evals/datasets/stage1-reproduction-v1.json',
CRLF will be replaced by LF the next time Git touches it
```

At first, this looked like a normal Windows Git warning.

Later, we discovered that the warning had become important because Reliora was using SHA-256 hashes to prove exactly which dataset produced an evidence report.

A verification command eventually produced:

```text
Staged dataset SHA256:
4f83b0ffa6a87b4f1dab68d27352b0f40d3556a87b4935e073bc59b0e38e1429

Report-recorded SHA256:
694c25333c2f9e58256b164361290181e0293706ba013055185d97fc6cd63802

Hashes match: False
```

That meant the evidence report was not fingerprinting the exact representation Git was preparing to commit.

This was a real provenance problem.

---

## What Are Line Endings?

Text files need hidden characters to represent the end of each line.

Two common conventions are:

### Windows

```text
CRLF
```

which means:

```text
Carriage Return + Line Feed
```

### Linux and Unix

```text
LF
```

which means:

```text
Line Feed
```

The visible text can look exactly the same in an editor even though the underlying bytes are different.

---

## Why This Usually Goes Unnoticed

A human may see:

```text
hello
world
```

regardless of whether the file internally uses:

```text
CRLF
```

or:

```text
LF
```

Most modern editors handle both automatically.

So for ordinary source-code editing, the difference can feel invisible.

---

## Why Line Endings Matter for Hashing

A cryptographic hash such as SHA-256 works on the exact bytes of a file.

It does not hash what the file looks like visually.

Therefore:

```text
same visible text
+
CRLF bytes
```

and:

```text
same visible text
+
LF bytes
```

produce different SHA-256 hashes.

Even one changed byte can produce a completely different fingerprint.

---

# What Is SHA-256?

SHA-256 is a cryptographic hashing algorithm.

It takes data as input and produces a 256-bit fingerprint, commonly displayed as 64 hexadecimal characters.

For example:

```text
4f83b0ffa6a87b4f1dab68d27352b0f40d3556a87b4935e073bc59b0e38e1429
```

A useful mental model is:

```text
file bytes
    ↓
SHA-256
    ↓
digital fingerprint
```

---

## What SHA-256 Is Useful For

SHA-256 can help prove that:

- a file has not changed
- an experiment used a particular dataset
- an evidence report corresponds to a particular artifact
- two byte-for-byte representations are identical

It is especially useful for:

```text
provenance
auditability
reproducibility
integrity checking
```

---

## SHA-256 Is Not Encryption

Hashing and encryption are different.

Encryption is designed so protected data can later be decrypted with the correct key.

Hashing is designed as a one-way fingerprinting process.

In this project, SHA-256 was used for identity and provenance rather than secrecy.

---

# What Happened in Reliora

Reliora contains:

```text
evals/datasets/stage1-reproduction-v1.json
```

This is the versioned Stage-1 reproduction dataset.

The reproduction runner:

```text
evals/runners/run_stage1_reproduction.py
```

loads that dataset and produces:

```text
evidence/generated/stage1-reproduction-v1-report.json
```

The report records a SHA-256 fingerprint for the dataset.

The intended relationship is:

```text
dataset
    ↓
SHA-256
    ↓
hash stored in report
```

This allows the report to identify exactly which dataset produced the experimental result.

---

# The Initial Problem

The reproduction runner originally read the ordinary Windows working-copy file.

That working copy used Windows-style:

```text
CRLF
```

line endings.

The runner calculated the dataset hash from those bytes.

Conceptually:

```text
Working copy dataset
       ↓
     CRLF
       ↓
   SHA-256
       ↓
hash written into report
```

---

# What Git Was Doing

Reliora also had repository-level line-ending rules.

The `.gitattributes` file instructed Git to normalize repository text files to:

```text
LF
```

Therefore, when Git staged the dataset, the representation became:

```text
Working-copy file
      CRLF
        ↓
Git normalization
        ↓
Staged Git representation
       LF
```

The staged file therefore contained different bytes from the file that the runner originally hashed.

---

# Why Git Displayed the Warning

During `git add`, Git displayed:

```text
CRLF will be replaced by LF the next time Git touches it
```

In simple English, Git was saying:

> The file on Windows currently uses CRLF line endings, but this repository is configured to store the text using LF line endings.

The warning itself did not mean `git add` failed.

The files were staged successfully.

---

# Why the Warning Became Important

For ordinary Python source code, line-ending normalization may not change program behaviour.

But Reliora's evidence system was using byte-level fingerprints.

Therefore:

```text
CRLF bytes
!=
LF bytes
```

which meant:

```text
working-copy SHA-256
!=
staged Git SHA-256
```

That created an inconsistency in the evidence chain.

---

# The Provenance Problem

The report was intended to say:

> This experimental result was produced from this exact dataset.

But if the report stored the hash of the CRLF version while Git committed the LF version, then a future verification could produce:

```text
Report claims dataset hash:
ABC...

Actual committed dataset hash:
XYZ...
```

An auditor could reasonably ask:

> If these fingerprints differ, how do we know the committed dataset is the dataset that produced this report?

That would weaken the evidence provenance.

---

# What Is Provenance?

Provenance means being able to trace where an artifact or result came from.

In this case:

```text
Frozen Stage-1 evidence
        ↓
Reproduction dataset
        ↓
Evaluator execution
        ↓
Generated evidence report
```

A trustworthy provenance chain should make it possible to connect each output to its exact input.

---

# Working Copy vs Staging Area

This issue also reinforced an important Git concept.

There were two representations involved:

```text
Working Directory
```

and:

```text
Git Staging Area
```

The working directory contained Windows-style bytes.

The staging area contained the normalized Git representation.

They looked visually identical but were not byte-for-byte identical.

---

# How We Diagnosed the Problem

Instead of assuming the warning was harmless, we compared the exact staged dataset with the hash recorded inside the exact staged evidence report.

The verification used Git's staged-file syntax.

For example:

```text
:evals/datasets/stage1-reproduction-v1.json
```

The leading colon means:

> Read the version currently stored in Git's staging area.

This is different from simply opening the ordinary working-copy file.

---

## Verification Command Used

We ran a Python command that:

1. read the staged dataset
2. calculated its SHA-256
3. read the staged evidence report
4. extracted the dataset hash recorded in the report
5. compared the two values

The result was:

```text
Staged dataset SHA256:
4f83b0ffa6a87b4f1dab68d27352b0f40d3556a87b4935e073bc59b0e38e1429

Report-recorded SHA256:
694c25333c2f9e58256b164361290181e0293706ba013055185d97fc6cd63802

Hashes match: False
```

---

# What `Hashes match: False` Meant

This was not merely a cosmetic warning.

It meant:

> The exact dataset Git was preparing to commit did not have the same fingerprint that the evidence report claimed.

That was enough reason to stop before committing.

---

# Why We Did Not Ignore the Problem

One tempting response could have been:

> The JSON looks the same, so the difference does not matter.

That would have been inappropriate because the system explicitly used hashes for evidence integrity.

If the hash contract is meaningful, the bytes being hashed must be clearly defined.

Otherwise, the evidence mechanism becomes unreliable.

---

# The Correct Fix: Canonical Representation

We changed the reproduction runner so text is normalized before hashing.

The canonical hashing rule became:

```text
UTF-8 encoded text
+
LF line endings
```

Before calculating SHA-256, the runner converts:

```text
CRLF → LF
CR   → LF
```

This gives the hashing process one consistent representation.

---

## Why Canonicalization Matters

Without canonicalization:

```text
Windows
   ↓
CRLF
   ↓
Hash A
```

while:

```text
Linux
   ↓
LF
   ↓
Hash B
```

With canonicalization:

```text
Windows CRLF ─┐
Linux LF      ├──→ canonical UTF-8 + LF ──→ same hash
CI runner     ┘
```

This makes the fingerprint more reproducible across environments.

---

# The Function We Added

The reproduction runner gained logic conceptually equivalent to:

```python
def normalize_text_bytes_for_hashing(raw_bytes: bytes) -> bytes:
    text = raw_bytes.decode("utf-8")
    normalized_text = text.replace("\r\n", "\n").replace("\r", "\n")
    return normalized_text.encode("utf-8")
```

The purpose is:

```text
input text bytes
      ↓
decode UTF-8
      ↓
normalize line endings
      ↓
encode UTF-8
      ↓
SHA-256
```

---

# Why This Logic Belongs in the Runner

The dataset itself should remain normal version-controlled text.

The reproduction runner is responsible for defining how experiment provenance is calculated.

Therefore, the canonical hashing rule belongs with the evidence-generation logic.

---

# We Also Documented the Hash Basis

The generated report was changed to explicitly state the hashing rule.

Conceptually, the report records:

```text
hash_basis:
UTF-8 text normalized to LF line endings before SHA-256
```

This is important because a hash value by itself does not explain:

> Hash of exactly what representation?

The report now documents that contract.

---

# Why Documenting the Hash Basis Matters

Consider two engineers:

```text
Engineer A:
hashes raw Windows working-copy bytes

Engineer B:
hashes normalized LF text
```

They may get different results even if the visible content looks identical.

By explicitly documenting the hash basis, future engineers know exactly how to reproduce the fingerprint.

---

# Regenerating the Evidence

After changing the hashing logic, the reproduction report had to be generated again.

Why?

Because the report itself contains the dataset hash.

Changing the hashing rule changes the value the report must contain.

The experiment was rerun and still reported:

```text
Total cases: 4
Executable cases: 3
Documented non-executable cases: 1
Detected known defects: 3
Mismatches: 0
All executable expectations met: True
```

The behavioural experiment had not changed.

Only the provenance calculation had been corrected.

---

# Restaging Was Also Necessary

The runner and evidence report had already been staged before the fix.

Editing them afterward did not automatically update the staging area.

Therefore, we ran `git add` again for the changed files.

This replaced the staged copies with the corrected versions.

---

# Final Verification

After normalizing the hash basis, regenerating the report, and restaging the files, we repeated the staged comparison.

The result was:

```text
Staged dataset SHA256:
4f83b0ffa6a87b4f1dab68d27352b0f40d3556a87b4935e073bc59b0e38e1429

Report-recorded SHA256:
4f83b0ffa6a87b4f1dab68d27352b0f40d3556a87b4935e073bc59b0e38e1429

Hashes match: True
```

---

# What `Hashes match: True` Proved

It proved that:

> The exact canonical dataset representation Git was preparing to preserve matched the dataset fingerprint recorded inside the evidence report.

That restored the intended provenance relationship.

---

# Another Hash We Calculated

We also calculated SHA-256 for the generated evidence report itself.

For example, after one regeneration the report hash was:

```text
918FE4D3E59A86C03A07C29048CA98A76944AAAC470EBE25529877DCEB536A6B
```

This fingerprint can identify that particular generated report artifact.

However, if the report is regenerated or its contents change, its SHA-256 will also change.

---

# Why Earlier Report Hashes Changed

During this work, the evidence-report SHA-256 changed more than once.

This was expected because we changed:

- dataset text
- report metadata
- hashing behaviour

SHA-256 is sensitive to any byte-level change.

Therefore:

```text
changed file
    ↓
new bytes
    ↓
new SHA-256
```

This is expected behaviour, not instability in SHA-256.

---

# Another Related Issue: Stage-1 Hashes

Earlier, we also discovered that Stage-1 working-copy hashes differed from the hashes previously recorded during the project audit.

At first, that could have suggested that the files were different.

We investigated instead of assuming corruption.

The Windows Git configuration had:

```text
core.autocrlf=true
```

which converted line endings in the checked-out working copy.

When we hashed the exact Git objects directly from the frozen commit, the hashes matched the previously recorded canonical values.

For example:

```text
output_eval_dataset.jsonl
adbca1abe644698a1de34262353b40b0358df816a9d9bdcd3c9dfdeb7dbabb68

harness-tests.json
69b61558ee815be95c411f274f090aa7cc8b3475137d99dd21759e6445a1418c

system_prompt.txt
08d51fa123bcf73e088aceaf5ba71a427c4185af92f3b005fef14e8746dfc8d2
```

This showed that the canonical Git content was correct.

The discrepancy came from line-ending conversion in the Windows working copy.

---

# Why `.gitattributes` Matters

Reliora uses a `.gitattributes` file to define repository-level line-ending behaviour.

Its purpose includes normalizing repository text files to LF.

This makes the repository's intended representation explicit rather than relying only on each developer's global Git configuration.

A useful mental model is:

```text
Global Git settings
      +
Repository .gitattributes
      ↓
How Git handles files
```

Repository rules can make project behaviour more predictable across different developer machines.

---

# Why `.editorconfig` Is Related but Different

Reliora also uses:

```text
.editorconfig
```

This helps editors such as VS Code follow formatting conventions such as:

- character encoding
- indentation
- line endings

The distinction is roughly:

```text
.editorconfig
→ helps editors create consistent files

.gitattributes
→ tells Git how repository files should be handled
```

Both improve consistency, but they operate at different layers.

---

# Warning vs Error

This incident also taught an important debugging lesson.

Git initially displayed:

```text
warning:
CRLF will be replaced by LF
```

A warning is not automatically a failed command.

The `git add` operation succeeded.

However, the warning contained information that became significant because of the system's evidence requirements.

Therefore:

```text
warning
!=
automatic failure
```

but also:

```text
warning
!=
automatically safe to ignore
```

---

# Context Determines Severity

The same warning can have different importance depending on the system.

### Ordinary source file

CRLF/LF normalization may have little practical effect.

### Cryptographically hashed evidence dataset

CRLF/LF normalization changes bytes and therefore changes the SHA-256 fingerprint.

The correct engineering response is:

> Understand what the warning means in the context of the system.

---

# Troubleshooting Process We Followed

A useful troubleshooting pattern from this incident was:

```text
Observe warning
      ↓
Do not immediately assume failure or harmlessness
      ↓
Understand what Git is changing
      ↓
Identify whether byte identity matters
      ↓
Compare staged bytes with recorded evidence
      ↓
Confirm mismatch
      ↓
Define canonical representation
      ↓
Regenerate evidence
      ↓
Restage changed files
      ↓
Verify hashes again
      ↓
Hashes match: True
```

This is more reliable than guessing.

---

# Commands and Concepts Involved

## Check a File's SHA-256

PowerShell:

```powershell
(Get-FileHash "path\to\file" -Algorithm SHA256).Hash
```

### What It Does

Reads the file and computes its SHA-256 fingerprint.

### Why We Used It

To identify the generated evidence report.

### Side Effect

None. It is read-only.

---

## Stage Files

```powershell
git add "path/to/file"
```

### What It Does

Places the current version of the file into the Git staging area.

### Why It Mattered Here

The corrected dataset, runner, and report had to replace older staged versions.

---

## Inspect Staged Content

Git can reference staged content with:

```text
:path/to/file
```

This is useful when byte-for-byte verification must reflect exactly what Git is preparing to commit.

---

# Repository Files Related to This Incident

```text
reliora-ai-support-platform/
│
├── .gitattributes
├── .editorconfig
│
├── evals/
│   ├── datasets/
│   │   └── stage1-reproduction-v1.json
│   │
│   └── runners/
│       └── run_stage1_reproduction.py
│
└── evidence/
    └── generated/
        └── stage1-reproduction-v1-report.json
```

---

# Why Each File Matters

## `.gitattributes`

Defines repository-level Git handling rules, including LF normalization.

## `.editorconfig`

Helps editors create files using consistent text-formatting rules.

## `stage1-reproduction-v1.json`

The evaluation input whose identity must be traceable.

## `run_stage1_reproduction.py`

Defines the experiment execution and canonical dataset hashing logic.

## `stage1-reproduction-v1-report.json`

Records the experimental result and dataset fingerprint.

---

# Why This Is Important for Production Engineering

Evidence and reproducibility become increasingly important in systems involving:

- AI evaluation
- regulated workflows
- model validation
- audit trails
- release governance
- incident analysis
- scientific experiments
- security evidence

A production-quality evidence system should clearly define:

```text
what was run
which input was used
which code version was used
how artifacts were fingerprinted
what output was produced
```

---

# Why This Is Important for AI/ML Engineering

AI systems are probabilistic, so evaluation artifacts become especially important.

Without strong provenance, someone may ask:

> Which prompt version produced this result?

> Which dataset version was evaluated?

> Which model version was used?

> Has the evaluation dataset changed?

> Does this report still correspond to the current test set?

Cryptographic fingerprints are one tool for answering those questions.

---

# Important Lessons

1. Visually identical text can contain different bytes.
2. CRLF and LF produce different SHA-256 hashes.
3. Git may normalize line endings when staging files.
4. Working-copy bytes and staged Git bytes can therefore differ.
5. A warning is not automatically an error.
6. A warning should not automatically be ignored either.
7. When byte identity matters, verify the actual staged representation.
8. Evidence reports should define the exact basis used for hashing.
9. Canonicalization improves cross-platform reproducibility.
10. Editing a staged file requires restaging if the new version should enter the commit.
11. SHA-256 changes when file bytes change; that is expected.
12. Provenance is stronger when inputs, code, execution, and outputs can all be traced.

---

## Interview Explanation

> While implementing evaluation provenance in Reliora, I discovered that Windows CRLF working-copy line endings caused the reproduction runner to hash different bytes from the LF representation Git was preparing to commit. I verified the mismatch against the staged Git content instead of ignoring the warning, then defined a canonical UTF-8/LF hashing contract and documented the hash basis in the generated evidence report. After regenerating and restaging the artifacts, the staged dataset hash matched the report-recorded hash exactly. The experience reinforced that warnings have to be interpreted in system context, especially when byte-level provenance is part of the reliability design.