# Seams 101

**SEAMS: Supervised Engineering via Adversarial Model Steps**

- **S**upervised = human in the loop, not autonomous
- **E**ngineering = the full process, not just coding
- **A**dversarial = the cross-model review pattern
- **M**odel = AI models doing the work
- **S**teps = one commit at a time

## What Is Seams?

SEAMS is an opinionated workflow for using AI coding agents to make changes in production codebases. It works for greenfield and brownfield alike, but is best suited to **brownfield, complex, or critical** source code -- anywhere a wrong step compounds and can't be easily undone.

It sits on top of the RALPH (Recursive Autonomous Loop with Planning & Human-oversight) concept but adds structure that RALPH alone doesn't provide: explicit approval tiers, small reviewable commits, cross-model review, and a human always in the loop at the right moments.

The name comes from the core idea: instead of asking an agent to deliver a whole user story in one pass, you work **one commit seam at a time**. Each seam is a small, logical, reviewable slice of work -- like a seam in fabric that joins two pieces together without trying to be the whole garment.

## Where SEAMS Sits

Agentic coding exists on a spectrum:

```
Move fast, break things ◄────────────────────────► Failure is not an option
(Prototypes, MVPs)                                  (Aviation, medical devices)

Vibe coding ──► RALPH ──► SEAMS ──► Safety-critical formal verification
```

SEAMS sits toward the right but it's not at the extreme. It's for projects where **bugs have real consequences but you're not certifying flight software**. Production SaaS, payment systems, auth flows, data migrations -- things where shipping a mistake means angry customers, data loss, or a 3am incident, not a plane falling out of the sky.

The approval tiers are the dial that lets you calibrate where on the spectrum you sit for a given project. More on those below.

## Why SEAMS Exists

Vanilla RALPH works well for greenfield projects where the agent can generate a whole feature from scratch. But as complexity and risk increase:

- The agent doesn't have enough understanding of what's already built to make safe, large changes in one pass.
- Context windows, no matter how large, can't hold every relevant detail about a mature codebase.
- A single bad decision compounds -- the agent keeps building on top of it.
- The human has no visibility into what happened during a multi-hour agent session.

SEAMS addresses this by breaking the "engineering loop" into smaller, inspectable steps with clear decision points about when the agent can keep going and when it must stop for human review.

## The Repeating Pattern

One pattern runs through every phase of SEAMS: **create, then cross-review with a different model until convergence**. It applies to PRDs, plans, and code equally. The human drives the creative/divergent thinking; the agents handle the convergent refinement. You know a phase is done when the reviewer is reduced to nitpicking trivia.

This is not the same as "get a second opinion." It's a structured loop where findings go back to the author for action, the author re-presents, and the reviewer checks again. The goal is convergence on a tighter artifact -- not just a list of suggestions.

## The Process

### Phase 1: PRD Creation & Adversarial Review

Before any code is written, create a solid PRD (Product Requirements Document):

1. **Draft the PRD** with one model (e.g., Claude). Use an interactive session -- let it ask questions, unravel ambiguities, and flesh out the requirements.
2. **Send it to a different model** (e.g., Codex) for adversarial review. Tell it: "Don't touch it, just pick it apart. Be super picky."
3. **Bounce the feedback back** to the original model to address the issues.
4. **Repeat** until the reviewer is nitpicking trivia (e.g., "change 'columns' to 'column'"). That's your convergence signal.

The key insight: the human provides the divergent thinking (the creative vision, the business context). The review loop's job is **convergence** -- tightening ambiguities and closing gaps so the implementation agent won't have to guess.

### Phase 2: Plan & Working Agreement

Turn the PRD into an actionable plan:

- Break it into a **checklist** of items with stable IDs (e.g., `PR1A-001`, `CH-001`).
- Order them by user journey flow -- build from what the user can do first, then layer on complexity.
- Classify each item by **tier** (risk level) and **merge strategy** (independent or stacked).

Then, just like the PRD phase, **cross-review the plan between two models until they converge**. One model drafts the plan; the other reviews it for gaps, ambiguities, missing checklist items, incorrect ordering, and unrealistic scope boundaries. The plan bounces back and forth until the reviewer is down to nitpicks.

This matters because the plan is what the implementing agent will follow seam-by-seam. Ambiguities in the plan become bad decisions in code. The cross-review catches those before any code is written -- which is orders of magnitude cheaper than catching them after three seams have built on a flawed assumption.

### Phase 3: Seam Execution Loop

This is the core loop. Each seam produces two artifacts:

- A **seam plan** -- written before implementation, reviewed before any code is touched. Captures what was discovered, what will change, what risks exist, and any proposed modifications to the plan itself.
- A **seam report** -- written after implementation, reviewed before commit. Captures what actually changed, what was verified, and how to review it.

For each checklist item:

1. **Discover** -- investigate the code, understand what's already in place.
2. **Write a seam plan** -- propose the changes, classify the tier, assess risks.
3. **Plan review** -- a different model reviews the plan. For Tier 3, the human must also approve before implementation starts.
4. **Implement** the seam per the approved plan.
5. **Verify** -- run the smallest test loop that gives meaningful confidence.
6. **Write a seam report** -- document what changed, what was verified, how to review.
7. **Report review** -- the reviewer validates claims against the actual diff.
8. **Commit** after approval.
9. **Push PR** -- wait for CI and automated reviewers. Address feedback, re-push until green.
10. **Merge and propose the next seam.**

Each iteration of this loop can be a **fresh agent session**. The agent catches up by reading the plan, all existing seam plans and reports in order, and git state. This cold-start pattern is deliberate -- it prevents context window degradation and forces the documentation to be good enough for a "new hire" to pick up mid-stream.

**Plan-only seams** are valid. When discovery reveals a checklist item is already satisfied or the plan needs correcting, the seam is just a plan update -- a seam plan with no corresponding code change or report. It still goes through review and gets a sequence number.

## The Three Approval Tiers

The approval tiers are the dial on the spectrum. They determine how much autonomy the agent gets based on the risk of each change. You configure them for your project's risk profile:

- **Startup MVP?** Most things are Tier 1/2. You rarely block. It feels close to RALPH.
- **Mature product with paying customers?** More Tier 2/3. The agent keeps moving on safe stuff but stops for anything that touches money or user data.
- **Regulated industry?** Almost everything is Tier 3. The agent proposes, the human approves each step.

What counts as "risky" is project-specific. The examples below are illustrative -- you define your own risk categories.

### Tier 1 -- Auto-commit
**What**: Import path fixes, doc updates, test renames, config changes, test-only changes.

**Policy**: After plan review and report review pass, commit and continue. The human reviews asynchronously.

### Tier 2 -- Commit-and-continue
**What**: Mechanical code moves with no behavior change, PR review cleanup, new integration tests that don't alter runtime code.

**Policy**: After plan review and report review pass, commit, push, and start the next seam. The human reviews asynchronously.

### Tier 3 -- Stage-and-block
**What**: Auth flow changes, schema migrations, resolution logic changes, anything that changes database state after a user action. Anything touching billing or authentication.

**Policy**: Plan review must pass AND the human must approve before implementation starts. Report review must pass AND the human must approve before commit.

**Default**: If unsure which tier, default to Tier 3.

The reviewer agent also critiques the tier classification -- if the implementing agent says Tier 2 but the reviewer thinks it should be Tier 3, that disagreement surfaces before anything is committed.

## Seam Artifacts

Every seam produces structured markdown artifacts saved to `.seams/{issue}/` (gitignored). They are sequentially numbered and never overwritten:

- `0001-CH-001-auth-cleanup-plan.md` -- the seam plan (what will change and why)
- `0001-CH-001-auth-cleanup-report.md` -- the seam report (what actually changed)

Both share the same sequence number. The plan comes first; the report follows after implementation.

These are review artifacts, not committed code. They serve as the audit trail of what was planned, what was reviewed, and what was done. Any future agent session can reconstruct the full history by reading them in order.

## Cross-Model Review

The same adversarial review pattern from Phases 1 and 2 continues into implementation. A second agent, running a **different model**, acts as the reviewer for each seam:

- It reviews **seam plans** before implementation starts -- validating the approach, scope, tier classification, and risk assessment.
- It reviews **seam reports** after implementation -- performing claims-vs-reality validation against the actual code diff.
- Feedback bounces back to the implementing agent for fixes. The implementing agent addresses the findings, updates the artifact, and re-presents.
- This typically takes one or two passes before the seam is clean.

The diversity matters: models are trained differently, so they genuinely catch different things. This is the agent equivalent of diversity of perspective in a human team.

The review is not a rubber stamp. In real sessions, reviewers have flagged stale comments that described pre-refactor behavior, identified cross-provider collision bugs the implementer missed, caught duplicated test infrastructure that should be shared, pushed back on tier classifications, and challenged plan assumptions before any code was written.

## Scope Control and Discovery

During implementation, the agent will inevitably discover things that weren't anticipated in the plan. The SEAMS process handles this explicitly:

- The agent **does not silently add new work** to the checklist. It flags what was discovered, explains why it matters, and states whether it blocks the current seam or should be deferred.
- Proposed scope changes go in the seam plan's "Proposed Plan Changes" section and are reviewed before taking effect.
- The human confirms before anything new enters the checklist. New items get a stable ID and are tracked the same way as everything else.
- The implementing agent is expected to **proactively investigate** before proposing a seam -- reading the relevant code, checking what's already in place, and reporting a concrete assessment before writing any code.

If the implementer discovers mid-implementation that the approved plan is wrong, it must **stop and flag** -- not quietly adjust. An amended seam plan is written, re-reviewed, and then implementation continues.

## Deferred Cleanup Tracking

Brownfield work often requires temporary bridge code -- dual-write logic, compatibility shims, vendor-specific branches that should eventually become trait-driven. SEAMS tracks these explicitly:

- Every piece of temporary code gets a `// TODO @agent: CH-XXX will remove this` comment in the code itself, naming the specific checklist item responsible for cleanup.
- The seam plan identifies temporary code that will be introduced. The seam report lists every instance in a Deferred Cleanup section.
- The reviewer verifies that the target checklist item actually covers the cleanup. If it doesn't, it gets flagged for a plan update.

This creates a chain of accountability across seams. No temporary code exists without a named owner and a removal plan.

## Key Principles

**One commit at a time.** Not one feature. Not one user story. One logical, reviewable commit. This is the fundamental difference from RALPH.

**Review before and after.** Seam plans are reviewed before implementation starts. Seam reports are reviewed before commit. This catches bad approaches upstream (cheap to fix) and validates execution downstream (catches reality vs intent gaps).

**Cross-review at every phase.** PRDs, plans, seam plans, and seam reports all get adversarially reviewed by a different model. The pattern is always the same: create, review, address findings, re-review, converge.

**The human is the divergent body.** The agent helps converge (implement, refine, review). The human provides creative direction, business context, and final judgment. The review loop is convergent by design -- it tightens, it doesn't wander.

**Every session is day one.** Each agent iteration starts fresh. It re-reads the plan, the seam history, and git state. This avoids context window decay and forces the documentation to be good enough for a "new hire" to pick up.

**Supervised, not autonomous.** SEAMS is explicitly a supervised process. The human is always nearby, reviewing seam plans and reports, approving Tier 3 changes. Bots never merge. The goal is to make the human's review time efficient, not to eliminate it.

**No direct plan edits.** The plan document is only modified through the seam process. Plan changes are proposed in seam plans, reviewed, and applied after approval. This keeps the plan accurate and auditable.

**Fix upstream, not downstream.** If the agent produces bad plans, don't critique the plan at the implementation step. Go fix how plans get generated. If the agent keeps misclassifying tiers, fix the tier definitions. Push corrections to the source.

**Brownfield is the real test.** Greenfield is easy -- low entropy, the model knows what "average code" looks like. Everything becomes brownfield eventually. SEAMS is designed for the reality that comes after the first few passes.

## How SEAMS Differs from RALPH

SEAMS builds on RALPH, so someone could reasonably say "that's just RALPH." And they'd be right -- in the same way that Scrum is "just iterative development." The primitives are the same: a loop, a plan, fresh sessions. What SEAMS adds is opinion about where the human belongs, when the agent must stop, and how risk flows through each step.

| | RALPH | SEAMS |
|---|---|---|
| **Unit of work** | One user story / spec per iteration | One commit per seam |
| **When review happens** | After the iteration completes (post-hoc) | Before implementation (plan) and after (report) |
| **Commit policy** | Auto-commit after each iteration | Tiered -- auto-commit trivial changes, block on risky ones |
| **Human involvement** | Human writes specs; loop runs autonomously | Human supervises continuously via structured approval tiers |
| **Plan creation** | Single-pass or interactive with one model | Adversarial cross-model review until convergence |
| **Plan maintenance** | Not a defined step | Plan changes go through seam plans and review |
| **Scope control** | Defined upfront in the spec | Agent proactively flags discoveries mid-flight, human confirms before adding |
| **Risk awareness** | Not a first-class concept | Explicit risk matrix and decision levels per seam |
| **Deferred cleanup** | Not tracked | Tracked with in-code TODO comments and cross-seam accountability |
| **Target environment** | General-purpose (strongest in greenfield) | Designed specifically for brownfield |

The fundamental tension is throughput vs. safety. A pure RALPH loop maximises throughput -- it'll chew through a greenfield plan fast with minimal human attention. SEAMS trades some of that speed for correctness in codebases where a wrong database migration or auth change can't be easily undone.

## What's Next

Read [seams-in-practice.md](seams-in-practice.md) for the operational guide -- how to set up a SEAMS session, the file structure, the roles and agents, the dispatch orchestration layer, and what to expect on your first run.
