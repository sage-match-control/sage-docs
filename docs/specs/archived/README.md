# Archived specs

Specs that **will not be built**: superseded by another design, or dropped.
None of it describes code that exists. They are kept for their reasoning and
measurements, which later specs still cite.

Read them for background only. Every cell reference, file path and function
name in here is a *proposal* — do not go looking for it in the repos.

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Proposed | Why archived |
| --- | --- | --- |
| [Fast data delivery](fast-data-delivery-spec.md) | Cloudflare R2 in the live data path plus pointer polling, cutting edit-to-visible from ~35–40s to ~5–7s | Superseded by [Live push delivery](../implemented/durable-object-push-spec.md), which was built instead |
| [Fast data delivery — explainer](fast-data-delivery-explainer.md) | The same, in plain language. No instructions to implement | As above |
