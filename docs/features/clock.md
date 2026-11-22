# Clock

**Clock** is a full-screen wall clock for a spare screen at the venue: a TV by the desk, a tablet on a
stand, a phone propped up at a court. It lives at `/tools/clock`, opens from Control Center's **Pages and
tools** row (with the selected event's name above the time) and from the tools roster, and is linked from
no public page. Anyone with the link can open it; it needs no sign-in and reads no event data.

## What it shows

- **The time**, as large as the screen allows, with the seconds in green. It shows the device's own time,
  so set the device's clock and time zone correctly.
- **The date** under it, in the device's language.
- **A label** above it, when there is one: the event's name when it was opened from Control Center, or
  anything typed in the settings (a court, a venue).

On a wide screen the time reads across one line. On a tall one (a phone held upright) hours, minutes and
seconds stack, so the digits use the height. The digits keep the same size from minute to minute; they
change size only when the window does or a setting changes.

## Controls

A small bar at the bottom holds **Settings**, **Full screen** and **Hide**. It fades out, with the mouse
pointer, after three seconds without a move or a tap, and comes back on the next one.

- **Full screen** fills the whole screen, browser bars and all. Press it again, or **Esc**, to leave.
  Double-clicking the clock does the same. An iPhone's browser cannot go full screen: the button hides the
  bar instead, and **Share → Add to Home Screen** opens the clock from the home screen without the browser's
  bars.
- **Hide** hides the bar at once, for a clean screen without going full screen.
- **Settings**:
    - **12-hour** (with AM/PM) or **24-hour**.
    - **Theme**: **Navy** (the S.A.G.E. navy, the default), **Black** (pure black, kindest to an OLED screen
      left on all day) or **Light**.
    - **Show seconds** and **Show date**.
    - **Keep screen on**: stops the screen from dimming or locking while the clock is showing (on by
      default). A browser that can't do this says so; set the device's own screen timeout instead.
    - **Label above the time**: empty for none.
    - **Reset to defaults**.

Keys: **F** full screen, **H** hide the bar, **S** settings, **Esc** close the settings.

The settings are remembered by this browser, so a screen set up once keeps its look. Opening the clock from
Control Center sets the label to the event's name each time.

## Links that set the clock

The address can set any setting, which suits a bookmark on a device that always shows the clock the same
way:

| Part | Does |
| --- | --- |
| `?title=Court 3` | the label (empty for none) |
| `?h24=1` / `?h24=0` | 24-hour / 12-hour |
| `?seconds=0`, `?date=0` | hides the seconds, the date |
| `?theme=black` | `navy`, `black` or `light` |
| `?wake=0` | lets the screen sleep |

For example `/tools/clock?title=Court%203&theme=black&h24=1`.

How it is built: [Clock (technical)](../technical/clock.md).
