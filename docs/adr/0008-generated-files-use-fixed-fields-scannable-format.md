# Generated files use a fixed-fields, table-driven format, not prose

`FINDINGS.md` / `STACK.md` / `SURFACE.md` / `GATE.md` are all written in a scannable house style: glanceable table at the top, fixed-fields blocks per entry (Title / What / Why-it-ships-you-broken / Evidence / Acceptance / Blocked-by for Holes; equivalents for the others), no narrative paragraphs. The shape of a Hole block mirrors matt's `/to-issues` template field-for-field so each entry is mechanically convertible into an AFK ticket with no reformatting.

Trade-off: less room for nuanced prose, sometimes feels stilted to write. Accepted because the first run produced files that were "lobotomous to read" — humans couldn't sift gold from dirt, and `/to-prd` / `/to-issues` couldn't slice them cleanly. Predictable structure beats expressive prose for documents that exist to drive downstream action.
