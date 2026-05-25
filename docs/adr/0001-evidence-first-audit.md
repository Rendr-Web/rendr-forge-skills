# The audit proves findings with exploit tests, not by interviewing the user

`plug-the-holes` confirms every Hole with a re-runnable exploit test rather than asking the user what might be wrong. Ground truth for "is this exploitable" lives in the running system, not the user's head; the user genuinely cannot tell you whether tenant isolation holds, only a test can. Interviewing is concentrated in `setup-deslop`, where the unknowns (the stack, what a tenant is) really do live with the user.

This is a deliberate divergence from interview-first skills. A reader expecting a grilling session in the audit core should know it was a deliberate choice.
