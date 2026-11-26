---
kind: agents
---

# Orchestration Workflow

This project uses a multi-agent orchestration framework. It provides
specialized agents, a 4-phase orchestrated execution loop, and quality
gates that enforce TDD, coverage thresholds, and spec-driven development.

## Starting Work

The default entry point is the **`start-task`** skill. Load it via
`SKILL_MANAGE` (`action: load`, `id: start-task`) before starting
any new task. It primes context, guides complexity assessment, and routes
to the right level of process.

For complex features (multi-file, spec-driven), the full pipeline is:

Research → Plan → Design Review Gate → Work Unit Decomposition →
Orchestrated Execution (4-phase loop per unit) → Final Review → PR

Load the **`orchestrated-execution`** skill (`orchestrated-execution`)
for the full 4-phase loop: IMPLEMENT → VALIDATE → ADVERSARIAL REVIEW → COMMIT.

## Available Skills

| Skill | Purpose |
|---|---|
| `start-task` | Begin tracked work — complexity assessment, routing |
| `prime` | Load relevant knowledge before starting |
| `requirements-capture` | Write the frozen scope baseline before design |
| `brainstorming-extension` | Bridge brainstorming into the design review gate |
| `design-review-gate` | Parallel 5-agent design review |
| `orchestrated-execution` | 4-phase execution loop per work unit |
| `plan-review-gate` | Adversarial plan review (3 reviewers must PASS) |
| `acceptance-gate` | Verify the running system against the requirements document |
| `create-issue` | Create a well-structured GitHub Issue |
| `external-tools` / `external-tools-health` | Delegate to Codex/Gemini CLIs; cross-model review |
| `handling-pr-comments` | Handle PR review comments |
| `pr-shepherd` | Monitor a PR through to merge |
| `ci-fix` | Diagnose CI failures without blowing up context |
| `visual-review` | Screenshot-driven UI review |
| `self-reflect` | Extract learnings after a PR merge |
| `handoff` | Write a resumable handoff document |
| `todo-md-maintenance` | Targeted TODO.md edits |
| `setup` | Interactive guided setup — detects project, configures orchestration |
| `haskell-coder` / `haskell-reviewer` | Language skills — Haskell implementation and review |

## Agents

21 specialized agents live in `agents/`. Each is a self-contained role
definition with its own scope, inputs, outputs, and quality bar. The
`issue-orchestrator` coordinates them for a single issue; the
`swarm-coordinator` runs several issues in parallel. Rubrics that agents cite
live in `rubrics/`; the multi-agent coordination protocol lives in
`guides/agent-coordination.md`.

## Quality Gates

- **Design Review Gate**: Parallel 5-agent review after design is drafted.
  Load `design-review-gate`.
- **Plan Review Gate**: Adversarial review after any implementation plan is
  drafted. Spawns 3 independent reviewers (Feasibility, Completeness, Scope
  & Alignment) — ALL must PASS before the plan is presented to the user.
  Load `plan-review-gate`.
- **Requirements Baseline**: Written before design, reviewed by the human
  (~2 min), then frozen. User-facing behaviour only. It is the baseline both
  review gates measure scope against. Load `requirements-capture`.
- **Acceptance Gate**: After the final comprehensive review, a fresh `qa-agent`
  verifies the *running system* against the frozen requirements document — item
  by item, with quoted output as evidence. It never reads the diff. Verdicts are
  ACCEPTED / REJECTED / INCOMPLETE. Load `acceptance-gate`.
- **Coverage Gate**: Reads `.coverage-thresholds.json` and runs the
  enforcement command — BLOCKING gate before PR creation.

## Workflow Enforcement (MANDATORY)

These rules override any conflicting instructions. They ensure the full
orchestration pipeline is followed regardless of which skill initiated the work.

**The pipeline, in order:**
```
[complex tasks only] requirements capture -> HUMAN APPROVES (the only doc
  they read) -> freeze -> brainstorm? -> design -> Design Review Gate
  -> plan -> Plan Review Gate -> execution -> final review
  -> ACCEPTANCE GATE -> self-reflect -> PR
```

- **Complex tasks only — before design**: WRITE the requirements document
  (`docs/requirements/<date>-<slug>-requirements.md`) —
  user-facing behaviour only, no implementation detail. Present flagged issues
  CONCISELY (one line each, every one with a default) and get human approval,
  or record an explicit auto-approval and list the defaults taken in the PR.
  Then FREEZE it. Load `requirements-capture`.

  Skip it for Simple tasks (bug fixes, single-file changes, copy edits, config
  tweaks). A requirements doc for a one-line fix is the same over-process this
  phase exists to prevent, pointed the other way. Use the `start-task`
  complexity assessment as the gate; if the assessment is genuinely ambiguous,
  ask the operator rather than defaulting to Complex.
- **Scope is frozen after approval**: A reviewer finding may NEVER add scope on
  its own authority. Triage every finding into exactly one bucket — IN SCOPE
  (fix the design), CONSTRAINT CORRECTION (fix the prompt), PRE-EXISTING (file
  an issue; one line in the design), or NEW REQUIREMENT (needs human sign-off;
  default is decline).
- **After brainstorming**: STOP — do NOT proceed to planning or
  implementation. RUN the Design Review Gate. WAIT for all 5 reviewers to
  approve. ONLY THEN proceed.
- **After any plan is created**: STOP — do NOT present the plan to the user
  or begin implementation. RUN the Plan Review Gate. WAIT for all 3
  reviewers to PASS. ONLY THEN present the plan for approval.
- **Execution method choice**: When a plan is ready, always ask the user
  which execution approach they want (orchestrated execution vs.
  subagent-driven vs. parallel session). Do NOT auto-select.
- **Before finishing a branch**: RUN `self-reflect` to capture
  learnings, commit knowledge base updates, THEN proceed to PR creation.
- **Subagent discipline**: NEVER use `--no-verify` on commits. NEVER
  force-push. ALWAYS follow TDD. NEVER self-certify — the orchestrator
  validates independently. STAY within declared file scope.
- **Context recovery**: Approved plans, project context, and execution
  state persist to disk. If context is lost mid-execution, reload from
  the persisted plan and execution-state files.
- **Before self-reflect**: RUN the `acceptance-gate` on any task that has a
  requirements document. A fresh `qa-agent` — not the implementer, not the
  orchestrator — walks every requirement item against a reachable running
  system and quotes the observed output. Do NOT hand it the diff, the plan, or
  the DoD items. REJECTED returns defects to implementation as normal work
  units; INCOMPLETE goes to the human with the blocker. The PR is not marked
  ready until the verdict is ACCEPTED.
- **Diagnose before implementing**: For complex tasks (multi-file, non-obvious
  behavior, performance issues, anything where the root cause is unclear), add
  debug/logging statements or diagnostic tests first to confirm your
  understanding of the actual problem before writing the implementation. Do not
  implement based on assumptions — instrument, observe, then fix.

