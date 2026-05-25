# Mapping Pattern

How to translate the stack-agnostic audit into the codebase in front of you, for **any** stack, including one nobody's heard of yet. This is the reference `/setup-deslop` uses to interview you and write your project's `STACK.md`. You rarely read this directly; you read the `STACK.md` it produces.

The audit categories never change. What changes per stack is the *answer to five questions*. Get those answers and every checklist category has a concrete target.

## The five questions

Answer these and you've mapped the stack. Everything in the audit derives from them.

1. **What is the surface?** How does untrusted input reach the backend, and what's the unit to enumerate? REST routes? RPC / server functions? Resolvers? Server actions? Webhook handlers? → *defines what "an endpoint" means and what the public-vs-internal check enumerates.*
2. **How is something marked not-public?** What's the mechanism for "server-to-server / admin only, not callable by a stranger"? Route middleware? A separate internal service? A naming/visibility convention (e.g. `internal*` functions)? → *defines the public-vs-internal check.*
3. **Where does verified identity come from?** What proves who the caller is, and how does a handler read that identity *safely*; i.e. from a verified token/session, never from request args the client controls? → *defines authentication, and the "never trust identity from the body" rule across authorisation and tenant isolation.*
4. **Where is authorisation meant to live, and what is a tenant?** What's a tenant (org / team / user)? What field keys a row to its tenant? Is there a single shared seam that enforces scoping, or is it re-done per handler? → *defines authorisation and tenant isolation and the "scope once at a seam" target.*
5. **What costs money, and how does money move?** Which calls hit a paid API per request (LLM, SMS, email)? What's the payment processor, and how are its webhooks authenticated? Is there an entitlement / rate-limit layer? → *defines cost & abuse and the data-integrity webhook-idempotency check.*

Plus two housekeeping answers: **where do secrets live** (and what's the client/server env convention, so you know what leaks to the browser), and **what's the ops story** (backups+restore, prod/dev split, rollback, error tracking).

> Whatever the answers name (a different auth provider, payment processor, prototyping tool), the *questions* are identical and the *categories* don't move. Changing auth vendor changes question 3's answer and nothing else.

## Worked examples (by archetype, not by brand)

Three common shapes. Each is mapped against the five questions so you can see the form a good mapping takes. Match the codebase to the nearest archetype, then refine from the actual stack.

### Archetype A: RPC backend with a managed auth provider
*(backend is a set of callable server functions; a hosted service handles login)*
1. **Surface:** every exported server function callable from the client. Enumerate all of them.
2. **Not-public:** a visibility tier for internal-only functions (server-to-server / scheduled). The Hole is a sensitive function left in the public tier.
3. **Identity:** read from the verified auth context the provider populates, never from function args. A function taking a user/tenant id as an *argument* is the tell.
4. **Authz/tenant:** typically **no automatic row security**; every function scopes itself. Target seam: one helper that returns the verified tenant id from the identity, with every data function routing through it and filtering on a tenant-keyed index. Audit = find public data functions that *don't*.
5. **Money:** paid/AI calls live in server actions. Must be authed + spend-capped *before* the call. Processor webhooks arrive as HTTP handlers: verify signature, make idempotent.
- **Secrets:** the server/client env convention decides what ships in the bundle. Check the built bundle, not just source.

### Archetype B: Server-rendered framework with a relational database
*(framework with API routes + server actions; SQL store; a session library)*

Same five questions, different shape. Server actions are the easy-to-forget surface, and middleware that's *defined* but not *applied* is the classic Hole.

1. **Surface:** API route handlers + server actions. Enumerate both.
2. **Not-public:** middleware applied to a path group. The Hole is middleware defined but not actually applied to a route.
3. **Identity:** the server session helper. Reading a user id from the request body instead is the tell.
4. **Authz/tenant:** scoping is a `WHERE tenant_id = $session_tenant` on every query, or pushed into the DB with Row-Level Security so it can't be forgotten. Target seam: a scoped data-access layer, or RLS policies. Audit = find queries that interpolate an id with no tenant clause (also an input-validation injection check).
5. **Money:** processor SDK calls + a webhook route, signature-verified and idempotent. Per-request paid calls need a rate limit.
- **Secrets:** the framework's public-env prefix decides browser exposure. Anything sensitive with a public prefix is a Secrets Hole.

### Archetype C: Standalone API service with an ORM
*(separate backend service; ORM over a database; self-managed sessions)*

The route table is the surface. RBAC tends to be UI-only here, so the hidden button still has a live endpoint behind it.

1. **Surface:** the route table(s). Enumerate every `router.*`; hunt for undocumented `/debug`, `/admin`, `/seed`.
2. **Not-public:** auth middleware on a router. Hole: a route mounted outside the protected router.
3. **Identity:** the session middleware populating `req.user`. Trusting a header/body field instead is the tell.
4. **Authz/tenant:** explicit checks in each handler or a shared guard. Target seam: a `scopeToTenant(req)` guard with ORM queries always filtered by it.
5. **Money:** processor SDK + webhook endpoint (raw-body signature check). Paid per-request calls need throttling.
- **Secrets:** `.env` committed or hardcoded keys are the usual Secrets catastrophe; rotate anything ever exposed.

## Migrating between stacks
If an app is being ported (one stack → another), the shipped app is the *destination*. Derive `STACK.md` for the destination and audit that. Pin the source's critical-path behaviour with characterisation tests first so the port has a target, and re-run exploit tests on both sides: a Hole closed in the source can reopen in the port under a different auth/scoping model.
