# Per-vendor stack mapping files

The repo will not ship a mapping file for every framework, auth provider, or payment processor. Mapping a codebase to the audit is done by `/setup-deslop` — five derivation questions plus three archetypes (see `skills/setup-deslop/references/MAPPING-PATTERN.md`) — which writes a per-project `STACK.md`.

## Why this is out of scope

Enumerating vendors is combinatorial and goes stale the day a new tool ships. The agent running the skill already knows current tools and can derive a mapping from the archetypes on demand, so a maintained per-vendor catalogue would be high-effort, perpetually outdated, and redundant. See `docs/adr/0002-stack-specifics-in-per-project-stack.md`.

## Prior requests

_None yet — this records a deliberate boundary, not a response to a request._
