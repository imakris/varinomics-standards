# Varinomics Review Scope

This document governs what reviewers - human or LLM - must *not* flag when reviewing Varinomics code. It applies in addition to the coding-style documents; the coding-style docs govern what you *write*, this document governs what you *critique*.

## The rule is narrow on purpose

This rule applies to **elective features** - things a project chooses to do or not to do. i18n, accessibility, cross-platform portability beyond the declared target, specific architectural patterns, specific feature completeness. A project adopts these, or it doesn't; a reviewer must check which before flagging absence.

The rule does **not** apply to baseline software quality. Correctness, data integrity, security hygiene, performance sanity, resource hygiene, maintainability - every project is implicitly committed to these. A reviewer never has to "check if the project signed up for correctness." Baseline-quality findings are always in scope.

This distinction matters because conflating the two is dangerous. Telling a reviewer not to flag security concerns because "the project hasn't adopted security as a requirement" would be catastrophic - especially for products like financial software, where security is foundational to the domain. The rule below is about features, not about quality.

## The governing rule

**For elective features, review against the project that exists - not against a project the reviewer imagines it could become.**

A project's elective commitments come from the project: its stated goals, its observable conventions, and its committed constraints. Reviewers know many electives a project *could* be doing - supporting more platforms, shipping in more languages, adopting particular architectural patterns, conforming to particular compliance regimes. That knowledge is useful when the project has chosen to value the thing. When it hasn't, applying the knowledge as a criticism is miscalibration - the reviewer has substituted an imagined project for the real one.

When a reviewer flags the absence of an elective feature the project has not adopted, the reviewer is doing advocacy, not review. Advocacy is a legitimate activity; it just belongs in a product discussion, not in a code-review report.

## Baseline vs elective

A finding falls into one of two categories, and they are treated differently:

### Baseline quality - always in scope

These are not adopted; they are assumed. Every project is committed to them by default:

- **Correctness.** The code does what it claims to do. Matches its documentation, matches its tests, matches its interface.
- **Data integrity.** No lost writes, no silent corruption, no torn updates under expected workloads.
- **Security hygiene.** No obvious exploits, no SQL injection, no hardcoded secrets, no unvalidated untrusted input, no credentials in logs. The *specific* threat model is domain-dependent (see below), but the baseline of "no obvious bugs that an attacker can walk through" is never opt-out.
- **Performance sanity.** No O(n²) where O(n) is trivial, no UI-thread blocking for seconds, no unbounded memory growth, no leaking resources. The *specific* performance targets are project-dependent, but "egregiously slow / wasteful" is never acceptable.
- **Resource hygiene.** Files closed, handles freed, threads joined, sockets shut down, locks released.
- **Maintainability baseline.** A new reader can understand the code in reasonable time. Names mean what they say. Structure is not actively hostile.

A reviewer flagging any of these has not overstepped scope. They may be wrong (the suspected bug may not exist), but the category is legitimate.

### Elective features - in scope only when adopted

These are choices a project makes. A reviewer must check whether the project has adopted them before flagging their absence:

- **Internationalization.** Shipping in multiple languages, translation workflow, locale-aware formatting.
- **Accessibility.** Screen-reader support, keyboard-only navigation, high-contrast modes, `Accessible.*` properties. *(See "Domain sets the baseline" below - a11y may be a legal baseline for some product categories.)*
- **Cross-platform portability beyond the declared target.** If the project targets Windows x86_64, "this would break on Linux" and "this would break on ARM" are not findings.
- **Specific architectural patterns.** Dependency injection, hexagonal architecture, plugin models, event sourcing. In scope when the project uses them; out of scope when it doesn't.
- **Specific compliance regimes.** FIPS, SOC2, PCI-DSS, GDPR-specific features. In scope when the project has adopted them; out of scope when the domain doesn't require them.
- **API / ABI stability.** Backwards compatibility obligations. In scope when external consumers exist; out of scope when the project's own `AGENTS.md` rules them out (e.g. pre-launch).
- **Feature completeness.** Cancel buttons, dark mode, undo history, configurability. In scope when part of the design; out of scope when the reviewer just thinks they'd be nice.

## Domain sets the baseline

The baseline is not universal - it rises with the project's domain. A CLI utility's baseline is "doesn't crash, handles common inputs, doesn't leak memory." A banking product's baseline is the utility baseline *plus* adversarial-user threat modeling, audit trails, financial-precision arithmetic, and regulatory compliance appropriate to the jurisdiction. A government-facing service's baseline may include accessibility by law.

Galanthus does not need to "adopt" banking-level security for banking-level security concerns to be in scope. The domain implies it. Similarly, a health-data product does not need to "adopt" HIPAA-equivalent handling for such handling to be baseline - the domain implies it.

The practical test: would a reasonable operator of software in this domain consider the behavior acceptable? If the answer is clearly no, the finding is baseline and in scope, regardless of whether the repo's docs mention it. If the answer is yes, and the concern is about a feature the project could reasonably choose not to provide, the finding is elective and requires adoption to be in scope.

## What counts as evidence that an elective feature is adopted

- **Explicit statements** in `CLAUDE.md`, `AGENTS.md`, `README`, architecture docs, or the user's direct instructions.
- **Conventions used consistently** across the code. A pattern applied in three or more places is evidence of commitment; a pattern applied once is not.
- **Build, deployment, and CI configuration.** A CI matrix with multiple OS targets implies cross-platform scope. A `.ts` file implies i18n. A fuzz-testing target implies fuzz testing is in scope.
- **Absence over a wide area.** If no file in a large codebase uses feature X, feature X is not in the project's ambition.

## What does not count

- **What the reviewer would do on a greenfield project.** The project is not greenfield and is not the reviewer's.
- **General best practice for "category Y of software."** Best practices are useful inputs to product decisions, not automatic requirements.
- **A single stray instance.** A leftover `tr()` call from a scaffolding experiment is not evidence that i18n is in scope. One `Accessible.name` on one button is not evidence that a11y is in scope. Isolated code does not establish a commitment.
- **What would "also" be good.** "This would also work in dark mode" is not a finding unless dark mode is in scope.

## When the target is unclear

If a concern might be elective *or* baseline, err toward baseline when the consequence of the bug is harm (data loss, exploit, user-visible breakage), and toward elective when the consequence is merely "feature missing." When in doubt:

1. Read `CLAUDE.md`, `AGENTS.md`, `README`, and any architecture docs.
2. Look at what the code consistently uses.
3. Consider the product domain. Banking ≠ CLI utility ≠ game ≠ government service - baselines differ.
4. If still unclear, **ask the user** what's in scope before writing the report.

## Found failures must be reported, regardless of provenance

The scope rules above govern *what categories* of issue a reviewer flags. This section governs *how findings are reported once made*. A reviewer or executor who has observed a failing test, a build error, a crash, a corrupted output, a leaked resource, or any other concrete failure must say so plainly in the report - whether or not the failure appears to have been introduced by the change under review.

The phrasing "X fails, but it's pre-existing" is forbidden. So is its softer cousin "X fails on master too, so out of scope." Pre-existence is not a license to omit. It is, at most, context to attach to the same finding.

The minimum a report must do when a failure is observed:

- **Name the failure.** Test name, error message, stack trace location, or the exact symptom. Not "some tests fail."
- **Attach it to the report's findings list.** Not as a footnote, not in conversation, not "for what it's worth." A finding.
- **State whether the change under review introduced it, perpetuates it, or merely exposed it.** All three categories are fair game; pretending none of them exists is not.

The failure mode this rule guards against is the cumulative one: every reviewer who silently passes over a pre-existing failure makes it harder to find later, and contributes to a culture where the next failing test is also "not my batch's problem." The cost of recording a finding is one paragraph. The cost of letting failures accumulate unmentioned, across many reviews, is a project where nobody knows what is actually broken.

### Multiple failures escalate

When a single review surfaces more than one failure - including a mix of "introduced by this change" and "pre-existing" - the report must do more than list them:

1. **Be louder about the count.** A summary line stating the total number of observed failures belongs at the top of the report, not buried in the findings list. "Three failing tests, one new and two pre-existing" is the kind of headline the next reader needs.
2. **Group findings.** Distinguish failures that block the change under review from failures that pre-date it. Both are reported; only the categorisation differs.
3. **Propose a concrete remediation plan.** For each failure, name the file and the rough fix shape. If the fix is genuinely unknown, say so explicitly and point at what would have to be investigated. Vague reassurance ("should be looked at later") does not count.
4. **Suggest delegating the remediation to an agent.** Multiple-failure cleanups are exactly the workload an automated executor handles well: mechanical, well-scoped, repetitive. A review that surfaces several failures should close with a recommendation to feed the remediation plan to an agent, not with a hand-off to "future cleanup time."

A review that observes failures but routes them anywhere other than the findings list is a defective review, regardless of what the diff itself looks like.

## Summary

Two buckets:

- **Baseline quality** (correctness, data integrity, security hygiene, performance sanity, resource hygiene, maintainability). Always in scope. Domain raises the baseline, never lowers it. A reviewer does not need to check whether the project "adopted" correctness.
- **Elective features** (i18n, a11y where not legally baseline, cross-platform beyond target, specific architectural patterns, specific compliance regimes, feature completeness). In scope only when adopted, by explicit statement or consistent convention. Advocacy for un-adopted electives belongs in a product discussion, not in a review.

The failure mode the rule guards against is the reviewer imposing an elective that the project has chosen not to take on. It does not, and must not, protect bugs or vulnerabilities - those are baseline concerns regardless of what any document says.

A separate failure mode, addressed in "Found failures must be reported, regardless of provenance" above, is the reviewer who *does* see a real failure but omits it because the failure is pre-existing or seems out of scope. Every observed failure goes in the findings list. Multiple observed failures escalate to a remediation plan with a delegation suggestion.
