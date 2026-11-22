# Clock

`tools/clock.html` is one self-contained page: an inline `<style>` and one inline `<script>`, Google Fonts
(Barlow Condensed for the digits, Inter for the controls), and no engine module. It reads no event data and
calls no API. Control Center's **Open clock** tile (`renderScheduleLauncher()`) links to
`/tools/clock?title=<the event's title from events.json>`, and the tools roster links to `/tools/clock`.
How it is used: [Clock (features)](../features/clock.md).

## Settings

`S` is `{ h24, seconds, date, theme, wake, label }`: the defaults, then what this browser saved in
`localStorage` under `sage.clock.v1`, then the URL (`?h24`, `?seconds`, `?date`, `?wake` as booleans,
`0`/`false`/`no`/`off` being false; `?theme` one of `navy`, `black`, `light`; `?title` the label, at most 80
characters). Every change in the settings sheet saves the whole of `S`, so values that came from the URL
are saved with it. Storage that throws (a private window) leaves the defaults and the URL.

The theme is `data-theme` on `<html>`; each theme sets the colour tokens (`--bg`, `--fg`, `--sub`,
`--accent`, `--panel` …) the rules use, and the `theme-color` meta follows it.

## Ticking

`tick()` renders and schedules the next call at the next whole second (`1000 - Date.now() % 1000`, plus
5 ms), so the seconds change with the device clock's and a late timer does not drift. The hours and minutes
are written only when they change. `visibilitychange` back to visible renders at once. The tab's title is
the time.

## Sizing

`fit()` sizes the digits so the widest time the current format can show fits the stage, never the current
one, so the size does not change from minute to minute. A hidden `.time` element at 100px holds
`88:88:88 PM` (without `:88` when seconds are off, without `PM` in 24-hour); its box gives the width and
height per 100px. The stage is the viewport (`100dvh`) less a 16px padding or the safe-area insets.

- **Stacked** when the stage is more than 1.15 times as tall as it is wide: the stage gets `.stack`, the
  digits a column (`88` / `88` / `88`, then AM/PM), and the colons are hidden.
- The font size is the smaller of 94% (96% stacked) of the stage's width over the template's width, and 96%
  of the stage's height over the template's height plus room for the label (0.22 of the size) and the date
  (0.28). The dock floats over the clock and fades when idle, so it takes no room from the digits.
- The date is 0.2 and the label 0.16 of the size; either is then shrunk to fit the stage's width if it is
  wider (a long date on an upright phone).

A `ResizeObserver` on the stage refits on any size change (rotation, full screen, a phone's toolbars), and
`document.fonts.ready` once the font has loaded.

## Full screen, the dock and the wake lock

- **Full screen** calls `requestFullscreen` (or `webkitRequestFullscreen`) on `<html>`, with
  `navigationUI: 'hide'`, and the matching exit; `fullscreenchange` swaps the button's label and icon. Where
  neither exists (an iPhone's Safari) the button hides the dock and says why in a toast. The page carries
  `apple-mobile-web-app-capable`, so Add to Home Screen opens it without Safari's bars.
- **Idle**: three seconds after the last pointer move, tap or key, `body.idle` fades the dock out and hides
  the cursor; the next one removes it. The timer does not run while the settings sheet is open. **Hide** sets
  `idle` at once.
- **Keep screen on** takes a Screen Wake Lock (`navigator.wakeLock.request('screen')`) while the setting is
  on, takes it again when the page becomes visible (a browser releases it when the tab is hidden), and
  releases it when the setting is turned off. Without the API the checkbox is disabled with a note.
