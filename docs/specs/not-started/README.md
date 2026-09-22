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
| [Fast data delivery](fast-data-delivery-spec.md) | Cloudflare R2 in the live data path plus pointer polling, cutting edit-to-visible from 40–60s to ~5–7s | Nothing technical. A Cloudflare account and a custom domain (§9.4) |
| [Fast data delivery — explainer](fast-data-delivery-explainer.md) | The same, in plain language. No instructions to implement | — |
