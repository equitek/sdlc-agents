---
name: acceptance-gate
description: Verify the running system against the frozen requirements document before a PR is marked ready — the operator's acceptance test, run as a user rather than a reviewer. Use when a Complex feature has passed final comprehensive review and needs to prove it delivers what was approved.
auto_activate: false
triggers:
  - "acceptance"
  - "acceptance test"
  - "does it meet the requirements"
  - "verify against requirements"
  - "UAT"
  - "user acceptance"
  - "ready for review"
---

# Acceptance Gate

**Core principle**: Two reviews already verified construction. This one is the
first time anyone checks the promise.

## Why this gate exists

The pipeline verifies, in order, that the design is sound (Design Review Gate),
that the plan is executable (Plan Review Gate), and that each work unit meets its
Definition of Done (adversarial review, per unit). All three are **construction**
checks: they prove the thing was built as specified.

None of them ask whether the operator got what they approved. A work unit can
pass adversarial review — implementation present, test present, `file:line`
cited — while the feature is unreachable from the entry point, wired to the
wrong handler, or rendering `Error: undefined` where the requirements document
committed to `last failed: no such tab z`. Tests are written against the plan.
The requirements document is written against the user. Those are different
oracles, and only one of them has ever been checked mechanically.

`requirements-capture` already did the expensive half of this work: it forced
literal rendered examples, numbered "Done when" criteria, and explicit negative
constraints before design began. That is a UAT script. This gate walks it.

The requirements document currently has one downstream consumer: the Scope &
Alignment reviewer, which uses it as a **fence** — "is this finding in scope?"
The fence is the cheap use. The oracle is the valuable one.

---

## When this gate runs

| Task assessment | Acceptance gate? |
| --- | --- |
| Complex — requirements document exists | **Yes, mandatory** |
| Simple — no requirements document | No — nothing to verify against |

**The gate is gated by the artifact, not by confidence.** If a requirements
document exists with status `approved` or `auto-approved`, run the gate. If one
does not exist, the task was assessed Simple and there is no promise to check.

Do not run it on every work unit — it verifies the feature, not the unit. Do not
run it before the final comprehensive review — reviewing an unbuilt feature
wastes a pass.

**Position in the pipeline:**

```
requirements capture -> human approves -> freeze
  -> design -> Design Review Gate
  -> plan -> Plan Review Gate
  -> execution (4-phase loop per work unit)
  -> final comprehensive review
  -> ACCEPTANCE GATE            <-- here
  -> self-reflect -> PR ready
```

It runs **before** `self-reflect` and before the PR is marked ready, because a
FAIL sends work back to implementation and the learnings should come from the
resolved state — not from a defect that is about to be fixed anyway.

---

## Step 0: Confirm the artifact

```bash
ls docs/requirements/*-requirements.md
```

Read the document. Check the header:

- `**Status**: approved (<date>, <who>)` or `auto-approved (<reason>)` → proceed
- `**Status**: awaiting human approval` → **STOP.** The gate has no authority
  document. Load `requirements-capture` and get approval first.
- No document → task was Simple. The gate does not apply. Say so and stop.

For `auto-approved` documents, every default recorded in the flagged-issues block
becomes a check — and every one must also appear in the PR description. Verify
both.

---

## Step 1: Establish a reachable system

Acceptance requires a running system. Before reviewing anything, get one, and
record exactly what you are testing:

```markdown
**System under test**: <local build | staging URL | deployed env>
**Commit**: <sha>
**How reached**: <exact command or URL>
```

For a UI, confirm `visual-review` prerequisites (Playwright + chromium) before
committing to screenshot-based checks — if the browser is unavailable, that is a
known constraint to report, not a surprise at item 12.

If the system cannot be started, **stop and report INCOMPLETE immediately** with
the blocker. Do not proceed to infer behaviour from source. A gate that quietly
degrades into a code read is worse than no gate, because it reports a pass.

---

## Step 2: Build the checklist

Convert the requirements document into an enumerated checklist. Every item in
these four sections becomes one row, and nothing outside them is a check:

| Section | Becomes |
| --- | --- |
| `## What the user sees` | One item per literal rendered example |
| `## Behaviour` | One item per documented command / interaction |
| `## Constraints` | One item per "never does X" invariant |
| `## Done when` | One item per numbered criterion |

Sections that are **not** checks:

- `## Out of scope` — pre-existing gaps and non-goals. Verifying these would be
  scope creep in reverse. They are context, not promises.
- `## Notes for the design phase` — explicitly not requirements.
- `## Scope` / header metadata — descriptive.

**Count the items before starting.** If the document yields no machine-checkable
items, that is a REQUIREMENT DEFECT worth reporting: the document failed its own
filter and cannot be verified.

---

## Step 3: Verify — cheapest and most decisive first

Order matters. Exact-output checks take seconds; a full visual sweep takes
minutes. Front-load the cheap ones so a REJECTED verdict is reached quickly and
the expensive work is skipped when the feature is already going back.

### 3a. Literal output (cheapest, most decisive)

The document quoted exact text. Produce that text and diff it character by
character, whitespace included.

```bash
bin/chat send "/last z"
# compare observed output to the document's stated error string
```

Do not accept "semantically equivalent." The document committed to a string. If
the exact string was a bad choice, that is a REQUIREMENT DEFECT for the human —
not permission to pass a near-miss.

### 3b. Behaviour and argument grammar

Invoke each documented form **literally**, including the ones that resolve an
earlier ambiguity. If the approved document says `/last 3` means tab 3, open
three tabs and confirm tab 3 was selected — not that three messages came back.

The flagged-issues block in the requirements document recorded decisions the
human accepted by default. Those decisions are now behaviour commitments. Test
them.

### 3c. Constraint probes (the only adversarial part)

Negative invariants cannot be verified by normal use. You must **attempt the
violation**:

| Invariant form | How to probe it |
| --- | --- |
| "never sends X to Y" | Seed X into the input path; inspect what crossed Y |
| "never modifies Z" | Snapshot Z, exercise the feature, re-snapshot, diff |
| "never changes the focused tab" | Record focus, run the command, re-check focus |
| "no secrets in logs" | Feed a known secret; grep the log output for it |

An unattempted constraint is **UNVERIFIABLE, not PASS.** This is the discipline
that makes the gate worth running — a negative invariant asserted in a document
and never probed in code is exactly the failure mode acceptance exists to catch.

### 3d. "Done when" walk

Each numbered criterion, observed independently. One observation must not be
claimed for several criteria — that is how a five-item list gets "verified" by
one happy-path run.

### 3e. UI checks (most expensive — last)

If the document makes claims about what is rendered on screen, use
`visual-review` to capture the state and compare it to the document's
description. Do this last: it needs a browser, and a text-level FAIL already
sends the feature back.

---

## Step 4: Classify findings honestly

Not every problem found is an acceptance failure. Three distinct classes, and
conflating them is how this gate would become the scope-growth failure the
requirements phase exists to prevent:

| Finding | Meaning | Effect on verdict |
| --- | --- | --- |
| **FAIL** | System behaves differently from a stated requirement | REJECTED |
| **REQUIREMENT DEFECT** | The document is wrong, contradictory, or unbuildable | Not a FAIL — flag for human |
| **DOC GAP** | Document silent on something you observed | Never a FAIL — observation |
| **PRE-EXISTING** | Real defect, not this feature's, hit incidentally | One line, filed. No verdict effect |

A REQUIREMENT DEFECT is a win, not a nuisance: the pipeline caught a bad promise
before a user did. Report it prominently and separately from the verdict.

**Never let an observation become a requirement.** If you notice something that
would improve the feature but the document does not mention it, it goes in
Observations. Turning it into a FAIL is scope creep by reviewer — the precise
failure the frozen baseline was built to prevent.

---

## Step 5: Report the verdict

Follow the output format in `rubrics/acceptance-review-rubric.md`. Three
possible verdicts:

| Verdict | Condition | Next step |
| --- | --- | --- |
| **ACCEPTED** | Every item PASS | Proceed to `self-reflect` and PR |
| **REJECTED** | One or more item FAIL | Return defects to implementation |
| **INCOMPLETE** | No FAIL, but one or more UNVERIFIABLE | Human decision required |

There is no ACCEPTED with caveats.

### On REJECTED

Acceptance does **not** loop. It is not a review-iterate cycle like the two
gates. The reviewer reports defects and stops:

```markdown
## Acceptance Gate — REJECTED

### Defects (return to implementation)

1. **Error text does not name the failure**
   - Requirement: `last failed: no such tab z`
   - Found: `Error: undefined`
   - File scope hint: error construction path (implementer locates it)

### Re-acceptance

After the fix, re-run acceptance on the **failed items only**. Passing items
need not be re-verified — the system was observed to satisfy them at commit
`<sha>`, and the fix is scoped to the defect.
```

The fix is a new work unit through the normal 4-phase loop: the implementer does
not skip adversarial review because acceptance found the defect. Acceptance
found it by using the product; the fix still has to meet its contract.

### On INCOMPLETE

An INCOMPLETE verdict means a human must decide — supply access, confirm a test
counts as evidence, or accept a requirement defect. Present it as a decision,
never as a soft pass:

```markdown
## Acceptance Gate — INCOMPLETE

### Blocked items (human decision required)

1. **Secret boundary not observable** (requirement: "No secrets are ever sent
   to the chat platform")
   - Attempted: seeded `OPENAI_API_KEY` into input path; inspected argv and
     outbound log
   - Blocker: could not capture the platform payload locally
   - Options: (a) provide one captured outbound request, (b) accept the existing
     `cbInput` strip unit test as evidence, (c) defer to staging verification
   - Default: (b) — the unit test asserts the strip directly
```

---

## Reviewer isolation

The acceptance reviewer is a **fresh instance** with no prior context:

- No access to the design doc, the plan, the DoD items, or the diff
- No knowledge of what implementers believed they built
- **Does not read `git diff`** — the diff is construction evidence, and this gate
  verifies behaviour, not construction

That last rule is the load-bearing one. The moment an acceptance reviewer reads
the implementation, it stops being a user and becomes a third code reviewer — and
it will start passing items because "the code looks right." The whole value of
the gate is that it knows nothing except what the operator was promised and what
the system actually did.

Give the reviewer exactly two inputs: the requirements document, and a reachable
system.

---

## Reporting

```markdown
## Acceptance Gate — ACCEPTED (1 pass)

**Requirements**: `docs/requirements/2026-03-04-last-command-requirements.md`
**System under test**: local build, commit `a1b2c3d`
**Coverage**: 17 items — 17 PASS, 0 FAIL, 0 UNVERIFIABLE

All 17 requirement items observed to behave as documented. Constraints probed:
2 of 2 (secret boundary attempted with seeded key; focused-tab invariant checked
across 3 commands).

Proceeding to self-reflect, then PR.
```

Keep it short. If the report is longer than the requirements document, the review
was overdone.

---

## Proportionality

This gate costs one pass over one page or two of requirements. On a well-written
document most items are mechanical, and the whole thing is minutes of runtime.

It is cheap because `requirements-capture` paid the cost upfront — literal
examples, numbered criteria, explicit invariants. That is the trade the pipeline
makes: harder requirements writing, mechanical acceptance later. If acceptance is
expensive or ambiguous, the requirements document was underspecified, and the
finding to report is a REQUIREMENT DEFECT, not more reviewer effort.

**Do not run this gate on Simple tasks.** A bug fix has no requirements document
and needs none. Running a UAT pass on a one-line fix is the over-process this
pipeline spends most of its design effort avoiding — pointed the other way.

---

## Relationship to other skills

| Skill | Relationship |
| --- | --- |
| `requirements-capture` | Produces the authority document. No document, no gate. |
| `design-review-gate` | Uses the document as a **fence** for design findings. Upstream. |
| `plan-review-gate` | Uses the document as a **scope baseline**. Upstream. |
| `orchestrated-execution` | Runs after this gate on REJECTED — defects become work units. |
| `visual-review` | Used for UI claims (step 3e). Most expensive, run last. |
| `self-reflect` | Runs after ACCEPTED. Not after REJECTED. |
| `handoff` | Include acceptance status in any handoff mid-feature. |

---

## Anti-patterns

| # | Anti-pattern | Why it is wrong | Instead |
| --- | --- | --- | --- |
| 1 | Reading the diff | Turns a user into a third code reviewer; passes items because code "looks right" | Two inputs only: requirements + running system |
| 2 | "Tests pass" as acceptance evidence | Tests are written against the plan, not the user's document | Observe the behaviour |
| 3 | Semantic-equivalence passes | Document committed to exact text | Literal match, or REQUIREMENT DEFECT |
| 4 | Skipping constraint probes | Negative invariants cannot be verified by normal use | Actively attempt each violation |
| 5 | One run, five criteria | The classic way a checklist gets falsely signed off | One observation per item |
| 6 | UNVERIFIABLE as a soft pass | Hides unverified promises behind a euphemism | Evidence of the attempt + what would unblock |
| 7 | Observations turned into FAILs | Scope creep by reviewer — the failure the baseline prevents | Observations section, no verdict effect |
| 8 | Iterating the gate | Acceptance is not a review loop; it reports defects | Return defects, re-run once on failed items |
| 9 | Running on Simple tasks | A UAT pass for a bug fix is over-process | Gate on presence of a requirements document |
| 10 | Running before final comprehensive review | Verifies a feature that may still change | After final review, before self-reflect |
| 11 | Inferring behaviour from source when the system won't start | Silent degradation to a code read, reported as a pass | Report INCOMPLETE with the blocker |
| 12 | Bundling requirement defects with FAILs | Buries a bad promise as an implementation bug | Separate section, human decision |
