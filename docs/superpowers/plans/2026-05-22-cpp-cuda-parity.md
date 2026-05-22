# C/C++/CUDA Mapping & Validation Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring clawpatch's C/C++/CUDA feature mapping up to par with the mature language mappers — cover loose source files, emit conservative validation commands, map build-config files, and tag CUDA features with the `concurrency` trust boundary.

**Architecture:** Four independent changes on the `cuda-support` branch. C/C++/CUDA language-classification helpers are centralised in `src/mappers/shared.ts`; a new `src/mappers/c-cpp-groups.ts` adds residual source-group slicing; `src/detect.ts` gains a conservative C/C++/CUDA validation-command branch driven only by project-declared workflows; `src/mappers/config.ts` learns the build-config files.

**Tech Stack:** TypeScript (ESM, `.js` import specifiers), Vitest, oxlint, oxfmt. Spec: `docs/superpowers/specs/2026-05-22-cpp-cuda-parity-design.md`.

**Branch:** Continue on `cuda-support` (it already carries the spec and the extension-level CUDA support this builds on).

**Commands:** This environment has no global `pnpm`; every command below is run via `npx --yes pnpm@latest exec …`. Test steps run the whole `src/mapper.test.ts` file (≈7s) rather than name filters, because several intended test names share substrings with existing tests; read the per-test `✓`/`✗` lines in the output for the named test.

---

### Task 1: Centralise C/C++/CUDA classification + CUDA `concurrency` boundary

Moves the language-classification helpers into `shared.ts` so the new groups module can reuse them, adds a `withCudaConcurrency` helper, and tags every CUDA feature with the `concurrency` trust boundary.

**Files:**
- Modify: `src/mappers/shared.ts` (add classification helpers)
- Modify: `src/mappers/c-cpp.ts` (drop the local helpers, import from shared, apply the boundary)
- Test: `src/mapper.test.ts`

- [ ] **Step 1: Write the failing test**

Add this test in `src/mapper.test.ts` immediately after the existing `it("detects CUDA projects from .cu sources", …)` test:

```ts
  it("tags CUDA build targets with the concurrency trust boundary", async () => {
    const root = await fixtureRoot("clawpatch-cuda-concurrency-");
    await writeFixture(
      root,
      "CMakeLists.txt",
      "project(gpuapp CUDA)\nadd_executable(gpuapp src/main.cu)\n",
    );
    await writeFixture(root, "src/main.cu", "int main(void) { return 0; }\n");

    const project = await detectProject(root);
    const result = await mapFeatures(root, project, []);
    const gpuapp = result.features.find((feature) => feature.title === "CMake binary gpuapp");

    expect(gpuapp?.trustBoundaries).toContain("concurrency");
  });
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: `tags CUDA build targets with the concurrency trust boundary` FAILS — `gpuapp.trustBoundaries` is `["user-input","filesystem","process-exec"]`, missing `"concurrency"`. All other tests pass.

- [ ] **Step 3: Move the classification helpers into `shared.ts`**

In `src/mappers/c-cpp.ts`, delete this block (added by the earlier CUDA commit, just below `isCMake`):

```ts
type LanguageTag = "c" | "cpp" | "cuda";

function languageTag(path: string): LanguageTag {
  if (/\.cuh?$/iu.test(path)) {
    return "cuda";
  }
  return /\.(?:C|H)$/u.test(path) || /\.(?:cc|cpp|cxx|hh|hpp|hxx)$/iu.test(path) ? "cpp" : "c";
}

function languageLabel(tag: LanguageTag): string {
  return tag === "cuda" ? "CUDA" : tag === "cpp" ? "C++" : "C";
}
```

In `src/mappers/shared.ts`, add this block immediately after the `isCOrCppTestPath` function (`TrustBoundary` is already imported at the top of the file):

```ts
export type LanguageTag = "c" | "cpp" | "cuda";

export function languageTag(path: string): LanguageTag {
  if (/\.cuh?$/iu.test(path)) {
    return "cuda";
  }
  return /\.(?:C|H)$/u.test(path) || /\.(?:cc|cpp|cxx|hh|hpp|hxx)$/iu.test(path) ? "cpp" : "c";
}

export function languageLabel(tag: LanguageTag): string {
  return tag === "cuda" ? "CUDA" : tag === "cpp" ? "C++" : "C";
}

export function withCudaConcurrency(
  boundaries: TrustBoundary[],
  tag: LanguageTag,
): TrustBoundary[] {
  if (tag !== "cuda" || boundaries.includes("concurrency")) {
    return boundaries;
  }
  return [...boundaries, "concurrency"];
}
```

In `src/mappers/c-cpp.ts`, update the `./shared.js` import to add the three new names (keep the existing names):

```ts
import {
  isSafeFile,
  isCOrCppTestPath,
  isSampleProjectPath,
  languageLabel,
  languageTag,
  normalize,
  packageTrustBoundaries,
  shouldSkip,
  stripLineComments,
  walk,
  withCudaConcurrency,
} from "./shared.js";
```

- [ ] **Step 4: Run the full suite to verify the move is behaviour-preserving**

Run: `npx --yes pnpm@latest exec vitest run`
Expected: every test passes **except** `tags CUDA build targets with the concurrency trust boundary`, which still FAILS (the helper exists but is not applied yet). A behaviour-preserving move leaves all pre-existing tests green.

- [ ] **Step 5: Apply `withCudaConcurrency` at every C/C++ seed site**

In `src/mappers/c-cpp.ts`, wrap the `trustBoundaries` value at each of the six seed-creation sites. Find each by its `source:` field; the transformation depends on the current `trustBoundaries` expression:

| Seed `source` | Current `trustBoundaries:` | New `trustBoundaries:` |
|---|---|---|
| `autotools-bin` | `["user-input", "filesystem", "process-exec"]` | `withCudaConcurrency(["user-input", "filesystem", "process-exec"], tag)` |
| `autotools-lib` | `packageTrustBoundaries(target)` | `withCudaConcurrency(packageTrustBoundaries(target), tag)` |
| `cmake-test` | `[]` | `withCudaConcurrency([], languageTag(testEntryPath))` |
| `cmake-bin` | `["user-input", "filesystem", "process-exec"]` | `withCudaConcurrency(["user-input", "filesystem", "process-exec"], tag)` |
| `cmake-lib` | `packageTrustBoundaries(target)` | `withCudaConcurrency(packageTrustBoundaries(target), tag)` |
| `c-main` | `["user-input", "filesystem", "process-exec"]` | `withCudaConcurrency(["user-input", "filesystem", "process-exec"], tag)` |

For `autotools-bin`, `autotools-lib`, `cmake-bin`, `cmake-lib`, and `c-main` a local `const tag = languageTag(...)` already exists in scope. The `cmake-test` seed has no `tag` variable — pass `languageTag(testEntryPath)` directly as shown.

- [ ] **Step 6: Run the tests to verify they pass**

Run: `npx --yes pnpm@latest exec vitest run`
Expected: PASS — all tests green, including the new concurrency test.

- [ ] **Step 7: Format and commit**

```bash
npx --yes pnpm@latest exec oxfmt --write src/mappers/shared.ts src/mappers/c-cpp.ts src/mapper.test.ts
git add src/mappers/shared.ts src/mappers/c-cpp.ts src/mapper.test.ts
git commit -m "Tag CUDA features with the concurrency trust boundary

Centralise C/C++/CUDA language classification in shared.ts and apply a
withCudaConcurrency helper at every C/C++ seed site so CUDA targets carry
the concurrency trust boundary the review prompt already consumes.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Map build-config files as config features

**Files:**
- Modify: `src/mappers/config.ts`
- Test: `src/mapper.test.ts`

- [ ] **Step 1: Write the failing test**

Add this test in `src/mapper.test.ts` after the test added in Task 1:

```ts
  it("maps CMake and autotools build files as config features", async () => {
    const root = await fixtureRoot("clawpatch-build-config-");
    await writeFixture(root, "CMakeLists.txt", "project(app CXX)\nadd_executable(app main.cpp)\n");
    await writeFixture(root, "CMakePresets.json", '{"version":6}\n');
    await writeFixture(root, "configure.ac", "AC_INIT([app],[1.0])\n");
    await writeFixture(root, "main.cpp", "int main(void) { return 0; }\n");

    const project = await detectProject(root);
    const result = await mapFeatures(root, project, []);
    const titles = result.features.map((feature) => feature.title);

    expect(titles).toContain("Project config CMakeLists.txt");
    expect(titles).toContain("Project config CMakePresets.json");
    expect(titles).toContain("Project config configure.ac");
  });
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: `maps CMake and autotools build files as config features` FAILS — none of the three `Project config …` titles are present. All other tests pass.

- [ ] **Step 3: Add the build-config files to the candidate list**

In `src/mappers/config.ts`, the `candidates` array ends with `"Makefile",`. Replace that single line with:

```ts
    "Makefile",
    "CMakeLists.txt",
    "CMakePresets.json",
    "configure.ac",
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: `maps CMake and autotools build files as config features` PASSES. All other tests pass.

- [ ] **Step 5: Format and commit**

```bash
npx --yes pnpm@latest exec oxfmt --write src/mappers/config.ts src/mapper.test.ts
git add src/mappers/config.ts src/mapper.test.ts
git commit -m "Map CMake and autotools build files as config features

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Conservative validation commands — `Makefile`

Adds a C/C++/CUDA branch to `languageDefaultCommands` that emits `make` commands only when a root `Makefile` exists, and a `test` command only when the Makefile declares a `check`/`test` target.

**Files:**
- Modify: `src/detect.ts`
- Test: `src/mapper.test.ts`

- [ ] **Step 1: Write the failing tests**

Add these two tests in `src/mapper.test.ts` after the Task 2 test:

```ts
  it("defaults C/C++ validation to make when the Makefile declares a check target", async () => {
    const root = await fixtureRoot("clawpatch-cpp-makefile-check-");
    await writeFixture(root, "Makefile", "all:\n\tcc -o app main.c\n\ncheck:\n\t./app\n");
    await writeFixture(root, "main.c", "int main(void) { return 0; }\n");

    const project = await detectProject(root);

    expect(project.detected.commands.typecheck).toBe("make");
    expect(project.detected.commands.test).toBe("make check");
  });

  it("defaults C/C++ validation to make with no test command when the Makefile has none", async () => {
    const root = await fixtureRoot("clawpatch-cpp-makefile-notest-");
    await writeFixture(root, "Makefile", "all:\n\tcc -o app main.c\n");
    await writeFixture(root, "main.c", "int main(void) { return 0; }\n");

    const project = await detectProject(root);

    expect(project.detected.commands.typecheck).toBe("make");
    expect(project.detected.commands.test).toBeNull();
  });
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: both new `defaults C/C++ validation to make …` tests FAIL — `typecheck` is `null` (no C/C++ branch exists in `languageDefaultCommands` yet). All other tests pass.

- [ ] **Step 3: Add the C/C++/CUDA branch and the Makefile helper**

In `src/detect.ts`, in `languageDefaultCommands`, add this branch immediately before the final `return { typecheck: null, lint: null, format: null, test: null };`:

```ts
  if (
    languages.includes("c") ||
    languages.includes("cpp") ||
    languages.includes("cuda")
  ) {
    return cOrCppDefaultCommands(root);
  }
```

Then add these functions immediately after the `rubyDefaultCommands` function:

```ts
async function cOrCppDefaultCommands(root: string): Promise<ProjectCommands> {
  const makefileCommands = await makefileDefaultCommands(root);
  if (makefileCommands !== null) {
    return makefileCommands;
  }
  return { typecheck: null, lint: null, format: null, test: null };
}

async function makefileDefaultCommands(root: string): Promise<ProjectCommands | null> {
  if (!(await pathExists(join(root, "Makefile")))) {
    return null;
  }
  const source = await readFile(join(root, "Makefile"), "utf8").catch(() => "");
  const test = makefileHasTarget(source, "check")
    ? "make check"
    : makefileHasTarget(source, "test")
      ? "make test"
      : null;
  return { typecheck: "make", lint: null, format: null, test };
}

function makefileHasTarget(source: string, target: string): boolean {
  return new RegExp(`^${target}\\s*:(?!=)`, "mu").test(source);
}
```

The `(?!=)` negative lookahead keeps a `:=` variable assignment (e.g. `check := something`) from being mistaken for a `check:` rule.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: both `defaults C/C++ validation to make …` tests PASS. All other tests pass.

- [ ] **Step 5: Format and commit**

```bash
npx --yes pnpm@latest exec oxfmt --write src/detect.ts src/mapper.test.ts
git add src/detect.ts src/mapper.test.ts
git commit -m "Default C/C++/CUDA validation to declared Makefile targets

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Conservative validation commands — `CMakePresets.json`

Extends `cOrCppDefaultCommands` to emit a CMake command only when the project declares its build workflow in `CMakePresets.json`.

**Files:**
- Modify: `src/detect.ts`
- Test: `src/mapper.test.ts`

- [ ] **Step 1: Write the tests**

Add these four tests in `src/mapper.test.ts` after the Task 3 tests. The first drives the new code; the last three are regression guards for the conservative boundary (they verify clawpatch does **not** invent a command):

```ts
  it("emits a CMake workflow preset validation command when one is declared", async () => {
    const root = await fixtureRoot("clawpatch-cmake-preset-");
    await writeFixture(root, "CMakeLists.txt", "project(app CXX)\nadd_executable(app main.cpp)\n");
    await writeFixture(root, "main.cpp", "int main(void) { return 0; }\n");
    await writeFixture(
      root,
      "CMakePresets.json",
      JSON.stringify({ version: 6, workflowPresets: [{ name: "default", steps: [] }] }),
    );

    const project = await detectProject(root);

    expect(project.detected.commands.typecheck).toBe("cmake --workflow --preset default");
  });

  it("emits no C/C++ validation command for a CMake project without presets", async () => {
    const root = await fixtureRoot("clawpatch-cmake-nopreset-");
    await writeFixture(root, "CMakeLists.txt", "project(app CXX)\nadd_executable(app main.cpp)\n");
    await writeFixture(root, "main.cpp", "int main(void) { return 0; }\n");

    const project = await detectProject(root);

    expect(project.detected.commands.typecheck).toBeNull();
    expect(project.detected.commands.test).toBeNull();
  });

  it("emits no C/C++ validation command for ambiguous CMake presets", async () => {
    const root = await fixtureRoot("clawpatch-cmake-ambiguous-preset-");
    await writeFixture(root, "CMakeLists.txt", "project(app CXX)\nadd_executable(app main.cpp)\n");
    await writeFixture(root, "main.cpp", "int main(void) { return 0; }\n");
    await writeFixture(
      root,
      "CMakePresets.json",
      JSON.stringify({
        version: 6,
        workflowPresets: [
          { name: "debug", steps: [] },
          { name: "release", steps: [] },
        ],
      }),
    );

    const project = await detectProject(root);

    expect(project.detected.commands.typecheck).toBeNull();
  });

  it("emits no C/C++ validation command for an autotools-only project", async () => {
    const root = await fixtureRoot("clawpatch-autotools-nullcmd-");
    await writeFixture(root, "Makefile.am", "bin_PROGRAMS = app\napp_SOURCES = main.c\n");
    await writeFixture(root, "main.c", "int main(void) { return 0; }\n");

    const project = await detectProject(root);

    expect(project.detected.commands.typecheck).toBeNull();
    expect(project.detected.commands.test).toBeNull();
  });
```

- [ ] **Step 2: Run the tests to verify status**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: `emits a CMake workflow preset validation command when one is declared` FAILS (`typecheck` is `null`). The three `emits no … command` guards already PASS — C/C++ defaults to `null` today; they exist to catch a future over-aggressive change. All other tests pass.

- [ ] **Step 3: Add the CMake preset helper**

In `src/detect.ts`, change `cOrCppDefaultCommands` (added in Task 3) to fall through to the preset helper:

```ts
async function cOrCppDefaultCommands(root: string): Promise<ProjectCommands> {
  const makefileCommands = await makefileDefaultCommands(root);
  if (makefileCommands !== null) {
    return makefileCommands;
  }
  const presetCommands = await cmakePresetDefaultCommands(root);
  if (presetCommands !== null) {
    return presetCommands;
  }
  return { typecheck: null, lint: null, format: null, test: null };
}
```

Then add these functions immediately after `makefileHasTarget`:

```ts
type CMakePresetSets = {
  workflowPresets: string[];
  configurePresets: string[];
  buildPresets: string[];
  testPresets: string[];
};

async function cmakePresetDefaultCommands(root: string): Promise<ProjectCommands | null> {
  if (!(await pathExists(join(root, "CMakePresets.json")))) {
    return null;
  }
  const presets = await readCMakePresets(root);
  if (presets === null) {
    return null;
  }
  const testPreset = singlePresetName(presets.testPresets);
  return {
    typecheck: cmakeBuildCommand(presets),
    lint: null,
    format: null,
    test: testPreset === null ? null : `ctest --preset ${testPreset}`,
  };
}

function cmakeBuildCommand(presets: CMakePresetSets): string | null {
  const workflow = singlePresetName(presets.workflowPresets);
  if (workflow !== null) {
    return `cmake --workflow --preset ${workflow}`;
  }
  const configure = singlePresetName(presets.configurePresets);
  const build = singlePresetName(presets.buildPresets);
  if (configure !== null && build !== null) {
    return `cmake --preset ${configure} && cmake --build --preset ${build}`;
  }
  return null;
}

function singlePresetName(names: string[]): string | null {
  return names.length === 1 ? (names[0] ?? null) : null;
}

async function readCMakePresets(root: string): Promise<CMakePresetSets | null> {
  let parsed: unknown;
  try {
    parsed = JSON.parse(await readFile(join(root, "CMakePresets.json"), "utf8"));
  } catch {
    return null;
  }
  if (typeof parsed !== "object" || parsed === null) {
    return null;
  }
  const record = parsed as Record<string, unknown>;
  return {
    workflowPresets: cmakePresetNames(record.workflowPresets),
    configurePresets: cmakePresetNames(record.configurePresets),
    buildPresets: cmakePresetNames(record.buildPresets),
    testPresets: cmakePresetNames(record.testPresets),
  };
}

function cmakePresetNames(value: unknown): string[] {
  if (!Array.isArray(value)) {
    return [];
  }
  const names: string[] = [];
  for (const entry of value) {
    if (typeof entry !== "object" || entry === null) {
      continue;
    }
    const preset = entry as { name?: unknown; hidden?: unknown };
    if (
      typeof preset.name === "string" &&
      preset.hidden !== true &&
      /^[A-Za-z0-9._-]+$/u.test(preset.name)
    ) {
      names.push(preset.name);
    }
  }
  return names;
}
```

`cmakePresetNames` skips `hidden` presets (they cannot be invoked with `--preset`) and any preset whose name is not a plain identifier (a conservative guard against unusual names in a shell command). `CMakeUserPresets.json` is intentionally not read — it is a user-local, typically gitignored file.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: all four new tests PASS. All other tests pass.

- [ ] **Step 5: Format and commit**

```bash
npx --yes pnpm@latest exec oxfmt --write src/detect.ts src/mapper.test.ts
git add src/detect.ts src/mapper.test.ts
git commit -m "Default CMake validation to declared CMakePresets workflows

Emit a CMake validation command only from a project-declared
CMakePresets.json workflow; never invent a configure step.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: C/C++/CUDA residual source-group slicing

Adds a new mapper module that groups C/C++/CUDA source files not owned by any build target into bounded review slices, and wires it into `cCppSeeds`.

**Files:**
- Modify: `src/mappers/grouping.ts` (export `chunkFiles`)
- Create: `src/mappers/c-cpp-groups.ts`
- Modify: `src/mappers/c-cpp.ts` (call the new module)
- Test: `src/mapper.test.ts`

- [ ] **Step 1: Write the failing tests**

Add these three tests in `src/mapper.test.ts` after the Task 4 tests:

```ts
  it("maps loose C++ sources with no build target as a source-group feature", async () => {
    const root = await fixtureRoot("clawpatch-cpp-group-");
    await writeFixture(root, "lib/parser.cpp", "int parse(void) { return 0; }\n");
    await writeFixture(root, "lib/lexer.cpp", "int lex(void) { return 0; }\n");

    const project = await detectProject(root);
    const result = await mapFeatures(root, project, []);
    const group = result.features.find((feature) => feature.title === "C/C++ source group lib");

    expect(group?.kind).toBe("library");
    expect(group?.source).toBe("c-cpp-group");
    expect(group?.ownedFiles).toEqual([
      { path: "lib/lexer.cpp", reason: "source group member" },
      { path: "lib/parser.cpp", reason: "source group member" },
    ]);
  });

  it("excludes files already owned by a CMake target from source groups", async () => {
    const root = await fixtureRoot("clawpatch-cpp-group-exclude-");
    await writeFixture(root, "CMakeLists.txt", "add_executable(app src/main.cpp)\n");
    await writeFixture(root, "src/main.cpp", "int main(void) { return 0; }\n");
    await writeFixture(root, "src/helper.cpp", "int help(void) { return 0; }\n");

    const project = await detectProject(root);
    const result = await mapFeatures(root, project, []);
    const group = result.features.find((feature) => feature.title === "C/C++ source group src");

    expect(group?.ownedFiles).toEqual([
      { path: "src/helper.cpp", reason: "source group member" },
    ]);
  });

  it("maps a loose CUDA kernel directory as a CUDA source group with concurrency", async () => {
    const root = await fixtureRoot("clawpatch-cuda-group-");
    await writeFixture(
      root,
      "kernels/reduce.cu",
      "__global__ void reduce(float *x) { x[0] = 0; }\n",
    );
    await writeFixture(root, "kernels/reduce.cuh", "__global__ void reduce(float *x);\n");

    const project = await detectProject(root);
    const result = await mapFeatures(root, project, []);
    const group = result.features.find(
      (feature) => feature.title === "CUDA source group kernels",
    );

    expect(group?.tags).toContain("cuda");
    expect(group?.trustBoundaries).toContain("concurrency");
    expect(group?.ownedFiles).toEqual([
      { path: "kernels/reduce.cu", reason: "source group member" },
      { path: "kernels/reduce.cuh", reason: "source group member" },
    ]);
  });
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: all three new tests FAIL — no `c-cpp-group` features exist (`group` is `undefined`). All other tests pass.

- [ ] **Step 3: Export `chunkFiles` from `grouping.ts`**

In `src/mappers/grouping.ts`, change the `chunkFiles` declaration line from:

```ts
function chunkFiles(label: string, files: string[], maxFiles: number): FileGroup[] {
```

to:

```ts
export function chunkFiles(label: string, files: string[], maxFiles: number): FileGroup[] {
```

(Only the `export` keyword is added; the body is unchanged.)

- [ ] **Step 4: Create the source-group module**

Create `src/mappers/c-cpp-groups.ts` with exactly this content:

```ts
import { chunkFiles, partitionFileGroups } from "./grouping.js";
import { isCOrCppPath, isCOrCppTestPath, languageTag, withCudaConcurrency } from "./shared.js";
import { FeatureSeed } from "./types.js";

const sourceGroupMaxFiles = 12;

export function cCppGroupSeeds(sourceFiles: string[], ownedPaths: Set<string>): FeatureSeed[] {
  const residual = sourceFiles.filter(
    (path) => isCOrCppPath(path) && !ownedPaths.has(path) && !isCOrCppTestPath(path),
  );
  if (residual.length === 0) {
    return [];
  }
  const byTopDir = new Map<string, string[]>();
  for (const path of residual) {
    const slash = path.indexOf("/");
    const topDir = slash === -1 ? "" : path.slice(0, slash);
    byTopDir.set(topDir, [...(byTopDir.get(topDir) ?? []), path]);
  }
  const seeds: FeatureSeed[] = [];
  for (const [topDir, files] of [...byTopDir.entries()].toSorted(([left], [right]) =>
    left.localeCompare(right),
  )) {
    const groups =
      topDir === ""
        ? chunkFiles(".", files.toSorted(), sourceGroupMaxFiles)
        : partitionFileGroups(topDir, files, sourceGroupMaxFiles);
    for (const group of groups) {
      seeds.push(groupSeed(group.label, group.files));
    }
  }
  return seeds;
}

function groupSeed(label: string, files: string[]): FeatureSeed {
  const sorted = files.toSorted();
  const tags = [...new Set(sorted.map(languageTag))];
  const isCuda = tags.includes("cuda");
  return {
    title: `${isCuda ? "CUDA" : "C/C++"} source group ${label}`,
    summary: `C/C++/CUDA source files under ${label} not owned by a build target.`,
    kind: "library",
    source: "c-cpp-group",
    confidence: "low",
    entryPath: sorted[0] ?? label,
    symbol: label,
    route: null,
    command: null,
    tags: [...tags, "source-group"],
    trustBoundaries: withCudaConcurrency(["filesystem"], isCuda ? "cuda" : "cpp"),
    ownedFiles: sorted.map((path) => ({ path, reason: "source group member" })),
  };
}
```

- [ ] **Step 5: Wire the module into `cCppSeeds`**

In `src/mappers/c-cpp.ts`, add this import beside the existing `import { FeatureSeed, SeedFileRef } from "./types.js";` line:

```ts
import { cCppGroupSeeds } from "./c-cpp-groups.js";
```

In the `cCppSeeds` function, replace these final two lines:

```ts
  seeds.push(...(await mainFunctionTargets(root, files, alreadySeeded)));
  return dedupeByEntry(seeds);
```

with:

```ts
  seeds.push(...(await mainFunctionTargets(root, files, alreadySeeded)));
  const ownedPaths = new Set(
    seeds.flatMap((seed) => [
      seed.entryPath,
      ...(seed.ownedFiles?.map((file) => file.path) ?? []),
    ]),
  );
  seeds.push(...cCppGroupSeeds(files.filter(isCOrCppSource), ownedPaths));
  return dedupeByEntry(seeds);
```

`ownedPaths` is built from every prior seed (autotools, cmake, cmake-test, and `c-main`), so source groups never double-own a file already covered by a build target.

- [ ] **Step 6: Run the tests to verify they pass**

Run: `npx --yes pnpm@latest exec vitest run src/mapper.test.ts`
Expected: all three new source-group tests PASS. All other tests pass.

- [ ] **Step 7: Run the full suite**

Run: `npx --yes pnpm@latest exec vitest run`
Expected: PASS — every test green across all test files.

- [ ] **Step 8: Format and commit**

```bash
npx --yes pnpm@latest exec oxfmt --write src/mappers/grouping.ts src/mappers/c-cpp-groups.ts src/mappers/c-cpp.ts src/mapper.test.ts
git add src/mappers/grouping.ts src/mappers/c-cpp-groups.ts src/mappers/c-cpp.ts src/mapper.test.ts
git commit -m "Map residual C/C++/CUDA sources into bounded source groups

Source files not owned by any CMake/autotools/main() target are grouped
per directory into bounded, low-confidence review slices.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 6: Documentation and final verification

**Files:**
- Modify: `README.md`
- Modify: `docs/feature-mapping.md`
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Update `README.md`**

In the "What It Maps Today" list, replace the C/C++/CUDA bullet:

```markdown
- C/C++/CUDA standalone `main()` files, CMake `add_executable` / `add_library`
  targets, and autotools `bin_PROGRAMS` / `lib_LTLIBRARIES` targets, including
  CUDA `.cu` / `.cuh` sources
```

with:

```markdown
- C/C++/CUDA standalone `main()` files, CMake `add_executable` / `add_library`
  targets, autotools `bin_PROGRAMS` / `lib_LTLIBRARIES` targets, and residual
  source groups for files outside any build target, including CUDA `.cu` /
  `.cuh` sources
```

- [ ] **Step 2: Update `docs/feature-mapping.md`**

Replace the C/C++ paragraph:

```markdown
C/C++ mapping covers generic project shapes only: standalone source files with
`main()`, CMake `add_executable` / `add_library`, and autotools `bin_PROGRAMS` /
`lib_LTLIBRARIES`. It deliberately avoids project-specific C dialects such as
php-src extension metadata. CUDA `.cu` / `.cuh` files are mapped through the same
C/C++ shapes, including the legacy `FindCUDA` `cuda_add_executable` /
`cuda_add_library` commands; CUDA targets are tagged `cuda`, and a repository with
`.cu` / `.cuh` sources is detected as a `cuda` project.
```

with:

```markdown
C/C++ mapping covers generic project shapes only: standalone source files with
`main()`, CMake `add_executable` / `add_library`, and autotools `bin_PROGRAMS` /
`lib_LTLIBRARIES`. It deliberately avoids project-specific C dialects such as
php-src extension metadata. CUDA `.cu` / `.cuh` files are mapped through the same
C/C++ shapes, including the legacy `FindCUDA` `cuda_add_executable` /
`cuda_add_library` commands; CUDA targets are tagged `cuda`, and a repository with
`.cu` / `.cuh` sources is detected as a `cuda` project. Source files not owned by
any build target are grouped per directory into bounded, low-confidence source
groups. C/C++/CUDA validation commands are emitted only when the project declares
them — a root `Makefile` `check`/`test` target, or a `CMakePresets.json` build
workflow — and stay null otherwise.
```

- [ ] **Step 3: Update `CHANGELOG.md`**

Under the `## 0.3.1 - Unreleased` heading, immediately after the existing line that begins `- Added CUDA support to C/C++ mapping`, add these three lines:

```markdown
- Added residual C/C++/CUDA source-group mapping so source files outside any CMake/autotools/`main()` target are grouped per directory into bounded review slices.
- Added conservative C/C++/CUDA validation command defaults from a root `Makefile` `check`/`test` target or a declared `CMakePresets.json` build workflow, and mapped `CMakeLists.txt`, `CMakePresets.json`, and `configure.ac` as config features.
- Added the `concurrency` trust boundary to CUDA build targets and source groups.
```

- [ ] **Step 4: Run the full verification suite**

```bash
npx --yes pnpm@latest exec vitest run
npx --yes pnpm@latest exec tsc -p tsconfig.json --noEmit
npx --yes pnpm@latest exec oxlint . --config oxlint.json
npx --yes pnpm@latest exec oxfmt --check .
```

Expected: all tests pass; `tsc` exits 0; oxlint reports 0 warnings / 0 errors; oxfmt reports all files correctly formatted. If oxfmt reports an issue, run `npx --yes pnpm@latest exec oxfmt --write .` and re-check.

- [ ] **Step 5: Commit**

```bash
git add README.md docs/feature-mapping.md CHANGELOG.md
git commit -m "Document C/C++/CUDA mapping and validation parity

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Notes for the implementer

- **Module placement deviation from the spec.** The spec described `withCudaConcurrency` as living in `c-cpp.ts`. The plan places it (with `languageTag`/`languageLabel`/`LanguageTag`) in `shared.ts` instead, so the new `c-cpp-groups.ts` can reuse them without a circular `c-cpp.ts` ↔ `c-cpp-groups.ts` import. Behaviour is identical.
- **Guard tests.** Three Task 4 tests (`emits no … command` for no-presets, ambiguous presets, and autotools-only) pass from the start — C/C++ defaults to null today. They are deliberate regression guards for the conservative boundary, not red-first drivers.
- **TDD.** Every other test must be observed failing before its implementation step. Do not write implementation code ahead of its test.
