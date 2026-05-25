# Method vocabulary lives in the skill's GLOSSARY; domain vocabulary lives in the project's CONTEXT.md

The skill's `GLOSSARY.md` defines *method* terms (Hole, Surface, exploit test): project-independent, shipped with the skill, identical on every codebase. A project's `CONTEXT.md` defines *domain* terms (Order, Shipment): project-specific, living in the audited repo, shared with the architecture/grilling skills used in Phase 2.

They aren't two sources of truth for the same thing, so there's no drift. One describes how the audit works, the other describes what the app is about, and findings compose them ("the Order export path is a tenant-isolation Hole"). `deslop` reads `CONTEXT.md` when present and contributes durable coined terms back into it, rather than inventing private synonyms, so the audit and Phase 2 share one domain vocabulary. Recorded because "why two glossaries?" is the obvious question a contributor will ask.
