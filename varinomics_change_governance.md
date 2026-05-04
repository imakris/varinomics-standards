# Varinomics Change Governance

This document governs how Varinomics work that spans more than one commit is planned, executed, and reviewed. It applies to API migrations, refactors, multi-step features, and any other change too large to land atomically.

The coding-style and review-scope documents tell you *how to write* and *what to critique*. This one tells you *how to keep a multi-batch change from drifting* — getting larger than planned, accumulating duplicate ways to do the same thing, leaving legacy behind that nobody ever deletes, or silently reducing the contract you said you were delivering.

## Why this exists

Multi-step changes drift. The reasons are predictable:

- **The original plan grows.** Each batch surfaces "while we're at it" cleanup that the planner never scoped. The change ships at 2x to 5x the planned size, often with the original goal diluted by accumulated scope.
- **Old and new coexist forever.** Each migrated call site replaces the old shape with the new one. The old shape stays public for "compatibility" or "until everyone migrates." Nobody migrates. The codebase ends up offering two ways to do the same thing, indefinitely.
- **Helpers are orphaned but never deleted.** A migration removes the only caller of helper `X`, but the migration commit doesn't remove `X`. A future reader sees `X` and assumes it's still load-bearing. It rots.
- **"Deferred" means "forgotten."** A reviewer or planner says "let's defer the typed handles / the binding work / the cleanup to a later batch." The later batch inherits the same caution and defers it again. The deferred work is never done unless someone forces it.
- **Silent degradation hides contract reduction.** When the new contract can't be delivered for some inputs (allocation failure, internal limit, missing capability), the easiest thing is to fall back to the old shape and pretend the call succeeded. The caller now gets something different from what they asked for, with no signal that anything went wrong.
- **Self-review misses recurring failure modes.** The same agent (human or LLM) that wrote the code is the worst reviewer of that code. Migration-discipline failures in particular cluster in blind spots that need an independent reader to catch.

The rules below are what we have learned to do about each of these.

## The rules

### 1. One canonical public way per capability

Every capability — function, command, configuration entry point — has exactly one public way to invoke it. When a migration introduces a new shape, the old shape is removed in the same commit.

No `_v2`, `_raw`, `_legacy`, `_old`, `_classic`, `_compat` aliases. No "preserved for external callers" stubs without a named external caller. No `// removed:` comments where the symbol used to be.

If an external caller depends on the old shape, they get a build break and migrate. The audit history makes the migration visible. If the project explicitly cannot tolerate a build break (e.g., a published ABI with installed third-party consumers), that exception is documented at the top of the migration plan and approved by the owner before the migration starts; it is not a default.

### 2. Hard-fail, never silent-degrade

When the new contract cannot be delivered, surface an explicit error. Never substitute a different result shape and report success.

Concretely: if a migrated function is documented to return a typed handle and the typed handle cannot be allocated (out of memory, internal limit, capability gap), return an error result with a non-OK status. Do not fall back to "JSON only" or "the old representation." The caller asked for the new contract; either deliver it or surface failure.

A degraded success looks the same as a real success to the caller. That is the bug.

### 3. Sibling code paths share their producers

Whenever two or more code paths produce the same logical result, they share the producer. The new code path does not get its own private builder while the old one keeps the original.

Concrete example: a synchronous "do operation" call and an asynchronous "resume the operation after a TAN prompt" call must both go through the same result builder. Otherwise the resumed call returns a different result shape than the synchronous one, and the contract bifurcates the moment the user hits a TAN prompt.

The general rule: if path A and path B end at "produce a `Foo`," they share the `build_foo` function. They do not each get their own copy.

### 4. Dead-code sweep happens in the same commit as the deletion that orphans it

When a migration removes the only caller of an internal helper, the helper is deleted in the same commit. Not "in a follow-up." Not "tracked for cleanup later." The same commit.

This is checked by `git grep` after the migration: every helper newly orphaned by the change is removed before the commit lands. The build will catch anything missed.

The reasoning is mechanical: if cleanup is deferred to a future commit, the future commit has to re-prove the helper is unused. That proof is harder later (more callers may have been added in the meantime) and it competes with the next batch's work for attention. So it never happens. Doing the deletion in the same commit costs nothing extra and removes a recurring source of dead code.

### 5. Refactor at N=3, not at N=6

When a pattern repeats across implementations, refactor the shared part out before the third occurrence — not after the sixth. Pattern recognition gets cheaper with each instance, but consolidation gets more expensive: each new instance is more code to back-fit into the abstraction.

Concretely: if the second time you write `populate_typed_X` looks 80% like the first one, factor the shared part out *before* writing the third. If you wait until the sixth, you are signing up to either rewrite all six or live with the duplication.

This rule is about cost, not aesthetics. Three copies is the inflection point where the refactor is cheaper than continuing to copy.

### 6. Cross-cutting infrastructure is shared, not forked

Dispatch, formatting, serialization, error mapping, logging, and similar cross-cutting concerns live in one shared helper per concern. New call sites use the helper. They do not reimplement it.

If a call site needs behavior the helper doesn't yet support, the helper grows a parameter or a callback. It does not get forked.

### 7. A "convenience" migration that grows the call site is not delivering the convenience

If the migrated call site is longer than the pre-migration call site, the design is not delivering the value it claimed. Stop and revisit before push.

The point of an envelope, a builder, a fluent interface, or any other "convenience" surface is to reduce caller boilerplate. If callers are now doing more work, the abstraction is in the wrong place.

This is a hard metric: count the lines, including any new local helpers the caller had to write to use the new API.

### 8. The defer-trap is real; treat "defer" as "drop unless explicitly scheduled"

Anything that gets "deferred to a later batch" must come with both:
- a concrete predecessor blocker (the technical reason it can't go in *this* batch), AND
- a concrete successor batch named (the next batch where it lands, with the file list).

Without both, the deferred work is functionally cancelled. Reviewers should treat any `// TODO: in a later batch`, `defer typed handles`, or `clean this up next time` comment as cancelled work unless it points at a named future commit.

This is the rule that produced the most surprise across past projects. Agents and humans both consistently underestimate how reliably "deferred" becomes "never."

### 9. Independent review before push, with round-trip until clean

A multi-batch change is not push-ready until an independent reader has reviewed it. "Independent" means a different model (for LLM reviews) or a different person (for human reviews) — not the same actor that wrote the code, even if framed as a "fresh look."

Self-review by the same agent misses the failure modes the agent is predisposed to make. The point of independent review is exactly to catch those.

Round-trip until clean. Two or three review cycles per batch is the floor, not the ceiling.

### 10. Gate the batch on observable signals before declaring it complete

Before a batch is considered done, the executor runs the project's gate: regenerate generated artifacts, run the relevant test target (not the broad suite — the relevant target for what changed), run drift checks (schema, exported symbols, snapshot tests), run integration tests for the migrated surface.

The reviewer's job includes verifying the gate was actually run, not just reading the diff. A batch where the gate was skipped is not push-ready, even if the diff looks clean.

### 11. Smaller is better; if a batch can't honor the rules, split it

If the batch as scoped cannot honor rules 1-10, split it. Splitting is preferable to bending the rules. There is no batch size that justifies leaving dead code, public aliases, silent degradation, or skipped gates.

This is the rule that backstops all the others. Most violations of rules 1-10 happen because "this batch is already big enough; we'll do that part separately." That logic is exactly what produces drift. The correct response is to split the batch, not to defer the rule.

## Reviewer's role

A reviewer of a multi-batch change verifies:

1. **Rule compliance.** Each of rules 1-11 was honored in the diff, not just intended.
2. **Gate execution.** The artifacts, tests, and drift checks the project requires were actually run, not skipped.
3. **No new orphans.** `git grep` for symbols touched by the diff confirms no dead helpers were left behind.
4. **No defer-trap.** Anything described as "deferred" is named in a successor batch with a file list. If not, treat as cancelled and flag.
5. **The contract.** The migrated surface delivers what its documentation says, not a quietly reduced version of it.

A reviewer who finds a rule violation produces specific, actionable feedback — not a request to "consider" or "perhaps look at" the issue. The rules are non-negotiable; either the diff honors them or it doesn't.

## When this document does not apply

Single-commit changes. Hotfixes. Documentation-only edits. Anything that lands atomically and is reviewed against the change as a whole.

This document specifically targets the failure modes that emerge when a change spans batches. If the change is small enough to land in one commit, the discipline is automatic — there is no "later batch" for things to drift into.
