# After the event

**Who:** coordinator, with the developer · **When:** the days after the last day · **You need:**
the final results on the Hub and the event's workbooks

## Final results

1. Make sure every score is in and **Awards** shows every category's podium. Export the
   images from the **Awards** tab ([step 8](run-the-day.md#end-of-day)) if you haven't.
2. Check **Standings** one more time against the workbooks.
3. Nothing needs turning off. The Hub **stays up** at the address the QR code points to,
   showing the final scores, standings and match lists.

**Keep the event's entry in `events.json`.** Every page of the event reads it. Removing a
finished event's entry blanks its Hub, schedule board, scorer page and desk page.

## Turn off live push for the event's pages

!!! note "Developer task — needs GitHub access to the site repo"
    Set `LIVE_BASE_URL = ''` in the settings script of the event's `index.html`,
    `schedule.html` and `scorer.html` (if it has one), and change nothing else. Nothing is
    published for the event any more, so an open connection would only cost requests
    against a daily cap. With the constant empty, the pages read their snapshot from GitHub
    and show the final results. The mechanics are in
    [Adding a new event § After the event](../technical/adding-a-new-event.md#after-the-event).

Check afterwards that the Hub and schedule board still load and still show the final
results. Later, the developer can archive the event's folder, which is a separate step.

## What to keep

- The event's **workbooks**, in the event's folder under **1. TOURNAMENTS**.
- The **draw text files** and images, in the folder's `BRACKETS` folder. The text file
  carries the seed and every pair's fingerprint, so the draw can still be re-checked.
- The **schedule PDF**.
- A note of anything that went wrong on the day and the rough time. `event-data`'s git
  history has a timestamped commit for every sync and override.

## Check it worked

- [ ] The Hub, schedule board and Awards still show the final results.
- [ ] The workbooks, draw files and schedule PDF are filed in the event's folder.
- [ ] The developer has turned off live push for the event's pages.

**Back to** [the Usage guide](README.md).
