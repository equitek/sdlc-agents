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

## When this gate does not run — three distinct cases

These get conflated constantly, and only one of them is a waiver. Keeping them
apart is most of what makes the gate trustworthy.

| Case | What it means | Decided by | Recorded |
| --- | --- | --- | --- |
| **Not applicable** | No requirements document → Simple task, no promise exists | Complexity assessment (`start-task`) | Nowhere — nothing was promised |
| **Blocked** | Document exists, behaviour verifiable, environment not reachable | Reviewer, at gate time → **INCOMPLETE** | Escalation; human decides how to unblock |
| **Waived** | Behaviour is **inherently unobservable by an agent**, ever | **Human, at approval time** | Requirements document + PR description |

### Not applicable

No requirements document, no gate. The task was assessed Simple at `start-task`
and there is no promise to verify. Record it and move on. Do not manufacture a
UAT pass for a one-line fix.

### Blocked is not a waiver

A missing credential, a staging URL that will not resolve, a service the agent
cannot start — none of these are reasons to waive the gate. They are reasons the
gate cannot run *yet*. The verdict is **INCOMPLETE**, which puts a question to
the human; it does not reach a conclusion.

**The failure mode to avoid:** an agent near the end of a long task hits an
environment blocker, reclassifies it as "acceptance isn't possible here," and
waves the gate through. That is indistinguishable from skipping the gate.
Environment problems get solved (someone supplies a URL) or escalated. They are
never waived.

### Waived — decided in advance, by the human, per item

A waiver is warranted when the behaviour **cannot be observed by an agent at
all**, regardless of environment. Not "hard to observe." Not "observable only in
production" — a QA agent can be pointed at production. Truly unobservable:

| Category | Example | Why an agent cannot |
| --- | --- | --- |
| Physical world | Print output, LED state, hardware button, sensor reading | No physical presence |
| Third-party system | App Store review, bank settlement, a vendor's UI | Cannot drive someone else's system |
| Long-duration | "stays up 30 days", "no memory growth over a week" | Task completes in minutes |
| Human judgement | "the wording reads well", "the chart is legible" | Not a mechanical property |
| Legal / compliance | Signature, regulatory filing, licence click-through | Requires an accountable human |
| Destructive at scale | "survives loss of a production replica" | Cannot ethically or practically trigger |

#### The four rules of a waiver

All four must hold. Any one missing means it is not a waiver.

1. **Granted at approval time, never at gate time.** The human marks items
   waived when they approve the requirements document — before any
   implementation effort exists. At gate time there is sunk cost, a finished
   feature, and an agent that would like to be done. The decision must precede
   the incentive to make it.
2. **Per item, never per gate.** One unobservable item does not waive the other
   sixteen. If every item is waived, that is not a gate decision — it is a sign
   the task was mis-assessed and should never have had a requirements document.
   Report that as an assessment defect instead.
3. **A waiver is a substitute, not an absence.** Every waived item names the
   evidence that *does* stand in for it: an accepted unit test, a human
   post-merge checklist, a staging-only probe, a monitored production signal. A
   waived item with no named substitute is not a waiver — it is an unchecked
   promise wearing a waiver's clothes.
4. **The substitute is verified at gate time.** The reviewer confirms the named
   evidence exists and passes. A waiver redirects the work; it does not delete
   it.

#### Recording a waiver

In the requirements document, annotated on the item itself:

```markdown
## Done when

1. `/last` prints `<tab>: <text>` for the focused tab.
2. **WAIVED (2026-03-04, Mighty)** — "Stays responsive with 10k-message tabs."
   Unobservable by an agent (long-duration interactive performance).
   **Substitute**: `bench/last-seek.sh` in CI; human spot-check post-merge.
```

Document header records the count: `**Waivers**: 1 of 5 (Done when #2)`.

Waived items are **listed in the PR description** with reason and substitute —
same principle as auto-approved defaults. The human sees the decision after the
fact even if they did not examine it before.

### With no waivers

If a requirements document exists with status `approved` or `auto-approved`,
run the gate on every non-waived item. The gate is gated by the artifact, not by
confidence.

Do not run it on every work unit — it verifies the feature, not the unit. Do not
run it before the final comprehensive review — verifying an unsettled feature
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

Read the `**Waivers**` count from the header. For each waived item:

- The annotation **must** name a substitute. No substitute → not a valid waiver;
  report it as a REQUIREMENT DEFECT and mark the item UNVERIFIABLE.
- The substitute **must** exist and pass. Verify it like any other check.
- A waiver granted at gate time rather than approval time **is not a waiver**.
  Treat the item as in-scope and verify it, or report INCOMPLETE if you cannot.

If every item is waived, stop: the task should not have had a requirements
document. Report an assessment defect rather than a vacuous ACCEPTED.

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

**Waived items stay in the checklist** with verdict WAIVED — they are not
removed. Removing them hides the fact that a promise went unobserved by the
reviewer, and the whole point is that the decision stays visible in the report.
Each WAIVED row names its substitute in the Evidence column.

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

### 3f. Waiver substitutes

A waiver redirects work, it does not delete it. For every item annotated
`**WAIVED**`, verify the named substitute instead of the behaviour:

| Substitute named | How to verify |
| --- | --- |
| A test or benchmark | Run it; confirm it passes |
| A human post-merge checklist | Confirm the checklist exists and is referenced in the PR |
| A staging-only probe | Run the probe, if the environment is reachable |
| A monitored production signal | Confirm the alert/dashboard exists and covers the behaviour |

Verdict is **WAIVED** with the substitute outcome as evidence:

```markdown
| 5 | Done when | Stays responsive with 10k-message tabs | WAIVED | substitute:
`bench/last-seek.sh` run, 42ms mean over 10k msgs (budget 100ms) |
```

If the substitute **fails**, that is a FAIL — the waiver was granted on the
condition that the substitute holds. If the substitute **does not exist**, that
is a REQUIREMENT DEFECT (an invalid waiver), and the item is UNVERIFIABLE.

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

Follow the output format in `rubrics/acceptance-review-rubric.md`. Four
possible verdicts:

| Verdict | Condition | Next step |
| --- | --- | --- |
| **ACCEPTED** | Every item PASS | Proceed to `self-reflect` and PR |
| **ACCEPTED (with waivers)** | Every non-waived item PASS, every waived item's substitute verified | Proceed to `self-reflect` and PR |
| **REJECTED** | One or more item FAIL | Return defects to implementation |
| **INCOMPLETE** | No FAIL, but one or more UNVERIFIABLE | Human decision required |

There is no ACCEPTED with caveats. Waivers are not caveats: a waived item has a
verified substitute, so something was checked. An item with nothing checked is
UNVERIFIABLE, which is INCOMPLETE.

Report waived items explicitly even on success — `**Waived**: 1 of 17 (Done when
#2, substitute verified)`. The waived count travels into the PR description.

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
| 13 | Waiving at gate time instead of approval time | The decision lands with sunk cost and an agent that wants to be done | Waivers are granted when the human approves requirements |
| 14 | Waiving the whole gate | One unobservable item does not waive the other sixteen; all-waived means the task was mis-assessed | Waive per item, report an assessment defect instead |
| 15 | Waiving to an absence | A waived item with no named substitute is an unchecked promise | Every waiver names the evidence that stands in |
| 16 | Treating a blocker as a waiver | "No staging URL" and "no credentials" are solvable, not impossible | INCOMPLETE, escalate; never waive |
| 17 | Skipping the substitute check | A waiver redirects work, it does not delete it | Verify the named substitute like any other check |
