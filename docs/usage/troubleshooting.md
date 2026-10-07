# 9. When something's wrong

**Who:** operator · **When:** any time, but mostly on the day · **You need:** Control
Center signed in, and the workbook open

Find the symptom, read the cause, do the fix. For a message shown *inside* a workbook,
**SAGE → Help** explains each one and lists this workbook's current status and the
version of each SAGE script.

> **Check these two first.** Most trouble is one of them.
>
> 1. **Is the workbook's General access still *Anyone with the link: Viewer*?** The sync
>    reads it with an API key, which needs that.
> 2. **Does Mission Control show a fresh *Synced* for the venue?** If not, click
>    **Resync this day now**.

## The data isn't reaching the site

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| A venue reads stale in **Facility sync status**, or **Sync failed** | The workbook's General access changed; the workbook was edited in a way that broke a tab; the live sync is not set up | Check **General access** first. Click **Resync this day now**. In the workbook run **SAGE → Sync now**: the words after **Sync failed** say why, and **SAGE → Help** explains each one. |
| A score or court entry hasn't appeared within about 2 minutes | A sync lost a race with another venue's sync, or another event's on the same day. Facility sync status only turns amber after 5 minutes. | Click **Resync this day now**. |
| **Live updates** reads *polling GitHub (push not connected)* | The live push service isn't connected to this page | Updates still arrive, 30–60 seconds slower. Keep going and tell whoever maintains the system. To stop relying on it, switch **Sync method** to **GitHub only**, and back when healthy. |
| Resync fails for one venue | Sheets API trouble | Tick **Use CSV export fallback** and resync again. |
| **Check connection** says live push is off | The sync method was switched, or the service is unavailable | Mission Control → **Sync method**. |
| The Hub loads but is empty | A column name in `CSV` or `STANDINGSCSV` changed. The names are exact and case-sensitive. | Matches tab: `matchNumber`, `teamCode1`, `team1Player1`, `team1Player2`, `teamCode2`, `team2Player1`, `team2Player2`, `Schedule`, `team1Score`, `team2Score`, `CourtAssignment`, `court`. Standings tab: `teamCode`, `player1`, `player2`, `wins`, `loss`, `quotient`, `bracket`. Without `court`, every court sits on "No match playing". |
| **Set up live sync** won't accept the day key or venue | The registration and the workbook disagree. Venue names are case-sensitive. A new day key takes about a minute to be live. | Fix whichever is wrong, wait a minute, and run setup again. |

## The site shows something wrong

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| A bad score is already public | Typo in the sheet | **Public site status → Force hidden**, correct the sheet, resync, set it back to **Auto**. Control Center keeps showing everything throughout. |
| A warning banner names an unmapped code | A team code's division or event prefix isn't in the registration's labels | Correct the code in the sheet, or ask the admin to add the label to `display`. |
| Playoff matches read **TBD** | The qualifier draw or the seed cells haven't been filled | See [Playoff hand-offs](run-the-day.md#playoff-hand-offs). |
| Names show as codes (`ND_1`) | The roster and the codes haven't met | Check `STEP 1` and `STEP 3` on the category tab. |
| *Lineup not set* (team) | The matchup's lineup isn't on `MatchUps` yet, or `MatchUps` isn't ticked in **Live sync settings** | Enter the lineup, and tick `MatchUps` in **SAGE → Live sync settings**. |
| The Hub hides scores and standings | The day isn't live yet | Public site status is **Auto**, live about 4 hours before the first match. Click **Force live** to show it sooner. |

## A generator refuses to run

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| It lists problems and stops | The plan is the wrong format, clubs are uneven, a court label is missing or repeated, no category is ticked, a playoff is too deep, a bracket has over 13 pairs | Fix the plan, make a **fresh copy** of the master, and run again. It checks everything before writing anything. |
| A tab already exists, or the workbook was already generated | Each generator runs once per workbook | Make a **fresh copy** of the master. |
| A run stops partway (the bar turns red) | A failure part way through | The tabs it finished stay, and the copy can't be generated again. Make a fresh copy. |
| **Fill match numbers** writes nothing | A slot has a team on one side only, or a code is on a slot's second row | Fix `SCHEDULE` and run it again. |
| **Import bracket draws** refuses a file | Wrong number of pairs, brackets or bracket sizes; a tab already has names | The message names the file and both shapes. Redraw, or tick **Replace existing names**. |
| **Shuffle roster codes** refuses a tab | The tab has no roster | Paste the names into `STEP 1` first. |

## Score entry

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| A save is refused with a message naming the service account | The workbook isn't shared with it | Share the workbook with `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` as **Editor**, or move it into the event's folder under **1. TOURNAMENTS**. |
| *Sign-in expired* | The operator's 12-hour sign-in ended | Sign in on Mission Control and save again. |
| The dialog says the **sheet changed** | Someone typed into the sheet, or another operator or scorer saved | Choose **Keep the sheet's score** or **Replace with yours**. |
| *Publishing failed* | The score reached the sheet but couldn't be published | Click **Resync this day now**. |
| Nothing in Match Finder is clickable | Signed out, or the event has no `scoreEntry` | Sign in. If it still isn't, the admin must add `scoreEntry`. |
| The dialog is slow | A big workbook | Wait. Saving again is safe, because a score already in the sheet isn't written twice. |

## Scorer links and desk links

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| *Ask the operator for a scorer link.* | No link on this device, or it isn't a scorer link | Send the link again. |
| *This scorer link has expired.* | It ran past its 24 hours | Issue a new link and send it. |
| *Scorer links are stopped for this event.* | The operator set the switch to **Stopped** | Set **Scorer links** back to **Accepting**. Scorers reload. |
| **Issue scorer link** is off | The event has no scorer page, or the switch is **Stopped** | The admin adds `scorer.html`. Or set **Accepting**. |
| *Ask the operator for a new desk link.* | The event was switched to `"console"`, or the link's day ended | Issue a new desk link. |
| A desk link opens a missing page | The event has no `attendance.html` | The admin adds it. |

## Attendance

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| A switch flips back and a message says it was not saved | A dropped connection, or the workbook isn't shared with the service account | Try again. If the message names the account, share the workbook with it. |
| *No roster yet* | The roster hasn't been built | Press **Update roster**. |
| *The ATTENDANCE tab's header row is not …* | Someone changed the first row of the tab | Put it back, or delete the tab and press **Update roster**. |
| A name appears twice | Two spellings of one person | Fix the spelling on the category tab. See **Needs attention**. |

---
**Features:** [Control Center](../features/control-center.md) · [Score entry](../features/score-entry.md) ·
[Event attendance](../features/event-attendance.md) · [The scoring workbook](../features/scoring-workbook.md)
