# `/plug-the-holes` is audit-only; fixes happen between sessions

`/plug-the-holes` sweeps, builds evidence (runnable exploit tests where feasible, code-inspection where not), and stops. It never edits production code. Fix work moves to a separate session — `/tdd` against the RED exploit tests, or matt's `/to-prd` → `/to-issues` chain feeding AFK agents — so each closed Hole is its own small reviewable PR instead of one catastrophic 18-file structural diff.

Trade-off: shipping fixes spans multiple sessions and (typically) multiple PRs, which is slower for a solo vibe-coder than the old one-shot flow. Accepted because review-blockability is itself a launch blocker; the first real run (PipeInspect, 2026-05-25) produced an unreviewable diff that nobody could safely merge.
