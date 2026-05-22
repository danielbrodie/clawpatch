# CUDA Review Competence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make clawpatch's review and fix prompts CUDA-aware — inject CUDA-specific reviewer guidance for any feature that owns `.cu`/`.cuh` sources, so clawpatch evaluates and patches CUDA code with domain knowledge instead of a generic checklist.

**Architecture:** One file, `src/prompt.ts`. Two new local helpers — `featureIncludesCuda(feature)` (true when an entrypoint/owned path is `.cu`/`.cuh`) and `cudaGuidance()` (a static CUDA-hazard block). The review prompt (`buildReviewPromptBundle`) injects the guidance when the feature is CUDA and the mode is `default`; the fix prompt (`buildFixPrompt`) injects it whenever the feature is CUDA. No finding-schema change, no mapper change.

**Tech Stack:** TypeScript (ESM, `.js` import specifiers), Vitest, oxlint, oxfmt. Spec: `docs/superpowers/specs/2026-05-22-cuda-review-competence-design.md`.

**Branch:** Continue on `cuda-review-competence` (it already carries the spec).

**Commands:** This environment has no global `pnpm`; run every pnpm command via `npx --yes pnpm@latest exec …`. Task test steps run the whole `src/prompt.test.ts` file (small, fast); read the per-test `✓`/`✗` lines for the named tests.

**Note on the guidance text:** the spec showed the `cudaGuidance()` block with markdown backticks around identifiers. This plan uses the same content as plain text (no backticks) — the clawpatch review prompt is plain text throughout (see the "Review categories" list and `reviewModeInstructions`), and a template literal full of escaped backticks would be needlessly noisy. Content and intent are identical.

---

### Task 1: CUDA-aware review prompt

Adds the `featureIncludesCuda` and `cudaGuidance` helpers and injects the guidance into `buildReviewPromptBundle`.

**Files:**
- Modify: `src/prompt.ts`
- Test: `src/prompt.test.ts`

- [ ] **Step 1: Write the failing tests**

In `src/prompt.test.ts`, add a new `describe` block immediately **after** the existing `describe("review prompt provenance", …)` block closes (its closing `});`) and **before** the `function project(` declaration:

```ts
describe("CUDA prompt guidance", () => {
  it("includes CUDA review guidance for a feature that owns a .cu file", async () => {
    const root = await fixtureRoot("clawpatch-prompt-cuda-review-");
    await writeFixture(root, "src/kernel.cu", "__global__ void k(void) {}\n");
    const cudaFeature: FeatureRecord = {
      ...feature(),
      entrypoints: [],
      ownedFiles: [{ path: "src/kernel.cu", reason: "kernel" }],
      contextFiles: [],
    };
    const bundle = await buildReviewPromptBundle(root, project(root), cudaFeature, defaultConfig());

    expect(bundle.prompt).toContain("CUDA hazards");
  });

  it("omits CUDA review guidance for a non-CUDA feature", async () => {
    const root = await fixtureRoot("clawpatch-prompt-noncuda-review-");
    await writeFixture(root, "src/index.ts", "export const value = 1;\n");
    const tsFeature: FeatureRecord = {
      ...feature(),
      entrypoints: [],
      ownedFiles: [{ path: "src/index.ts", reason: "primary" }],
      contextFiles: [],
    };
    const bundle = await buildReviewPromptBundle(root, project(root), tsFeature, defaultConfig());

    expect(bundle.prompt).not.toContain("CUDA hazards");
  });

  it("omits CUDA review guidance in deslopify mode even for a CUDA feature", async () => {
    const root = await fixtureRoot("clawpatch-prompt-cuda-deslopify-");
    await writeFixture(root, "src/kernel.cu", "__global__ void k(void) {}\n");
    const cudaFeature: FeatureRecord = {
      ...feature(),
      entrypoints: [],
      ownedFiles: [{ path: "src/kernel.cu", reason: "kernel" }],
      contextFiles: [],
    };
    const bundle = await buildReviewPromptBundle(
      root,
      project(root),
      cudaFeature,
      defaultConfig(),
      "deslopify",
    );

    expect(bundle.prompt).not.toContain("CUDA hazards");
  });

  it("includes CUDA review guidance for a mixed feature whose entrypoint is C++ but owns a .cu file", async () => {
    const root = await fixtureRoot("clawpatch-prompt-cuda-mixed-");
    await writeFixture(root, "src/main.cpp", "int main(void) { return 0; }\n");
    await writeFixture(root, "src/kernel.cu", "__global__ void k(void) {}\n");
    const mixedFeature: FeatureRecord = {
      ...feature(),
      entrypoints: [{ path: "src/main.cpp", symbol: "main", route: null, command: null }],
      ownedFiles: [
        { path: "src/main.cpp", reason: "host" },
        { path: "src/kernel.cu", reason: "kernel" },
      ],
      contextFiles: [],
    };
    const bundle = await buildReviewPromptBundle(root, project(root), mixedFeature, defaultConfig());

    expect(bundle.prompt).toContain("CUDA hazards");
  });
});
```

- [ ] **Step 2: Run the tests to verify status**

Run: `npx --yes pnpm@latest exec vitest run src/prompt.test.ts`
Expected: the two `includes CUDA review guidance …` tests FAIL (`bundle.prompt` has no `"CUDA hazards"` text yet). The two `omits …` tests already PASS — no CUDA guidance exists, so they are regression guards, not drivers. All pre-existing tests pass.

- [ ] **Step 3: Add the `featureIncludesCuda` and `cudaGuidance` helpers**

In `src/prompt.ts`, find the end of the `reviewModeInstructions` function and the start of `buildRevalidatePrompt`:

```ts
  throw new Error(`Unsupported review mode: ${mode}`);
}

export async function buildRevalidatePrompt(root: string, findingJson: string): Promise<string> {
```

Replace it with (the two helpers inserted between):

```ts
  throw new Error(`Unsupported review mode: ${mode}`);
}

function featureIncludesCuda(feature: FeatureRecord): boolean {
  const paths = [
    ...feature.entrypoints.map((entrypoint) => entrypoint.path),
    ...feature.ownedFiles.map((file) => file.path),
  ];
  return paths.some((path) => /\.cuh?$/iu.test(path));
}

function cudaGuidance(): string {
  return `CUDA hazards (this feature includes CUDA .cu/.cuh sources) — inspect for:
- Kernel data races; missing, divergent, or conditionally-reached __syncthreads()/__syncwarp() barriers.
- Unchecked CUDA runtime calls (cudaMalloc, cudaMemcpy, cudaFree, async copies) and missing cudaGetLastError()/cudaDeviceSynchronize() after a kernel launch.
- Host vs. device pointer confusion: dereferencing device memory on the host, or passing the wrong memory space or copy direction to cudaMemcpy.
- Out-of-bounds or uncoalesced global-memory access, shared-memory bank conflicts, and blockIdx/threadIdx-derived indices used without bounds checks.
- Stream and event synchronization errors, including use-after-free across asynchronous copies.
- Device-memory leaks: allocations not freed on every return path.
Map findings to the existing categories (concurrency, bug, data-loss, performance). Report only hazards visible in the included code; do not speculate about GPU runtime behavior you cannot see.`;
}

export async function buildRevalidatePrompt(root: string, findingJson: string): Promise<string> {
```

- [ ] **Step 4: Compute `cudaBlock` in `buildReviewPromptBundle`**

In `src/prompt.ts`, in `buildReviewPromptBundle`, find:

```ts
  const validEvidencePaths = [
    ...new Set(includedFiles.filter((file) => file.readable).map((file) => file.path)),
  ];
  const prompt = `You are reviewing one semantic feature for clawpatch.
```

Replace it with:

```ts
  const validEvidencePaths = [
    ...new Set(includedFiles.filter((file) => file.readable).map((file) => file.path)),
  ];
  const cudaBlock =
    mode === "default" && featureIncludesCuda(feature) ? `\n${cudaGuidance()}\n` : "";
  const prompt = `You are reviewing one semantic feature for clawpatch.
```

- [ ] **Step 5: Inject `cudaBlock` into the review prompt template**

In the same `buildReviewPromptBundle` template string, find the line:

```ts
${reviewModeInstructions(mode)}
```

Replace it with:

```ts
${reviewModeInstructions(mode)}${cudaBlock}
```

When the feature is not CUDA, or the mode is `deslopify`, `cudaBlock` is the empty string and the prompt is byte-identical to before.

- [ ] **Step 6: Run the tests to verify they pass**

Run: `npx --yes pnpm@latest exec vitest run src/prompt.test.ts`
Expected: all four `CUDA prompt guidance` review tests PASS. All pre-existing tests pass.

- [ ] **Step 7: Typecheck, lint, format**

Run each; all must be clean:
```bash
npx --yes pnpm@latest exec tsc -p tsconfig.json --noEmit
npx --yes pnpm@latest exec oxlint . --config oxlint.json
npx --yes pnpm@latest exec oxfmt --write src/prompt.ts src/prompt.test.ts
```

- [ ] **Step 8: Commit**

```bash
git add src/prompt.ts src/prompt.test.ts
git commit -m "Inject CUDA reviewer guidance into the review prompt

CUDA-owning features get a CUDA-hazard guidance block in the default-mode
review prompt, so the provider evaluates kernels for CUDA-specific bug
classes instead of a generic checklist.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: CUDA-aware fix prompt

Injects the same `cudaGuidance()` into `buildFixPrompt`. Reuses the `featureIncludesCuda` and `cudaGuidance` helpers added in Task 1.

**Files:**
- Modify: `src/prompt.ts`
- Test: `src/prompt.test.ts`

- [ ] **Step 1: Write the tests**

In `src/prompt.test.ts`, add these two tests inside the `describe("CUDA prompt guidance", …)` block created in Task 1 — immediately before that block's closing `});`:

```ts
  it("includes CUDA guidance in the fix prompt for a CUDA feature", async () => {
    const root = await fixtureRoot("clawpatch-prompt-cuda-fix-");
    await writeFixture(root, "src/kernel.cu", "__global__ void k(void) {}\n");
    const cudaFeature: FeatureRecord = {
      ...feature(),
      entrypoints: [],
      ownedFiles: [{ path: "src/kernel.cu", reason: "kernel" }],
      contextFiles: [],
    };
    const prompt = await buildFixPrompt(
      root,
      finding("src/kernel.cu"),
      cudaFeature,
      defaultConfig(),
    );

    expect(prompt).toContain("CUDA hazards");
  });

  it("omits CUDA guidance in the fix prompt for a non-CUDA feature", async () => {
    const root = await fixtureRoot("clawpatch-prompt-noncuda-fix-");
    await writeFixture(root, "src/index.ts", "export const value = 1;\n");
    const tsFeature: FeatureRecord = {
      ...feature(),
      entrypoints: [],
      ownedFiles: [{ path: "src/index.ts", reason: "primary" }],
      contextFiles: [],
    };
    const prompt = await buildFixPrompt(root, finding("src/index.ts"), tsFeature, defaultConfig());

    expect(prompt).not.toContain("CUDA hazards");
  });
```

- [ ] **Step 2: Run the tests to verify status**

Run: `npx --yes pnpm@latest exec vitest run src/prompt.test.ts`
Expected: `includes CUDA guidance in the fix prompt for a CUDA feature` FAILS (no `"CUDA hazards"` in the fix prompt yet). `omits CUDA guidance in the fix prompt for a non-CUDA feature` already PASSES (regression guard). All other tests pass.

- [ ] **Step 3: Compute `cudaBlock` in `buildFixPrompt`**

In `src/prompt.ts`, in `buildFixPrompt`, find:

```ts
  for (const path of fixPromptPaths(finding, feature, config)) {
    fileBlocks.push(await rawFileBlock(root, path));
  }
  return `You are clawpatch applying one small repair in the current repository.
```

Replace it with:

```ts
  for (const path of fixPromptPaths(finding, feature, config)) {
    fileBlocks.push(await rawFileBlock(root, path));
  }
  const cudaBlock = featureIncludesCuda(feature) ? `\n${cudaGuidance()}\n` : "";
  return `You are clawpatch applying one small repair in the current repository.
```

- [ ] **Step 4: Inject `cudaBlock` into the fix prompt template**

In the same `buildFixPrompt` template string, find:

```ts
  "validationCommands": ["string"]
}

Finding:
```

Replace it with:

```ts
  "validationCommands": ["string"]
}
${cudaBlock}
Finding:
```

When the feature is not CUDA, `cudaBlock` is the empty string and the prompt is byte-identical to before.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `npx --yes pnpm@latest exec vitest run src/prompt.test.ts`
Expected: both new fix-prompt tests PASS. All other tests pass.

- [ ] **Step 6: Typecheck, lint, format**

Run each; all must be clean:
```bash
npx --yes pnpm@latest exec tsc -p tsconfig.json --noEmit
npx --yes pnpm@latest exec oxlint . --config oxlint.json
npx --yes pnpm@latest exec oxfmt --write src/prompt.ts src/prompt.test.ts
```

- [ ] **Step 7: Commit**

```bash
git add src/prompt.ts src/prompt.test.ts
git commit -m "Inject CUDA guidance into the fix prompt

CUDA-owning features get the CUDA-hazard block in the fix prompt so
patches respect CUDA correctness, not just generic correctness.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Documentation and final verification

**Files:**
- Modify: `docs/code-review.md`
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Update `docs/code-review.md`**

At the end of `docs/code-review.md`, find the closing paragraph:

```markdown
Review does not edit files. Use `clawpatch fix --finding <id>` for the explicit
patch loop.
```

Replace it with:

```markdown
## CUDA-aware review

When a feature owns CUDA `.cu` / `.cuh` sources, `clawpatch review` (in the
default mode) and `clawpatch fix` add CUDA-specific guidance to the provider
prompt: kernel data races and synchronization barriers, unchecked CUDA runtime
calls and missing post-launch error checks, host versus device pointer
confusion, unsafe global- and shared-memory access, stream and event
synchronization, and device-memory leaks. Findings still use the existing
categories — there is no CUDA-specific category. Deslopify mode is unaffected.

Review does not edit files. Use `clawpatch fix --finding <id>` for the explicit
patch loop.
```

- [ ] **Step 2: Update `CHANGELOG.md`**

In `CHANGELOG.md`, under `## 0.4.1 - Unreleased`, find the line:

```markdown
- Added the `concurrency` trust boundary to CUDA build targets and source groups.
```

Add this line immediately after it:

```markdown
- Made `clawpatch review` and `clawpatch fix` CUDA-aware by injecting CUDA-specific reviewer guidance (kernel races, unchecked CUDA runtime calls, host/device pointer confusion, memory-access hazards, synchronization mistakes) into the prompt for features that own `.cu` / `.cuh` sources.
```

- [ ] **Step 3: Run the full verification suite**

```bash
npx --yes pnpm@latest exec vitest run
npx --yes pnpm@latest exec tsc -p tsconfig.json --noEmit
npx --yes pnpm@latest exec oxlint . --config oxlint.json
npx --yes pnpm@latest exec oxfmt --check .
```

Expected: all tests pass; `tsc` exits 0; oxlint reports 0 warnings / 0 errors; oxfmt reports all files correctly formatted. If oxfmt reports an issue, run `npx --yes pnpm@latest exec oxfmt --write .` and re-check.

- [ ] **Step 4: Commit**

```bash
git add docs/code-review.md CHANGELOG.md
git commit -m "Document CUDA-aware review and fix guidance

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Notes for the implementer

- **Guard tests.** Three tests (`omits CUDA review guidance for a non-CUDA feature`, `omits CUDA review guidance in deslopify mode …`, `omits CUDA guidance in the fix prompt for a non-CUDA feature`) pass from the start — there is no CUDA guidance to leak yet. They are deliberate regression guards. The driver tests are the three `includes …` tests.
- **TDD.** Each `includes …` test must be observed failing before its implementation step. Do not write implementation code ahead of its test.
- **Marker phrase.** Tests assert on the substring `"CUDA hazards"`, the opening words of `cudaGuidance()`. If you reword that opening, update the tests.
