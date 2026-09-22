# Not-started specs

Specs with **nothing built**. These are proposals and plans: the reasoning is
worth keeping, but none of it describes code that exists.

Read them for design intent, not as documentation. Every cell reference, file
path and function name in here is a *proposal* — do not go looking for it in
the repos.

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Proposes | Blocked on |
| --- | --- | --- |
| [Bracket draw name import](bracket-draw-name-import-spec.md) | Uploading the Bracket Generator's exported text files into a generated workbook to fill each category's rosters, and its `STEP 3` codes, from the draw | Nothing. Needs the Standard Tournament Generator, which is built |
| [Bracket Generator workbook handoff](bracket-generator-workbook-handoff-spec.md) | The outbound half: a SAGE menu route from a scoring workbook into the bracket tool (both formats), and how a dual meet's `STEP 3` gets filled | Nothing for the menu route. The dual-meet output mode is still undecided (§5, §6); the standard tournament's inbound half is [Bracket draw name import](bracket-draw-name-import-spec.md) |
| [Fast data delivery](fast-data-delivery-spec.md) | Cloudflare R2 in the live data path plus pointer polling, cutting edit-to-visible from 40–60s to ~5–7s | Nothing technical. A Cloudflare account and a custom domain (§9.4) |
| [Fast data delivery — explainer](fast-data-delivery-explainer.md) | The same, in plain language. No instructions to implement | — |
