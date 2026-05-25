# Stack specifics live in a per-project STACK.md, never hardcoded in the skills

The skills are stack-agnostic. What each audit category *means* in a given codebase is captured in a per-project `STACK.md` that `/setup-deslop` generates from five derivation questions, rather than being baked into the skills.

Trade-off: a generic engine plus a thin generated overlay is less immediately powerful than hardcoding expert mappings for one stack, but it survives vendor swaps (auth / payments / framework) and lets maintainers keep their hand-tuned overlay private. Reject requests to add per-vendor mapping files to the skills (see `.out-of-scope/enumerate-every-stack.md`).
