# Not-started specs

Specs with **nothing running**. These are proposals and plans: the reasoning
is worth keeping, but none of it describes behaviour any workbook or site has
yet.

Read them for design intent, not as documentation. Unless a spec's own status
block says otherwise, every cell reference, file path and function name in
here is a *proposal* — do not go looking for it in the repos.

One caveat: a spec can be code-complete and still live here, because Apps
Script ships by pasting into a master rather than deploying. Check the status
block at the top of each.

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Proposes | Blocked on |
| --- | --- | --- |
| [Bracket draw name import](bracket-draw-name-import-spec.md) | Uploading the Bracket Generator's exported text files into a generated workbook to fill each category's rosters, and its `STEP 3` codes, from the draw. Plus §11's menu route into the tool and §12's dual-meet roster shuffle | **Code written and green in the harnesses**, but not pasted into either master and never run against a real workbook (§9.9 step 3) |
| [Fast data delivery](fast-data-delivery-spec.md) | Cloudflare R2 in the live data path plus pointer polling, cutting edit-to-visible from 40–60s to ~5–7s | Nothing technical. A Cloudflare account and a custom domain (§9.4) |
| [Fast data delivery — explainer](fast-data-delivery-explainer.md) | The same, in plain language. No instructions to implement | — |
