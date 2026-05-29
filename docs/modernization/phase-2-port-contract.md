# Phase 2 Port Contract — Per-Class Porting Workflow

| Field          | Value                                                                                 |
|----------------|---------------------------------------------------------------------------------------|
| Status         | Active reference                                                                      |
| Date           | 2026-05-29                                                                            |
| Author/Model   | claude-sonnet-4-6 (Claude Code agent, UPR worker, story UPR-14)                      |
| Repo           | https://github.com/TimvanderWal504/universal-pokemon-randomizer-fvx                  |
| Branch         | upr/UPR-14                                                                            |
| Commit SHA     | (see Resume Point — SHA recorded after commit)                                        |
| Related issues | UPR-2 (parent epic), UPR-7 (target-language ADR), UPR-8 (test harness), UPR-10 (hazard checklist), UPR-12 (phase 2 planning), UPR-14 (this document) |
| Jira epic      | UPR-2 — AI Modernisation                                                              |

---

## Purpose

Phase 2 contains many structurally identical porting stories — one per `utils`
class.  This document is the workflow contract that every per-class story
(2.4.x and later) must follow.  It is self-contained: an agent dispatched on an
arbitrary 2.4.x story with no prior context can execute it using only this
document plus the story description and the references it links.

Do not re-state the hazard checklist here.  The definitive list is in
**UPR-10** (`docs/upr/UPR-10-porting-notes.md`), and that document is the
Phase 2 preamble that every worker must read before writing any code.

---

## 1. Workflow

The following numbered steps take a story from "dispatched" to "branch ready
for human merge."  Execute them in order; do not skip steps.

### Step 1 — Read the story description in full

Before touching any file, read the complete Jira story (passed to you as
context by the orchestrator).  Note:

- The exact Java class name and its package path (e.g.
  `utils/src/main/java/compressors/DSCmp.java`).
- The target C# namespace and file path (e.g.
  `port/src/UniversalPokemonRandomizer.Core/Compressors/DSCmp.cs`).
- Which public methods the story asks you to port (may be all, or a subset).
- The fixture directory names the tests will load (e.g. `DSCmp_LZ10`,
  `DSCmp_LZ11`).  These are listed under "Acceptance criteria" or "Scope".

### Step 2 — Read UPR-10 Phase 2 preamble before writing any code

Open `docs/upr/UPR-10-porting-notes.md` and read:

1. The "Phase 2 Checklist / Prompt Preamble" section — copy it verbatim into
   your working context before generating any C# code.
2. The six "Semantic Hazards" sections — internalise the unsigned-byte masking
   rule, nibble recombination, integer overflow, LZ11 nibble-mask dispatch,
   off-by-one guard, and endianness hazard.

Do not skip this step.  An agent that ports code without reading UPR-10 will
reproduce the known Java-to-C# semantic bugs the checklist was written to
prevent.

### Step 3 — Read the Java source class in full

Open the Java source file at the path identified in Step 1.  For each method
to be ported:

1. Identify every `byte[]` read site.  Note which reads feed arithmetic
   expressions (those require an `int` intermediate in C#).
2. Identify any inner-loop break guards — apply the `>=` correction if the
   Java source uses `>` against the buffer length (see UPR-10, Hazard 5).
3. Note all method signatures.  Public methods called by the test file must be
   `public` in C# (see UPR-10, AI Workflow — Systematic Mistakes).

### Step 4 — Create the C# source file

Create a new file at:

```
port/src/UniversalPokemonRandomizer.Core/<Package>/<ClassName>.cs
```

Where `<Package>` is the C# equivalent of the Java package (e.g. the Java
`compressors` package maps to the `Compressors` directory).  Use PascalCase
for directory names.

Rules:
- Follow the structural template of the reference port:
  `port/src/UniversalPokemonRandomizer.Core/Compressors/DSDecmp.cs`
  (commit `da5dfa5b` on branch `upr/UPR-9`).
- Apply every rule in the UPR-10 Phase 2 Checklist on every line.
- Do not port methods outside the story's scope.
- Do not create any other files at this step.

### Step 5 — Add the xUnit differential test file

Create a new test file at:

```
port/tests/UniversalPokemonRandomizer.Tests/Fixtures/<ClassName>DifferentialTests.cs
```

The file must follow the pattern established by
`port/tests/UniversalPokemonRandomizer.Tests/Fixtures/DSDecmpDifferentialTests.cs`.
Specifically:

1. One `*Fixtures` static class per fixture directory, using
   `FixtureLoader.LoadForFunction("<FunctionName>")` where `<FunctionName>`
   matches the fixture directory name exactly (e.g. `"DSCmp_LZ10"`).
2. One `*_DifferentialTests` sealed class per function, with a single
   `[Theory]` / `[MemberData]` test method that calls the ported C# method
   and passes the output to `DifferentialAssert.BytesEqual`.
3. The test method name must follow the pattern
   `<MethodName>_MatchesGoldenVector` (e.g. `CompressLZ10_MatchesGoldenVector`).
4. Do not add stub tests (CorrectStub / WrongStub).  Those live only in
   `DifferentialTests.cs` and are not duplicated per class.

### Step 6 — Run `dotnet test port/`

This is the definition-of-done command.  Run it and capture the full output.

```
dotnet restore port/ --configfile port/NuGet.config
dotnet test port/
```

Expected passing state — see Section 2 (Definition of Done) for the exact
description of what output constitutes a pass.

### Step 7 — Commit on `upr/<ISSUE-KEY>` and report back

Stage only the two files created in Steps 4 and 5.  Commit with a message
describing the class ported.  Never stage or commit files outside those two.

Report back to the orchestrator with:

- Branch name and tip commit SHA.
- A one-paragraph summary of what was ported.
- The exact `dotnet test` output (copy-pasted, not paraphrased).
- Any deviations, assumptions, or open questions.

---

## 2. Definition of Done

### Verification command

```bash
dotnet restore port/ --configfile port/NuGet.config
dotnet test port/
```

### Passing output

A story is done when `dotnet test port/` exits 0 and the output satisfies all
of the following:

1. **All `<ClassName>` fixture tests pass.** Every test in the newly added
   `<ClassName>DifferentialTests.cs` file is listed as `Passed`.  For example,
   after porting `DSCmp` with LZ10 and LZ11 support:
   - `DSCmp_LZ10_DifferentialTests.CompressLZ10_MatchesGoldenVector` —
     all cases pass.
   - `DSCmp_LZ11_DifferentialTests.CompressLZ11_MatchesGoldenVector` —
     all cases pass.

2. **Wrong-stub failures are unchanged.** The 6 intentional failures from the
   UPR-8 wrong-stub harness tests (`DSCmp_LZ10_WrongStubTests.WrongStub_Fails`)
   remain.  These are expected failures — the test suite is designed to report
   them as failures.  Any change to this count indicates a problem:
   - More than 6 wrong-stub failures: a new wrong-stub was accidentally added,
     or an existing passing test broke.
   - Fewer than 6: a wrong-stub test was accidentally modified or deleted.

3. **Total count increases by the right amount.** If the ported class adds
   `N` LZ10 fixture cases and `M` LZ11 fixture cases, the `Passed` count
   increases by exactly `N + M` compared to the pre-story baseline.

Example passing summary line (after adding DSCmp with 6 LZ10 + 6 LZ11
fixtures):

```
Failed!  - Failed: 6, Passed: 38, Skipped: 0, Total: 44, Duration: ...
```

(The `Failed!` label is correct — those 6 are intentional wrong-stub proofs.)

The current baseline before any 2.4.x story runs:
- Failed: 6 (intentional wrong-stub, DSCmp_LZ10)
- Passed: 26
- Total: 32

### Hazard-checklist confirmation

Before declaring done, mentally (or literally) tick each item in the
**Phase 2 Checklist** from `docs/upr/UPR-10-porting-notes.md`:

- [ ] Every `byte` read that feeds arithmetic uses an `int` intermediate.
- [ ] No arithmetic intermediate is stored back into a `byte` before a shift
  or OR.
- [ ] Inner-loop break guard uses `>=` not `>` against buffer length.
- [ ] All methods called by the test file are `public`.
- [ ] C# naming conventions applied (PascalCase methods, camelCase locals).
- [ ] `dotnet test port/` output captured and shows the correct pass/fail
  counts (real output, not self-certified).

If any item is unchecked, the story is not done.

---

## 3. File-Ownership Boundaries

Each per-class story owns exactly two files.

### MAY touch (new files only)

| File | Description |
|------|-------------|
| `port/src/UniversalPokemonRandomizer.Core/<Package>/<ClassName>.cs` | The C# port of the Java class |
| `port/tests/UniversalPokemonRandomizer.Tests/Fixtures/<ClassName>DifferentialTests.cs` | The xUnit Theory tests for the ported class |

Both files must be **new** (not modifications of existing files).

### MUST NOT touch

| File / Directory | Reason |
|-----------------|--------|
| `utils/fixtures/` (any subdirectory) | Fixture corpus is frozen. Adding, removing, or modifying `.in` or `.out` files invalidates the differential baseline for all stories. |
| `port/tests/UniversalPokemonRandomizer.Tests/Fixtures/DifferentialTests.cs` | This file contains the shared stub tests and harness types. It is modified only by the story that owns the harness. |
| `port/tests/UniversalPokemonRandomizer.Tests/Fixtures/DSDecmpDifferentialTests.cs` | Owned by UPR-9. Do not modify. |
| Any other existing C# source or test file | Modifying an existing file is out of scope for a per-class porting story. |
| The Java source tree (`utils/src/`) | Java source is read-only reference material. Never modify it. |
| `port/src/.../DSDecmp.cs` (or any already-ported class) | Each class is ported once. Refactoring is a separate story. |
| `.claude/`, `docs/upr/`, `docs/modernization/` | Documentation stories own documentation files. Porting stories do not. |
| `port/UniversalPokemonRandomizer.sln`, `port/**/*.csproj` | Project structure changes require a separate story and human review. |

If you find yourself needing to touch a file outside the two owned files,
stop and raise it as a deviation in your report rather than proceeding.

---

## 4. Disqualifiers

The following are automatic failure conditions for a porting story.  A story
exhibiting any of these cannot be marked Done, regardless of apparent output.

### D1 — Modifying or skipping a fixture

Any change to a file under `utils/fixtures/` — adding, removing, renaming, or
altering a `.in` or `.out` file — disqualifies the story.  The fixture corpus
is immutable across all Phase 2 porting stories.  If the corpus is missing
coverage for a specific path, that is an open question to escalate, not a
reason to add fixtures mid-story.

### D2 — Self-certifying without real `dotnet test` output

Stating "all tests pass" or "the implementation is correct" without running
`dotnet test port/` and capturing its output is a disqualifier.  The
verification command must be run in the working environment, and the actual
terminal output (including pass/fail counts) must be included in the report.
Prose assertions, code review, or reasoning about correctness do not substitute
for running the command.

### D3 — Touching files outside the story's class scope

Committing changes to any file other than the two owned files (Section 3)
disqualifies the story.  This includes refactoring existing code, adjusting
project files, updating documentation, or making "while I was here" fixes.
Those belong in separate stories.

### D4 — Porting a different class than specified

The story dispatches you to port a specific Java class (e.g. `DSCmp.java`).
Porting a different class, or porting additional classes not named in the story,
disqualifies the story.  If the story's scope is ambiguous, raise the ambiguity
in the report rather than guessing.

### D5 — Using `sbyte` in ported code

C#'s `sbyte` type (signed byte, -128..127) mirrors Java's `byte` semantics and
reintroduces all the sign-extension hazards that UPR-10 documents.  Any
occurrence of `sbyte` in ported code is an automatic disqualifier.  Use `byte`
(unsigned) and `int` intermediates throughout.

### D6 — Failing to read UPR-10 before writing code

An agent that generates C# without first loading the UPR-10 Phase 2 preamble
into its context cannot confirm it applied the hazard checklist.  If the
report does not reference UPR-10 and confirm the checklist was applied, the
story is treated as unverified.

---

## 5. Worked Example — DSCmp (LZ10 and LZ11 Compressor)

This section walks through the upcoming 2.4.x story for `DSCmp` on paper to
illustrate how the workflow above applies to a real class.  No code is written
here; the purpose is to demonstrate how a worker agent should reason before
touching a file.

### Java source location and key method signatures

File: `utils/src/main/java/compressors/DSCmp.java`
Package: `compressors`

Public methods to port (from the Java source):

```java
// Compresses decompressed bytes using LZ10 format.
// Returns the compressed byte array including the 0x10 header byte and
// 3-byte (or 4-byte extended) little-endian decompressed length.
public static byte[] compressLZ10(byte[] decompressed)

// Compresses decompressed bytes using LZ11 format.
// Returns the compressed byte array including the 0x11 header byte and
// 3-byte (or 4-byte extended) little-endian decompressed length.
public static byte[] compressLZ11(byte[] decompressed)
```

Private helpers (referenced internally; must be ported as private static):

```java
// Returns { matchLength, matchDisplacement } for a back-reference at `current`.
private static int[] getOccurenceInfo(byte[] decompressed, int current, int pending, int finished)

// Returns the 4-byte little-endian representation of val.
private static byte[] getBytes(int val)

// Returns the 3-byte little-endian representation of val.
private static byte[] getBytes24(int val)
```

Constants (translate as `public const int`):

```java
public static final int LZ10  = 0x10;
public static final int LZ11  = 0x11;
public static final int HUFF4 = 0x24;
public static final int HUFF8 = 0x28;
public static final int RLE   = 0x30;
```

### C# file to create

```
port/src/UniversalPokemonRandomizer.Core/Compressors/DSCmp.cs
```

Namespace: `UniversalPokemonRandomizer.Core.Compressors`
Class declaration: `public static class DSCmp`

Note: `compressLZ10` → `CompressLZ10`, `compressLZ11` → `CompressLZ11` (PascalCase).
The private helpers `getOccurenceInfo`, `getBytes`, `getBytes24` → `GetOccurrenceInfo`,
`GetBytes`, `GetBytes24` (PascalCase, fix spelling while renaming).

### UPR-10 hazards that apply to DSCmp

Unlike DSDecmp (a decompressor that reads compressed bytes), DSCmp is a
**compressor** that reads raw (decompressed) bytes and writes compressed bytes.
The byte-arithmetic hazards from UPR-10 still apply but manifest differently:

1. **Unsigned-byte masking** — `decompressed[curIn]` reads are used to write
   flag bytes and back-reference tokens.  The cast `(byte)(...)` is used
   extensively to truncate intermediate `int` values back to bytes for the
   output buffer.  Confirm each cast is intentional truncation, not an
   accidental sign error.

2. **Nibble recombination** — LZ10 packs length and displacement into two bytes
   as nibbles: `((occLength - 3) & 0xF) << 4` for the high nibble of byte 0,
   and `((occDisp - 1) >> 8) & 0xF` for the low nibble of byte 0.  Verify
   these remain as `int` intermediates before the final `(byte)` cast for the
   output buffer write.

3. **No inner-loop off-by-one** — DSCmp's main loop uses `curIn < decompressed.length`
   as its while condition, which is correct.  No `>` vs `>=` fix is needed here,
   but confirm after reading the source.

4. **`ByteArrayOutputStream` → `List<byte>` or `MemoryStream`** — Java's
   `ByteArrayOutputStream` with `write(int)` and `write(byte[], int, int)` maps
   to either a `List<byte>` (with `Add` and `AddRange`) or a `MemoryStream`.
   Using a `List<byte>` is idiomatic for this pattern; `MemoryStream` is an
   alternative.  Either is acceptable; choose one and apply consistently.

### Fixture directories to wire up

The fixture corpus (frozen in `utils/fixtures/`) contains:

```
utils/fixtures/DSCmp_LZ10/   ← 6 golden vector pairs (.in / .out)
utils/fixtures/DSCmp_LZ11/   ← 6 golden vector pairs (.in / .out)
```

The test file must load both:

```csharp
FixtureLoader.LoadForFunction("DSCmp_LZ10")
FixtureLoader.LoadForFunction("DSCmp_LZ11")
```

The `.in` file is the **decompressed** data (input to the compressor).
The `.out` file is the **compressed** data (expected output).

### Test method skeleton

```csharp
// File: port/tests/UniversalPokemonRandomizer.Tests/Fixtures/DSCmpDifferentialTests.cs

using UniversalPokemonRandomizer.Core.Compressors;

namespace UniversalPokemonRandomizer.Tests.Fixtures;

public static class DSCmp_LZ10Fixtures
{
    private static readonly IReadOnlyList<FixtureCase> Cases =
        FixtureLoader.LoadForFunction("DSCmp_LZ10");

    public static IEnumerable<object[]> All =>
        Cases.Select(c => new object[] { c });
}

public static class DSCmp_LZ11Fixtures
{
    private static readonly IReadOnlyList<FixtureCase> Cases =
        FixtureLoader.LoadForFunction("DSCmp_LZ11");

    public static IEnumerable<object[]> All =>
        Cases.Select(c => new object[] { c });
}

public sealed class DSCmp_LZ10_DifferentialTests
{
    [Theory]
    [MemberData(nameof(DSCmp_LZ10Fixtures.All), MemberType = typeof(DSCmp_LZ10Fixtures))]
    public void CompressLZ10_MatchesGoldenVector(FixtureCase fixture)
    {
        var input    = File.ReadAllBytes(fixture.InPath);   // decompressed bytes
        var expected = File.ReadAllBytes(fixture.OutPath);  // compressed bytes

        var actual = DSCmp.CompressLZ10(input);

        DifferentialAssert.BytesEqual(fixture, expected, actual);
    }
}

public sealed class DSCmp_LZ11_DifferentialTests
{
    [Theory]
    [MemberData(nameof(DSCmp_LZ11Fixtures.All), MemberType = typeof(DSCmp_LZ11Fixtures))]
    public void CompressLZ11_MatchesGoldenVector(FixtureCase fixture)
    {
        var input    = File.ReadAllBytes(fixture.InPath);   // decompressed bytes
        var expected = File.ReadAllBytes(fixture.OutPath);  // compressed bytes

        var actual = DSCmp.CompressLZ11(input);

        DifferentialAssert.BytesEqual(fixture, expected, actual);
    }
}
```

Note: `DSCmp_LZ10Fixtures` and `DSCmp_LZ11Fixtures` already exist in
`DifferentialTests.cs` (UPR-8 harness), which defines them as data sources for
the wrong-stub tests.  The real implementation test classes above are **new and
separate**; they reference the same fixture loader but are not the same class.

Caution: inspect `DifferentialTests.cs` before writing to confirm there is no
naming collision.  If `DSCmp_LZ10Fixtures` already exists there, use a
distinct name such as `DSCmp_LZ10ImplementationFixtures` or a nested namespace
to avoid a duplicate-class compile error.

### Expected `dotnet test` result

After implementing `DSCmp.cs` and adding the test file above, the expected
summary from `dotnet test port/` is:

```
Failed!  - Failed: 6, Passed: 38, Skipped: 0, Total: 44, Duration: ...
```

Breakdown:
- 6 intentional wrong-stub failures (DSCmp_LZ10 wrong-stub, unchanged from baseline)
- 26 pre-existing passes (baseline: 17 DSDecmp differential + 6 DSCmp_LZ10 correct-stub + 3 harness)
- 6 new passes: `DSCmp_LZ10_DifferentialTests` (6 fixtures)
- 6 new passes: `DSCmp_LZ11_DifferentialTests` (6 fixtures)
- Total new = 12; 26 + 12 = 38 passing

If `dotnet test` shows a different failure count or fewer passing tests,
the implementation has a bug — do not mark the story Done.

---

## Resume Point

An agent picking up a Phase 2 porting story (UPR-14 or any 2.4.x) after this
document should:

1. Read this document from the top.
2. Check out or create the branch `upr/<ISSUE-KEY>` in a dedicated worktree.
3. Read the story description to identify the class, file paths, and fixture
   directories.
4. Follow Sections 1–4 of this document as the step-by-step guide.
5. The reference port is `port/src/UniversalPokemonRandomizer.Core/Compressors/DSDecmp.cs`
   (commit `da5dfa5b` on branch `upr/UPR-9`).
6. The hazard checklist is in `docs/upr/UPR-10-porting-notes.md` — read it
   before writing any C# code.
7. Run `dotnet restore port/ --configfile port/NuGet.config && dotnet test port/`
   and confirm the pass/fail counts match the definition of done.

Current repo HEAD on master: `88d92e38` (UPR-10 porting notes added).
Baseline test state: Failed 6 (intentional), Passed 26, Total 32.
