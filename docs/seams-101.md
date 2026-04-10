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

One pattern runs through every phase of seams: **create, then cross-review with a different model until convergence**. It applies to PRDs, plans, and code equally. The human drives the creative/divergent thinking; the agents handle the convergent refinement. You know a phase is done when the reviewer is reduced to nitpicking trivia.

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
- Create a **working agreement** that defines:
  - The process the agent must follow
  - Approval tiers (see below)
  - A risk matrix for decision-making
  - Verification policy
  - Branching strategy
  - How seam reports should be written

Then, just like the PRD phase, **cross-review the plan between two models until they converge**. One model drafts the plan; the other reviews it for gaps, ambiguities, missing checklist items, incorrect ordering, and unrealistic scope boundaries. The plan bounces back and forth -- the reviewer flags issues, the author addresses them, the reviewer checks again -- until the reviewer is down to nitpicks. The same convergence signal applies: when the feedback is trivial, the plan is done.

This matters because the plan is what the implementing agent will follow seam-by-seam. Ambiguities in the plan become bad decisions in code. The cross-review catches those before any code is written -- which is orders of magnitude cheaper than catching them after three seams have built on a flawed assumption.

### Phase 3: Seam Execution Loop

This is the core loop. For each checklist item:

1. **Investigate** -- understand the root cause or requirement in concrete terms.
2. **Propose the next seam** -- scope the smallest meaningful commit.
3. **Implement** the seam.
4. **Update the plan** with progress and traceability notes.
5. **Run verification** -- the smallest loop that gives meaningful confidence.
6. **Stage** only the files for this seam.
7. **Write a seam report** -- a structured markdown file documenting what changed, why, what was verified, and how to review it.
8. **Present for review** -- stop and wait (or continue, depending on the approval tier).
9. **Commit** after approval, then propose the next seam.

Each iteration of this loop is a **fresh agent session**. The agent catches up by:

1. Reading the working agreement (the process it must follow).
2. Reading the plan document (the full checklist, schema changes, locked decisions).
3. Reading all existing seam reports in order (what's been done, what's pending review).
4. Checking `git log` to see what's been committed vs. what's still staged.
5. Checking `git status` for any uncommitted work in progress.
6. Based on all of the above, identifying the next checklist item and proposing the next seam.

This cold-start pattern is deliberate. It prevents context window degradation across long-running work and forces the documentation to be good enough for a "new hire" to pick up mid-stream. If the seam reports and plan aren't clear enough for a fresh agent to continue, that's a signal the documentation needs improving -- not that the context window needs to be bigger.

## The Three Approval Tiers

The approval tiers are the dial on the spectrum. They determine how much autonomy the agent gets based on the risk of each change. You configure them for your project's risk profile:

- **Startup MVP?** Most things are Tier 1/2. You rarely block. It feels close to RALPH.
- **Mature product with paying customers?** More Tier 2/3. The agent keeps moving on safe stuff but stops for anything that touches money or user data.
- **Regulated industry?** Almost everything is Tier 3. The agent proposes, the human approves each step.

What counts as "risky" is project-specific. The examples below are illustrative -- you define your own risk categories in the working agreement.

### Tier 1 -- Auto-commit
**What**: Import path fixes, doc updates, test renames, config changes, test-only changes.

**Policy**: Commit directly after verification passes. Write the seam report and continue immediately. The human reviews asynchronously.

**Analogy**: So trivial it's not worth stopping for. Just keep going.

### Tier 2 -- Commit-and-continue
**What**: Mechanical code moves with no behavior change, PR review cleanup, new integration tests that don't alter runtime code.

**Policy**: Commit, push, write seam report, and start the next seam immediately. The human reviews asynchronously. If the human spots an issue, they can flag it and the agent will address it.

**Analogy**: Substantial enough to document, but safe enough to keep moving. The seam report gives the human visibility without blocking progress.

### Tier 3 -- Stage-and-block
**What**: Auth flow changes, schema migrations, resolution logic changes, anything that changes database state after a user action. Anything touching billing or authentication.

**Policy**: Stage the files, write the seam report, and **stop**. Do not commit. Wait for explicit human approval.

**Analogy**: This is the "if we get this wrong, it's bad and possibly irreversible" tier.

**Default**: If unsure which tier, default to Tier 3.

The reviewer agent also critiques the tier classification -- if the implementing agent says Tier 2 but the reviewer thinks it should be Tier 3, that disagreement surfaces before anything is committed.

## Seam Reports

Every seam produces a structured markdown report saved to `.seams/GH-{issue}/NNNN-{slug}.md`. These are review artifacts (gitignored), not committed code. They include:

- **Checklist IDs covered** and one-sentence purpose
- **What changed** -- grouped by domain (schema, auth, API routes, tests, etc.) with file paths, functions, and reasoning
- **Grounding and standards** -- what local guidance was consulted, what agreement was followed
- **Staged files** list
- **Verification completed** -- every command run and its result
- **Risk matrix** -- scenarios, consequences, and decision levels
- **Deferred cleanup** -- any temporary code left in place, with `// TODO @agent:` comments in the code
- **Manual review guide** -- exact steps for the human to verify
- **Proposed commit message** and next seam

## Cross-Model Review

The same adversarial review pattern from Phases 1 and 2 continues into implementation. A second agent, running a **different model**, acts as the reviewer for each seam:

- It has its own review instructions (stored in `.seams/GH-{issue}/review.md`) that define what to check and how to assess quality.
- The implementing agent writes the seam report; the reviewer reads both the report and the actual code.
- The reviewer performs **claims-vs-reality validation** -- does the code actually do what the report says it does? Are there changes the report didn't mention? Did the agent follow the standards it cited?
- The reviewer also critiques the tier classification, scope discipline, test coverage, and risk matrix.
- Feedback bounces back to the implementing agent for fixes. The implementing agent addresses the findings, updates the seam report, reruns verification, and presents again.
- This typically takes one or two passes before the seam is clean.

In practice, the implementing and reviewing roles can be assigned to whichever models suit the task. For example, Codex might implement (it's strong at mechanical code changes) while Claude reviews (it's thorough at reasoning about design implications). Or the roles can be swapped. What matters is that the two agents are different models -- they're trained on different data by different teams, so they genuinely catch different things. This is the agent equivalent of diversity of perspective in a human team.

The review is not a rubber stamp. In real sessions, reviewers have flagged stale comments that described pre-refactor behavior, identified cross-provider collision bugs the implementer missed, caught duplicated test infrastructure that should be shared, and pushed back on tier classifications. The implementing agent then addresses each finding before the seam can progress.

## Scope Control and Discovery

During implementation, the agent will inevitably discover things that weren't anticipated in the plan -- a legacy fallback that's still live, a missing test case, a dependency between two checklist items that wasn't obvious upfront. The seams process handles this explicitly:

- The agent **does not silently add new work** to the checklist. It flags what was discovered, explains why it matters, and states whether it blocks the current seam or should be deferred.
- The human confirms before anything new enters the checklist. New items get a stable ID and are tracked the same way as everything else.
- The implementing agent is expected to **proactively investigate** before proposing a seam. In practice, this means the agent reads the relevant code, checks what's already in place, and reports back with a concrete assessment before writing any code. For example: "PR1A-001 is largely already in place -- the session actor is already provider-neutral. The real gap is in the OAuth fallback branch which still calls `upsert_orgs_for_login()`."

This prevents two common failure modes: scope creep (the agent quietly adds features nobody asked for) and scope blindness (the agent implements something that's already done or doesn't need doing).

## Deferred Cleanup Tracking

Brownfield work often requires temporary bridge code -- dual-write logic, compatibility shims, vendor-specific branches that should eventually become trait-driven. Seams tracks these explicitly rather than hoping someone remembers to clean them up later:

- Every piece of temporary code gets a `// TODO @agent: CH-XXX will remove this` comment in the code itself, naming the specific checklist item responsible for cleanup.
- The seam report includes a **Deferred Cleanup** section listing every instance: where it lives, why it still exists, and which future checklist item will remove it.
- The reviewer verifies that the target checklist item actually covers the cleanup. If it doesn't, it gets flagged for a plan update.

This creates a chain of accountability across seams. No temporary code exists without a named owner and a removal plan.

## CI as a Gate

Every seam that gets pushed to a PR must pass CI before the agent can proceed:

- Automated code review tools (Cadence, GitHub Copilot, Sentry) review the PR.
- The agent must address or explicitly dismiss each piece of feedback.
- The PR cannot progress until all checks are green.

This mirrors the process a human engineer would follow -- the agent isn't exempt from the team's quality gates.

## The Automation Layer: seams.sh

A shell script (`seams.sh`) automates the iteration loop:

```bash
./seams.sh                      # defaults: claude, 10 iterations
./seams.sh --tool codex         # use Codex
./seams.sh --tool claude 20     # 20 iterations
```

It reads `.env.seams` for the issue number, then repeatedly launches the agent with a bootstrap prompt. Each iteration:

1. Tells the agent to read the bootstrap file and follow instructions.
2. The agent catches up from seam files and git state.
3. The agent does one (or a few) seams.
4. The script checks for signals:
   - `<seams>COMPLETE</seams>` -- all done, exit successfully.
   - Tier 3 keywords -- stop and tell the human to review.
5. If neither signal, loop to the next iteration.

## Key Principles

**One commit at a time.** Not one feature. Not one user story. One logical, reviewable commit. This is the fundamental difference from RALPH.

**Cross-review at every phase.** PRDs, plans, and code all get adversarially reviewed by a different model. The pattern is always the same: create, review, address findings, re-review, converge. This isn't optional polish -- it's structural. Ambiguities caught in a PRD review save hours of wasted implementation. A plan gap caught before coding saves days of rework.

**The human is the divergent body.** The agent helps converge (implement, refine, review). The human provides creative direction, business context, and final judgment. The review loop is convergent by design -- it tightens, it doesn't wander.

**Every session is day one.** Each agent iteration starts fresh. It re-reads the plan, the seam history, and git state. This avoids context window decay and forces the documentation to be good enough for a "new hire" to pick up.

**Supervised, not autonomous.** Seams is explicitly a supervised process. The human is always nearby, reviewing seam reports, merging PRs. Bots never merge. The goal is to make the human's review time efficient, not to eliminate it. Time zones can help here -- different humans can supervise different seams, so the agent doesn't have to wait for one person to wake up.

**Fix upstream, not downstream.** If the agent produces bad plans, don't critique the plan at the implementation step. Go fix how plans get generated. If the agent keeps misclassifying tiers, fix the tier definitions. Push corrections to the source.

**Brownfield is the real test.** Greenfield is easy -- low entropy, the model knows what "average code" looks like. Everything becomes brownfield eventually. Vibe coding without checks and balances just gets you to brownfield faster, without the business value to show for it. Seams is designed for the reality that comes after the first few passes.

## File Structure

```
project/
├── docs/plans/
│   ├── _working-agreement-template.md    # Process definition
│   └── my-feature-plan-2026-04-07.md     # The plan with checklist
├── .env.seams                            # SEAMS_ISSUE=GH-364
├── seams.sh                              # Automation script
└── .seams/                               # gitignored
    └── GH-364/
        ├── bootstrap.md                  # Agent startup instructions
        ├── review.md                     # Reviewer agent instructions
        ├── 0001-CH-001-first-seam.md     # Seam reports (never overwritten)
        ├── 0002-CH-002-second-seam.md
        └── ...
```

## How Seams Differs from RALPH

Seams builds on RALPH, so someone could reasonably say "that's just RALPH." And they'd be right -- in the same way that Scrum is "just iterative development." The primitives are the same: a loop, a plan, fresh sessions. What seams adds is opinion about where the human belongs, when the agent must stop, and how risk flows through each step.

| | RALPH | Seams |
|---|---|---|
| **Unit of work** | One user story / spec per iteration | One commit per seam |
| **When review happens** | After the iteration completes (post-hoc) | Inline, before each seam progresses |
| **Commit policy** | Auto-commit after each iteration | Tiered -- auto-commit trivial changes, block on risky ones |
| **Human involvement** | Human writes specs; loop runs autonomously | Human supervises continuously via structured approval tiers |
| **Plan creation** | Single-pass or interactive with one model | Adversarial cross-model review until convergence |
| **Plan review** | Not a defined step | Cross-reviewed the same way PRDs and code are |
| **Scope control** | Defined upfront in the spec | Agent proactively flags discoveries mid-flight, human confirms before adding |
| **Risk awareness** | Not a first-class concept | Explicit risk matrix and decision levels per seam |
| **Deferred cleanup** | Not tracked | Tracked with in-code TODO comments and cross-seam accountability |
| **Target environment** | General-purpose (strongest in greenfield) | Designed specifically for brownfield |

The fundamental tension is throughput vs. safety. A pure RALPH loop maximises throughput -- it'll chew through a greenfield plan fast with minimal human attention. Seams trades some of that speed for correctness in codebases where a wrong database migration or auth change can't be easily undone. If you're building something from scratch, RALPH will get you there faster. If you're refactoring an auth system with real users, seams gives you the guardrails.

## Where Dispatch Fits

Dispatch is the messaging layer that will eventually automate the manual copy-paste between agents. Instead of the human copying the seam report from the implementing agent and pasting it to the reviewer, dispatch lets agents send messages to each other -- by role, not by identity. "Send this to a PR reviewer" rather than "send this to agent #3."

This turns the seams workflow from a human-orchestrated process into a semi-autonomous one where the human only intervenes at Tier 3 checkpoints or when something goes wrong.
