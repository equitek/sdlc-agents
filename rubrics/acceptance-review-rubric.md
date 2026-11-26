# Acceptance Review Rubric

**Used By**: QA Agent (`acceptance-gate` skill)
**Purpose**: Verify the *running system* against the frozen requirements document
**Version**: 1.0

---

## Overview

This rubric is for **acceptance review** — checking whether the built system
does what the operator approved, from the operator's seat.

**It is not a third code review.** Two reviews already happened:

| Review | Question | Evidence surface |
| --- | --- | --- |
| Design Review Gate | Is this the right design? | The design doc |
| Adversarial Review | Does the code meet its DoD contract? | The diff, `file:line` |
| **Acceptance Review** (this) | **Does the system behave as the operator was promised?** | **The running system** |

The distinction is not stylistic. Adversarial review passes when every DoD item
is implemented and tested. Acceptance review passes only when you actually
observed the promised behaviour happen. A work unit can pass adversarial review
while the feature is unreachable, wired to the wrong handler, or rendering
`/last b: <empty>` where the requirements say `/last b: no messages`.

**Therefore: do not read the diff.** If you find yourself in `git diff`, you are
doing the wrong job. Construction is verified. Your job is to be a user.

---

## The authority document

The **requirements document** (`docs/requirements/<date>-<slug>-requirements.md`,
status `approved` or `auto-approved`) is the sole authority. Not the design, not
the plan, not the issue, not the DoD items, not your judgement of what the
feature ought to do.

If a requirement is wrong — bad design, contradicts another requirement,
unbuildable — that is a **REQUIREMENT DEFECT**, not an acceptance failure. See
"Findings that are not acceptance failures" below.

Every item in the document is a check. Nothing in the document may be skipped
and nothing outside it may be added.

---

## Verdicts

### Per item

Every requirement item gets exactly one verdict:

| Verdict | Meaning | Requires |
| --- | --- | --- |
| **PASS** | Observed to behave as the document states | Actual output, quoted |
| **FAIL** | Observed to behave differently | Expected vs. found, both quoted |
| **UNVERIFIABLE** | Could not reach or exercise the behaviour | Evidence of the attempt + what would be needed |

`UNVERIFIABLE` is not a soft FAIL and not a soft PASS. It means the check could
not be performed, and it is reported so a human can decide — not so the reviewer
can avoid work. It therefore requires **evidence of the attempt**: the command
you ran, the access you lacked, the service that would not start.

Rejected forms of UNVERIFIABLE — these are FAILs in disguise and must be treated
as such:

- "Tests cover this" — tests are construction verification, not observation
- "The implementation looks correct" — you were not supposed to look
- "Not tested due to time" — run it, or report the blocker
- "Requires credentials I don't have" without stating which, and without asking

### Overall

| Verdict | Condition |
| --- | --- |
| **ACCEPTED** | Every item PASS |
| **REJECTED** | One or more item FAIL |
| **INCOMPLETE** | No FAIL, but one or more UNVERIFIABLE |

There is no ACCEPTED with caveats. A single unobserved item is INCOMPLETE.

---

## Evidence requirements

Every verdict requires cited, quoted evidence — **the actual output, not a
description of it.**

### PASS

```markdown
**Item**: "`/last b` shows tab b's last message."
**Verdict**: PASS
**Evidence**:
- Ran: `bin/chat send "/last b"` against local instance (commit `a1b2c3d`)
- Observed:
  ```
  b: the deploy finished at 14:02
  ```
- Matches the document's rendered example character-for-character.
```

### FAIL

```markdown
**Item**: "Errors say `last failed: no such tab z`."
**Verdict**: FAIL
**Expected**: `last failed: no such tab z`
**Found**:
- Ran: `bin/chat send "/last z"`
- Observed:
  ```
  Error: undefined
  ```
**Gap**: error text does not name the failure; the document specifies the exact string.
```

### UNVERIFIABLE

```markdown
**Item**: "The dashboard shows a green banner when the queue is empty."
**Verdict**: UNVERIFIABLE
**Attempted**: `make dev` then `curl localhost:8080/health` → connection refused
**Blocker**: dashboard service requires `POSTGRES_URL`, not present in local env
**To verify**: human supplies a local DB URL, or point the reviewer at a running environment
```

### Invalid evidence

- "Tests pass" (construction, not acceptance)
- "Implementation at `src/Foo.hs:88` returns the right value" (you read the diff)
- "Looks correct" / "appears to work" (no observation)
- A paraphrase of expected output instead of pasted actual output
- Screenshot cited but not captured

---

## What to verify: the four requirement surfaces

The requirements document has four sections that make verifiable claims. Each
maps to a specific verification technique. Verify all four, in this order —
cheapest and most decisive first.

### 1. What the user sees (STRONGEST — mechanical)

The document contains **literal rendered examples**. These are exact-match
tests and they are the cheapest thing in the entire gate.

| Check | Classification | Criteria |
| --- | --- | --- |
| Literal output matches | BLOCKING if divergent | Character-for-character, whitespace included |
| Errors match literal text | BLOCKING if divergent | Documented error strings reproduced exactly |
| Absence also verified | BLOCKING if wrong | "shows X and not Y" — confirm Y is absent |

Do not accept "semantically equivalent." The document committed to a string; the
system produces a string. They match or they do not. If the exact text was a bad
choice, that is a REQUIREMENT DEFECT for the human, not a licence to pass a
near-miss.

### 2. Behaviour (command grammar)

Each documented command or interaction with its argument grammar:

| Check | Classification | Criteria |
| --- | --- | --- |
| Happy path per documented form | BLOCKING if broken | Each documented invocation performed |
| Argument grammar honoured | BLOCKING if divergent | `/last 3` means what the document says it means |
| Documented defaults hold | BLOCKING if divergent | Ambiguity the human resolved by default behaves as recorded |
| Adjacent forms not documented | WARNING | Note surprising behaviour, do not fail on it |

**Invoke the documented grammar literally.** If the document says `/last 3` is
"tab 3", run `/last 3` with three tabs open and confirm it selected tab 3 — not
that it returned three messages. Grammar ambiguity is exactly what the flagged
issues block resolved before approval; acceptance confirms the resolution landed.

### 3. Constraints (negative invariants)

The document's "never does X" statements. These are **negative assertions**, and
they cannot be verified by using the feature normally — you must *attempt the
violation*:

| Check | Classification | Criteria |
| --- | --- | --- |
| Violation attempted | BLOCKING if unattempted | Each "never" was actively provoked |
| Violation did not occur | BLOCKING if it did | Observed the invariant hold under attack |
| Enforcement point identified | WARNING | Note where the invariant is enforced |

This is the one place the acceptance reviewer behaves adversarially. "No secrets
are ever sent to the chat platform" is not verified by sending a normal message —
it is verified by seeding a secret into the input path and inspecting what
crossed the boundary. An unattempted constraint is UNVERIFIABLE, not PASS.

### 4. Done when (numbered acceptance criteria)

Each numbered criterion, independently observed:

| Check | Classification | Criteria |
| --- | --- | --- |
| Each criterion observed true | BLOCKING if unobserved | Direct observation, one at a time |
| Criteria verified independently | BLOCKING if batched | One observation must not be claimed for several criteria |
| Auto-approved defaults hold | BLOCKING if divergent | Every default listed in the PR description behaves as recorded |

The "Done when" list is the operator's definition of done. It is the closest
thing to a UAT script the process has. Walk it.

### Coverage arithmetic

Report the counts:

```
Items in document: N
  PASS: A    FAIL: B    UNVERIFIABLE: C
```

If C is large relative to N, the environment is not ready for acceptance and the
honest verdict is INCOMPLETE — not a partial pass. Say so plainly.

---

## Findings that are not acceptance failures

Three things can go wrong that are **not** the system failing to meet a
requirement. Classify them explicitly; do not silently turn them into FAILs and
do not silently bury them.

| Finding | Meaning | Action |
| --- | --- | --- |
| **REQUIREMENT DEFECT** | The document states something wrong, contradictory, or unbuildable | Report for human decision. Do not fail the item; do not pass it either — mark UNVERIFIABLE with reason, and flag the defect |
| **DOC GAP** | The document is silent on behaviour you observed | Report as observation. Never a FAIL — silence is not a promise |
| **PRE-EXISTING** | A real defect, not this feature's, encountered incidentally | File it (one line). Does not affect the verdict |

A REQUIREMENT DEFECT discovered here is a genuine win: it means the pipeline
found a bad promise before a user did. Report it prominently and keep it separate
from the gate verdict.

---

## Output format

```markdown
## Acceptance Review: <feature>

**Requirements**: `docs/requirements/<file>.md` (status: approved <date>)
**System under test**: <how it was reached — local build, staging URL, commit sha>
**Reviewer**: fresh instance, no prior context

### Verdict: ACCEPTED | REJECTED | INCOMPLETE

### Coverage

| Section | Items | PASS | FAIL | UNVERIFIABLE |
| --- | --- | --- | --- | --- |
| What the user sees | 6 | 6 | 0 | 0 |
| Behaviour | 4 | 3 | 1 | 0 |
| Constraints | 2 | 1 | 0 | 1 |
| Done when | 5 | 5 | 0 | 0 |
| **Total** | **17** | **15** | **1** | **1** |

### Per-item results

| # | Section | Requirement | Verdict | Evidence |
| --- | --- | --- | --- | --- |
| 1 | Sees | `/last` prints `tab: text` | PASS | `b: the deploy finished` |
| 2 | Sees | Error: `last failed: no such tab z` | FAIL | Found `Error: undefined` |
| 3 | Constraints | No secrets cross the boundary | UNVERIFIABLE | requires platform capture |

### FAILs (defects to fix)

1. **Error text does not name the failure.** Document specifies
   `last failed: no such tab z`; system emits `Error: undefined`.
   Expected vs. found above. Return to implementation.

### UNVERIFIABLE (human decision required)

1. **Secret boundary not observable.** Attempted: seeded `OPENAI_API_KEY` into
   input path, checked process argv and outbound log. Could not inspect the
   platform payload locally.
   - To verify: provide a capture of one outbound request, or confirm the
     existing `cbInput` strip test is accepted as evidence.

### Requirement defects found

1. **`/last 3` — tab 3 or three messages?** The document's Behaviour section says
   "tab 3"; the rendered examples show three messages. The approved doc is
   self-contradictory. Human must decide; both readings were verified and both
   work, so this is a documentation defect, not an implementation one.

### Observations (not failures)

1. `/last` accepts `/last b extra` and ignores `extra`. Document is silent.

### Recommended action

REJECTED — 1 defect. Fix the error string, re-run acceptance on that item only.
The UNVERIFIABLE item needs a human decision before this feature is accepted.
```

---

## Reviewer conduct rules

1. **Be a user, not a reviewer.** You have no diff, no plan, and no DoD list. If
   it is not in the requirements document, it is not your business.
2. **Run it or report the blocker.** Never infer behaviour from source, tests,
   or commit messages.
3. **Quote, don't summarise.** Actual output in a code block. Every time.
4. **Match literally.** The document committed to text. Near-misses are FAILs.
5. **Provoke the constraints.** Negative invariants require active attempts.
6. **One observation, one item.** Do not let a single happy-path run stand in
   for five criteria.
7. **UNVERIFIABLE is not an escape hatch.** It requires evidence of an attempt
   and a statement of what would unblock it. Report it, don't hide behind it.
8. **Do not fix anything.** Read-only. You report; implementation fixes.
9. **Do not iterate.** There is no review loop here. FAILs go back to
   implementation as defects, and acceptance is re-run afterwards. You do not
   negotiate with the implementer.

---

## Proportionality

Acceptance review is one pass over a page or two of requirements. If your review
is longer than the document, you have overdone it.

The cost control is ordering: **mechanical checks first.** Exact-output checks
are cheap (run a command, compare a string). Constraint probes are medium
(seed a value, inspect a boundary). UI screenshots are expensive (browser,
render, capture). Do the cheap decisive checks first, so a REJECTED verdict is
reached in minutes rather than after a full visual sweep. Stop early when you
already have a FAIL that sends the feature back.

---

## Comparison: adversarial vs. acceptance

| Dimension | Adversarial (per work unit) | Acceptance (per feature) |
| --- | --- | --- |
| **Subject** | The diff | The running system |
| **Contract** | DoD items from the plan | Requirements document |
| **Evidence** | `file:line` | Observed output, quoted |
| **Stance** | Auditor of code | User of the product |
| **Verdicts** | PASS / FAIL | PASS / FAIL / UNVERIFIABLE |
| **Frequency** | Every work unit | Once per Complex feature |
| **Loops** | Up to 3 retries, fresh reviewer | No loop — return defects, re-run once |
| **Catches** | Doesn't meet the contract | Meets the contract, breaks the promise |
