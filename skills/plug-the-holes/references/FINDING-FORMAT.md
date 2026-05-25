# Finding Format

How to log what the audit turns up. One findings file per app, `FINDINGS.md`, appended to as you go. The format exists so `/launch-gate` can drive straight off it: it should be able to answer "is every Hole Closed?" by reading this file alone.

## Every finding has

- **ID**: `H-01`, `H-02` for Holes; `HD-01` for Hardening; `IM-01` for Improvements. Stable, referenceable.
- **Bucket**: Hole / Hardening / Improvement (see GLOSSARY).
- **State**: Suspected / Confirmed / Closed / Closed-by-control / Accepted-exception. (Hardening and Improvement skip straight to a parked state.)
- **Category**: which checklist section (1–9).
- **Claim**: the concrete bad outcome, falsifiably stated.
- **Exploit test**: path to the curl/script/test that demonstrates it. Holes only. No exploit test = stays Suspected.
- **Compensating control**: *(if any)* the claimed mitigation and the path to the test that proves it blocks the bad outcome. A green control test → Closed-by-control.
- **Accepted exception**: *(if any)* the owner's justification + name. Required before a Hole can be carried past the gate without a fix.
- **Fix**: what was changed and *at which seam*. Closed only.
- **Re-verify**: confirmation the exact exploit test went green.

## Template

*(Example uses an RPC + per-function-scoping stack; substitute your own surface unit and seam from `STACK.md`.)*

```markdown
### H-03 · Tenant isolation · CONFIRMED → CLOSED
**Category:** Tenant isolation
**Claim:** `api.invoices.list` collects all invoices with no org filter; a logged-in
user from org A reads org B's invoices.
**Exploit test:** `deslop/exploits/h03_cross_tenant_invoices.test.ts` - calls list as
org A's user, asserts zero org B rows. Initially RED (returned 14 org B rows).
**Fix:** routed `list` through `requireOrg(ctx)` + `.withIndex("by_org")`. Same seam
now used by `get`, `search`, `export` (H-04, H-05 closed by the same edit).
**Re-verify:** re-ran h03 test - GREEN (0 cross-tenant rows). 2026-05-25.
```

## Rules

- **One finding, one bucket.** If you're tempted to write "Hole-ish", it's Hardening. Decide.
- **Holes need the exploit-test line filled before they can be Confirmed**, and the re-verify line before they can be Closed. An empty exploit-test field on a "Confirmed" finding is a process error; downgrade to Suspected.
- **Group fixes that share a seam.** If one `requireOrg` wrapper closes H-03/04/05, say so. It documents that the fix was structural, and that's the signal the architecture got better, not just safer.
- **Park, don't fix, the non-Holes.** Hardening and Improvement findings are logged with a one-line note and a destination (`→ post-gate list` / `→ /improve-codebase-architecture` / `→ /diagnose`). They are not worked during the audit.

## Roll-up for the gate

At the top of `FINDINGS.md`, keep a live count so the gate decision is glanceable:

```markdown
## Status
Holes:        7 found · 6 Closed · 1 Closed-by-control · 0 open   ← gate needs 0 open
Exceptions:   1 Accepted (needs owner sign-off at the gate)
Hardening:    5 parked (post-gate)
Improvement:  9 parked (→ architecture / diagnose)
Last audit:   2026-05-25
```
