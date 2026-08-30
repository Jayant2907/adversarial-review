---
name: adversarial-review
description: >-
  Run after ANY agent/lane report, or before trusting/merging a change — a full
  source-verified adversarial review that COMBINES independent code-review
  channels (correctness, security, reuse) AND the cross-cutting feature-impact
  sweep (path divergence, denormalized-field drift, cross-screen staleness,
  duplicates/orphans, downstream/temporal impact, amount-to-amount across
  surfaces, UX traps), plus an architect cross-check for non-trivial fixes. It
  READS the actual changed source — it never trusts the report's claims or a
  green happy-path suite — and ends in a merge GO / NO-GO with file:line
  findings and severities. Invoke when the user hands over an agent report,
  says "review this / adversarial review / check impact / find workaround
  issues / make sure nothing else broke", or before merging any built or
  changed feature. Self-contained: needs no other skill to be wired in.
---

# Adversarial Review (with Impact Sweep)

A report saying DONE and a green test suite are **claims, not evidence**. This
skill turns "the agent says it works" into "I read the code and proved it works
everywhere the same data is computed, read, or shown — or I found where it
doesn't." It is the standing process to run **after every agent/lane report and
before every merge**, not a per-request favour.

**Why this exists (keep as the worked example):** On one change, an agent's 18
tests passed and the report read DONE, but reading the actual branch source
found `reactivate()` was not single-use — calling it twice in sequence created
a duplicate ACTIVE record with the same unique identifiers, dedup deliberately
bypassed. Neither the report nor the first reviewer caught it; source-reading
did. Separately, on a scheduling/period-shift build every own-suite test
passed, yet a downstream settlement path used a stale reference record for a
key divisor — a numeric bug found only by the impact sweep. **Happy-path green
hides the real holes. Both classes are what this review catches.**

## The one rule

Do **not** relay or trust the report. Fetch the branch, diff against base, and
read the changed source yourself. Everything below operates on the code, not the
write-up.

---

## Phase 0 — Ground truth

1. Fetch the branch / worktree; `git diff <base>...<head>` (or the working-tree
   diff for uncommitted work). List every changed file and every NEW file.
2. Read the actual changed source — not the report, not the comments, not the
   commit message. Note what the change **reads, writes, and displays**, and the
   codebase's #1 risk class (e.g. silent wrong-value / duplicate-record /
   under-counting — whatever this domain's costliest failure mode is).
3. Build an **impact map**: for each quantity/field/record the change touches,
   list *every other place* that same thing is read, written, or shown. This map
   drives the Impact channel below.

## Phase 1 — Fan out independent channels IN PARALLEL

Dispatch these as concurrent sub-agents (one message, multiple tool calls) and
reconcile. Each runs in its own context so none anchors on another's take.

| Channel | Agent | Hunts |
|---|---|---|
| **Correctness / quality** | `p-code-reviewer` (adversarial) | bugs, edge cases, wrong logic, test smells, dead code |
| **Security / data-safety** | `p-code-reviewer` (security lens) or `security-review` | injection, auth/RBAC gaps, secrets, unsafe deps, tenant isolation |
| **Reuse / simplification** | `p-code-review-reuse` (if available) or the reviewer | duplication, over-abstraction, dead code |
| **Impact sweep** | `general-purpose` running the 7 checks below | the cross-cutting classes a feature build misses |

Run the **Impact channel** whenever the change built or modified a feature: a
new/changed compute path, a denormalized/cached field, an effective-dated
resolver, or a multi-step / multi-screen flow. Return each channel's findings in
one schema: `{file, line, severity, failure_scenario, fix, channel}` where
`channel ∈ correctness | security | reuse | impact`.

## Phase 2 — Orchestrator goes DEEPER than the reviewers

On the #1 risk class, out-read the sub-agents. Explicitly probe:
- **Bypass paths** — does an import / bulk-upsert / admin endpoint reach the
  same write WITHOUT the guard the UI path has?
- **Idempotency** — can the action run twice and double the effect (a running
  total, a generated document, a ledger entry, a paid/settled flag)? Do reverse
  ops (reopen/cancel/delete) actually reverse side effects, or just flip a status?
- **Concurrency / TOCTOU** — is a guard enforced only in app code with no
  DB-level unique/constraint, so two near-simultaneous calls both pass?

## Phase 3 — Design the fix (non-trivial only)

For anything beyond a one-liner, have a `Plan` (architect) agent design the fix
and, if warranted, a senior-engineer pass sharpen it — *before* writing any fix
prompt. Fix at the **shared sink** (the lowest common function), guarded so
callers already doing the right thing are untouched.

## Phase 4 — Verdict

A finding is **Confirmed** when a second channel (or your own source-read)
independently reproduces it; otherwise mark it `(channel: X, unconfirmed)`.
Deliver:
- The **impact map**.
- A **ranked findings table**: `severity | file:line | failure scenario (concrete
  inputs → wrong output) | fix | channel(s)`, most-severe first.
- A **merge GO / NO-GO** per lane. Merge only the clean lanes; send the rest back
  with a precise fix prompt. Never merge on "tests pass" alone.

---

## The 7 Impact checks (the sweep, embedded)

Work these for the feature just built/changed. They are independent — fan them
out as parallel sub-agents for a large surface; auditing "every reader of field
X" is embarrassingly parallel.

### 1. Path divergence — one quantity, many resolvers
The #1 bug class. The same output (a divisor, rate, eligibility flag) is often
computed in several places: the live RUN path, a PREVIEW/SIMULATE path, a
FINAL-SETTLEMENT path, a RECOMPUTE path, an EXPORT/REPORT path.
- List every path that produces the quantity; `grep` the resolver + returned field.
- Confirm they share ONE resolver, or prove they agree. Common trap: the primary
  path threads the authoritative input (e.g. `getActiveRecord(asOfDate)`) but
  simulate/preview/settlement paths omit it and fall back to a default or stale
  field.
- Fix at the shared sink, not per-caller.

### 2. Denormalized / cached field drift
Any field duplicating an authoritative source WILL drift.
- Identify authoritative source (effective-dated table via resolver) vs the
  denormalized copy (a column on the parent entity).
- `grep` EVERY reader; classify SAFE (uses resolver / write / display-only) vs
  RISKY (feeds a computation, export, or decision off the stale copy). Rank RISKY
  by blast radius (money > metadata > display).
- Does the feature's write path update BOTH sources?

### 3. Cross-screen / cross-session state staleness
State mutated on screen A must show on screen B.
- Does every screen showing the state refetch on re-entry/focus, or cache/
  auto-restore stale state? Add refetch-on-focus + clear-on-stale.
- Watch for a status shown from one source on A and a different source on B.

### 4. Data integrity — duplicates & orphans
- Query for >1 live row per natural key (e.g. entity × cycle × period).
  Duplicates make ALL downstream verification unreliable — clean first.
- Do reverse operations (reopen/cancel/delete) reverse their side effects, not
  just flip a status?

### 5. Downstream / temporal impact
- Enumerate consumers of the output: any regulatory or compliance surface,
  accruals, final-settlement calculations, exports/registers, dashboards.
- For period-shifted / effective-dated features: calendar-boundary splits,
  fiscal-year boundaries, mid-period joiners/leavers, period-attribution in
  each consumer.

### 6. Amount-to-amount verification across surfaces
"It renders" is not verification. Pick ONE ground truth, diff everything to it.
- Ground truth = a real finalized record produced by the correct engine path.
- Confirm the SAME numbers in: preview, simulate/what-if, raw API/context
  endpoint, FE display, export/register. Any mismatch = check 1 or check 2,
  never "rounding".

### 7. UX / flow traps
- Disabled controls that strand the user; multi-step dead-ends; actions offered
  that will 409/400 downstream (offer the correct action for the state instead).

---

## Output contract
When invoked directly: deliver the impact map, ranked findings table, per-lane
GO/NO-GO, the fixes you were confident on (fixed at the shared sink, guarded,
re-tested + rebuilt/restarted before claiming verified), and residual
follow-ups you did not fix with why. When invoked as a review channel by another
orchestrator, return the standard findings JSON with the `channel` field set.
