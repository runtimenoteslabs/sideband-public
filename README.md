# sideband

sideband controls monitors on Windows from the taskbar or the command line. It reads and
changes the settings stored in the monitor itself: brightness, contrast, input source, colour
gains, volume and power.

It uses DDC/CI, a control channel carried alongside the picture on HDMI, DisplayPort and DVI
cables. Any monitor that supports DDC/CI works. sideband does not use a table of supported
models, so a monitor the manufacturer's software has dropped still works here.

Laptop screens do not expose DDC/CI. sideband changes their brightness through the Windows
display interface instead.

sideband also arranges windows. Zones divide a monitor into rectangles. Drag a window over one
and it resizes to fit. Fourteen layouts are included, and you can draw your own.

## Requirements

- Windows 10 or 11, 64-bit.
- A monitor with DDC/CI enabled in its on-screen menu.
- No administrator rights.

Laptop screens are reached a different way and expose brightness only.

## Install

Download `sideband.exe` from the [releases page](https://github.com/runtimenoteslabs/sideband-public/releases).
Run it to start sideband. Exit it to stop it. Delete it to remove it. There is no installer.

To run it by name from a terminal, put it in a directory on your `PATH`.

Releases are not code-signed yet. Windows shows a warning the first time you run the file.
[SECURITY.md](SECURITY.md) describes the warning and how to check the download against its
published SHA-256.

Settings are stored under `%APPDATA%\sideband`. Turning on **Start with Windows** creates one
registry entry. sideband installs no service.

## The tray

Run `sideband.exe`, or run `sideband tray` from a shell. An icon appears in the notification
area.

Left-click the icon for a brightness slider on each display. Right-click it for the full set
of controls each display reports. Where the MCCS standard defines a name, sideband uses it.
Values without a standard name are shown as their code.

`Ctrl+Alt+Up` and `Ctrl+Alt+Down` change brightness on every display without opening the menu.

The menu reads the values again when it opens. If you changed brightness using the monitor
buttons or another program, sideband shows the new value.

Input and power sit at the bottom of the menu and ask for confirmation. Changing either can
make the display unavailable.

**Start with Windows** creates one per-user startup entry. Task Manager lists it under Startup
Apps. Turning the option off removes the entry.

## Window layouts

Turn on **Snap windows while dragging** in the tray menu. Drag a window and sideband shows the
available zones over the screen. The zone under the cursor is highlighted. Release the window
and it resizes to fit. Drag towards the top of the screen to pick a different layout during
the drag.

Fourteen layouts are included: columns, rows, 2x2 and 3x3 grids, 2:1 and 3:1 splits, priority
grids and overlapping zones.

Choose **Edit zones** to draw your own over the monitor. Click a zone and press `V` or `H` to
split it. Drag an edge to resize it. Drag empty space to add a zone. Right-click a zone to
remove it. Give the layout a name and it appears in the menu and in the drag strip.

Zone positions are stored as proportions of the display rather than fixed pixels, so a layout
works at any resolution. Layouts and preferences are stored under `%APPDATA%\sideband`.

Windows has a layout picker of its own that appears when you drag to the top of the screen.
The two collide. Turn it off under Settings > System > Multitasking.

### Arranging open windows

**Arrange windows now** puts the windows already open on that monitor into the current layout.
Their positions are recorded first, so **Undo arrange** can put them back.

The same thing from the command line:

```
sideband arrange columns:3        three columns on this monitor
sideband arrange ratio:2:1        two thirds and a third
sideband arrange grid:2:2         two by two
sideband arrange undo             put them back
```

Layouts are `columns:<n>`, `rows:<n>`, `grid:<c>:<r>`, `grid:<n>`, `ratio:<a>:<b>`,
`priority:<n>` and `focus:<n>`. Add a monitor number to arrange one screen:
`arrange rows:2 2`.

Windows does not allow a normal process to move some windows belonging to applications running
as administrator. sideband does not request elevation. It leaves those windows where they are
and names them.

## Commands

| Command | Description |
|---|---|
| `sideband list` | List every display that responds. |
| `sideband get [display] [control]` | Read current values. Omit `control` to read all of them. |
| `sideband set <display> <control> <value>` | Write a value. |
| `sideband caps <display>` | Print what the display reports about itself. |
| `sideband arrange <layout>` | Move windows into a zone layout. |
| `sideband help` | Print usage. |

Name a display by index (`1`), or by any part of its name, maker or id (`aaa`, `mon`, `2400`,
`internal`). Use `all`, or omit the argument on `get`, to address every display.

```
> sideband list
1. AAA MON-2400             external  AAA-MON-2400-SERIAL0001
2. BBB PANEL-15             built-in  internal

> sideband get 1
[AAA MON-2400]
    brightness   73  (0-100)
    contrast     75  (0-100)
    input        0x19
    volume       50  (0-100)
    power        on
    inputs       0x19, dp1, hdmi1, hdmi2

> sideband set 1 brightness 60
AAA MON-2400: brightness -> 60
```

```sh
sideband set all brightness 40        # every display at once
sideband set mon-2400 input hdmi1     # name a display instead of numbering it
sideband set 1 0x62 30                # write a register directly
```

The exit code is `0` on success and `1` on failure. The same monitor controls are available
from scripts, Task Scheduler, Stream Deck and AutoHotkey.

## Controls

| Name | VCP code | Type | Notes |
|---|---|---|---|
| `brightness` | `0x10` | continuous | Backlight level. |
| `contrast` | `0x12` | continuous | |
| `preset` | `0x14` | enumerated | Colour temperature preset. |
| `red`, `green`, `blue` | `0x16`, `0x18`, `0x1A` | continuous | Individual colour gains. |
| `input` | `0x60` | enumerated | Active input source. |
| `volume` | `0x62` | continuous | Speaker volume. |
| `mode` | `0xDC` | enumerated | Preset picture mode. |
| `osd-language` | `0xCC` | enumerated | |
| `power` | `0xD6` | enumerated | DPMS power state. |
| `technology` | `0xB6` | read-only | Panel technology. |
| `controller` | `0xC8` | read-only | Controller ID. |
| `firmware` | `0xC9` | read-only | Firmware revision. |

Continuous controls take a number. The monitor supplies the maximum. Enumerated controls take
a name, such as `hdmi1` or `standby` or a raw value.

Named input sources run from `vga1` through `hdmi2`. Codes above `0x12` are not in the MCCS
table. Manufacturers use that range for USB-C and Thunderbolt, so sideband prints them as hex.

A laptop screen has `brightness` and nothing else. sideband refuses the other controls instead
of attempting them.

## Display names

Use the id from `list` when you save a display reference in a script. It takes one of four
forms. The first two are durable:

| Form | Example | Stability |
|---|---|---|
| EDID | `AAA-MON-2400-SERIAL0001` | Survives reboots, cable swaps and port changes. |
| Internal | `internal` | The laptop screen. A machine has one, so it cannot be confused. |
| Model | `model:MON-2400` | Stable per model. Two displays of the same model share it. |
| Index | `index:2` | Enumeration order only. Changes when displays are replugged. |

`list` prints a note when any display falls back to the last two forms.

The maker comes from the display's EDID record. sideband shows a name where it recognises the
three-letter manufacturer code, and the code itself where it does not.

A laptop screen reports the part number of the panel, which means nothing to most people, so
`list` marks it `built-in`. It also answers to `internal`.

## Safety

Two kinds of write can leave a display unusable. sideband restricts both.

**Input and power.** Changing either can leave you with no picture and no way back except the
buttons on the monitor. Both ask for confirmation, with No selected by default. Neither is
read back afterwards, because the monitor may no longer be there to answer.

**Manufacturer registers `0xE0`–`0xFF`.** Vendor-specific registers can mean different things
on different monitors. The register that selects USB upstream on one can drive a factory
calibration mode on another. sideband does not name them or expose them as ordinary controls.
Pass `--unsafe` to address one by register number.

Every other write is checked three ways. sideband checks the value against the list the
monitor advertises. It bounds continuous controls by the reported maximum. It reads the value
back afterwards and keeps what the monitor returned.

## Troubleshooting

**`list` reports no displays.** Many monitors have a DDC/CI option in their own on-screen
menu, and ship with it off. Look under `Others`, `Setup` or `Management`.

**A display works when connected directly but not through a dock, KVM or MST hub.** Some
docks, KVM switches and MST hubs pass video but not DDC/CI. No software can work around it.

**`get` shows fewer controls than `caps` advertises.** The monitor advertises codes its
firmware does not implement. sideband skips controls that return nothing.

**A change is accepted but nothing happens on screen.** Some monitors acknowledge a DDC/CI
write without applying it. sideband reads ordinary controls back after writing them, so it
reports the value that came back. Some firmware rejects values inside its own reported range.
One monitor tested during development reports a maximum of 100 for brightness and ignores
every value above 75.

Check these, in order:

- **The monitor's own menu.** Eco modes, dynamic contrast and some picture presets cap or
  lock brightness. This is the usual cause on an external monitor.
- **HDR.** With HDR on, many monitors hand brightness to the graphics driver and stop
  answering their own control.
- **Adaptive brightness and battery saver**, on a laptop screen. Both write the same control
  sideband writes, so either can undo a change a moment later. They are under Settings >
  System > Display and Settings > System > Power.

Night Light is not one of these. It filters colour in the graphics pipeline and never touches
the monitor's brightness control. The screen looks warmer and the reported value does not
change.

**A laptop screen and an external monitor disagree.** The Windows brightness slider writes the
same control sideband writes, so a change in either place shows up in the other. It does not
reach external monitors. A laptop at 50 and an external monitor at 40 is normal.

**The laptop screen is missing from `list`.** Its display driver answered neither route
sideband uses. Where the screen does appear, `sideband caps internal` names the route that
worked. That is the useful detail in a bug report.

## Source

The source is not published. Releases are built on Windows by a GitHub Actions workflow and
published to the releases page with their SHA-256 checksums.

## Further reading

- [Architecture](docs/DESIGN.md) covers the control channel, the shape of the code and the
  monitor behaviour the code is written to survive.
- [Security](SECURITY.md) lists every path sideband writes, how to check a download and how
  to report a vulnerability. sideband has no account, analytics, telemetry or update check.
  It makes no network requests.
- [VESA standards](https://vesa.org/vesa-standards/) publishes MCCS, the standard that says
  what each control code means.

## Licence

MIT. Copyright (c) 2026 RuntimeNotes Labs.
