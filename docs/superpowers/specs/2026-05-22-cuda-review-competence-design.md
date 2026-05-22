# CUDA review competence — design

- **Date:** 2026-05-22
- **Status:** Approved (pending written-spec review)
- **Scope:** Spec 2 of 2. Builds on the Spec 1 work (C/C++/CUDA mapping &
  validation parity), now merged to `main`; CHANGELOG section
  `0.4.1 - Unreleased`.

## Problem

Spec 1 made clawpatch **see** CUDA — it detects `.cu`/`.cuh`, maps kernels and
CMake/Make targets, tags features `cuda`, adds the `concurrency` trust boundary,
and emits build commands. It did not make clawpatch **judge** CUDA.

`src/prompt.ts` builds the prompt sent to the review provider. That prompt has a
fixed, generic "Review categories" list and exactly one guidance-injection point
— `reviewModeInstructions(mode)` for mode-specific text. There is no
language- or domain-specific guidance. A `.cu` kernel is therefore reviewed with
the same checklist as a Node route: nothing tells the model to look for kernel
races, unchecked CUDA runtime calls, host/device pointer confusion, memory-access
hazards, or synchronization mistakes. The same is true of `buildFixPrompt` —
clawpatch patches a CUDA finding with no CUDA awareness.

So clawpatch can map a CUDA repository but cannot evaluate it *as CUDA*. Closing
that gap is the readiness bar before contributing upstream.

## Decisions taken during brainstorming

- **Both prompts.** CUDA guidance is injected into the review prompt
  (`buildReviewPromptBundle`) **and** the fix prompt (`buildFixPrompt`) — clawpatch
  should both evaluate and patch CUDA with domain awareness.
- **Trigger on owned-file extension, not the `cuda` tag.** A feature is CUDA if
  any `entrypoint`/`ownedFile` path is `.cu`/`.cuh`. A mixed target whose `main()`
  is in a `.cpp` but which compiles `.cu` kernels is tagged `cpp`, yet still needs
  CUDA review — the extension check catches it; the tag would not.
- **Review guidance is gated to `default` mode.** `deslopify` mode instructs the
  model to report only simplification findings and explicitly *not* hunt bugs;
  injecting CUDA bug guidance there would contradict it.
- **No finding-schema change.** CUDA bug classes map onto the existing `category`
  enum (`concurrency`, `bug`, `data-loss`, `performance`). The guidance steers to
  those; there is no new `cuda` category. Leaving the strict schema untouched is
  the conservative, in-spirit choice.
- **`--prompt-file` is unchanged.** clawpatch's existing user-supplied reviewer
  guidance (`customPrompt`/`customBlock`) still composes on top; built-in CUDA
  guidance is automatic and per-feature, the user override is manual and
  per-run. They coexist.

## Components — all in `src/prompt.ts`

### 1. `featureIncludesCuda(feature)` — CUDA detection

A local helper: returns `true` when any path in `feature.entrypoints` or
`feature.ownedFiles` matches `/\.cuh?$/iu`. Used by both prompt builders. Kept
local to `prompt.ts` (a one-line regex) rather than importing a mapper util, so
`prompt.ts` keeps its current narrow import surface.

### 2. `cudaGuidance()` — the guidance block

Returns a bounded, static guidance block:

```
CUDA hazards — this feature includes CUDA `.cu`/`.cuh` sources. Attend to:
- Kernel data races; missing, divergent, or conditionally-reached
  `__syncthreads()`/`__syncwarp()` barriers.
- Unchecked CUDA runtime calls (`cudaMalloc`, `cudaMemcpy`, `cudaFree`, async
  copies) and missing `cudaGetLastError()`/`cudaDeviceSynchronize()` after a
  kernel launch.
- Host vs. device pointer confusion: dereferencing device memory on the host, or
  passing the wrong memory space or copy direction to `cudaMemcpy`.
- Out-of-bounds or uncoalesced global-memory access, shared-memory bank
  conflicts, and `blockIdx`/`threadIdx`-derived indices used without bounds
  checks.
- Stream and event synchronization errors, including use-after-free across
  asynchronous copies.
- Device-memory leaks: allocations not freed on every return path.
Map findings to the existing categories (`concurrency`, `bug`, `data-loss`,
`performance`). Report only hazards visible in the included code; do not
speculate about GPU runtime behavior you cannot see.
```

One shared block serves both prompts: the review prompt consumes it as added
review focus; the fix prompt consumes it as correctness constraints the patch
must respect. The closing anti-speculation line matches the review prompt's
existing "Evidence must point at included files" discipline.

### 3. Review-prompt injection — `buildReviewPromptBundle`

Build a `cudaBlock` string and interpolate `${cudaBlock}` into the prompt
template immediately after the existing `${reviewModeInstructions(mode)}`:

```ts
const cudaBlock =
  mode === "default" && featureIncludesCuda(feature) ? `\n${cudaGuidance()}\n` : "";
```

When the feature is not CUDA, or the mode is `deslopify`, `cudaBlock` is the
empty string and the prompt is byte-identical to today's.

### 4. Fix-prompt injection — `buildFixPrompt`

Build a `cudaBlock` the same way — no mode gate, since `buildFixPrompt` has no
review mode — and interpolate it after the "Fix only the finding below"
instructions, before the `Finding:` block:

```ts
const cudaBlock = featureIncludesCuda(feature) ? `\n${cudaGuidance()}\n` : "";
```

## Data flow

```
buildReviewPrompt → buildReviewPromptBundle → (default mode + CUDA feature)
                                              → prompt includes cudaGuidance()
buildFixPrompt → (CUDA feature) → fix prompt includes cudaGuidance()
```

No change to the finding schema, the review-mode set, the mapper, or
`--prompt-file` handling.

## Testing (TDD)

Failing-first tests in `src/prompt.test.ts`. A feature is constructed as a
`FeatureRecord`; the prompt string is asserted to contain or not contain a
stable marker phrase from `cudaGuidance()` (e.g. `"CUDA hazards"`).

1. Review prompt, `default` mode, feature owning a `.cu` file → prompt contains
   the CUDA guidance.
2. Review prompt, `default` mode, a pure-`.cpp`/non-CUDA feature → prompt does
   not contain it.
3. Review prompt, `deslopify` mode, a CUDA feature → prompt does not contain it
   (mode gate).
4. Review prompt, `default` mode, a mixed feature (entrypoint `.cpp`, an owned
   `.cu` file) → prompt contains it (owned-file trigger).
5. Fix prompt, a CUDA feature → prompt contains the CUDA guidance.
6. Fix prompt, a non-CUDA feature → prompt does not contain it.

Whole-suite `vitest run`, `tsc --noEmit`, `oxlint`, and `oxfmt --check` must all
pass.

## Readiness gate

After this lands, run `clawpatch map` and `clawpatch review` against a real CUDA
repository and confirm it surfaces genuine CUDA findings — the actual proof that
clawpatch can evaluate CUDA, and the precondition for any upstream contribution.
A live review needs a configured provider (`clawpatch doctor`); if none is
available in the working environment, this step is handed to the user.

## Out of scope

- **Kernel-level feature slicing** — making each `__global__` kernel its own
  review unit. Spec 1's source groups and targets already place CUDA code in
  front of the model; per-kernel slicing is a refinement, not the readiness gate.
- **`compute-sanitizer` / runtime validation** — GPU-runtime tooling, a separate
  concern from static review competence, and unrunnable without a GPU.

## Documentation to update on implementation

`docs/code-review.md` (note CUDA-aware review and fix guidance) and a
`CHANGELOG.md` entry under `0.4.1 - Unreleased`.
