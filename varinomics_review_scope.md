# Varinomics Review Scope

This document governs what reviewers - human or LLM - must *not* flag when reviewing Varinomics code. It applies in addition to the coding-style documents; the coding-style docs govern what you *write*, this document governs what you *critique*.

## The governing rule

**Review against the project that exists, not against a project the reviewer imagines it could become.**

A project is defined by what it has committed to - its stated goals, its observable conventions, and its committed constraints. Reviewers draw on broad experience and know many things a project *could* be doing: supporting more platforms, shipping in more languages, being more accessible, scaling to more users, hardening against more attacks, emitting more telemetry, conforming to more audit standards, following more architectural patterns. That knowledge is useful - but only when the project has chosen to value the thing.

When a reviewer flags the absence of something the project has not adopted, the reviewer has substituted an imagined project for the real one. That is advocacy, not review. Advocacy is a legitimate activity; it just does not belong in a code-review report.

## Why this matters

Review bandwidth is finite, and noise displaces signal. A report that mixes real issues with findings based on imagined requirements trains the reader to ignore the report, which in turn buries the real issues. A reviewer who files out-of-scope findings is not being thorough - they are being *miscalibrated*. Thorough means reporting every issue that matters to *this* project. Miscalibrated means reporting issues that would matter to *some other* project.

## What counts as evidence of scope

A concern is in scope when the evidence supports it:

- **Explicit statements** in `CLAUDE.md`, `AGENTS.md`, `README`, architecture docs, or the user's direct instructions.
- **Conventions used consistently** across the code. A pattern applied in three or more places is evidence of commitment; a pattern applied once is not.
- **Build, deployment, and CI configuration.** A CI matrix with multiple OS targets implies cross-platform scope. A `.ts` file implies i18n. A fuzz-testing target implies fuzz testing is in scope. A performance-regression CI job implies performance is in scope.
- **Absence over a wide area.** If no file in a large codebase uses feature X, feature X is not in the project's ambition. Silence is evidence.

## What does **not** count as evidence of scope

- **What the reviewer would do on a greenfield project.** The project is not greenfield and is not the reviewer's.
- **What industry "best practice" says for category Y of software.** Best practices are general; projects are specific. A best practice becomes a requirement for *this* project only when *this* project has adopted it.
- **A single stray instance.** A leftover `tr()` call from a scaffolding experiment is not evidence that i18n is in scope. One `Accessible.name` on one button is not evidence that accessibility is in scope. Isolated code does not establish a commitment.
- **What would "also" be good.** Many true statements are not findings. "This would also work in dark mode" is not a finding unless dark mode is in scope.

## Common categories where the rule applies

The rule above is the whole rule; the list below is illustrative, not exhaustive. When a reviewer encounters any of these dimensions, the first move is to check the project's actual commitment.

### Platform and architecture

Cross-platform portability is in scope when the project actually targets multiple platforms: cross-platform CI, abstraction layers used consistently, explicit porting goals in the docs. A Qt project deployed only to Windows, using `#ifdef Q_OS_WIN` and Win32 APIs directly, is a Windows project - treat it as one. A single `_mm_*` intrinsic in a Windows x86_64 target is not an ARM-portability bug. "This would break on Linux" is not a finding when Linux is not a target.

### Internationalization

i18n is in scope when the application ships in multiple languages: `.ts` files present and maintained, `qsTr()` / `tr()` used consistently across the UI, translation workflow in docs or CI, multiple locales declared in deployment. Absent that evidence, bare string literals are not a finding. A single stray `tr()` call is not evidence - the question is whether the app *actually* ships in multiple languages.

### Accessibility

Accessibility is in scope when the project states it as a requirement, or uses `Accessible.*` properties, keyboard-navigation affordances, and screen-reader hooks consistently across existing UI. A project that does not currently support accessibility has not made an error - it has made a product decision with real implementation cost. If you believe accessibility *ought* to be a requirement, raise it as a product question, not a review finding.

### Performance and scale

Performance concerns are in scope when the project has performance goals: latency SLOs, memory budgets, throughput targets, performance CI. "This allocation could be avoided" is a finding only when allocation cost matters in this code path, in this project. Micro-optimizations in a tool that runs once at startup and exits are not findings.

### Security and threat modeling

Security findings must name the threat they mitigate, and the threat must be in scope. "This is not constant-time" is a finding only when timing attacks are part of the threat model. "This doesn't sanitize X" is a finding only when X is a reachable untrusted input. A threat that the project has not accepted as in-scope is not grounds for a finding.

### Testing depth

Coverage expectations follow the project's own conventions. A project with unit tests for public APIs is not under-tested because it lacks fuzz harnesses, property-based tests, or mutation testing - unless those are stated goals. "This function has no test" is a finding against the project's own testing practice, not against a hypothetical 100%-coverage norm.

### Operational concerns

Logging, metrics, tracing, and alerting requirements vary by product. A command-line tool that prints and exits is not under-instrumented. A daemon running in production without tracing might be - but only if the project has committed to operational observability. Flag the gap only when the gap is real for *this* deployment.

### Architectural patterns

Dependency injection, hexagonal architecture, plugin models, event sourcing, clean-architecture layering, and so on are in scope when the project uses them. Flagging the absence of a pattern the project has not adopted is advocacy. "This should be an interface" is a finding only when interfaces are the project's idiom for this kind of thing.

### API and ABI stability

Backwards compatibility is in scope when external consumers exist. In a pre-launch repository where the project's `AGENTS.md` explicitly rules out compatibility obligations, "this breaks the API" is not a finding - the project has said it does not have an API in the stable sense yet. Reviewer-invented "users" do not establish compatibility obligations.

### Feature completeness

"This dialog lacks a Cancel button" is a finding only when Cancel was part of the design. "This error message could be friendlier" is a finding only when error-message polish is something the project has invested in elsewhere. Absence of a feature the reviewer thinks would be nice is not the same as absence of a feature the project committed to.

## When the target is unclear

If the project's scope on any dimension isn't obvious from the evidence:

1. Read the project's own governance files (`CLAUDE.md`, `AGENTS.md`, `README`).
2. Read the code: what is actually, consistently used?
3. If still unclear, **ask the user** what's in scope before writing the report. Do not fill the report with hypothetical findings based on guessed scope.

## Summary

Review findings fall into two buckets:

- **In scope.** The project has committed to X, either by explicit statement or by the consistent evidence of its code. A finding that X is missing, broken, or weaker than the project's commitment is valid.
- **Out of scope.** The reviewer believes X *would be* nice, useful, defensive, modern, safer, or more consistent with some other project's conventions - but the project itself has not committed to X. This is not a finding. It is advocacy, and it belongs in a product discussion, not in a review report.

Out-of-scope findings are not "nice-to-haves." They are noise that displaces signal. The test a finding must pass to belong in a review is not "is this a good idea in general?" but "is this a gap against what *this* project has adopted?"

If the answer to that second question is no, the reviewer has written a product proposal, not a review. Send it up the right channel; keep it out of the review report.
