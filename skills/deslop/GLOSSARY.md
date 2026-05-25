# Deslop Glossary

Shared language for the whole pipeline. Use these terms exactly, across findings, reports, the gate doc, and conversation. Drift ("vulnerability", "issue", "tech debt" used interchangeably) is how a deslop turns back into slop.

## The work

- **Slop**: code that runs but was never engineered. Symptoms: no input validation, authorisation missing or copy-pasted per-handler, two libraries doing the same job, secrets in the repo, no tests, no error handling past the happy path. Slop isn't "bad code"; it's *ungoverned* code. Removing slop means imposing boundaries, not rewriting.
- **Deslop**: the act of making slop safe to ship, then incrementally governable. Two phases, hard line between them.

## Severity - the only three buckets

Every finding lands in exactly one. The bucket decides *when* it gets fixed, not *whether*.

- **Hole**: a defect that gets you breached, sued, or hit with a runaway bill on day one. The **only** category that blocks launch. Examples: tenant A can read tenant B's data; an admin mutation callable unauthenticated; a service key in the client bundle; an unthrottled endpoint that bills per call.
- **Hardening**: reduces real risk but is survivable on launch day. Goes on the post-gate list. Examples: missing rate limit on a cheap endpoint, no audit log, weak-but-present validation.
- **Improvement**: quality, maintainability, architecture. Never blocks launch. Examples: mixed libraries, shallow modules, no tests around non-critical paths, duplication.

If you can't decide between Hole and Hardening, ask: *"If this shipped tonight and someone hostile found it within a week, what's the worst outcome?"* Breach / legal / financial = Hole. Embarrassment / slow fix = Hardening.

## Surface & isolation

- **Surface**: every way untrusted input reaches the system: HTTP routes, public backend functions / RPC endpoints, webhooks, file uploads, queue/cron consumers, anything callable from a browser or a stranger's curl.
- **Default-deny**: every point on the surface is public and hostile until *proven* otherwise. Audit from that assumption, not the reverse.
- **Authn (authentication)**: proving *who* the caller is. Usually handled by the auth provider.
- **Authz (authorisation)**: deciding *what* that caller may see or do. Almost always the app's job, and almost always where slop fails.
- **Tenant isolation**: the guarantee that tenant A can never observe or mutate tenant B's data. The single most important property of a multi-tenant SaaS, and the most commonly broken.

## Findings & proof

- **Exploit test**: a concrete, re-runnable demonstration that a Hole is real: the curl, the script, the failing test that reads the wrong tenant's row. The security analog of a feedback loop. **No exploit test → not a confirmed Hole, just a worry.**
- **Compensating control**: a claimed mitigation that makes a Suspected Hole safe without fixing it in the obvious place (edge auth in front of an unauthed route, a deliberate cross-tenant support console). A control is a *claim*: it gets its own exploit test that proves the control actually blocks the bad outcome, and only counts when that test is green.
- **Finding states**:
  - **Suspected**: you have a hypothesis, no proof yet.
  - **Confirmed**: exploit test is *red* (the bad thing happens). It's now a real Hole.
  - **Closed**: fix applied, exploit test is *green* (the bad thing no longer happens) and you re-ran it yourself.
  - **Closed-by-control**: not fixed, but a compensating control was *proven* to block the bad outcome (its exploit test is green). Counts as Closed at the gate.
  - **Accepted exception**: a Suspected/Confirmed Hole the owner deliberately chooses to carry, with a recorded justification and the owner's name, because the design is intentional and has no enforceable control. **Never silent**: it is surfaced at the gate for explicit sign-off, not closed.
- **The Gate**: the written, per-app line. Every Confirmed Hole is Closed (or Closed-by-control), and every Accepted Exception is signed off in writing = ship. Nothing on the Hardening or Improvement lists may move the line.

## Vocabulary layers (no drift with CONTEXT.md)

This glossary is **method vocabulary**: how the audit talks about defects (Hole, Surface, exploit test). It is project-independent, ships inside the skill, and is the same on every codebase you ever deslop.

A project's `CONTEXT.md` (built and read by the architecture/grilling skills) is **domain vocabulary**: how *that* app talks about itself (Order, Shipment, the materialisation cascade). It is project-specific and lives in the audited repo.

A finding uses both: *"the **Order export** path (domain) is a tenant-isolation **Hole** (method)."* `deslop` reads `CONTEXT.md` if it exists and names things with the project's domain terms; if recon has to coin a durable domain term, it writes it into `CONTEXT.md` rather than inventing a private synonym. One home per layer.

## Prime heuristics

- **Demonstrate, don't assert.** A Hole isn't real until you've reproduced it. This kills the endless theoretical-risk noise that audits of generated code drown in.
- **Money and identity first.** Auth, tenant isolation, and billing before anything else. Those are the breaches that end companies.
- **Authn ≠ authz.** Knowing who someone is tells you nothing about what they may touch. Check the data layer, not the login.
- **Scope once, at a seam.** Enforce tenant scoping through one shared wrapper, not re-implemented per function. This is where security and DRY are the same move.
- **The gate is a line, not a wish.** Write it down. Everything above ships; everything below waits. Gold-plating before launch is slop with better intentions.
