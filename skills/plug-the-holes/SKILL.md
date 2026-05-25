---
name: plug-the-holes
description: Disciplined security and ship-blocker audit for a vibe-coded or inherited app. Walks the attack surface money-and-identity-first, demonstrates every Hole with evidence (a re-runnable exploit test where feasible, or precise code-inspection where not), and STOPS. This skill does not edit production code or apply fixes — fix work happens between sessions via /tdd, matt's /to-prd → /to-issues chain, or AFK agents working from the tickets. Use when the user wants to find launch-blocking security issues — auth gaps, broken multi-tenant isolation, exposed secrets, unprotected endpoints, missing authorisation, runaway-cost endpoints — or asks "is this safe to ship". Run after setup-deslop and map-the-surface, before launch-gate. This is the core of the deslop pipeline.
---

# Plug The Holes

Find the defects that can hurt someone on launch day, document them with the strongest evidence the audit context can support, and **stop**. Everything else (the fix, the seam, the routing of handlers) happens in a *separate* session via `/tdd` or matt's skills, so each closed Hole is its own small reviewable PR rather than one catastrophic structural diff.

> **Demonstrate, don't assert.** A Hole is not real until you have evidence: either a **runnable exploit test** (preferred — a curl, a script, a failing test that makes the bad thing actually happen), OR a precise **code-inspection** pointer with reasoning when a runnable test is genuinely infeasible from the audit context. See `Evidence` in [../deslop/GLOSSARY.md](../deslop/GLOSSARY.md). Lazy "looks bad" assertions don't count.

This mirrors the feedback-loop discipline in `/diagnose`: the exploit test (or, where impossible, the documented code-inspection evidence) *is* the skill. Producing it is the audit's whole job. The fix is mechanical and belongs downstream.

Prerequisites: `STACK.md` (from `/setup-deslop`) and `SURFACE.md` (from `/map-the-surface`). Normally `/deslop` will have produced both before routing here; if you are invoking this skill directly and either is missing, stop and run `/deslop` first — auditing without them means guessing the stack and the surface. Read [../deslop/GLOSSARY.md](../deslop/GLOSSARY.md); severity buckets, finding states, and Evidence types are used verbatim.

## The order is the method

Audit in this sequence — **money-and-identity-first**. The categories that end companies come before the ones that merely embarrass them. Don't reorder for convenience.

1. **Secrets & credentials.** Exposed keys are the cheapest catastrophe to find and the worst to ship.
2. **Authentication.** Can you act as someone you're not?
3. **Tenant isolation.** Can tenant A touch tenant B's data? *The* SaaS Hole.
4. **Authorisation on every entry point.** Is each surface point locked to the right role, at the data layer?
5. **Public vs internal surface.** Is anything sensitive callable that shouldn't be?
6. **Input validation.** Can a malformed payload corrupt data or inject?
7. **Cost & abuse.** Can a stranger run up your bill (especially AI endpoints)?
8. **Data integrity & critical-path races.** Double-charge, double-submit, partial-write.
9. **Backups, env separation, observability.** Can you recover and see failures post-launch?

The exhaustive, stack-agnostic checks for each category live in [references/CHECKLIST.md](./references/CHECKLIST.md). **What each check means in your codebase** is in this project's `STACK.md` — read it first.

## The loop

### 1. Sweep
Go through every category, in order, and form a list of **Suspected** Holes by reading `SURFACE.md` against the matching `STACK.md` section. Don't build evidence yet. Each Suspected entry must be **falsifiable**: state the bad outcome concretely. *"`getInvoices` doesn't filter by the tenant key from STACK.md, so a logged-in user from org A can read org B's invoices by guessing IDs"* beats *"authorisation looks weak"*.

Suspected is **transient**. By the time you save `FINDINGS.md`, every Suspected entry has been promoted to Confirmed (with Evidence) or discharged out of the report. A delivered `FINDINGS.md` never contains a Suspected entry.

### 2. Exceptions checkpoint, once, against the whole list
Present the Suspected list to the user as a numbered list and ask, for each: *is there a compensating control or intended design I should know about before I attack it?* (e.g. "that endpoint's unauthed but sits behind edge auth", "cross-tenant reads there are deliberate, it's the support console").

This is **not** an "ignore these" switch. Classify each response:

- **Verifiable compensating control** → don't drop the finding; **redirect the exploit test at the control.** "Behind edge auth" → the test becomes *a request without the edge headers must be rejected*. If the control holds, the finding becomes **Closed-by-control** with the test as evidence (these don't block the gate). If it doesn't hold, treat it as a normal Suspected Hole and proceed.
- **Intended design with no enforceable control** → record as **Accepted Exception** with the owner's justification and name. Carried *visibly* to `/launch-gate` for explicit sign-off, never silently closed.
- **No / unsure** → audit as normal.

### 3. Build evidence
For each remaining Suspected Hole (and each claimed control from step 2), produce **Evidence** — exactly one of:

- **runnable-test** (preferred): the cheapest re-runnable thing that makes the bad outcome happen, or proves the control holds. A curl against a running dev server with a low-privilege token, a failing test that calls the entry point as the wrong tenant, a script that loops a paid endpoint unauthenticated. Put each test under `.deslop/exploits/<id>.<ext>` so the hand-off agent can find them. Tag any temporary instrumentation `[DESLOP-xxxx]` so cleanup is one grep.
- **code-inspection**: a precise pointer (`file:line`) plus a short reasoning paragraph showing the Hole is structurally present. Used **only** when a runnable test is genuinely infeasible from this skill's context — most often: an action runtime the audit harness can't load, a prod-env value not available, infra not stood up. The Hole is no less real; the audit just can't run a test for it from here. **Requires a `Why no runnable test` field** in the finding (see [FINDING-FORMAT.md](./references/FINDING-FORMAT.md)). The hand-off agent or fix-implementer then either builds the missing test rig OR fix-and-manually-verifies (curl from prod, etc.), but never silently closes.

Prefer `runnable-test`. Reach for `code-inspection` only when the alternative is shortcutting / shoehorning a test that risks a false negative. An honest "test infeasible here, evidence is the code" beats a contrived test that passes for the wrong reason.

### 4. Confirm or discharge
- **runnable-test goes RED** (bad thing happens) → **Confirmed Hole**. Record in FINDINGS.md with `Evidence: runnable-test: <path>`.
- **runnable-test goes GREEN** (couldn't make it happen, or the control held) → discharge it from the report. Either the code is safe, or your test is too weak; note which in a one-line comment near the test file if it stays around as regression. A discharged worry is a *good* outcome: noise removed.
- **code-inspection evidence is solid** → **Confirmed Hole**. Record in FINDINGS.md with `Evidence: code-inspection: <file:line>` + the `Why no runnable test` justification.
- **code-inspection evidence is hand-wavy** → don't deliver as Confirmed. Either build the runnable test, or discharge.

Log every Confirmed Hole using the fixed-fields block in [references/FINDING-FORMAT.md](./references/FINDING-FORMAT.md).

### 5. Note shared seams in the findings, do not write them
If multiple Holes close via one structural edit (the headline case: scope-once-at-a-seam — three handlers each forgetting to filter by tenant close together via one shared `requireTenant` wrapper), record that observation in the `Recommended seam` field of each affected Hole. **Do not write the seam yourself.** The hand-off (matt's `/to-issues` or `/tdd`) will see the blocked-by edges and naturally produce one "write the seam" ticket + N "route handler X through seam" tickets, each landing as its own reviewable PR. This preserves the architecture-improving insight without producing an unreviewable structural diff inside the audit session.

### 6. Triage the rest
Anything that isn't a Hole goes to its bucket and does not stop the audit:
- **Hardening** → parked, post-gate list.
- **Improvement** → parked, hand to `/improve-codebase-architecture`.
- **Latent bug / race that isn't a launch-day Hole** → parked, hand to `/diagnose`.

### 7. Stop. Hand off.
**Do not edit production code from this skill.** Fix work happens between sessions. End the run by detecting matt's skills in the environment and recommending one of two chains:

- **If matt's `/to-prd` and `/to-issues` are installed** (check `~/.claude/skills/to-prd/` and `~/.claude/skills/to-issues/` or equivalent agent-skills lookup), output:

  ```
  Next:
    1. /to-prd                   publish FINDINGS.md as a launch-readiness PRD
    2. /to-issues <prd-id>       slice the PRD into AFK-ready tickets (one per Hole or seam-group)
    3. /tdd <exploit-test-path>  for any Hole an AFK agent doesn't pick up
  Re-run /deslop when fixes are done → /launch-gate flips to SHIP if all exploit tests now GREEN.
  ```

- **Else**, output:

  ```
  Next:
    For each Confirmed Hole: /tdd <its .deslop/exploits/*.test.ts path>
    (or build the missing test rig / manually verify for code-inspection Holes)
  Re-run /deslop when fixes are done → /launch-gate flips to SHIP if all exploit tests now GREEN.

  Tip: install matt's skills for the PRD → tickets chain:
    npx skills add mattpocock/skills
  ```

## Output

`FINDINGS.md` per the format in [references/FINDING-FORMAT.md](./references/FINDING-FORMAT.md): a `Run log` block at the top, a `Status (current)` glance table, a `Hand-off` block, then every Confirmed Hole as a fixed-fields block. Plus `.deslop/exploits/*.test.ts` (runnable evidence) and any throwaway scripts in a clearly-marked `.deslop/exploits/` dir — they're the fix-implementer's red signal and your regression suite for the next audit.

## On re-run (idempotency)

`/plug-the-holes` is safely re-runnable. The re-sweep cross-references existing FINDINGS.md entries (state-transition table in `FINDING-FORMAT.md`), prepends a one-line `Run log` entry with the delta, and bumps per-Hole `Last changed` timestamps where evidence has changed. Never deletes a finding; history is the entire point. When all Confirmed Holes have flipped to Closed, the next `/launch-gate` call will SHIP.
