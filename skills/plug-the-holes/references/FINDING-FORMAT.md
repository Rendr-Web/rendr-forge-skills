# Finding Format

How `FINDINGS.md` is laid out, so it can be scanned in seconds AND mechanically converted into a `/to-prd` PRD or a per-Hole set of `/to-issues` tickets. The format is **fixed**: every Confirmed Hole has the same six fields in the same order, every time. Reasoned prose lives in those fields, not around them.

## File shape

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
- <N> Confirmed Holes. Recommended chain: `/to-prd` then `/to-issues` (matt's skills detected).
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

## Parked (do not block gate)

### Hardening
| ID    | Title                                 | Destination |
|-------|---------------------------------------|-------------|
| HD-01 | Import payload size cap               | post-gate   |
| HD-02 | Cascade delete chunking               | post-gate   |

### Improvement
| ID    | Title                                 | Destination                    |
|-------|---------------------------------------|--------------------------------|
| IM-01 | Strip obsolete `createdBy` args       | /improve-codebase-architecture |
| IM-02 | Confirm error-tracking wired          | /improve-codebase-architecture |

### Accepted exceptions
(empty by default; populated only after explicit owner sign-off at gate)
```

## The six required fields

Every Confirmed Hole has these in this order:

1. **What** — one-line claim, falsifiable. Names the surface unit (path / handler / endpoint) and the bad outcome. No background, no rationale.
2. **Why it ships you broken** — 1–2 lines of impact. The "ends companies" reason this is a Hole, not Hardening.
3. **Evidence** — exactly one of:
   - `runnable-test: <path>` — path to the RED test in `.deslop/exploits/`. Preferred.
   - `code-inspection: <file:line>` + reasoning paragraph. Used only when a runnable test is genuinely infeasible from the audit context.
4. **Why no runnable test** — **required when Evidence is `code-inspection`, absent otherwise.** One paragraph justifying why a test couldn't be built (runtime constraint, prod-env value, infra not available). If this field is hand-waving, the auditor was lazy and the entry should be downgraded to Suspected (and then either built or discharged before delivery).
5. **Acceptance (closes when)** — checkbox list. For `runnable-test` Evidence: the test going GREEN is the first checkbox. For `code-inspection` Evidence: two paths offered ("either build the test and pass it, or fix + manually verify with <specific check>") so the fix-implementer knows what closes the finding.
6. **Blocked by** — other finding IDs (H-03, H-05) or named pre-requisite tasks (`T-seam`, `T-prod-env-read`). Used by `/to-issues` to build the dependency graph.

Optional 7th field:

7. **Recommended seam** — present only when one structural edit closes a group of Holes. Names the seam (e.g. `withOrg(ctx, orgId)`) and lists which IDs close together. Carries the "scope once, at a seam" insight into the hand-off without the audit having to write the seam itself.

## Per-Hole timestamps

Every block starts with two lines under the heading:

```
First seen: Run N (YYYY-MM-DD)
Last changed: Run M (YYYY-MM-DD) — optional brief note
```

Re-opened Holes get an extra line:

```
Re-opened: Run N (YYYY-MM-DD) — was Closed in Run M
```

These let a reader judge staleness without consulting git.

## State transitions across re-runs

`/plug-the-holes` is idempotent. On every run it re-sweeps and cross-references existing findings:

| Situation                                                  | Action                                                                                  |
|------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| Existing Confirmed Hole, sweep finds same evidence         | unchanged (no `Last changed` bump)                                                      |
| Existing Confirmed Hole, sweep cannot reproduce evidence   | bump `Last changed`; add note `Evidence no longer reproducible — investigate`           |
| New Hole not in last run                                   | add fresh entry; `First seen: Run N`                                                    |
| Existing Closed Hole, exploit test still GREEN             | unchanged                                                                               |
| Existing Closed Hole, exploit test now RED                 | flip state back to Confirmed; add `Re-opened: Run N — was Closed in Run M`              |
| Existing Suspected Hole (only possible mid-run)            | resolve to Confirmed or discharge before saving; never deliver a Suspected entry        |

**Never delete a finding.** History is the entire point. Discharged sweeps disappear (they were Suspected, never delivered). Closed Holes stay in the file forever as a record.

## Roll-up rules

The `Status (current)` table at the top of `FINDINGS.md` is regenerated from the per-Hole blocks on every run. Counts must match. If the gate decision is glance-able from the `Status` table + the `Hand-off` block, the format is doing its job.

The `Run log` is append-prepend: newest run on top, one line per run. After ~5 entries it can be truncated with `(earlier runs: git log -p FINDINGS.md)` to keep the header skimmable.

## What does NOT go in FINDINGS.md

- Production code edits — `/plug-the-holes` doesn't make them, so nothing to log.
- Fix narratives, diffs, before/after — those belong in the PR that closes the Hole (driven by `/tdd` or AFK), not in FINDINGS.md.
- Background prose explaining the project — that's in `STACK.md` / `SURFACE.md`.
- Rationale for severity bucket choice — the GLOSSARY's Hole/Hardening/Improvement rule decides this; if it's not obvious from `Why it ships you broken`, the finding is in the wrong bucket.
