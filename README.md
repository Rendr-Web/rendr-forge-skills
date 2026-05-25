# Rendr Forge Skills

The open tooling behind **Rendr Forge**, Rendr's practice for rescuing vibe-coded apps and getting them safe to ship. The method is called *deslop*: turn a vibe-coded app into something safe to ship, then methodically better.

A small, composable set built in the spirit of [mattpocock/skills](https://github.com/mattpocock/skills). It hands off *to* his skills for the back half of the work rather than reinventing it.

**Opinionated about method, pluggable about stack.** The audit categories are universal; what each one *means* in your codebase lives in a per-project `STACK.md` that `/setup-deslop` generates by interviewing you. Switch your auth provider or payment processor and only `STACK.md` changes; the workflow doesn't.

## The pipeline

```
/setup-deslop        ← run once per repo. Interviews your stack → STACK.md (your house overlay).
/deslop              ← orchestrates the rest.
  ├── /map-the-surface   recon → SURFACE.md
  ├── /plug-the-holes    security/ship-blocker audit → FINDINGS.md   ← the core
  └── /launch-gate       the written go/no-go line → GATE.md
                         then hand off for Phase 2:
                         /improve-codebase-architecture · /diagnose · /tdd
```

## The one idea

Separate the defects that can hurt a customer on launch day (**Holes**) from the ones that only hurt you later (**Hardening**, **Improvement**). Close every Hole, write down the line, ship. Everything else is incremental and never blocks a launch.

Three heuristics carry most of the weight:
- **Demonstrate, don't assert.** A Hole isn't real until a re-runnable exploit test makes the bad thing happen.
- **Money and identity first.** Auth, tenant isolation, and billing before anything else.
- **Scope once, at a seam.** Enforce tenant scoping in one shared place; closing the Hole and removing the slop become the same edit.

## Reference

Run them in order; `/deslop` orchestrates and will call the others.

- **[deslop](./skills/deslop/SKILL.md)** is the orchestrator. It covers the holes-vs-improvement split, the run order, and the hand-off to architecture/diagnosis skills for Phase 2. Start here.
- **[setup-deslop](./skills/setup-deslop/SKILL.md)** runs once per repo. It interviews you about the stack (detecting what it can) and writes `STACK.md`, the overlay the audit reads.
- **[map-the-surface](./skills/map-the-surface/SKILL.md)** does recon: data models, entry points, data stores, tenant model, and money flows into `SURFACE.md`.
- **[plug-the-holes](./skills/plug-the-holes/SKILL.md)** is the core. A security/ship-blocker audit loop that demonstrates every Hole with a re-runnable exploit test, fixes at the seam, and re-verifies. Produces `FINDINGS.md`.
- **[launch-gate](./skills/launch-gate/SKILL.md)** writes the go/no-go line, verifies every Confirmed Hole is Closed, and makes the ship call. Produces `GATE.md`.

## Companion skills (Phase 2)

Deslop stops at the launch gate. The "make it methodically better" half (architecture work, diagnosing hard bugs, test-driven features, growing the project's domain docs) is already well covered by **Matt Pocock's skills**, so the pipeline hands off to them rather than reinventing them.

They're recommended companions for Phase 2. (Phase 1, the audit and the gate, stands alone and needs nothing else.) Install them from the source, which stays current as he adds or renames skills:

→ **https://github.com/mattpocock/skills** · `npx skills add mattpocock/skills`

See his README for the current skill list and install options. We deliberately don't enumerate specific skills or pin install flags here; naming someone else's skills in our repo is how instructions go stale. We also don't bundle or vendor them, so they stay current at their source.

## Layout

```
.claude-plugin/plugin.json   (plugin manifest - enables the marketplace install path)
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

These skills are distributed the way the agent-skills ecosystem works: **GitHub is the registry; there is no npm publish step.** Push this folder to a public repo, and anyone installs it with the `skills` CLI (no global install; `npx` fetches it):

```bash
# install the whole set into your agent (Claude Code, Codex, Cursor, etc.)
npx skills add Rendr-Web/rendr-forge-skills

# or just the audit core
npx skills add Rendr-Web/rendr-forge-skills --skill plug-the-holes --skill setup-deslop

# install into a specific agent
npx skills add Rendr-Web/rendr-forge-skills -a claude-code
```

Choose **Symlink** install when prompted; it points each agent at one canonical copy so updates are one step.

**Updates flow from the repo, not a package.** When you push changes, users pull them with:

```bash
npx skills check     # see what's outdated (tracked via skills-lock.json + git SHAs)
npx skills update    # pull the latest
```

So your release process is just: edit the skills, commit, push. No version bump, no publish. Commit `skills-lock.json` in consuming projects to pin/share exact versions across a team.

(It also works as a Claude Code plugin marketplace, via `/plugin marketplace add Rendr-Web/rendr-forge-skills`, if you'd rather distribute that way.)

## Credits

Workflow and house style inspired by `mattpocock/skills`.
