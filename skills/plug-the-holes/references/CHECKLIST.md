# Ship-Blocker Checklist

The exhaustive, stack-agnostic checks behind each audit category in `SKILL.md`. Work the categories in the order given there (money-and-identity-first). For *how* each check maps to a real codebase, pair this with `STACK-MAPPINGS.md`.

Each check is phrased as a **Hole to disprove**, not a box to tick. The job is to build an exploit test that makes the bad thing happen; if you can't, the check passes.

## Table of contents
1. Secrets & credentials
2. Authentication
3. Tenant isolation
4. Authorisation
5. Public vs internal surface
6. Input validation
7. Cost & abuse
8. Data integrity & critical-path races
9. Backups, env separation, observability

---

## 1. Secrets & credentials
- A secret (API key, DB URL, service token, signing secret) is committed to the repo, in git history, or readable in the **built client bundle**.
- A key that should be server-only is exposed to the browser (wrong public/private env prefix).
- The same key is shared across dev and prod, so a dev leak is a prod breach.
- No plan to **rotate** anything that was ever exposed. (Exposed once = compromised. Rotate, don't hide.)
- Third-party webhooks accept unsigned requests (no signature verification) - effectively a secretless write endpoint.

## 2. Authentication
- An endpoint that assumes a logged-in user doesn't actually verify the session/token.
- Tokens don't expire, can't be revoked, or are accepted after logout.
- User identity is taken from a request body/header the client controls (`userId` in the payload) rather than the verified session.
- Password reset / email verification / magic-link flows can be replayed or skipped.
- Auth provider is configured in test/dev mode in production.

## 3. Tenant isolation - *the big one*
- A read path (list/get/search/export) doesn't filter by the caller's tenant key → tenant A sees tenant B's rows by changing or guessing an ID.
- A write path (update/delete) doesn't check the target row belongs to the caller's tenant → cross-tenant mutation.
- A nested/related query joins through to another tenant's data (the parent is scoped, the child isn't).
- Aggregate/report endpoints compute over all tenants.
- File/blob URLs are guessable or unscoped (object storage with public read).
- The tenant key is trusted from the client instead of derived from the session.
- **Test every one of these as a low-privilege user of tenant A trying to reach tenant B.** Isolation is not a setting; it's a property you prove per path.

## 4. Authorisation (RBAC)
- A role check exists on the route but not on the underlying data function (so the function is reachable another way).
- Admin/elevated actions are gated only in the UI (button hidden) not on the server.
- Role is read from client-controlled input.
- A user can escalate their own role via a normal update endpoint (mass-assignment).
- "Internal" or "service" actions can be invoked by an ordinary authenticated user.

## 5. Public vs internal surface
- A function/route that should be internal-only is publicly callable (see `STACK.md` for how "internal" is marked in this stack).
- Debug, seed, migration, or admin routes left mounted in production.
- Verbose error responses leak stack traces, SQL, or internal IDs to the client.
- GraphQL introspection / API docs exposing private operations.
- CORS set to `*` on credentialed endpoints.

## 6. Input validation
- Any mutation accepts unvalidated shape/type → malformed data persists and corrupts later reads.
- String input reaches a query without parameterisation (SQL/NoSQL injection).
- User-supplied content is rendered without escaping (stored XSS).
- File uploads don't check type/size → storage abuse or malicious file served back.
- Numeric/enum fields accept out-of-range values that break invariants (negative quantities, unknown status).
- Redirect/URL params aren't allow-listed (open redirect).

## 7. Cost & abuse
- An endpoint that calls a paid API (LLM, SMS, email) is reachable unauthenticated or unthrottled → financial DoS by loop.
- No per-user/per-tenant rate limit on expensive or write-heavy paths.
- No cap on AI token spend per request/user/day.
- Unbounded queries (no pagination) let one caller pull or compute over everything.
- No protection on signup/login (credential stuffing, signup spam creating cost).

## 8. Data integrity & critical-path races
- A charge/refund/entitlement path can run twice for one intent (no idempotency key) → double-charge.
- Double-submit creates duplicate records (no uniqueness constraint or idempotency).
- Read-modify-write on a balance/counter/inventory without a transaction or atomic op → lost update.
- Multi-step operations have no rollback → partial writes leave inconsistent state.
- Webhook handlers aren't idempotent (providers retry; you process twice).
- *Only critical-path races are Holes here.* General race conditions and latent bugs go to `/diagnose`, parked.

## 9. Backups, env separation, observability
- No database backups, or backups never tested for restore.
- No prod/dev separation; dev work can read or clobber prod data.
- No way to roll back a bad deploy.
- No error-tracking service → you're blind to prod failures on code you didn't write.
- No structured logging on the money/identity paths → you can't reconstruct an incident.
- (These are launch-readiness Holes even though they're not "vulnerabilities" - shipping without them means the first incident is unrecoverable and invisible.)

---

### Compliance note (jurisdiction-dependent, flag don't guess)
If the app stores personal information, there are likely legal obligations (in AU: the Privacy Act / APPs; elsewhere GDPR/CCPA etc.). You are not the lawyer - **flag** PII handling, consent, and breach-notification gaps as findings for the owner to take advice on, rather than asserting compliance status.
