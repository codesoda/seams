# SEAMS in Practice

This is the operational guide for running SEAMS. It covers setup, the concrete workflow, roles, artifacts, and the dispatch orchestration layer. Read [seams-101.md](seams-101.md) first for the concepts.

## Prerequisites

You need:

- **Git** -- SEAMS is commit-oriented. You need a repo with a clean working tree.
- **Two AI coding agents** -- one to implement, one to review. They should be different models (e.g., Claude + Codex, Claude + Gemini). The cross-model diversity is what makes adversarial review effective.
- **A terminal** -- SEAMS runs in your shell.

No special tooling is required beyond what you already use for development. SEAMS is a process, not a platform. Dispatch automates the agent coordination but is not required to get started.

## Picking Your First Task

Choose something with these properties:

- **Small scope** -- a bug fix, a single API endpoint, a config change. Not a multi-week feature.
- **Real codebase** -- SEAMS works on greenfield and brownfield alike, but your first run will be more instructive on brownfield code where existing complexity gives the process something to do.
- **Low blast radius** -- your first run is about learning the workflow, not shipping something critical. Pick something where a mistake is recoverable.

## Setting Up

### 1. Write a PRD

Even for a small task, write a short PRD. This forces you to be explicit about what you're changing and why:

1. Open a session with your implementing model. Describe the task, let it ask questions.
2. Produce a short PRD covering: what's broken or missing, what the fix looks like, what's out of scope.
3. Send the PRD to your reviewing model: *"Review this PRD. Be picky. Flag ambiguities and unstated assumptions."*
4. Address the feedback, re-present. Stop when the reviewer is down to trivial nitpicks.

For a small task, this usually takes 1-2 rounds.

### 2. Create a Plan

Turn the PRD into a checklist:

1. Break the work into the smallest meaningful commits. Each one is a seam.
2. Give each item a stable ID (e.g., `CH-001`, `CH-002`).
3. Order by dependency -- what must exist before the next thing can be built.
4. Classify each item by tier (1/2/3) and merge strategy (independent or stacked).
5. Cross-review the plan with your second model until convergence.

Save the plan to `docs/plans/{issue}-plan.md`. The plan is a committed file -- it's the shared source of truth.

### 3. Set Up the File Structure

Create the SEAMS scaffolding in your repo:

```bash
# Create the seams directory for your issue (gitignored)
mkdir -p .seams/{issue}

# Add .seams to your .gitignore if it's not already there
echo ".seams/" >> .gitignore

# Copy the environment file from the template and fill in values
cp template/.env.seams .env.seams
```

If using dispatch, symlink the template's `prompts/` directory into your project's `.seams/` folder, copy `config/` and `dispatch.config.toml`, and customize the config files for your project.

The resulting structure:

```
project/
├── docs/plans/
│   └── {issue}-plan.md                  # The plan with checklist
├── .env.seams                           # Session configuration
├── prompts/                             # Agent prompts (from template)
│   ├── shared/                          # Methodology docs (all agents)
│   ├── config/                          # Overridable settings (_-prefixed)
│   └── agents/                          # Role-specific behavior
├── dispatch.config.toml                 # Dispatch cell definition
└── .seams/                              # gitignored
    └── {issue}/
        ├── 0001-CH-001-slug-plan.md     # Seam plans
        ├── 0001-CH-001-slug-report.md   # Seam reports
        └── ...
```

## The Execution Loop

Each seam has three phases:

```
Phase A: Plan and Review
─────────────────────────
1. Choose next item
2. Discover ──────────── investigate code before touching it
3. Write seam plan ────── what will change, why, risks, plan changes
   ┌─────────────────┐
   │ Reviewer reviews │
   │ the plan         │   ← approach, scope, tier, risks
   │                  │
   │ Tier 1/2: auto   │   ← reviewer approves → go
   │ Tier 3: human    │   ← human must approve before work starts
   └─────────────────┘

Phase B: Implement and Review
──────────────────────────────
4. Implement ──────────── code + tests per approved plan
5. Verify ─────────────── smallest loop giving meaningful confidence
6. Stage + seam report ── what actually changed, verification results
   ┌─────────────────┐
   │ Reviewer reviews │
   │ the report       │   ← claims vs reality, tier, deferred
   │                  │     cleanup, risk matrix
   └─────────────────┘
7. Approve → commit

Phase C: PR and CI
───────────────────
8. Push PR ────────────── to main or stack
   ┌─────────────────┐
   │ Wait for CI +    │
   │ reviewers        │   ← resolve locally, iterate with
   │                  │     reviewer, push again, rewait
   └─────────────────┘
9. Merge → next seam
```

The plan review (step 3) and report review (step 6) are both **iterate-until loops** -- they repeat until the reviewer approves. Step 8 is also a loop -- CI failures and reviewer feedback are addressed locally and re-pushed until everything is green.

**Plan-only seams**: Some seams are purely plan corrections with no code changes (e.g., discovery reveals a checklist item is already satisfied). These produce a seam plan but no seam report. They still go through plan review.

**Mid-implementation divergence**: If the implementer discovers during implementation that the approved plan is wrong, it must stop and flag -- not quietly adjust. An amended seam plan is written, re-reviewed, and then implementation continues.

## Roles

### Implementing Agent

Runs the seam loop: discovers, writes seam plans, implements, verifies, writes seam reports, stages. Starts each session cold by reading its prompt, the plan, all prior seam artifacts, and git state. Does not silently add scope -- flags discoveries and waits for confirmation.

### Reviewing Agent

A different model from the implementer. Reviews both seam plans (before implementation) and seam reports (after implementation):

- **Plan review**: Is the approach sound? Is the tier correct? Are risks identified? Is the scope right?
- **Report review**: Do claims match reality? Is the diff what the report says? Are deferred cleanup markers in place?

Sends numbered, actionable findings. Approves when findings are down to nitpicks.

### Test Runner Agent

Executes verification commands on behalf of the implementer and reports results honestly. Does not implement code or interpret results -- just runs commands and returns exact output. This separation means the implementer cannot fabricate test results.

### GitHub Agent

Manages the PR lifecycle: creates branches and PRs via `gh` CLI, monitors CI checks, handles automated review feedback (Copilot comments, Sentry reports, linter output), and reports status back to the coordinator.

### Coordinator Agent

The user's proxy inside the agent system. Orchestrates the other agents via dispatch, manages the two-phase review flow, enforces tier gates, and surfaces results, questions, and approval requests to the human. The coordinator is the only agent the user interacts with directly.

### Human Supervisor

Provides what agents cannot:

- **Scope authority** -- decides what goes in each seam, splits seams, defers items.
- **Domain expertise** -- provides architectural direction, catches semantic errors agents miss.
- **Tier 3 approval** -- must explicitly approve before implementation starts and before commit.
- **Merge authority** -- bots never merge.

The human is the divergent body. Agents converge (implement, refine, review). The human provides creative direction, business context, and final judgment.

## Approval Tiers

| Tier | Risk Level | Plan Gate | Report Gate |
|------|-----------|-----------|-------------|
| **1** | Trivial (docs, imports, test renames) | Reviewer approves → go | Reviewer approves → commit |
| **2** | Mechanical (refactors, no behavior change) | Reviewer approves → go | Reviewer approves → commit |
| **3** | Behavioral (auth, schema, billing) | Reviewer approves → **human approves** → go | Reviewer approves → **human approves** → commit |

**Default**: If unsure, default to Tier 3.

**Key nuance**: Even Tier 1 seams go through cross-agent review. The tier determines when the human is involved, not whether review happens.

## Branching Strategy

### Independent Seams

Branch from `origin/main`, implement, push, PR, merge. Each seam is a standalone branch.

### Stacked Seams

When seams have ordering dependencies:

1. Implement Seam A on a branch from `origin/main`.
2. Push a draft PR for Seam A.
3. Branch off Seam A's branch for Seam B.
4. Push a draft PR for Seam B.
5. When Seam A merges, rebase Seam B onto main.

Track the stack reference in the plan (e.g., `#444 -> #446 -> #447`).

The seam plan and report headers include a **Merge Strategy** field (`independent to main` or `stack on PR #NNN`) so the GitHub agent knows how to handle the PR.

## Seam Artifacts

Each seam produces up to two artifacts in `.seams/{issue}/` (gitignored, sequentially numbered, never overwritten):

- **Seam Plan** (`NNNN-{ids}-{slug}-plan.md`) -- reviewed before implementation.
- **Seam Report** (`NNNN-{ids}-{slug}-report.md`) -- reviewed before commit.

Both share the same sequence number. See the template's `prompts/shared/seam-plan-format.md` and `prompts/shared/seam-report-format.md` for the full section-by-section format.

**Reports are not written for PR fixups.** If CI fails or a reviewer requests changes on an existing PR, the agent fixes and pushes. No new report.

## Process Enforcement

These rules emerged from real sessions and exist because violating them caused problems:

- **No fabrication** -- never claim a command ran without running it. Fabricated claims are a blocking review finding.
- **No direct plan edits** -- plan changes go through seam plans and review. No agent modifies the plan document outside of a reviewed seam.
- **Discovery before implementation** -- read the code and assess before writing anything.
- **Scope discipline** -- flag discoveries, don't silently add work.
- **Staging discipline** -- seam artifacts are gitignored, never committed. Only code changes are staged.
- **Deferred cleanup accountability** -- every temporary code block gets a `// TODO @agent: CH-XXX` marker. The reviewer verifies the target item covers the cleanup.

## Configurable Defaults

The template includes config files (prefixed with `_`) that are designed to be edited per-project:

| File | What to customize |
|------|-------------------|
| `_tiers.md` | Tier definitions, what counts as Tier 3 for your project |
| `_verification.md` | Test commands, format/lint commands, fast-path vs slow-path |
| `_branching.md` | Branch naming, default to stacking or independent, squash-merge rules |
| `_review.md` | Review process, depth by tier, project-specific invariants |
| `_ci-review.md` | Your CI checks, review bot behaviors, what "all clear" means |

These files are loaded by agents at startup. Edit them to match your project's tooling and risk profile.

## Where Dispatch Fits

[Dispatch](https://github.com/codesoda/dispatch) is the local multi-agent orchestration layer that automates the handoffs between agents. It provides:

- **Message-driven coordination** -- agents send messages to each other by role, not identity.
- **Role-based discovery** -- "send this to a reviewer" rather than "send this to agent #3."
- **TTL-based liveness** -- workers heartbeat every 5 minutes or expire automatically.
- **No external dependencies** -- embedded Unix domain socket broker on a single machine.

### What Dispatch Automates

| Without dispatch (manual) | With dispatch |
|---------------------------|---------------|
| Human copies seam plan/report to reviewer session | Implementer sends artifact to `reviewer` role via `dispatch send` |
| Human relays reviewer findings back | Reviewer sends findings to `coordinator` role |
| Human monitors CI and relays failures | GitHub agent polls CI and sends status to `coordinator` |
| Human asks test runner to verify | Implementer sends verification request to `test-runner` role |
| Human shuttles context between sessions | Agents read each other's messages via `dispatch listen` |

### What Dispatch Does Not Replace

- **Tier 3 approval** -- the human must still explicitly approve.
- **Scope authority** -- the human decides what enters the checklist.
- **Domain judgment** -- architectural decisions, business context.
- **Merge authority** -- bots never merge.

### Dispatch Configuration

The template includes a `dispatch.config.toml` defining five agents:

```toml
[[agents]]
name = "implementer"
role = "implementer"
command = "claude --dangerously-skip-permissions --model opus"
prompt_file = "prompts/agents/implementer.md"

[[agents]]
name = "reviewer"
role = "reviewer"
command = "claude --dangerously-skip-permissions --model sonnet"
prompt_file = "prompts/agents/reviewer.md"

[[agents]]
name = "test-runner"
role = "test-runner"
command = "claude --dangerously-skip-permissions --model sonnet"
prompt_file = "prompts/agents/test-runner.md"

[[agents]]
name = "github-agent"
role = "github"
command = "claude --dangerously-skip-permissions --model sonnet"
prompt_file = "prompts/agents/github-agent.md"

[main_agent]
command = "claude"
model = "opus"
prompt_file = "prompts/agents/coordinator.md"
```

The coordinator is the main agent -- the one the user talks to. Everything else is a sub-agent that registers with dispatch, listens for work, and communicates via messages.

## Session Lifecycle

### Cold Start

Every agent session starts fresh. It catches up by reading:

1. Its role-specific prompt and the shared docs it references.
2. The config files (tiers, verification, branching, CI review).
3. The plan document (full checklist, locked decisions, current status).
4. All existing seam plans and reports in `.seams/{issue}/` in order.
5. `git log` and `git status`.

### Long-Lived Sessions

In practice, implementing sessions often run long (days, hundreds of turns). Context compaction happens automatically. The plan document and seam artifacts are the durable state that survives compaction -- the agent can always re-read them.

### Process Evolution

The config files are living documents. When the human corrects a process gap, the relevant config file is updated. Future sessions inherit the correction automatically since agents re-read configs at startup.

## Common Mistakes

- **Skipping the PRD for "simple" tasks.** The PRD loop is where you catch misunderstandings. Skipping it means the agent guesses, and you catch the problem three seams later.
- **Making seams too large.** If a seam touches more than 3-4 files or takes more than one conceptual step, split it. The review is only useful if the diff is small enough to actually read.
- **Rubber-stamping reviews.** If you're approving every seam without reading the plan or report, you're not doing SEAMS -- you're running an autonomous loop with extra steps. The review is the point.
- **Not reading seam artifacts in order.** When you come back after a break, read the plans and reports from the beginning. They're the narrative of what happened and why.
- **Over-engineering config upfront.** Start with the defaults. Add rules when you encounter a problem the defaults didn't cover. The config files are living documents.

## What to Expect

Your first SEAMS run will feel slow. That's normal. You're building the muscle memory for:

- Writing PRDs that are tight enough for an agent to implement from.
- Breaking work into seam-sized pieces.
- Reading seam plans and reports critically.
- Knowing when convergence has been reached.

By your third or fourth issue, the overhead drops significantly. The PRD and plan phases get faster because you develop an instinct for what level of detail the agent needs. The seam loop gets faster because you learn to scope seams well on the first try.
