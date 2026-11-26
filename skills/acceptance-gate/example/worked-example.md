# Worked Example: Acceptance Gate

A real run of the `acceptance-gate` procedure against a fixture, showing the
report format and the three failure classes in action.

Kept in the repo because the procedure's central rule — **never read the diff** —
is easy to state and easy to violate. This example shows what the output looks
like when the reviewer has done it correctly: every verdict carries observed
output, and the defect is found by *using the product*, not by reading code.

---

## The fixture

`/last` command for a chat CLI: three tabs, `a` focused. One planted defect:
the unknown-tab error does not match the requirements document's literal string.
A second planted trap: `/last 3` could plausibly mean "three messages" — the
requirements document resolves it to "tab 3", so the reviewer must honour the
resolution rather than its own reading.

## Step 0: Confirm the artifact

```
docs/requirements/2026-03-04-last-command-requirements.md
  **Status**: approved (2026-03-04, Mighty)   → proceed
```

## Step 2: Build the checklist

| # | Section | Item |
| --- | --- | --- |
| 1 | What the user sees | `/last` → `a: the deploy finished at 14:02` |
| 2 | What the user sees | `/last b` → `b: the deploy finished at 14:02` |
| 3 | What the user sees | `/last z` → `last failed: no such tab z` |
| 4 | Behaviour | `/last` shows focused tab's last message |
| 5 | Behaviour | `/last <tab>` shows that tab's last message |
| 6 | Behaviour | `/last 3` means **tab 3**, not three messages |
| 7 | Constraints | `/last` never changes the focused tab |
| 8 | Done when | 4 numbered criteria |

`## Out of scope` and `## Notes for the design phase` are **not** checks.

## Step 3: Verify

```
$ ./bin/chat /last
a: the deploy finished at 14:02                    # items 1, 4, 8.1 PASS

$ ./bin/chat /last b
b: the deploy finished at 14:02                    # items 2, 5, 8.2 PASS

$ ./bin/chat /last 2
b: the deploy finished at 14:02                    # item 6 PASS — tab 2 == b,
                                                   #   not two messages

$ ./bin/chat /last z
Error: undefined                                   # item 3 FAIL — expected
                                                   #   "last failed: no such tab z"

$ ./bin/chat status   # before: focused a
$ ./bin/chat /last; /last b; /last 2; /last z
$ ./bin/chat status   # after:  focused a          # item 7 PASS — constraint
                                                   #   actively probed across 4
                                                   #   invocations incl. the
                                                   #   failing one
```

## Step 5: Report

```markdown
## Acceptance Review: /last command

**Requirements**: `docs/requirements/2026-03-04-last-command-requirements.md`
  (status: approved 2026-03-04, Mighty)
**System under test**: local fixture build, commit `fixture`
**Reviewer**: fresh instance, no prior context, diff not read

### Verdict: REJECTED

### Coverage

| Section | Items | PASS | FAIL | UNVERIFIABLE |
| --- | --- | --- | --- | --- |
| What the user sees | 3 | 2 | 1 | 0 |
| Behaviour | 3 | 3 | 0 | 0 |
| Constraints | 1 | 1 | 0 | 0 |
| Done when | 4 | 3 | 1 | 0 |
| **Total** | **11** | **9** | **2** | **0** |

### Per-item results

| # | Section | Requirement | Verdict | Evidence |
| --- | --- | --- | --- | --- |
| 1 | Sees | `/last` → `a: the deploy finished at 14:02` | PASS | observed identical |
| 2 | Sees | `/last b` → `b: the deploy finished at 14:02` | PASS | observed identical |
| 3 | Sees | `/last z` → `last failed: no such tab z` | FAIL | observed `Error: undefined` |
| 4 | Behaviour | `/last` shows focused tab | PASS | focused `a`, printed `a:` |
| 5 | Behaviour | `/last <tab>` shows that tab | PASS | focused `a`, `/last b` printed `b:` |
| 6 | Behaviour | `/last 3` means tab 3 | PASS | `/last 2` → `b:`, tab index not message count |
| 7 | Constraints | never changes focused tab | PASS | `focused a` before and after 4 invocations |
| 8.1 | Done when | `/<tab>: <text>` for focused tab | PASS | as item 1 |
| 8.2 | Done when | `/last b` works when `a` focused | PASS | as item 2 |
| 8.3 | Done when | unknown tab prints documented error | FAIL | as item 3 |
| 8.4 | Done when | focus unchanged after any `/last` | PASS | as item 7 |

### FAILs (defects to fix)

1. **Error text does not name the failure.** Requirements specify
   `last failed: no such tab z`; system emits `Error: undefined`.
   Expected vs. found above.
   - Note: items 3 and 8.3 are two requirement expressions of the same defect.
     One fix resolves both. Re-acceptance covers both, once.

### UNVERIFIABLE (human decision required)

None.

### Requirement defects found

None. The `/last 3` ambiguity was resolved by the approved document (tab 3, not
three messages) and the system honours that resolution — item 6 PASS. Had the
reviewer trusted its own reading of "3", it would have reported a false failure.

### Observations (not failures)

1. `/last b extra` ignores the extra token. Document silent — observation only,
   no verdict effect.

### Recommended action

**REJECTED** — 1 defect (2 requirement expressions). Fix the error string, then
re-run acceptance on items 3 and 8.3 only at the new commit. Items 1, 2, 4-7,
8.1, 8.2, 8.4 were observed satisfied at this commit and need no re-verification.
```

## What the example demonstrates

| Rule | How this run honoured it |
| --- | --- |
| Never read the diff | Every verdict came from running `bin/chat`, not reading it |
| Quote actual output | Every PASS/FAIL carries the observed string verbatim |
| Match literally | `Error: undefined` vs `last failed: no such tab z` — no near-miss |
| Probe constraints | Focus snapshot taken before *and after* four invocations, including the failing one |
| Honour flagged-issue resolutions | `/last 3` tested as tab 3, not three messages |
| One observation, one item | Items 3 and 8.3 marked as two expressions of one defect, not two independent findings |
| No scope creep | `/last b extra` behaviour stayed an Observation, never a FAIL |
| No iteration loop | Reported the defect and stopped; re-acceptance is scoped to failed items |

## The cost

11 items, one pass, ~2 minutes of runtime against a local build. The only
expensive part is having a reachable system — which is why step 1 exists and why
an unreachable system is reported as INCOMPLETE rather than silently degrading
into a code read.
