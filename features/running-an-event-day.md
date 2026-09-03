# Running an event, day of

A plain-language walkthrough of what an operator actually does on tournament
day, using [Control Center](control-center.md). This assumes the event is
already set up — categories, days and venues registered, workbooks built and
wired to the sync. That half is [preparing an event](preparing-an-event.md).

## Before doors open

1. Get on the venue's wifi.
2. Open Control Center, select the event and today's day.
3. Sign in, under **Mission Control**.
4. Click **Check connection** — confirms everything's reachable before a
   single match depends on it.
5. Click **Resync this day now** — pulls a fresh copy of the schedule and
   doubles as a check that each venue's spreadsheet is actually wired up
   correctly before play starts.
6. Confirm **Facility sync status** shows a fresh "Synced" for every venue
   this event uses.
7. Open each venue's **Google Sheet** — the link sits right on its row in
   Facility Sync Status. This is where the actual day-of work happens:
   type scores into the matches tab, and mark which match is currently on
   which court in the **Court Control** column. That second part is what
   makes a match show up as live — on Control Center's Live Matches tab,
   the schedule board, and Tournament Hub — so it's worth opening this
   now and keeping it up throughout the day, not just visiting it once.

## Deciding when the public site goes live

Under **Public site status**: by default (`Auto`) the public site goes live
on its own, about 4 hours before the day's earliest scheduled match — no
action needed. If you want it visible earlier, e.g. so players can check
their schedule the moment they arrive, click **Force live**.

## Setting up the venue screens

- Click **Open schedule** to launch the wall display, and put that on
  whatever screen is mounted at the venue.
- Click **Open Tournament Hub** for the event's public page — useful on a
  second screen (showing Standings or Live Matches, whichever fits — the
  schedule board already covers the court-by-court view), or just to
  confirm it looks like what a player would actually see.

## During play

Everything below happens in the Google Sheet you opened in step 7, not in
Control Center — the console is where you *watch* the day, the sheet is
where you *run* it.

The rhythm at each court, as matches happen: mark the next match live on
that court as soon as it frees up (in Court Control), and replace it with
the *following* match number the moment it finishes — never leave a court
blank, since a blank court just shows as idle instead of telling anyone
what's coming up next. Enter the finished match's score right after.

Everything else happens on its own — scores and court status reach the
public site and the wall display within about ten seconds, with no extra
steps.

What to actually watch for:

- **Facility Sync Status**, periodically — it should read "Synced" from a
  few seconds to a couple of minutes ago, continuously. If it ever goes
  stale, click **Resync this day now** yourself rather than waiting.
- **A bad score or typo that needs a quiet fix** — set the public site to
  **Force hidden**, correct it, resync, then set it back. This only hides
  Tournament Hub; Control Center itself keeps showing everything the whole
  time, so you can verify the fix before putting it back in front of
  spectators.
- **A player asking where their match is** — use Match Finder right there in
  Control Center.

## End of day

Once the last Finals and Bronze matches are scored, check the **Awards**
tab — every category should show its medalists (or a warning naming a match
if a score looks off). Use **Export image** per category, or **Export whole
tournament** for one combined image, to hand results to the emcee or post
them.

Nothing needs turning off — once the public site has gone live, it just
stays that way.

---
**See also:** [Control Center](control-center.md)
