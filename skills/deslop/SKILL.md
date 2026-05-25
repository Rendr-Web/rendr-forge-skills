---
name: deslop
description: Orchestrates turning a vibe-coded app into something safe to ship, then methodically better. Routes through setup → recon → security/ship-blocker audit → launch gate → incremental hardening. Use this whenever the user wants to "de-slop", harden, take over, rescue, audit, productionise, or safely launch a vibe-coded / AI-generated / prototype codebase, including apps built with AI app-builders or vibe-coding tools, even if they only describe the symptoms ("a mate built this and it's a mess", "is this safe to launch", "inherited a prototype"). Start here; this skill decides which sub-skill to run and in what order.
---

# Deslop

Turn a vibe-coded app into something safe to ship, then incrementally better. The discipline is one idea: **separate the defects that can hurt someone on launch day from the ones that only hurt you later, and refuse to let the second kind block the first.**

Read [GLOSSARY.md](./GLOSSARY.md) before you start and use its language throughout. The severity buckets (**Hole / Hardening / Improvement**) are load-bearing; every finding is one of those three, and the word you pick decides when it gets fixed.

The pipeline is **stack-agnostic by design**. The categories ("is the surface authed", "is each tenant isolated", "can a stranger run up the bill") are universal; what differs per codebase is only their *instantiation*: what counts as an endpoint, where identity comes from, how webhooks are signed. That instantiation lives in one place, a per-project `STACK.md`, generated once by `/setup-deslop`. Swap the auth provider or payment processor and only `STACK.md` changes; the method doesn't.

## The shape of the work

Two phases, one hard line between them: **the Gate**.

```
  PHASE 1 - PLUG THE HOLES                   │ GATE │   PHASE 2 - DESLOP PROPER
  (everything that blocks launch)             │      │   (everything that doesn't)
                                              │      │
  setup → recon → audit → close every Hole    │ ship │   consistency → architecture
                                              │      │   → DRY → bugs → tech debt → features
```

Most teams get this backwards: they tidy the architecture (satisfying) while a tenant-isolation Hole sits open (fatal), or they gold-plate forever and never ship. The pipeline exists to stop both.

## Run order

Work top to bottom. Don't skip recon: **you cannot audit a surface you haven't mapped**, and generated codebases routinely hide entry points (stray webhooks, debug routes, a second database) the author forgot exist.

0. **`/setup-deslop`**: *First time in a repo only.* Interviews you about the stack and writes `STACK.md`, the overlay every later step reads. Skip if `STACK.md` already exists and the stack hasn't changed.
1. **`/map-the-surface`**: Recon. Produces `SURFACE.md`: data models, every entry point, where data lives, the tenant model, third-party services, where money moves.
2. **`/plug-the-holes`**: The security/ship-blocker audit loop. Walk the surface money-and-identity-first, **demonstrate every Hole with an exploit test**, fix, re-verify. Produces a findings log.
3. **`/launch-gate`**: Write the per-app line, verify every Confirmed Hole is Closed, make the ship / no-ship call. Resist gold-plating.

Then, and only after the gate, Phase 2 is **incremental and never blocks a ship**. Hand off to skills that already do this well rather than reinventing them. These live in `mattpocock/skills` (https://github.com/mattpocock/skills). Names below are indicative; defer to that repo for the current set.

- **`/improve-codebase-architecture`**: consistency, deep vs shallow modules, layout, DRY at the seams. Run it every few days, not once.
- **`/diagnose`**: for the latent bugs and race conditions surfaced during the audit but parked as non-Holes.
- **`/tdd`**: for new features and for pinning behaviour before you refactor.
- **`/grill-with-docs`**: extend the shared language (`CONTEXT.md`) for the app. Recon has already seeded it with the domain terms it needed; Phase 2 grows it. (Method vocabulary stays in the skill's glossary; see the "Vocabulary layers" note there.)

## Migration apps

When an app is being ported from one stack to another (e.g. a prototype to your production stack), the shipped app is the *destination*. Run `/setup-deslop` for the destination and audit that. Pin the source's critical-path behaviour with characterisation tests first (`/tdd`) so the port has a target, then re-run exploit tests on both sides: a Hole closed in the source can reopen in the port.

## What "done with Phase 1" means

It does not mean "the code is good". It means: `STACK.md` and `SURFACE.md` exist, every Confirmed Hole is Closed with a green exploit test, and the Gate doc says ship. Ugly-but-safe is a legitimate, intended state. Pretty-but-breachable is not.
