# Not-started specs

Specs with **nothing built**. These are proposals and plans: the reasoning is
worth keeping, but none of it describes code that exists.

Read them for design intent, not as documentation. Every cell reference, file
path and function name in here is a *proposal* — do not go looking for it in
the repos.

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Proposes | Blocked on |
| --- | --- | --- |
| [Standard Tournament Master](standard-tournament-master-spec.md) | A generator and master workbook for open-entry tournaments — the dual-meet generator's counterpart | Nothing. Needs the master workbook built first (§14) |
| [Fast data delivery](fast-data-delivery-spec.md) | Cloudflare R2 in the live data path plus pointer polling, cutting edit-to-visible from 40–60s to ~5–7s | Nothing technical. A Cloudflare account and a custom domain (§9.4) |
| [Fast data delivery — explainer](fast-data-delivery-explainer.md) | The same, in plain language. No instructions to implement | — |
| [Bracket Generator workbook handoff](bracket-generator-workbook-handoff-spec.md) | A SAGE menu route from a scoring workbook into the bracket tool, and a shuffled-codes output for the roster scaffold's STEP 3 | Was deferred until a standard tournament existed. [Standard Tournament Master](standard-tournament-master-spec.md) §15 now supplies the second case it was waiting for |
