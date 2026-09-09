# Implemented specs

Specs whose feature is **built and in use**. What each one describes exists
today; where the built thing departs from the text, the spec's own
*Divergences* or *As built* section records it — read that before trusting a
cell reference or a line number.

These are the specs to read when you need to understand something that is
already running. For what a tool does rather than why it is shaped that way,
start at [Features & Usage](../../features/README.md) or
[Technical](../../technical/README.md).

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Feature |
| --- | --- |
| [Runtime-fetched sync config](sync-config-runtime-spec.md) | `SyncConfigStore` fetching `event-data/config/events.json` at runtime |
| [Sync script configuration](sync-script-configuration-spec.md) | `sheets-sync.gs`'s **SAGE → Set up live sync** and Script Properties |
| [Match Control console](match-control-console-spec.md) | `tools/control-center.html` |
| [Awards tab](awards-podium-tab-spec.md) | The console's podium tab and PNG export |
| [Schedule screen](schedule-screen-spec.md) | `events/<key>/schedule.html`, since backported into both templates |
| [Scoresheet event picker](scoresheet-event-picker-spec.md) | `tools/scoresheet-generator.html`'s snapshot picker |
| [Calculator dual-meet fixes](calculator-dual-meet-spec.md) | "Pairs per club" and the dual bracket default |
| [Calculator PWA](calculator-pwa-spec.md) | `tools/sw.js` + `tournament-calculator.webmanifest` |
| [Bracket Generator](bracket-generator-spec.md) | `tools/bracket-generator.html` |
| [Sheet generator (Phase 1)](dual-meet-sheet-generator-spec.md) | `sheet-generator.gs` — category tabs, `Variables`, `Title`, `Reference for Players` |
| [Schedule generator (Phase 2)](dual-meet-schedule-generator-spec.md) | Its `SCHEDULE` tab |
| [Readouts generator (Phase 3)](dual-meet-readouts-generator-spec.md) | Its `Court Control`, `Timeline`, `CSV`, `STANDINGSCSV` |
| [Event site templates](event-templates-spec.md) | `_templates/dual-meet-template/` and `_templates/standard-tournament-template/` |
| [PNF × BUP dual meet](pnf-x-bup-dual-meet-spec.md) | The first real run of the dual-meet template |
