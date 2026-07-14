---
name: reduce
description: >
  Finds unnecessary seams in software that can be safely reduced without removing
  or changing functionality. Looks for obvious, low-risk consolidations in the spirit
  of practical DRY—duplicated logic, redundant wrappers, parallel abstractions,
  copy-paste blocks, and other simple reductions—not strict “never repeat yourself.”
  Use when the user wants to reduce duplication, simplify seams, collapse redundant
  structure, or runs /reduce.
compatibility: Designed for Agent Skills-compatible coding agents. Requires repository tools appropriate to the project.
metadata:
  author: Modoterra
  version: "1.0.0"
---

# Reduce

Find and apply **easy, safe reductions**: remove unnecessary seams and obvious duplication without changing or removing functionality.

This is practical DRY, not purist DRY. Prefer clarity and fewer moving parts over extracting every repeated token into an abstraction.

## Goals

1. Locate **seams that cost more than they buy** (wrappers, parallel paths, copy-paste blocks, redundant layers).
2. Propose only reductions that are **obvious and low-risk**.
3. Apply reductions that **preserve behavior** exactly.
4. Leave intentional variation, framework boundaries, and unclear cases alone.

## Non-goals

- Large redesigns or architecture rewrites
- Strict “never repeat yourself” extractions
- Premature abstraction of one-off or diverging logic
- Performance optimization unless the reduction is free and obvious
- Changing public APIs, contracts, or user-visible behavior
- Drive-by cleanups outside the agreed scope

## When to run

- User asks to reduce, simplify, collapse duplication, thin out seams, or runs `/reduce`
- After a feature lands and the code has grown parallel paths or copy-paste
- During review when duplication or redundant structure is the main maintainability issue

## Safety rules

You MUST preserve functionality:

- Same inputs → same outputs
- Same side effects, error cases, and edge handling
- Same public APIs and external contracts unless the user explicitly allows a break
- Same observable UI/UX behavior for user-facing code

You MUST NOT:

- Merge paths that only look similar but differ in edge cases
- Extract shared helpers when the instances are about to diverge for good reasons
- Collapse abstractions that enforce real domain or security boundaries
- Remove “duplication” that is intentional (tests, API surfaces, isolation at module edges)
- Reduce across package/module boundaries when that increases coupling without clear benefit

When uncertain whether two pieces are truly equivalent, treat them as **not** reducible until proven otherwise (read callers, tests, and history).

## What to look for (seams)

Scan for easy wins in this priority order:

### 1. Copy-paste blocks

Near-identical functions, methods, components, or config blocks that differ only by names, constants, or trivial parameters.

**Reduce by:** one shared helper/function/component with parameters, or a small data-driven table/map.

### 2. Redundant wrappers

Pass-through methods, classes, or modules that only forward calls without adding policy, validation, logging that matters, or a real boundary.

**Reduce by:** call the underlying unit directly; delete the empty shell.

### 3. Parallel implementations

Two code paths that implement the same rule for different entry points (CLI + HTTP, admin + public, A/B copies left behind, “v1” and “v2” that are identical).

**Reduce by:** one implementation shared by both entry points.

### 4. Duplicated conditionals and rules

The same business rule, validation, mapping, or status interpretation repeated in multiple places.

**Reduce by:** a single function, constant map, or shared type/guard at the natural seam—not a new framework.

### 5. Structural noise

- Nested if/else that can become early returns without changing logic
- Temporary variables that only rename once
- Flags or options that always take one value
- Dead branches behind impossible conditions (only when proven dead)
- Speculative extension points never used (empty interfaces, unused strategy hooks)

**Reduce by:** flatten, inline, or delete the unused seam.

### 6. Config / constants drift

Same magic strings, status codes, or thresholds defined in multiple files.

**Reduce by:** one source of truth already idiomatic for the project (existing constants module, enum, config key)—do not invent a new global registry unless the project already has one.

### 7. Test duplication (secondary)

Only reduce tests when shared setup is already the project’s pattern and the reduction keeps failures readable. Prefer clear tests over DRY tests.

## What not to force

Leave alone unless the user asks to go further:

- Similar-looking code that serves different domains
- Duplication at intentional boundaries (adapters, ports, generated code)
- Tiny one-liners where extraction would obscure call sites
- Framework-required boilerplate
- Generated, vendored, or third-party code
- “Almost the same” blocks that would need flags, modes, or complex options to unify

Small intentional duplication that preserves clarity is acceptable. A reduction must make the code **simpler to read and change**, not merely shorter.

## Workflow

### 1. Orient

- Confirm scope: whole repo, a package, a path, or the current diff/branch.
- Skim project conventions (`AGENTS.md`, architecture notes, existing shared utilities).
- Prefer reducing toward **patterns the codebase already uses**.

### 2. Discover

Search and read; do not propose from filenames alone.

Useful signals:

- Repeated string/identifier clusters
- Near-duplicate function bodies
- Thin files that only re-export or forward
- Multiple `switch`/`match`/maps over the same set of cases
- Comments like “same as X”, “copied from”, “temporary duplicate”
- Parallel directory trees with matching names

Trace callers for each candidate so the reduction is grounded.

### 3. Rank candidates

For each candidate, record:

| Field | Content |
|-------|---------|
| Location | files / symbols |
| Seam type | copy-paste, wrapper, parallel path, rule, noise, constants, … |
| Reduction | what collapses into what |
| Risk | low / medium / high |
| Confidence | high only when behavior equivalence is clear |
| Effort | trivial / small / larger |

Default to **high confidence + low risk + trivial/small effort**.

Present a short ranked list before large multi-file edits. For a clearly scoped, tiny win already in the user’s focus, you MAY apply it directly when the user asked to reduce.

### 4. Agree scope

- Default slice: the best **few** obvious reductions, not every possible cleanup.
- If the user named a path or symbol, stay there first.
- Ask before expanding into unrelated modules or public API changes.

### 5. Apply

For each accepted reduction:

1. Implement the smallest collapse that removes the seam.
2. Update all call sites consistently.
3. Remove dead code left by the collapse (old helpers, unused imports, empty files) when clearly obsolete.
4. Keep names explicit; do not invent clever micro-frameworks.
5. Prefer existing shared modules over new packages/layers.

### 6. Verify

- Run focused tests and diagnostics relevant to the touched area.
- If no tests cover the reduced logic, add only what is needed to lock behavior when practical—or state the verification gap.
- Diff-review for accidental behavior changes (condition order, defaults, error messages, types).

### 7. Report

Summarize:

- What was reduced (seams removed)
- What was left alone and why (briefly)
- Tests/diagnostics run
- Residual candidates not done (optional, short)

## Decision checklist (per candidate)

Only reduce when **all** are true:

- [ ] Behavior is equivalent for known callers and edge cases
- [ ] The shared concept is real (not a coincidental shape)
- [ ] The result is easier to read than the original
- [ ] No new abstraction layer is required beyond a simple extract
- [ ] Risk is low or the user accepted the risk
- [ ] Scope stays within the agreed slice

If any box fails, skip or propose as a separate larger refactor—do not smuggle it in as “reduce.”

## Communication

- Prefer concrete findings over abstract principles.
- Show before/after briefly for non-obvious reductions.
- Do not moralize about DRY; show the seam and the simpler shape.
- Stop when remaining issues need design judgment, not simple reduction.

## Constraints

- Functionality preservation is the primary constraint.
- Do not change behavior “while you’re here.”
- Do not add dependencies to enable a reduction.
- Do not edit generated/vendored/build artifacts.
- Do not expand into performance, feature, or style work unless asked.
- Prefer fewer, complete reductions over a scattershot of partial cleanups.
