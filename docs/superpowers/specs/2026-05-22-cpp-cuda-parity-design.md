# C/C++/CUDA mapping & validation parity — design

- **Date:** 2026-05-22
- **Status:** Approved (pending written-spec review)
- **Scope:** Spec 1 of 2. Builds on the `cuda-support` branch (commit `158b8d5`,
  which added `.cu`/`.cuh` extension recognition and the `cuda` language/tag).

## Problem

The C/C++ feature mapper — which CUDA now rides on — is the least developed of
clawpatch's language mappers; its own docs call it "generic project shapes
only." Compared with the mature mappers (Go, Rust, Python, JVM, .NET, …) it is
missing the capabilities that make a language feel first-class:

1. **No default validation commands.** `languageDefaultCommands` in `detect.ts`
   has no C/C++/CUDA branch, so every C/C++/CUDA feature maps with
   `test/typecheck/lint/format: null`. `clawpatch fix` and `revalidate` then
   have nothing to run.
2. **No loose source-group slicing.** The mapper only seeds CMake/autotools
   targets and standalone `main()` files. Source files outside a parseable
   build target — e.g. a CUDA kernel library, or sources added via
   `file(GLOB …)` (whose `${VAR}` expansion the CMake parser cannot resolve) —
   get **zero review coverage**.
3. **No build-config feature.** The config mapper maps `Makefile` but not
   `CMakeLists.txt`.
4. **CUDA features carry no `concurrency` trust boundary**, even though kernels
   are parallel by definition and the review prompt already treats
   `concurrency` and "race/concurrency bugs" as first-class.

## Decisions taken during brainstorming

- **Scope:** lift the whole C/C++/CUDA path together — the parity gap is the
  C/C++ base, not a CUDA-specific veneer.
- **Decomposition:** two specs. This one (Spec 1) closes the coverage and
  operational gaps. Spec 2 (later) adds CUDA semantic depth.
- **Validation commands:** conservative — clawpatch emits a command only when
  the project itself declares it: a `Makefile` `check`/`test` target, or a
  `CMakePresets.json` build workflow. No declared workflow → `null`, matching
  the .NET/autotools stance. No build command is ever invented.
- **Structure:** the new source-group logic lives in a sibling module, not
  inside the already-largest mapper file.

## Components

### 1. Source-group slicing — `src/mappers/c-cpp-groups.ts` (new)

`cCppSeeds` keeps its autotools → cmake → standalone-`main()` passes, then calls
a new `cCppGroupSeeds(root, sourceFiles, ownedPaths)`:

- `sourceFiles` — the `isCOrCppSource` files already discovered by `cCppSeeds`.
- `ownedPaths` — a `Set<string>` of every path owned by prior seeds
  (`entryPath` plus `ownedFiles[].path`). The current `alreadySeeded` set only
  tracks cli-commands and cmake-tests; it is widened so library/test seeds are
  also covered.
- **Residual files** = `sourceFiles` minus `ownedPaths`, minus
  `isCOrCppTestPath` files. (Sample/dependency paths are already excluded by the
  `cCppSeeds` walk.)
- Residual files are grouped by top-level directory; each directory's files are
  partitioned with the existing `partitionFileGroups` helper at **12 files per
  group** (the constant Node uses). Files at the repo root form a root group.
- Each `FileGroup` becomes one seed:
  - `kind: "library"`, `source: "c-cpp-group"`, `confidence: "low"`
  - `title`: `"CUDA source group <label>"` if the group contains any
    `.cu`/`.cuh` file, else `"C/C++ source group <label>"`
  - `summary`: notes these are source files not owned by a build target
  - `entryPath`: first file in the sorted group
  - `symbol`: the group label — the stable identity component of the feature ID
    (the same approach `node-source-group` uses; the ID is keyed on
    `kind + source + entryPath + symbol`, and a non-null `symbol` also keeps the
    library-collision disambiguator from rewriting it)
  - `route/command`: `null`
  - `tags`: the set of `languageTag` values present in the group, plus
    `"source-group"`
  - `trustBoundaries`: `["filesystem"]`, plus `"concurrency"` if any `.cu`/`.cuh`
    file is present
  - `ownedFiles`: every group file, reason `"source group member"`
- Group seeds pass through the existing `dedupeByEntry` in `cCppSeeds`. The
  distinct `source: "c-cpp-group"` prevents collisions with target seeds.

This closes the coverage gap: globbed CUDA kernel libraries and loose `.cpp`/
`.cu` trees with no `main()` become reviewable.

### 2. Validation commands — `detect.ts` `languageDefaultCommands`

A new `c`/`cpp`/`cuda` branch, evaluated last (before the all-`null` return),
delegating to a `cOrCppDefaultCommands(root)` helper:

- **Root plain `Makefile` present** → `typecheck: "make"`; `test: "make check"`
  if the Makefile declares a `check:` target, else `"make test"` if it declares
  a `test:` target, else `null`.
- **Else a root `CMakePresets.json`** declaring an unambiguous build workflow:
  - exactly one `workflowPresets` entry → `typecheck:
"cmake --workflow --preset <name>"`;
  - else exactly one `configurePresets` entry **and** one `buildPresets` entry →
    `typecheck: "cmake --preset <cfg> && cmake --build --preset <build>"`;
  - exactly one `testPresets` entry → `test: "ctest --preset <name>"`;
  - anything ambiguous (multiple competing presets, or none of the above) →
    `null`. `CMakeUserPresets.json` is ignored — it is a user-local, typically
    gitignored file and must not drive a shared command.
- **Otherwise** (a `CMakeLists.txt` with no `CMakePresets.json`, autotools-only
  `Makefile.am`/`Makefile.in`, or no clear build root) → all `null`.
- `lint` and `format` → always `null` for Spec 1. Project-wide C/C++
  lint/format has no reliable invocation without a compile database; deferred.

clawpatch never invents a CMake invocation — it emits one only when the project
declares the workflow itself, the same rule it already applies to `package.json`
scripts and Nx/Turbo targets. The `Makefile` target check is a regex on the raw
file (a commented-out `check:` is an accepted edge case); `CMakePresets.json` is
parsed as JSON. First-match-wins in `languageDefaultCommands` is unchanged: a
polyglot repo (e.g. Rust + some C) keeps the higher-priority language's
commands.

### 3. `CMakeLists.txt` config feature — `config.ts`

Add `"CMakeLists.txt"`, `"CMakePresets.json"`, and `"configure.ac"` to the
config mapper's `candidates` list. Like `Cargo.toml`, these are matched at the
repository root only. Build files become reviewable `config` features.

### 4. CUDA `concurrency` trust boundary — `c-cpp.ts`

A helper `withCudaConcurrency(boundaries, tag)` returns `boundaries` with
`"concurrency"` appended (deduped) when `tag === "cuda"`. It is applied at every
seed-creation site in `c-cpp.ts` that sets `trustBoundaries` with a known
language tag: `c-main`, cmake bin/lib, cmake test, autotools bin/lib. Source
groups apply the same rule directly in `c-cpp-groups.ts`. `"concurrency"` is
already a valid `TrustBoundary`; no prompt or schema change is needed — the
generic review prompt consumes it.

## Data flow

```
cCppSeeds: walk → autotools pass → cmake pass → main pass → group pass → dedupe
detectProject → detectCommands → languageDefaultCommands (+ C/C++/CUDA branch)
config mapper → picks up root CMakeLists.txt / configure.ac
```

No changes to the mapper registry, the review prompt, or the feature schema.

## Testing (TDD)

Failing-first tests added to `src/mapper.test.ts`, alongside the existing CUDA
tests:

1. A residual C++ source directory (no `main()`, no CMake target) produces a
   `"C/C++ source group"` feature owning those files.
2. A source group excludes files already owned by a CMake `add_executable` /
   `add_library` target.
3. A residual `.cu`/`.cuh` directory produces a `"CUDA source group"` feature
   tagged `cuda` with a `concurrency` trust boundary.
4. A root `Makefile` with a `check:` target →
   `detected.commands.typecheck === "make"`, `test === "make check"`.
5. A root `CMakePresets.json` with one `workflowPresets` entry →
   `detected.commands.typecheck === "cmake --workflow --preset <name>"`.
6. A root `CMakeLists.txt` with `project()` but no `CMakePresets.json` (and no
   root `Makefile`) → C/C++ validation commands all `null`; likewise a
   `CMakePresets.json` with multiple competing presets → `null`.
7. An autotools-only repo (`Makefile.am`, no `Makefile`) → C/C++ validation
   commands all `null`.
8. A root `CMakeLists.txt` produces a `"Project config CMakeLists.txt"` feature.
9. A CUDA CMake binary has `concurrency` in its `trustBoundaries`.

Whole-suite `vitest run`, `tsc --noEmit`, `oxlint`, and `oxfmt --check` must all
pass.

## Risks

- **Source-group volume is structural, not a defect.** Groups are residual-only
  — they cover only files no CMake/autotools/`main()` target already owns — so
  group count scales inversely with build-file quality and is near zero in a
  well-structured repo. Each group is bounded at 12 files and split by
  directory/filename family by `partitionFileGroups`, the same shape Node, JVM,
  and .NET groups already ship. `confidence: "low"` labels them honestly. No
  extra gate is warranted.
- **No invented build commands.** CMake commands come only from a project's own
  `CMakePresets.json`; a preset's configured `binaryDir` is generated by the
  project's declared workflow, not by a command clawpatch guessed, and
  clawpatch's `shouldSkip` already ignores `build`-named directories. A project
  with no declared workflow gets `null` — the honest result, and never a flaky
  invented configure step.

## Out of scope — Spec 2

`__global__` kernel-aware slicing, a per-language review-guidance hook,
`compute-sanitizer`-style validation, and C/C++ `lint`/`format` commands.

## Documentation to update on implementation

`README.md` ("What It Maps Today") and `docs/feature-mapping.md` (C/C++ section),
plus a `CHANGELOG.md` entry under `0.3.1 - Unreleased`.
