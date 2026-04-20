# Running a SEAMS Session

This document describes the concrete user experience of running a SEAMS session with dispatch.

## One-Time Setup (per project)

```bash
cd ~/projects/my-project

# Symlink the universal prompts into .seams/
mkdir -p .seams
ln -s ~/projects/seams/template/prompts .seams/prompts

# Copy the config files (these you'll edit)
cp -r ~/projects/seams/template/config .seams/config
cp ~/projects/seams/template/dispatch.config.toml .seams/dispatch.config.toml

# Fill in project-specific values
cat > .seams/.env.seams << 'EOF'
SEAMS_ISSUE=GH-42
SEAMS_PLAN=docs/plans/my-feature-plan.md
SEAMS_BRANCH_PREFIX=feat/gh42-
EOF

# Make sure .seams is gitignored
echo ".seams/" >> .gitignore

# Edit the config files for your project
# - .seams/config/_verification.md  → your test/lint/format commands
# - .seams/config/_ci-review.md     → your CI tools and review bot signals
# - .seams/config/_review.md        → project-specific invariants
# - .seams/config/_tiers.md         → adjust tier definitions if needed
# - .seams/config/_branching.md     → adjust branching strategy if needed
```

## Starting a Session

```bash
cd ~/projects/my-project

# Start the dispatch broker and all sub-agents (implementer, reviewer, test-runner, github-agent)
dispatch serve

# Start the coordinator — this is YOUR terminal, you talk here
claude --model opus --prompt-file .seams/prompts/agents/coordinator.md
```

`dispatch serve` reads `dispatch.config.toml`, starts the broker, and launches all configured sub-agents. The coordinator is the only agent you start manually because it's your interactive session.

The coordinator starts up, reads its prompt, registers with dispatch, and presents the current status:

```
SEAMS session for GH-42.
Current status: CH-003 committed, CH-004 next.
Plan: docs/plans/my-feature-plan.md

What would you like to work on?
```

## The Working Session

Once everything is running, you only interact with the coordinator terminal.

### Tier 1/2 Flow (informational)

```
You:  Let's start CH-004.

Coordinator:  Sending to implementer for discovery...
              [waits for dispatch message]

              Implementer completed discovery for CH-004.
              Seam plan written: .seams/GH-42/0004-CH-004-cleanup-imports-plan.md
              Sending to reviewer...
              [implementer ↔ reviewer iterate until plan approved]

              Reviewer approved the plan (Tier 1 — docs/config/test-only).
              Summary: Clean up unused imports across 3 files.
              Proceeding with implementation.

              [implementer works, sends verification to test-runner...]
              [implementer continues working while tests run...]
              [test-runner reports results back to implementer...]

              Seam report written: .seams/GH-42/0004-CH-004-cleanup-imports-report.md
              [implementer ↔ reviewer iterate until report approved]
              Reviewer approved the report.

              Staged files:
              - src/auth/session.rs
              - src/routes/onboarding.rs
              - src/routes/billing.rs

              Committing.
              Draft PR #468 created (independent to main).
              Monitoring CI...

              [github-agent monitors checks and automated reviews
               per _ci-review.md guidance...]

              CI passed. Automated reviews clear.
              PR #468 is ready. Next seam: CH-005.
```

### Tier 3 Flow (blocks for human approval)

The coordinator pauses twice — once before implementation starts, once before commit:

```
You:  Let's start CH-009.

Coordinator:  Sending to implementer for discovery...
              [waits for dispatch message]

              Seam plan for CH-009 approved by reviewer.
              Tier 3 — behavior/schema change.

              Summary: Add user_identity_id column to user_orgs table,
              backfill existing rows from user_identities.
              Merge strategy: stack on PR #453.

              Risk:
              - Migration on 50K-row table with FK constraint.
              - Backfill could fail if orphaned user rows exist.

              Full plan: .seams/GH-42/0009-CH-009-identity-column-plan.md

              Approve to start implementation?

You:  yes

              [implementer works, sends verification to test-runner...]
              [implementer continues while tests run...]
              [test-runner reports results...]
              [implementer ↔ reviewer iterate on report until approved...]

              Seam report for CH-009 approved by reviewer.
              Tier 3 — staging for your approval.

              What changed:
              - migrations/20260415_add_user_identity_id.up.sql — adds column + FK
              - crates/db/src/repos/org.rs — dual-write to new column
              - crates/api/tests/org_identity_tests.rs — 3 new integration tests

              Staged files:
              - migrations/20260415_add_user_identity_id.up.sql
              - migrations/20260415_add_user_identity_id.down.sql
              - crates/db/src/repos/org.rs
              - crates/api/tests/org_identity_tests.rs
              - docs/plans/my-feature-plan.md

              Diff: +87 -12 across 5 files
              Full report: .seams/GH-42/0009-CH-009-identity-column-report.md

              Approve to commit?

You:  approved

              Committed: "Add user_identity_id to user_orgs with backfill"
              Pushing to feat/gh42-ch009-identity-column...
              Draft PR #467 created, stacked on #453.
              Monitoring CI...

              [github-agent monitors checks and automated reviews
               per _ci-review.md guidance...]

              CI checks passed. Copilot reviewed — no issues.
              Sentry — no new issues flagged.
              PR #467 is ready for human review.

              Next seam: CH-010 (migrate reads to use user_identity_id).
```

### Plan-Only Seams

When discovery reveals a checklist item is already satisfied:

```
You:  What's next?

Coordinator:  Next is CH-011. Sending to implementer for discovery...
              [waits for dispatch message]

              Seam plan for CH-011 — plan correction only.
              Tier 1 — no code changes.

              Discovery found that CH-011 is already satisfied by earlier seams.
              The onboarding runtime already uses canonical user_id lookups.
              Proposed plan change: mark CH-011 as complete, next seam is CH-011b.

              Reviewer approved.
              Updating plan document.
              Done — CH-011 marked complete. Moving to CH-011b.
```

### Review Iteration

When the reviewer requests changes:

```
Coordinator:  Seam report for CH-005 reviewed.
              Reviewer verdict: Request changes.

              Issues found:
              1. Claims vs reality: report says auth_handler.rs is unchanged,
                 but the diff shows a modified error path at line 142.
              2. Deferred cleanup: missing // TODO @agent: CH-008 marker
                 at src/auth/session.rs:67.

              Sending findings back to implementer...
              [implementer fixes, re-verifies, re-presents...]

              Updated report reviewed. Reviewer approved.
              Staged files: [list]
              Approve to commit?
```

## Summary

Two commands to start a SEAMS session:

```bash
dispatch serve    # Starts broker + all sub-agents
claude --model opus --prompt-file .seams/prompts/agents/coordinator.md  # Your interactive session
```

From there, you talk to the coordinator. It drives everything else.
