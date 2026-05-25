# Evidence may be a runnable test OR a precise code-inspection finding

A Confirmed Hole's `Evidence` field carries exactly one of: `runnable-test: <path>` (preferred; a RED test/curl/script in `.deslop/exploits/`) or `code-inspection: <file:line>` plus reasoning and a required `Why no runnable test` justification. Both block the gate; both must be discharged before SHIP. This refines [ADR 0001](./0001-evidence-first-audit.md) — the spirit ("Hole isn't real until demonstrated") is preserved, but code is accepted as a form of demonstration when the audit context genuinely cannot run a test (action runtime, prod-env value, infra not stood up).

Trade-off: `code-inspection` is more easily abused as a lazy shortcut than a RED test. Mitigated by requiring the explicit `Why no runnable test` field — if it reads as hand-waving, the entry must be downgraded to Suspected and either built or discharged before delivery.
