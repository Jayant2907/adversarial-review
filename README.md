# adversarial-review

> **Agent Harness** — a merge-review harness for Cursor and Claude Code.

A **source-verified merge-review harness** that does not trust a "done" report or a green test suite. It reads the changed code, runs independent review channels in parallel, and ends in a **GO / NO-GO** for merge.

Compatible with **Claude Code** and **Cursor** (both use a `SKILL.md` skill folder).

## What this skill does

After any agent/lane report, or before you merge a change, the skill:

1. Diffs the branch (or working tree) against the base and reads the **actual** changed source.
2. Builds an **impact map**: every quantity, field, or record the change touches, and every other place it is read, written, or shown.
3. Fans out independent channels in parallel:
   - **Correctness** — bugs, edge cases, wrong logic, test smells
   - **Security** — injection, auth/RBAC gaps, secrets, unsafe deps, tenant isolation
   - **Reuse** — duplication, over-abstraction, dead code
   - **Impact sweep** — path divergence, denormalized-field drift, cross-screen staleness, duplicates/orphans, downstream/temporal impact, amount-to-amount mismatches, UX traps
4. Probes bypass paths, idempotency, and concurrency on the highest-risk class.
5. Delivers a ranked findings table (`severity | file:line | failure scenario | fix | channel`) and a per-lane **merge GO / NO-GO**.

This skill is self-contained. It does not require other skills to be installed, though `feature-impact-sweep` is the same 7-check sweep as a standalone skill.

## When to use it

Invoke it when you:

- hand over an agent report and want it challenged
- say "adversarial review", "review this", "check impact", or "make sure nothing else broke"
- are about to merge a built or changed feature

## Install

Clone this repo into your personal skills folder, or into a project's skills folder.

### Cursor

**User-level** (every project on this machine):

```bash
git clone https://github.com/Jayant2907/adversarial-review.git ~/.cursor/skills/adversarial-review
```

Windows (PowerShell):

```powershell
git clone https://github.com/Jayant2907/adversarial-review.git "$env:USERPROFILE\.cursor\skills\adversarial-review"
```

**Project-level** (shared with anyone who clones the project):

```bash
git clone https://github.com/Jayant2907/adversarial-review.git .cursor/skills/adversarial-review
```

### Claude Code

**User-level:**

```bash
git clone https://github.com/Jayant2907/adversarial-review.git ~/.claude/skills/adversarial-review
```

Windows (PowerShell):

```powershell
git clone https://github.com/Jayant2907/adversarial-review.git "$env:USERPROFILE\.claude\skills\adversarial-review"
```

**Project-level:**

```bash
git clone https://github.com/Jayant2907/adversarial-review.git .claude/skills/adversarial-review
```

Alternatively, copy the folder (it must contain `SKILL.md`) into the same locations.

Restart the agent session (or start a new chat) so the skill is picked up.

## How to run it

In Cursor or Claude Code, ask for an adversarial review, for example:

- `Run adversarial-review on this branch`
- `Adversarial review before merge`
- `Don't trust the agent report — review the actual diff`

The agent should load `SKILL.md` and follow the workflow there.

## What's in this repo

```
adversarial-review/
├── SKILL.md    # Skill metadata + full review procedure
└── README.md   # This file
```

## Agent Harness collection

Part of the **Agent Harness** suite by [@Jayant2907](https://github.com/Jayant2907) — installable skills that wrap structured agent workflows (review harnesses, QA harnesses, impact sweeps) for Cursor and Claude Code.

| Harness | Role |
|---|---|
| [adversarial-review](https://github.com/Jayant2907/adversarial-review) | Merge-review harness (this repo) |
| [p-code-review](https://github.com/Jayant2907/p-code-review) | Whole-codebase review harness |
| [feature-impact-sweep](https://github.com/Jayant2907/feature-impact-sweep) | Cross-cutting impact harness |
| [j-QA](https://github.com/Jayant2907/j-QA) | Manual UI QA harness |
| [j-user-readiness](https://github.com/Jayant2907/j-user-readiness) | Visual demo-readiness harness |
| [j-module-audit](https://github.com/Jayant2907/j-module-audit) | Field-level module audit harness |
