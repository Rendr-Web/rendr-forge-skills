---
name: launch-gate
description: Define and enforce the explicit, written "safe to ship" line for an app, then make the ship / no-ship call. Verifies every Confirmed Hole is Closed, confirms launch-readiness essentials (backups, rollback, error tracking), and deliberately refuses to let hardening or polish block the launch. Use at the end of the plug-the-holes phase, when the user asks "are we ready to launch / is it safe to ship / what's left before go-live", or whenever a security/deslop pass needs a clear go/no-go decision. Consumes FINDINGS.md and SURFACE.md.
---

# Launch Gate

Draw the line, then hold it. The gate exists to answer one question honestly: *is this safe to put in front of customers?* It stops two opposite failure modes: shipping with a Hole open, and never shipping because there's always one more thing to polish.

> **The gate is a line, not a wish.** It is written down, per app, before the final check. Everything above it must be true to ship. Nothing below it may move it.

Read [../deslop/GLOSSARY.md](../deslop/GLOSSARY.md). Consumes `FINDINGS.md` (from `/plug-the-holes`) and `SURFACE.md` (from `/map-the-surface`).

## Process

### 1. Write the gate, *before* the final check
Create `GATE.md` for the app. The criteria are deliberately a closed list. If it's not on here, it does not block launch. **Scannable format** — checklist with inline current state in the right margin, numbered open-blocker commands underneath. Default shape:

```markdown
# GATE — <app name>   ·   target launch: <date>
Re-verified: YYYY-MM-DD by running, not by reading.

## Decision: **SHIP** (or **NO-SHIP**)
Reason: <one line — the single blocking item, or "all line items true">.

## Must be TRUE to ship
- [ ] Every Confirmed Hole in FINDINGS.md is Closed or Closed-by-control                    (currently: __ / __)
- [ ] Every Accepted Exception is signed off below                                          (currently: __)
- [ ] SURFACE.md Unknowns section is empty                                                   (currently: __ unknown)
- [ ] No secrets in repo / history / dist bundle                                            (currently: __)
- [ ] Tenant isolation proven on every tenant-data path                                      (covered by exploit suite)
- [ ] Paid/AI endpoints authed AND rate-limited/capped                                       (covered by H-__)
- [ ] Database backups exist + restore tested                                                (currently: __)
- [ ] Bad deploy can be rolled back                                                          (currently: __)
- [ ] Error tracking live; structured logs on money/identity paths                          (currently: __)

## Open blockers (commands to run)
1. **H-__** — `<exact command to verify or close, e.g. bash .deslop/exploits/h__.sh>`
2. **H-__** — `<…>`
3. **HD-__** — `<launch-task command, e.g. "one rehearsed restore + evidence to ops/restore-YYYY-MM-DD.md">`

## Accepted exceptions
<one line per Accepted Exception: finding ID · what it is · why it's accepted · compensating control if any · OWNER who signed off>
<empty is the healthy default; every line here is a risk you are choosing to ship with>

## Explicitly NOT blocking
- Hardening list (HD-*): <count> → post-gate
- Improvement list (IM-*): <count> → /improve-codebase-architecture
- Parked bugs/races: <count> → /diagnose
```

Tune the list per app, but tune it *now*, deliberately, not item-by-item under pressure later. Adding a criterion mid-launch "just to be safe" is gold-plating wearing a hard hat.

On re-run after fix work: **update the existing `GATE.md` in place** (re-write the `Decision`, the `Re-verified` date, and the inline `(currently: …)` cells). Don't create a new file per attempt — the diff history in git is the audit trail.

### 2. Verify by re-running, not re-reading
For each "must be true" item, **confirm by running, not by trusting the log.** For Holes: re-run the exploit tests in `.deslop/exploits/` and watch them go green, including the *control* tests behind any Closed-by-control finding. A findings log that *says* Closed is evidence. A green exploit test is proof. They can disagree (a later edit reopened something); trust the test.

**Accepted Exceptions are not a quiet bypass.** Each one needs a named owner and a written justification in the gate doc. An exception with no owner, or one you (the auditor) would not put your own name to, is treated as an open Hole: NO-SHIP. The point of the bucket is to make carried risk *visible and owned*, never to make it disappear.

If anything is red or unknown, the gate is **NO-SHIP**. Name the exact failing item. Don't soften it.

### 3. Make the call
- **All true → SHIP.** Say so plainly. Parked Hardening/Improvement work does not reduce confidence in the ship decision; that's the whole point of the buckets. Hand the parked lists to Phase 2.
- **Any false → NO-SHIP.** State the single blocking line (or the few). Route each back: open Hole → fix it via `/tdd <exploit-test-path>` (or matt's `/to-prd` → `/to-issues` chain feeding AFK), then re-run `/deslop`; missing backup/rollback/tracking → it's a launch task, not a deferral, do it now. Give the shortest path to green, not a lecture.

### 4. Resist the drift
Two pressures will try to move the line, both wrong:
- *"While we're here, let's also fix…"* No. That's Phase 2. Log it, ship, come back.
- *"It's basically fine, we'll sort the auth thing after launch."* No. An open Hole is the one thing the line will not bend for. "After launch" for a tenant-isolation Hole means "after the breach".

The gate's authority comes from being boring and fixed.

## After the gate
Shipping on an ugly-but-safe app is success. Phase 2 (`/improve-codebase-architecture`, `/diagnose`, `/tdd`) now runs *incrementally, in production, without a gate*, because none of it can hurt a customer on day one.
