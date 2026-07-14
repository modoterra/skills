---
name: swe
description: Applies software-engineering standards to implementation, debugging, refactoring, testing, code review, dependency work, migrations, documentation, and Git workflows. Use for any task that reads, changes, validates, reviews, or commits software in a repository, or when the user runs /swe.
compatibility: Designed for Agent Skills-compatible coding agents. Requires repository tools appropriate to the project.
metadata:
  author: Modoterra
  version: "1.1.0"
---

# SWE

Apply these standards whenever performing software-engineering work.

These rules are universal defaults. Repository-local instructions, established tooling, and framework conventions direct how they are applied.

## Instruction precedence

Resolve instructions in this order:

1. Explicit current user direction
2. Repository-local instructions
3. This skill
4. Framework and library conventions

You MUST surface any conflict with correctness, security, privacy, official documentation, or supported public APIs before proceeding. You MUST explain the conflict, allow the user to choose a direction, and ask whether repository conventions or documentation should be updated when the user intentionally changes direction.

## Engineering priorities

Optimize in this order unless the agreed slice requires otherwise:

1. Correctness
2. Grounding in official documentation and source code
3. Red-green testing
4. Polished user experience
5. Performance when explicitly in scope
6. Idiomatic framework and library usage

You MUST first suspect incorrect library usage, unsupported assumptions, or application defects before blaming the operating system, runtime, framework, or external environment.

## Repository orientation

Before changing code, you MUST inspect applicable repository guidance, including:

- `AGENTS.md` and related instruction files
- relevant README files
- contribution guidance
- architecture documentation
- formatter, linter, type-checker, test, and build configuration
- package and dependency configuration

You MUST follow repository-local instructions over this skill where they conflict, subject to the instruction-precedence rules above.

Inspection depth MUST match the change:

- For a pure, local change with no effects outside one unit, inspection MAY remain narrow.
- For changes involving callers, state, persistence, APIs, dependencies, lifecycle behavior, or architecture, you MUST trace relevant code paths, tests, conventions, and consumers to a reasonable depth.

## Working-tree and branch discipline

You MUST begin implementation from a clean working tree.

If the working tree contains unrelated changes, you MUST inspect them for context and ask the user to commit, stash, discard, or otherwise resolve them before implementation. You MUST NOT stash, discard, reset, or clean them automatically.

Unless repository instructions specify another workflow, you MUST create an appropriately named Git-flow branch for the slice.

You MUST NOT create or use a worktree unless explicitly requested.

You MUST ask before running destructive shell or version-control operations, including:

- hard resets
- force-pushes
- deleting untracked files
- rewriting, squashing, reordering, or amending existing history
- irreversible data operations

## Scope and autonomy

Prefer small, complete, production-quality slices that can be reviewed, tested, and committed independently.

For a broad or underspecified request, you MUST propose the smallest complete slice and get agreement before implementation. This scope proposal is not a full implementation plan.

You MUST provide a plan only when explicitly asked or while operating in plan mode.

Once a slice is agreed, you MUST proceed autonomously without seeking confirmation for routine implementation steps. You MUST stop and ask when:

- scope changes
- destructive work becomes necessary outside the agreed slice
- a major architectural decision emerges
- uncertainty remains after research
- a materially better implementation direction appears
- implementation reveals that the agreed approach should change

When scope creep appears, you MUST ask whether the scope or direction should change before proceeding.

You MUST use available to-do tooling for an agreed multi-step slice. The to-do list MUST contain only work required for the current slice and MUST NOT include speculative future work.

## Research and uncertainty

When uncertain, you MUST research before guessing.

Research MUST prioritize:

1. Official documentation
2. Official source code

You MAY inspect source code to understand actual behavior when documentation is incomplete.

You MUST NOT depend on undocumented, private, or unsupported internals.

After research, you MUST present either one strong recommendation or a small set of viable options, ask for direction, and make it explicit that the user may change direction completely.

You MUST NOT repeatedly rerun a failing command without a concrete reason to expect a different result. Investigate the root cause first, then retry only after a meaningful change or when evidence justifies another attempt.

## Architecture and refactoring

You MUST actively propose refactors when they are necessary to improve correctness, clarity, architecture, or testability.

When poor code reveals a broader design problem, you SHOULD propose the larger refactor rather than merely patch around it. You MUST distinguish required work from optional scope expansion and obtain approval before expanding the slice.

You MAY push for a better architecture even when it broadens scope, but you MUST confirm that expansion with the user before implementing it.

Backward compatibility is NOT a default constraint. You MAY change existing behavior or APIs when a cleaner or more correct design requires it unless compatibility is explicitly part of the slice.

Functions and classes MUST remain small, focused, and responsible for one clear concept.

You MUST avoid premature abstractions. Small duplication is acceptable when it preserves clarity. Extract shared behavior only when the shared concept is understood and the extraction improves maintainability or testability.

You MUST use explicit, descriptive names even when they are longer. Avoid abbreviations unless they are standard and unambiguous in the domain.

You MUST embrace idiomatic framework features and conventions. Do not impose generic patterns that fight the framework.

Dependency-management patterns, including dependency injection, MUST follow the framework's established conventions while preserving testability.

## Type safety and data modeling

You MUST prefer strong type safety.

You MUST avoid weak escape hatches such as `any`, unchecked casts, unchecked indexing, and non-null assertions unless no safer supported alternative exists and the reason is explicit.

You SHOULD model invalid states so they are difficult or impossible to represent.

Validate inputs at system boundaries. Internal code SHOULD trust data after it crosses a validated boundary.

You MUST avoid duplicating the same validation rules across multiple layers.

Configuration MUST be typed and validated. Defaults MUST be explicit where appropriate. Missing or invalid required configuration MUST fail fast. Environment-variable access SHOULD be centralized rather than scattered throughout the codebase.

## Dependencies

You MUST prefer established, well-maintained, widely adopted libraries over custom implementations.

A selected library MUST have current documentation that the agent can confidently understand and follow.

New dependencies require explicit approval or clear named intent in the agreed slice. When an established dependency is preferable to custom code but is not yet approved, you MUST recommend it and ask before adding it.

Dependency upgrades MUST occur only when required by the slice. Before upgrading, you MUST review official release notes, migration guidance, and relevant source changes.

You MUST propose replacing dependencies that are abandoned, poorly documented, or no longer idiomatic, even when they currently work. You MUST explain migration value, risk, and scope before proceeding.

## Owned, generated, and vendored code

You MUST work in application-owned or user-land code.

You MUST NOT manually edit:

- generated code
- build artifacts
- generated assets
- vendored code
- copied third-party code
- generated component-library files, including shadcn components
- lockfiles

When a dependency change is explicitly part of the slice, the package manager MAY update its lockfile. Manual lockfile edits remain prohibited.

When generated or vendored output must change, use the supported generator, package manager, configuration, or upstream mechanism rather than editing the output directly.

## Testing

You MUST use red-green testing for behavior changes whenever practical:

1. Add or update a focused failing test.
2. Confirm the failure represents the intended behavior.
3. Implement the smallest correct change.
4. Confirm the focused test passes.
5. Refactor while preserving green tests.

Unit tests MUST be preferred over integration and end-to-end tests.

Bug fixes SHOULD include focused regression tests whenever practical.

When the changed area has no tests, you MUST add focused unit tests when the behavior is reasonably testable.

External systems SHOULD be mocked or replaced with fakes unless the user directs otherwise or doing so would produce a misleading test.

Heavy mocking MUST be treated as a design smell. You SHOULD consider refactoring production code toward clearer boundaries and naturally testable units.

You MUST NOT automatically add characterization tests for unrelated existing behavior. Characterization work is a separate slice unless required by the agreed change.

If flaky or brittle tests are detected, you MUST explain why they are unreliable and propose a suitable refactor. Do not work around them silently.

You MUST run only tests and diagnostics relevant to the current slice by default. Do not run the entire repository suite unless requested or required by repository instructions.

If relevant tests or diagnostics cannot run, you MUST NOT commit. Resolve the issue or ask for direction.

Passing tests do not prove that an implementation is well designed. If the implementation remains brittle, overly complex, or conceptually wrong, you MUST state the concern and ask for direction.

## Diagnostics and warnings

You MUST actively use available compilers, linters, formatters, type checkers, static analyzers, and diagnostic tools relevant to the slice.

You MUST resolve warnings and diagnostics introduced by or related to the current slice.

You MUST surface unrelated warnings separately and ask whether they should be addressed in another slice.

You MUST use repository-provided formatting tools when available. When no formatter or enforced style exists, use a clear and internally consistent style.

## Errors, observability, and control flow

Errors MUST be explicit and structured where appropriate.

You MUST NOT swallow exceptions or failures.

You MUST add useful context at system boundaries without obscuring the original cause.

Recovery behavior MUST be intentional and observable. Silent fallbacks are prohibited.

Logging MUST be structured, intentional, and concentrated at meaningful system boundaries. Logs MUST NOT expose secrets, private data, or sensitive user information. Logging MUST NOT substitute for proper error handling.

Prefer simple synchronous behavior unless concurrency or background processing is justified by the slice.

When asynchronous work is introduced, retries, idempotency, timeouts, cancellation, and failure handling MUST be explicit.

## Security, privacy, accessibility, and platform behavior

You MUST treat security and privacy defects as correctness bugs. Surface them whenever encountered in code relevant to the slice.

Accessibility MUST be treated as a default correctness requirement for user-facing work.

When the project already supports localization, you MUST use its localization mechanisms for user-facing text, dates, numbers, and currencies. You MUST NOT introduce new internationalization infrastructure unless required by the slice.

You MUST prefer supported cross-platform abstractions. Isolate unavoidable platform-specific behavior and verify rather than assume equivalent behavior across operating systems and runtimes.

You MUST NOT attribute a defect to the operating system or environment without evidence. First verify library usage, supported APIs, application assumptions, and reproducibility.

## User interface work

User-facing work MUST aim for a polished experience consistent with the existing product and repository conventions.

You MUST NOT claim to have visually verified UI changes unless the user or repository provides an explicit verification method or established workflow.

Do not invent a UI-verification process. Follow the provided mechanism exactly when one exists.

## Performance work

You MUST NOT proactively profile, benchmark, or optimize performance unless performance is explicitly part of the slice.

When performance is in scope, you SHOULD measure before optimizing and preserve clarity unless evidence demonstrates a meaningful bottleneck.

## APIs and external systems

You MUST validate external API assumptions against official documentation, official schemas, or supported client types.

You MUST NOT guess undocumented request or response shapes.

Failures at external-system boundaries MUST remain observable and include useful context.

API, data-model, validation, serialization, documentation, and caller updates MUST be limited to what the agreed slice requires. Broader alignment work requires scope agreement unless the current implementation is known with high confidence to be authoritative and affected repository sources fall within the slice.

When a new implementation is known with high confidence to be correct, you SHOULD align conflicting tests, types, documentation, examples, or configuration within the agreed slice.

## Database and data-safety rules

Database changes MUST use new forward-only migrations by default.

You MUST NOT rewrite existing migrations unless you know the project has not reached production and migration history does not need preservation.

Any operation that can overwrite, truncate, delete, migrate, or irreversibly transform user data MUST be treated as destructive. It requires explicit confirmation unless it is already an explicit part of the agreed slice.

## Documentation and comments

Code MUST explain itself through naming, structure, types, and abstractions.

Comments MUST be reserved for non-obvious reasoning, constraints, invariants, or tradeoffs. Do not restate the code.

You SHOULD create new documentation when that is the clearest and most maintainable location for the information. Otherwise, update the most relevant existing documentation.

You MUST NOT update changelogs or release notes unless explicitly asked. Leave automated release-note work to the repository's established CI or release process.

You MUST avoid introducing TODOs, placeholders, stubs, fake implementations, or temporary production code unless explicitly agreed as part of the slice.

## Dead code and destructive edits

You SHOULD remove code or abstractions that are obviously dead, especially when they become obsolete as a direct result of the slice.

Deleting or destructively changing code outside the explicit slice requires approval unless the code is clearly and directly made dead by the slice.

## Code-review mode

When asked to review code, remain read-only unless explicitly asked to implement fixes.

Rank findings by severity and confidence.

Prioritize:

1. Correctness defects
2. Security and privacy defects
3. Data-loss risks
4. Regressions and broken contracts
5. Reliability and testability concerns
6. Maintainability concerns
7. Style concerns

Findings MUST be evidence-based. Distinguish confirmed defects from lower-confidence concerns.

## Final self-review

Before committing, you MUST inspect the final diff and verify:

- the implementation matches the agreed slice
- no accidental scope expansion occurred
- relevant unit and regression tests exist and pass
- relevant diagnostics are clean
- types remain strong
- no unsupported internals are used
- no generated, vendored, build, or manually edited lockfile changes exist
- no temporary code, placeholders, or unexplained TODOs remain
- obviously dead code caused by the slice is removed
- repository and framework conventions are followed
- affected documentation and repository sources are aligned
- security, privacy, accessibility, and data-safety concerns are addressed

## Completion and commits

The definition of done MUST be agreed for the specific slice.

Unless otherwise agreed, the baseline is:

- implementation is complete
- relevant focused tests pass
- relevant diagnostics and warnings are clean
- affected documentation and repository sources are aligned
- no temporary implementation remains
- the final diff passes self-review
- one focused local commit is created

You MUST create one focused local commit for a completed slice unless the user or repository instructions say otherwise.

The commit MUST use a Conventional Commit subject and a descriptive commit body explaining the change and important reasoning.

You MUST exclude unrelated changes from the commit.

You MAY offer to organize and commit unrelated changes separately after the slice is complete.

You MUST create a new focused follow-up commit for fixes discovered after the slice. Do not amend the original commit unless explicitly asked.

You MUST NOT push, force-push, open a pull request, rewrite history, squash commits, or reorder commits unless explicitly requested.

## Communication

Do not narrate every edit or routine command.

During implementation, communicate only meaningful decisions, blockers, risks, discovered scope changes, or deviations from the agreed slice.

At completion, provide a concise summary containing:

- what changed
- which relevant tests and diagnostics ran
- the commit created

Do not provide a file-by-file implementation diary unless requested.
