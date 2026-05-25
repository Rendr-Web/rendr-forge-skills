# Rendr Forge Skills

The open tooling behind **Rendr Forge**, Rendr's practice for rescuing vibe-coded apps and getting them safe to ship. The method is called *deslop*: turn a vibe-coded app into something safe to ship, then methodically better.

A small, composable set built in the spirit of [mattpocock/skills](https://github.com/mattpocock/skills). It hands off *to* his skills for the back half of the work rather than reinventing it.

**Opinionated about method, pluggable about stack.** The audit categories are universal; what each one *means* in your codebase lives in a per-project `STACK.md` that `/setup-deslop` generates by interviewing you. Switch your auth provider or payment processor and only `STACK.md` changes; the workflow doesn't.

## The pipeline

You only ever type `/deslop`. It is the one entry point and it's **idempotent** — running it once gives you the audit + a NO-SHIP gate. Running it again after fix work re-verifies and flips to SHIP.

```
/deslop                       ← the one entry point. Orchestrates everything below.
  ├── /setup-deslop           step 0: first time in a repo only. Interviews your stack → STACK.md
  ├── /map-the-surface        recon → SURFACE.md
  ├── /plug-the-holes         security/ship-blocker audit → FINDINGS.md   ← the core
  │                           STOPS after building RED exploit tests; does NOT edit production code
  └── /launch-gate            re-runs the exploit suite → SHIP if all GREEN, NO-SHIP if any RED
                              produces / updates GATE.md
```

Fix work happens *between* `/deslop` runs, not inside one. After the audit hands you `FINDINGS.md` + a RED exploit suite, you pick one of two chains and come back when fixes are done:

```
audit (today)                                            re-verify (later, same /deslop command)
   │                                                              │
   ▼                                                              │
FINDINGS.md + RED exploit tests                                   │
   │                                                              │
   ├── matt's skills present → /to-prd → /to-issues → AFK / /tdd ─┤
   │                                                              │
   └── matt's skills absent  → /tdd <exploit-test-path> per Hole ─┘
```

Each closed Hole is its own small reviewable PR. No catastrophic 18-file structural diff in the audit session.

## The one idea

Separate the defects that can hurt a customer on launch day (**Holes**) from the ones that only hurt you later (**Hardening**, **Improvement**). Close every Hole, write down the line, ship. Everything else is incremental and never blocks a launch.

Three heuristics carry most of the weight:
- **Demonstrate, don't assert.** A Hole isn't real until you have evidence: a re-runnable exploit test (preferred), or a precise code-inspection finding where a runnable test is genuinely infeasible from the audit context. Lazy "looks bad" assertions don't count.
- **Money and identity first.** Auth, tenant isolation, and billing before anything else.
- **Scope once, at a seam.** Enforce tenant scoping in one shared place; closing the Hole and removing the slop become the same edit. The audit *names* the seam in `FINDINGS.md`; the fix-side skill (or AFK agent) *writes* it as one ticket plus N "route handler X" tickets.

## Reference

`/deslop` is the only command you need to invoke. The others are the steps it runs; documented here so you can read what each does (and so you can call one directly if you ever want to re-run a single step).

- **[deslop](./skills/deslop/SKILL.md)** is the orchestrator and the only entry point. It covers the holes-vs-improvement split, the run order, and the hand-off to architecture/diagnosis skills for Phase 2. Start here.
- **[setup-deslop](./skills/setup-deslop/SKILL.md)** is step 0 inside `/deslop`, run automatically the first time in a repo (skipped on later runs if `STACK.md` exists). Interviews you about the stack and writes `STACK.md`, the overlay the audit reads.
- **[map-the-surface](./skills/map-the-surface/SKILL.md)** does recon: data models, entry points, data stores, tenant model, and money flows into `SURFACE.md`.
- **[plug-the-holes](./skills/plug-the-holes/SKILL.md)** is the core. A security/ship-blocker audit loop that demonstrates every Hole with evidence (a re-runnable exploit test where feasible, or a precise code-inspection finding where not) and **stops there**. Produces `FINDINGS.md` plus a `.deslop/exploits/` suite of RED tests. Does not edit production code — fix work happens in a separate session via `/tdd` or matt's `/to-prd` → `/to-issues` chain.
- **[launch-gate](./skills/launch-gate/SKILL.md)** writes the go/no-go line, verifies every Confirmed Hole is Closed, and makes the ship call. Produces `GATE.md`.

## Companion skills (Phase 2)

Deslop stops at the launch gate. The "make it methodically better" half (architecture work, diagnosing hard bugs, test-driven features, growing the project's domain docs) is already well covered by **Matt Pocock's skills**, so the pipeline hands off to them rather than reinventing them.

They're recommended companions for Phase 2. (Phase 1, the audit and the gate, stands alone and needs nothing else.) Install them from the source, which stays current as he adds or renames skills:

→ **https://github.com/mattpocock/skills** · `npx skills add mattpocock/skills`

See his README for the current skill list and install options. We deliberately don't enumerate specific skills or pin install flags here; naming someone else's skills in our repo is how instructions go stale. We also don't bundle or vendor them, so they stay current at their source.

## Layout

```
.claude-plugin/
  plugin.json                (plugin manifest - what the plugin contains)
  marketplace.json           (marketplace catalog - what /plugin marketplace add reads)
.out-of-scope/               (deliberate-boundary notes for rejected feature requests)
.gitignore
CLAUDE.md                    (repo conventions for contributors)
LICENSE                      (MIT)
README.md
docs/adr/                    (architecture decision records for the repo's own design)
scripts/
  list-skills.sh             (enumerate skills in this repo)
  link-skills.sh             (symlink skills into ~/.claude/skills for local dev)
skills/
  deslop/            SKILL.md + GLOSSARY.md   (shared language - read first)
  setup-deslop/      SKILL.md
                     references/MAPPING-PATTERN.md   (5-question derivation + archetype examples)
                     references/STACK-TEMPLATE.md    (the blank STACK.md the interview fills)
  map-the-surface/   SKILL.md
  plug-the-holes/    SKILL.md
                     references/CHECKLIST.md         (exhaustive, stack-agnostic)
                     references/FINDING-FORMAT.md
  launch-gate/       SKILL.md
```

`STACK.md`, `SURFACE.md`, `FINDINGS.md`, and `GATE.md` are *per-project artifacts* the skills write into the repo being audited, not part of this bundle.

## Install & update

### Quickstart (30-second setup)

1. Run the installer:

   ```bash
   npx skills@latest add Rendr-Web/rendr-forge-skills
   ```

2. When prompted, **pick the skills to install** (the safe default is all 5) and **pick the agents to install them into** (Claude Code, Codex, Cursor, Gemini CLI, Warp, …). Make sure your agent is checked — if it's not in the list, it wasn't auto-detected; pass it explicitly with `-a <agent>`, e.g. `npx skills add Rendr-Web/rendr-forge-skills -a claude-code`.

3. Choose **Symlink** when asked for the install method. Each agent then points at one canonical copy, so updates are one step.

4. Restart your agent so the skill list reloads. You should see `/deslop` available.

Then just run `/deslop` and it walks the rest of the pipeline.

### Updates

See what's outdated:

```bash
npx skills check
```

Pull the latest:

```bash
npx skills update
```

Release process is just: edit the skills, commit, push. No version bump, no publish. Commit `skills-lock.json` in consuming projects to pin/share exact versions across a team.

### Alternative: Claude Code plugin marketplace

For Claude Code users who'd rather use the native plugin path (no third-party CLI, skills auto-namespaced as `/rendr-forge-skills:deslop`):

```bash
/plugin marketplace add Rendr-Web/rendr-forge-skills
/plugin install rendr-forge-skills@rendr-forge
```

Update later with `/plugin marketplace update rendr-forge`.

## Credits

Workflow and house style inspired by `mattpocock/skills`.
