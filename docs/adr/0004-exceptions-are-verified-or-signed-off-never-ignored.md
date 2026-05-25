# Declared exceptions are verified or signed off, never silently ignored

The exceptions checkpoint in `plug-the-holes` lets a user flag a Suspected Hole as intentional or compensated. But a claimed mitigation is treated as a *claim that gets its own exploit test*, and an intentional design with no enforceable control becomes an **Accepted Exception** carried to the gate for a named owner's sign-off.

There is deliberately no "ignore this finding" path. The obvious version of the feature ("here are the holes, untick any you don't care about") is a loophole that lets a real breach (a cross-tenant leak waved away as "intentional") vanish without evidence or accountability. Verifying the control, or forcing an on-record sign-off, keeps "demonstrate, don't assert" intact and keeps carried risk visible at the gate.
