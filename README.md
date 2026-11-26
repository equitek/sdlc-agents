# SDLC Agents

A multi-agent software development process you can drop into any project.

This repo is the process itself, extracted from a working project so it can be
reused across many. It contains no product code. It defines **who does what,
in which order, and what has to be true before work is called done** — as 21
agent role definitions, 21 skills that the agents and humans invoke, and the
review rubrics that make the quality gates objective.

## Why a process repo

Agent-assisted development fails in predictable ways. A model handed a
one-line request will happily produce a 900-line diff, mark its own homework,
and report success on code that never compiled. The usual fix is a longer
prompt, which does not scale past a few instructions.

This repo takes the opposite approach: encode the process the way you would
for a human team — explicit roles, an ordered pipeline, and gates that block
rather than advise. The gates are the point. A ruleset that says "please run
tests" gets skipped under pressure; a gate that says "the plan cannot be shown
to the user until three independent reviewers pass it" does not.

The process exists to make the *right* path the *easy* path.

## The core problem it solves

The single most common failure is **scope growth during review**. A design doc
reviewed by agents grows without bound, because it is the only written
statement of intent — so every reviewer finding becomes a requirement by
default, and nothing can be ruled out of scope. A ~200-line change reaches 11
design revisions, ~50 findings, and a 1200-line spec, roughly nine in ten
findings being implementation detail or pre-existing defects.

The fix is a **requirements document written before design**, containing only
user-facing behaviour, approved by a human, then frozen. Reviewer findings are
then triaged against it rather than absorbed into it. See
`skills/requirements-capture/`.

## The pipeline

```
[complex tasks only] requirements capture -> HUMAN APPROVES -> freeze
  -> brainstorm? -> design -> Design Review Gate
  -> plan -> Plan Review Gate -> execution -> final comprehensive review
  -> ACCEPTANCE GATE -> self-reflect -> PR
```

Simple tasks — bug fixes, single-file changes, copy edits, config tweaks —
skip the front half entirely. A requirements document for a one-line fix is
the same over-process this pipeline exists to prevent, pointed the other way.

### Gates

| Gate | What it blocks | Skill |
|---|---|---|
| Requirements baseline | Design work starts before a human approved scope | `requirements-capture` |
| Design Review Gate | Planning starts before 5 agents approve the design | `design-review-gate` |
| Plan Review Gate | The plan is shown to the user before 3 reviewers PASS | `plan-review-gate` |
| Adversarial review | A work unit commits without independent verification | `orchestrated-execution` |
| Acceptance Gate | A PR is marked ready before the running system was checked against the requirements document | `acceptance-gate` |
| Coverage Gate | A PR opens below the project's coverage threshold | `setup` |

Every gate names who decides, what evidence is required, and what happens on
disagreement. None of them are advisory.

## Execution model

Work is decomposed into **work units** — the smallest piece that delivers
verifiable behaviour — and each unit runs a 4-phase loop:

```
IMPLEMENT -> VALIDATE -> ADVERSARIAL REVIEW -> COMMIT
```

The orchestrator never trusts a subagent's self-report. Validation runs the
real quality gates; adversarial review checks each Definition-of-Done item
against `file:line` evidence. A unit that fails review is sent back with the
failure history, up to three times, then escalated to a human with what was
tried.

### Verification, not just validation

The Design Review Gate, the Plan Review Gate, and adversarial review all check
**construction** — they read artifacts: a design doc, a plan, a diff. Each
answers "was this built as specified?"

The **Acceptance Gate** is the only gate that runs the product, and the only one
that answers "did the operator get what they approved?" A fresh `qa-agent` takes
exactly two inputs — the frozen requirements document and a reachable system —
and walks every requirement item, quoting the observed output as evidence.

It deliberately **never reads the diff**. An acceptance reviewer that reads the
implementation stops being a user and becomes a third code reviewer, and will
start passing items because the code "looks right." The gate's whole value is
that it knows only what was promised and what actually happened.

It is cheap because `requirements-capture` already forced literal rendered
examples, numbered acceptance criteria, and explicit negative constraints. That
is a UAT script; this gate walks it.

### When acceptance is impossible

Some behaviour simply cannot be observed by an agent: physical output, another
company's system, thirty-day uptime, whether prose reads well. Forcing a check
there produces a gate that gets quietly ignored on exactly the task types where
it cannot run — so those items can be **waived**, under rules that keep the
waiver from becoming a skip:

- **Granted by the human at requirements approval, never by an agent at gate
  time.** At gate time there is sunk cost and an agent that wants to be done.
- **Per item, never per gate.** One unobservable item does not waive the other
  sixteen; all-waived means the task was mis-assessed and needs no requirements
  document at all.
- **A substitute, not an absence.** Every waiver names the evidence that stands
  in — a benchmark, a human post-merge checklist, a production signal — and the
  reviewer verifies it.

The distinction that does the real work: **a blocker is not a waiver.** Missing
credentials, an unreachable staging URL, a service that will not start — those
make the gate *INCOMPLETE*, a question for the human, not a bypass. Only genuine
unobservability is waivable.

## Layout

```
agents/          21 role definitions — scope, inputs, outputs, quality bar
skills/          21 skills — the procedures agents and humans invoke
rubrics/         6 review rubrics — what "PASS" means, per gate
guides/          multi-agent coordination protocol
references/      per-platform tool mappings (Claude Code, Codex, Gemini, OpenCode)
AGENTS.md        the workflow contract — read this first
```

`AGENTS.md` is the entry point. It states the mandatory workflow rules and the
pipeline order; everything else in the repo elaborates on it.

## Using it in a project

1. Copy `agents/`, `skills/`, `rubrics/`, `guides/`, and `references/` into
   your project's `.agents/` (or wherever your platform discovers skills).
2. Copy `AGENTS.md` into your project root — or merge its workflow section
   into an existing `AGENTS.md`.
3. Run the `setup` skill once. It detects your stack, asks a few questions,
   and writes the project-local files the gates depend on —
   `.coverage-thresholds.json`, CI scaffolding, and platform instruction files.
4. Start work with `start-task`. It assesses complexity and routes you to the
   right level of process; you do not have to remember the pipeline.

The agents and skills assume no particular language or product. Where a
project has language-specific needs, a language skill overrides the
language-neutral baseline — `haskell-coder` and `haskell-reviewer` are
included as worked examples of that pattern.

## Conventions the process assumes

These are the choices that make the gates enforceable. They are process
requirements, not implementation preferences:

- **TDD.** Failing test first, then minimal implementation, then commit.
- **Draft PR opened early**, pushed as work proceeds, marked ready only when
  the full local gate passes.
- **No force-pushes, no `--no-verify`, no skipped gates** — by agents or by
  humans. A gate that can be bypassed under time pressure is not a gate.
- **Plans and context persist to disk**, so an interrupted session resumes
  from the plan rather than from memory.
- **Humans write the rules; agents write the code.** Issues, design docs, and
  human-facing prose are authored by people. This repo is the exception that
  proves it: it is written in the imperative because it is the ruleset.

## Status

Extracted from a private project and generalised. The process is in active
use; the repo is a snapshot, not a library with a stability guarantee. Names
and structure may change as the process does.
