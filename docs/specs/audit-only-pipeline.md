# Spec — Audit-only pipeline + scannable house format

> Draft: 2026-05-25. Captures decisions from the PipeInspect post-mortem grilling session. Pending review before any SKILL.md edits.

## 1. Problem

The first real `/deslop` run (PipeInspect, 2026-05-25) was successful but exposed two structural problems:

1. **Fixes happened inside the audit session.** `/plug-the-holes` swept, built exploit tests, *and* shipped an 18-file structural edit closing 11 Holes — producing a single catastrophically large PR that's a blocker to review. The README already says Phase 2 (fixing) is delegated to matt's skills; the audit shouldn't be doing it.
2. **Generated files were lobotomous to read.** `FINDINGS.md` in particular mixed prose paragraphs, inline diffs, and inconsistent per-entry structure. Hard to scan, hard to convert into PRD/tickets.

This spec resolves both.

## 2. Decisions (grilled, locked)

| # | Decision | Rationale |
|---|---|---|
| D1 | `/plug-the-holes` stops after RED exploit tests. No production-code edits. | Preserves "demonstrate, don't assert"; fix work becomes a separate reviewable step. |
| D2 | Hand-off chain is recommended, not auto-run. Skill detects matt's skills; if present, recommends `/to-prd` → `/to-issues`. Else recommends `/tdd` per Hole + install hint. | Deslop stays standalone; leans on matt's skills when available. |
| D3 | `/deslop` orchestrator is idempotent. Full chain runs every time. First run = NO-SHIP + report; later runs = SHIP if all exploit tests now GREEN. | No new commands or flags; same UX. |
| D4 | State machine stays binary (Suspected / Confirmed / Closed) plus Closed-by-control + Accepted-exception. **New `Evidence` field** captures proof type: `runnable-test: <path>` OR `code-inspection: <file:line>` + `Why no runnable test:` justification. Both count as Confirmed; both block the gate. | Honest evidence trail; no false-negative downgrade; no new state to learn. |
| D5 | `FINDINGS.md` (and STACK / SURFACE / GATE) adopt scannable fixed-fields format. Each Hole = identical 6-field block mirroring `/to-issues` template. Top-of-file run log + per-Hole "First seen" / "Last changed". | Scannable AND mechanically convertible. |

## 3. New pipeline shape

```
/deslop  (idempotent, full chain every run)
  ├── /setup-deslop          skip if STACK.md exists
  ├── /map-the-surface       always re-run (drift detection)
  ├── /plug-the-holes        sweep → RED exploit tests OR code-inspection evidence → STOP
  └── /launch-gate           re-run all exploit tests → SHIP if all GREEN, NO-SHIP if any RED

After NO-SHIP, /plug-the-holes output recommends a hand-off path:

  if ~/.claude/skills/to-prd/ exists and tracker configured:
      → "Next: /to-prd, then /to-issues, then /tdd or AFK"
  else:
      → "Next: /tdd <exploit-test-path> per Hole. Or install matt's skills: npx skills add mattpocock/skills"
```

## 4. State machine

```
                  build exploit test
                  OR document code-inspection evidence
   Suspected ───────────────────────────────────────→ Confirmed
                                                          │
                                            fix work (outside /plug-the-holes)
                                            via /tdd / AFK / human
                                                          │
                                                          ▼
                                                       Closed  (re-verified GREEN)
                                                          │
                                                          │  OR
                                                          ▼
                                                 Closed-by-control  (compensating control test GREEN)
                                                          │
                                                          │  OR
                                                          ▼
                                                 Accepted-exception  (signed off at gate)
```

`Suspected` is fleeting: every Hole moves to Confirmed (with one of the two evidence types) or is discharged during the sweep. `Suspected` should not appear in a delivered FINDINGS.md.

## 5. House format — FINDINGS.md

```markdown
# FINDINGS — <app name>

## Run log
- Run 3 · 2026-05-27 · +2 Suspected (H-14, H-15) · −7 Closed (H-03..H-09) · 4 still Confirmed · 0 Re-opened
- Run 2 · 2026-05-26 · +0 Suspected · −0 Closed
- Run 1 · 2026-05-25 · +13 Suspected (initial)

## Status (current)
| ID    | Hole                       | State      | Evidence        | Blocked by   |
|-------|----------------------------|------------|-----------------|--------------|
| H-02  | R2 bucket public-URL       | CONFIRMED  | code-inspection | (env check)  |
| H-05  | Cross-tenant read          | CONFIRMED  | runnable-test   | T-seam       |
| H-13  | Clerk prod issuer          | CONFIRMED  | code-inspection | (env check)  |
| ...                                                                                |

## Hand-off
- 11 Confirmed Holes. Recommended chain: `/to-prd` then `/to-issues` (matt's skills detected).
- Or per-Hole: `/tdd .deslop/exploits/<id>.test.ts`.
- Shared-seam group: H-03..H-12 close together via `convex/_lib/auth.ts` (`withOrg` helper).

---

### H-05 · Cross-tenant read · CONFIRMED
First seen: Run 1 (2026-05-25)
Last changed: Run 1 (no change since)

**What**
Every `list*` / `getById` in `convex/projects.ts` returns any org's data when caller passes an arbitrary `organisationId`.

**Why it ships you broken**
Tenant A reads tenant B's invoices by ID enumeration. Breach in week 1.

**Evidence**
runnable-test: `.deslop/exploits/h05.test.ts` — RED ("Alice cannot list Bravo projects" fails).

**Acceptance (closes when)**
- [ ] `.deslop/exploits/h05.test.ts` — GREEN
- [ ] Same-org positive control still passes

**Blocked by**
T-seam (write `convex/_lib/auth.ts` `withOrg` helper).

**Recommended seam**
`withOrg(ctx, orgId)` — closes H-05/06/07/08 together as one structural edit.

---

### H-11 · R2 presign unauthed · CONFIRMED
First seen: Run 1 (2026-05-25)
Last changed: Run 1

**What**
`videos.getUploadUrl` mints 1-hour presigned R2 PUTs without checking caller identity. Direct cost-runaway path.

**Why it ships you broken**
Stranger loops the call, fills the bucket with arbitrary blobs. Cost-runaway on R2 storage + class-A ops.

**Evidence**
code-inspection: `convex/videos.ts:30-50`. Handler signature `(_ctx, args)` — `ctx` unused; no `ctx.auth.getUserIdentity()` call before `getSignedUrl`.

**Why no runnable test**
`convex-test`'s `'use node'` runtime doesn't load cleanly under `edge-runtime`; building a node rig is a hand-off task, not audit-scope.

**Acceptance (closes when)**
- [ ] Either: build a node-runtime test that proves unauthed caller is rejected, AND it passes
- [ ] Or: fix + paste curl output from prod showing 401 for unauthed `getUploadUrl` call

**Blocked by**
T-seam (same `actionWithOrg` helper covers actions).

---

(repeat per Hole)

---

## Parked (do not block gate)

### Hardening
| ID    | Title                                 | Destination |
|-------|---------------------------------------|-------------|
| HD-01 | Import payload size cap               | post-gate   |
| HD-02 | Cascade delete chunking               | post-gate   |
| ...                                                          |

### Improvement
| ID    | Title                                 | Destination |
|-------|---------------------------------------|-------------------------------|
| IM-01 | Strip obsolete `createdBy` args       | /improve-codebase-architecture |
| IM-02 | Confirm error-tracking wired          | /improve-codebase-architecture |
| ...                                                                          |

### Accepted exceptions
(empty by default; populated only after explicit owner sign-off at gate)
```

**Rules**
- Every Confirmed Hole has all six fixed fields. No optional prose.
- `Why no runnable test` field is **required** when Evidence is `code-inspection`; absent otherwise.
- `Recommended seam` field is optional; present only when one structural edit closes a group.
- `Run log` block is newest-first, one line per run.
- `Status (current)` table is regenerated each run from the per-Hole blocks below.
- Re-opened Holes get an explicit `Re-opened: Run N (date) — was Closed in Run M` line under the timestamps.

## 6. House format — STACK.md

Same scannable principle. Replace prose with a 9-section table:

```markdown
# STACK — <app name>
Generated: 2026-05-25 by /setup-deslop. Re-run when stack changes.

## Stack summary
| Layer          | Choice                              |
|----------------|-------------------------------------|
| Frontend       | React 19 + Vite + TanStack Router   |
| Backend        | Convex (RPC server functions)       |
| Database       | Convex document store               |
| Auth provider  | Clerk                               |
| Payment        | (none)                              |
| Paid per-call  | (none — R2 storage only)            |
| Hosting        | Convex cloud + Tauri desktop shell  |

## Audit categories (the overlay)
| Category        | What it means here                                                   | Audit target                                  |
|-----------------|----------------------------------------------------------------------|-----------------------------------------------|
| Secrets         | Convex env (server) + VITE_ prefix (client-safe)                     | grep dist/ for non-VITE_ secret names         |
| Authentication  | ctx.auth.getUserIdentity() in every public handler                   | enumerate handlers without it                 |
| Tenant scoping  | (target) one withOrg(ctx, orgId) seam routing all org-keyed queries  | enumerate handlers not routed through seam    |
| Authorisation   | requireRole(member, ['supervisor']) on destructive ops               | enumerate destructive ops without role check  |
| Internal surface| internalMutation / role-gate                                         | enumerate public mutations doing internal work|
| Input validation| convex/values v.* on every arg                                       | enumerate v.any() / loose args                |
| Cost & abuse    | actionWithOrg before R2 client construction                          | R2 presign actions without auth               |
| Data integrity  | Convex transactions (atomic)                                         | cross-mutation invariants                     |
| Ops             | Convex backups, prod/dev env split, Sentry on identity paths         | verify-before-ship items                      |

## Gate defaults for this stack
- (any stack-specific must-be-true items beyond the standard line)

## Unknowns (resolved during audit)
- (e.g. R2_PUBLIC_URL setting in prod; CLERK_JWT_ISSUER_DOMAIN in prod)
```

## 7. House format — SURFACE.md

Existing table format is mostly good; tighten prose, drop narrative sections that have no decision content. Each table row = one auditable thing.

```markdown
# SURFACE — <app name>
Generated: 2026-05-25 by /map-the-surface. Inventory, no grading.

## App
| Field    | Value                                                            |
|----------|------------------------------------------------------------------|
| Purpose  | CCTV pipe-inspection toolset for civil/utility contractors        |
| Users    | operator / supervisor / client                                   |
| Tenant   | organisation, keyed by organisationId on every domain row         |
| Stage    | Pre-launch (no external users)                                   |

## Data models
| Model         | Tenant key      | PII? | Notes                                |
|---------------|------------------|------|--------------------------------------|
| organisations | (root)           | yes  |                                      |
| projects      | organisationId   | yes  | clientName/email/siteAddress         |
| ...                                                                         |

## Entry points (the surface)
| Entry point                            | Type      | Auth expected? | Tenant-scoped? | Money? |
|----------------------------------------|-----------|----------------|----------------|--------|
| convex/projects.ts:listByOrganisation  | query     | yes (claim)    | yes (claim)    | no     |
| convex/videos.ts:getUploadUrl          | action    | yes (claim)    | yes (claim)    | yes    |
| ...                                                                                          |

## Data stores
| Store               | Purpose                  | Backed up? |
|---------------------|--------------------------|------------|
| Convex prod         | source of truth          | yes (claim)|
| Cloudflare R2       | video/photo blobs        | unknown    |

## Third-party services
| Service | Used for           | Key location  | Costs per call? |
|---------|--------------------|---------------|-----------------|
| Convex  | backend            | dashboard env | no              |
| Clerk   | auth               | dashboard env | no              |
| R2      | storage            | Convex env    | yes (per op)    |

## Money & identity flows
- Authn: Clerk (UI), but no ctx.auth.getUserIdentity() in handlers
- Authz: meant to live in handlers; missing
- Charging / entitlement: none

## Unknowns
- R2_PUBLIC_URL setting in prod Convex env
- CLERK_JWT_ISSUER_DOMAIN in prod Convex env
- R2 lifecycle / CORS / versioning policy
- Backup restore test evidence
- Error-tracking provider in code
```

## 8. House format — GATE.md

```markdown
# GATE — <app name>   ·   target launch: <date>
Re-verified: 2026-05-27 by running, not by reading.

## Decision: **NO-SHIP** (or **SHIP**)
Reason: 4 Confirmed Holes still RED (see FINDINGS.md).

## Must be TRUE to ship
- [ ] Every Confirmed Hole in FINDINGS.md is Closed or Closed-by-control       (currently: 7/11)
- [ ] Every Accepted Exception is signed off below                              (currently: 0)
- [ ] SURFACE.md Unknowns section is empty                                      (currently: 5 unknown)
- [ ] No secrets in repo / history / dist bundle (h01_bundle_secrets.sh GREEN)  (currently: GREEN)
- [ ] Tenant isolation proven on every tenant-data path                          (covered by exploit suite)
- [ ] Paid/AI endpoints authed AND capped                                        (covered by H-11/H-12)
- [ ] Database backups exist + restore tested                                   (currently: claimed, no evidence)
- [ ] Bad deploy can be rolled back                                              (currently: unknown)
- [ ] Error tracking live; structured logs on money/identity                    (currently: unknown)

## Open blockers (commands to run)
1. **H-02** — `bash .deslop/exploits/h02_r2_public_bucket.sh` against prod, or confirm `R2_PUBLIC_URL` unset
2. **H-13** — `bash .deslop/exploits/h13_clerk_issuer.sh` once Convex prod login available
3. **HD-05** — one rehearsed Convex restore (evidence to ops/restore-2026-05-27.md)
4. **IM-02** — identify error-tracking provider; confirm UnauthorizedError events land

## Accepted exceptions
(empty — none signed off)

## Explicitly NOT blocking
- Hardening list (HD-*): <count> → post-gate
- Improvement list (IM-*): <count> → /improve-codebase-architecture
- Parked bugs/races: <count> → /diagnose
```

## 9. Re-run behaviour

`/plug-the-holes` on second run:
1. Re-runs the Suspected sweep from scratch (drift detection) — produces a fresh candidate list.
2. Cross-references with existing `FINDINGS.md`:
   - Existing Confirmed Hole, sweep finds same evidence → unchanged (no `Last changed` bump).
   - Existing Confirmed Hole, sweep cannot reproduce → mark `Last changed: Run N` and add note `Evidence no longer reproducible — investigate`.
   - New Hole not in last run → add as Suspected, then build evidence as normal.
   - Existing Closed Hole, exploit test still GREEN → unchanged.
   - Existing Closed Hole, exploit test now RED → mark `Re-opened: Run N — was Closed in Run M`, state back to Confirmed.
3. Prepends a one-line `Run log` entry summarising the delta.

Never deletes a finding. History is the entire point.

## 10. ADRs to write at implementation time

- **ADR 0006** — `/plug-the-holes` is audit-only (D1, D3).
- **ADR 0007** — Evidence may be runnable-test or code-inspection (D4, refines ADR 0001).
- **ADR 0008** — Generated files use the fixed-fields scannable format (D5).

ADRs follow the existing convention: title + 1–3 sentences.

## 11. Files to update at implementation time

- `skills/plug-the-holes/SKILL.md` — remove steps 5 (fix) + 6 (re-verify); rewrite step 4 (Confirm or discharge) to capture Evidence type explicitly; add hand-off detection logic to step 7.
- `skills/plug-the-holes/references/FINDING-FORMAT.md` — replace template entirely with the fixed-fields block.
- `skills/plug-the-holes/references/CHECKLIST.md` — no change.
- `skills/setup-deslop/SKILL.md` + `references/STACK-TEMPLATE.md` — adopt scannable house format.
- `skills/map-the-surface/SKILL.md` — adopt scannable house format; tighten SURFACE.md template.
- `skills/launch-gate/SKILL.md` — adopt scannable GATE.md format; preserve all rules; add the re-run-detection note (re-run after fixes = re-verify, no new gate doc needed; just update existing).
- `skills/deslop/SKILL.md` — update "Run order" to call out idempotency + the new audit-stops-after-RED stopping point; update hand-off recommendation.
- `skills/deslop/GLOSSARY.md` — add `Evidence` term with the two types; clarify Suspected is transient.
- `README.md` — update pipeline diagram to show the audit / fix-elsewhere / re-verify shape.
- `docs/adr/0006`, `0007`, `0008` — per above.

## 12. Migration

The PipeInspect run's `FINDINGS.md` does not need to be regenerated by the user — its current `Closed` states reflect real work. When PipeInspect next runs `/deslop`, the new format will be produced fresh and the old file overwritten. The 18-file structural edit that already shipped becomes the "fix work that happened during the experimental first run"; future projects won't see it inside the audit session.

## 13. Out of scope for this redesign

- Anything in Phase 2 (architecture / diagnose / tdd) — those are matt's skills, untouched.
- An audit-native `/to-prd` competitor — we recommend matt's; we don't replace.
- Auto-publishing to issue trackers from `/plug-the-holes` — that's `/to-issues`'s job, kept separate.
- Cross-project audit aggregation (org-level dashboards). Out of scope unless requested.
