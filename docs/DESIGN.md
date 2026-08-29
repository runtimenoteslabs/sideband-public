# sideband architecture

**Status:** current · **Last updated:** 2026-08-30 · **Owner:** RuntimeNotes Labs

How sideband is put together. It is written for someone using it seriously, or deciding
whether to install it. The source is not published, so this document is the account of what
the program does and why.

For commands and options, see the [README](../README.md).

---

## 1. What sideband does

sideband reads and changes the settings stored inside a monitor: brightness, contrast, input
source, colour gains, volume and power. Those settings live in the monitor's own firmware.
There are two ways to reach them: the buttons on the bezel, and a control channel called
DDC/CI. sideband uses the channel.

It also changes the brightness of a laptop screen. A laptop screen has no DDC/CI channel and
is reached a different way. Section 5 covers it.

sideband holds no list of supported models. A monitor is controllable when it answers the
standard. That is why a monitor the manufacturer's software has dropped still works here.

sideband arranges windows too. Zones divide a monitor. A window dropped into one takes its
shape. Layouts are drawn by hand. Section 7 covers it.

Two things are out of scope. sideband does not calibrate colour, which needs separate
instruments and separate expertise. It makes no network requests.

---

## 2. DDC/CI

Four properties of the control channel shape everything else in the tool.

**It is an I²C bus on the video cable.** DDC/CI runs over two pins that every DisplayPort,
HDMI and DVI connector already carries. It runs at roughly 100 kbit/s. It is slow, it is
shared and it has no useful flow control. A monitor asked too quickly answers with nonsense,
or stops answering until it is unplugged.

**Controls are numbered, not named.** The VESA Monitor Control Command Set (MCCS) gives each
control a one-byte VCP code. `0x10` is brightness, `0x60` is input source, `0xD6` is power
mode. Codes from `0xE0` to `0xFF` are reserved for manufacturers, who use them for whatever
they like. sideband carries a catalogue mapping the standard codes to names.

**Controls come in two kinds.** A continuous control takes any value from zero to a maximum
the monitor reports: brightness, contrast, volume. An enumerated control takes one of a fixed
set of values: input source, power state. The difference decides how a written value is
checked. It is also why a range means something for brightness and nothing for input source.

**A monitor describes itself.** Ask a monitor for its capabilities and it returns a
parenthesised string. The string names its model and lists the codes it implements.
Enumerated codes carry their permitted values.

```
(prot(monitor)type(lcd)model(MON-2400)vcp(02 10 12 14(01 04 05) 60( 19 0F 11 12))mccs_ver(2.1))
```

That string is the closest thing to an API description a monitor offers. It is also the most
malformed data the tool handles. Section 9 says why.

---

## 3. Architecture

Six crates. The first split is what can be tested without hardware. The second is what a front
end needs against what a front end is. The last is which front end runs.

```
crates/core/     parsing, validation and the control catalogue.
                 No platform API, no external crates. Builds and tests anywhere.
    vcp          MCCS catalogue, value naming, write planning
    capabilities balanced-paren parser for the self-description string
    edid         EDID base-block parser
    identity     display identity and EDID pairing
    zones        zone geometry, layouts and the format they are saved in

crates/win/      Windows FFI. Every call that touches hardware.
    ddc          paced session over one monitor handle
    enumerate    logical monitors to physical monitor handles
    registry     EDID blocks from the device registry
    panel        laptop screen, via SetupAPI and video IOCTLs
    wmi          the fallback for the laptop screen, over COM
    desktop      top-level windows, monitors and the projection in use
    store        layouts, settings and the last arrangement, on disk

crates/display/  discovery, addressing and guarded writes over either transport

crates/cli/      the command surface: arguments, output, exit codes

crates/tray/     the notification-area surface
    tray         notification icon, hidden window, hotkeys, message loop
    menu         the right-click menu
    panel        the brightness flyout, on system trackbars
    worker       the thread that owns the displays
    icon         the tray image, drawn at run time
    startup      the per-user start-with-Windows entry
    snap         drag detection and what a drop lands in
    overlay      the zones drawn over a monitor during a drag
    bar          the layout picker that drops from the top
    editor       drawing zones

crates/sideband/ the `sideband` binary: the entry point, and nothing else
    console      attaching to the caller's console, when there is one
```

The first split is for testability, not tidiness. Almost every subtle failure here is a
parsing or validation failure. All of that logic sits in `core`, which compiles and runs on
any machine with no monitor attached. `crates/win` holds the calls that need real hardware.
It deliberately holds no logic worth testing.

One rule enforces the split: **no type in `core` may name a Windows handle, error code or
API.** That rule keeps the core testable. It is also what would let a second transport be
added underneath without disturbing anything above it.

`crates/display` holds the part of "set brightness to 60" that has nothing to do with a
command line. Find the displays. Work out which one was meant. Decide whether the write is
allowed, bound it, send it and confirm the monitor acted on it. `Display::apply` is the only
way to write a display, so a second front end inherits every guardrail. A front end keeps
argument parsing, output and asking the user questions.

Both front ends prove the split. The tray builds its menu from what each monitor advertised.
It sends every change through `Display::apply`. It asks the same question before an input
change that the command line asks, using the same check to decide when. The tray asks with a
`MessageBox` and the command line asks with a prompt. That difference is the part a front end
owns.

### 3.1 What happens during a command

`sideband set 1 brightness 60` runs through these stages.

1. **Discovery.** `EnumDisplayMonitors` yields logical monitors. Each expands to one or more
   physical monitor handles. Every handle is asked for its capability string. A handle that
   answers neither a capability string nor a brightness read has no usable channel and is
   dropped. That is the ordinary outcome for a laptop screen.

2. **Identity.** EDID blocks are read from the registry and paired against the model name each
   monitor reported. The laptop screen is found separately and claims its own EDID record.
   Section 6 covers both.

3. **Resolution.** The target `1` selects a display by index. A non-numeric argument matches
   display names and ids by substring, so `mon` and `internal` both work. `all` selects
   everything.

4. **Planning.** `Display::apply` turns `brightness` and `60` into a write, using only the
   catalogue and what the monitor advertised. It refuses a read-only control, an unknown
   control, a manufacturer register without `--unsafe` and a value outside the list an
   enumerated control advertised. Nothing has touched the wire yet.

5. **Bounding.** Brightness is continuous, so the only available bound is the maximum the
   monitor reports for that code. That costs one read. It is what stops an out-of-range write
   being swallowed in silence.

6. **Writing.** The transport writes, with pacing and retries.

7. **Verification.** The code is read back and compared. A monitor that acknowledged the
   command and ignored it is reported as a failure.

Every stage that can refuse does so as early as it can. Stages 1 to 4 need no transaction at
all, which matters on a channel this slow.

### 3.2 The tray, and why it owns a thread

The interface is built from the system's own widgets: a shell notification icon, a popup menu
and trackbar controls on a flyout. Dragging, clicking a groove, arrow keys and the light and
dark themes all arrive with them. The alternative was a toolkit. That would have brought a
large dependency tree to a program whose main claim is that it has one dependency.

Every transaction with a monitor is slow by the standards of an interface. Discovery reads a
capability string from each monitor and takes seconds. A single write costs a read, a write
and a read back. On the thread pumping window messages, that is an icon that takes seconds to
appear and a slider that lurches. So the displays live on a thread of their own.

`Display` owns a monitor handle, so it is not `Send`. That settled the design. The displays
are created on the worker and never leave it. The interface holds a snapshot of what they last
reported, and asks for changes over a channel.

Two consequences matter:

- **Writes to the same control supersede each other.** A slider dragged across its groove
  produces a request per pixel. A monitor accepts about six writes a second. The worker keeps
  the newest request and discards what it overtook. The hardware receives the positions it can
  act on.

- **A lock is never held across anything that pumps messages.** `TrackPopupMenu` and
  `MessageBoxW` run message loops of their own. Those loops dispatch queued messages on the
  calling thread. A snapshot lock held across either can be asked for again by the window
  procedure, one frame down the same stack. The lock is not reentrant, so the second attempt
  never returns. The tray freezes with no error anywhere. That is how this was found. The
  interface copies what it needs, drops the guard and only then calls.

### 3.3 One executable, two surfaces

Both front ends ship in one `sideband.exe`. A user downloads one file and puts it in one
place. `crates/cli` and `crates/tray` are libraries. `crates/sideband` links them and decides
which one runs.

The decision costs something. The subsystem field in a PE header is set at link time and
applies to the whole image. Windows gives a console-subsystem process a console before any of
its code runs. That would flash a console window on every double-click, and on every login
through the startup entry. A windows-subsystem process gets no console, so `println!` writes
into nothing until the program finds one.

sideband is linked as windows-subsystem and calls `AttachConsole(ATTACH_PARENT_PROCESS)`. That
adopts the console the calling shell already owns. Three things follow:

- **Whether the call succeeds tells sideband what the user meant.** Explorer, the startup
  entry and Task Scheduler all launch with no console. A shell has one. So `sideband` with no
  arguments prints usage when someone typed it at a prompt, and starts the tray when it was
  double-clicked. `sideband tray` starts the tray from a shell.

- **A standard handle that is already valid is left alone.** A shell fills those in before the
  process starts when output is redirected. Pointing them at the console every time would send
  `sideband list > out.txt` to the screen and leave the file empty.

- **A shell does not wait on a windows-subsystem process.** It prints its next prompt
  immediately, so output arrives underneath it. Redirection, pipes and exit codes are
  unaffected. It is cosmetic, and it is the price of the tray never flashing a console.

---

## 4. Talking to a monitor

One module owns one physical monitor handle and every transaction against it.

Pacing is enforced there, not at call sites. MCCS asks for roughly 40 ms of quiet after a read
and 50 ms after a write. The session records the earliest moment the next transaction may
begin, and sleeps until then. A call site that forgot the delay would produce a monitor
returning garbage, or one that stops answering until it is physically reconnected. The fault
would surface far from the code that caused it.

Failed transactions are retried three times with a widening gap. Transient I²C failures are
ordinary. A code that goes unanswered after three tries is ordinary too. Monitors routinely
refuse codes they advertise, so the read returns nothing and the caller treats that as
information.

A monitor handle has to be released explicitly. `DestroyPhysicalMonitors` runs from `Drop`,
so an early return cannot leak one and leave the monitor unreachable to whatever runs next.

Two quirks are absorbed at this layer. Some firmware echoes a one-byte value into both halves
of the reply word, so `0x19` arrives as `0x1919`. The low byte is authoritative for enumerated
controls. Continuous controls legitimately use the full range, so they are left alone. And a
monitor that publishes no capability string is probed for a small set of codes that are almost
universally implemented: brightness, contrast, input source, volume and power. It is written
off as uncontrollable only after that.

---

## 5. The laptop screen

A laptop screen has no video cable, so it has no DDC/CI channel. None of section 4 applies to
it. Its brightness belongs to the display driver, and there are two ways to ask.

**The video IOCTLs, tried first.** SetupAPI enumerates the monitor device interfaces present.
Each is asked for its supported-brightness table. External monitors refuse that call, so the
first interface that answers is the laptop screen. Reads and writes then go through
`IOCTL_VIDEO_QUERY_DISPLAY_BRIGHTNESS` and `IOCTL_VIDEO_SET_DISPLAY_BRIGHTNESS`. This path is
plain FFI, with no COM apartment and no marshalling.

**WMI, when a driver declines them.** Some drivers answer the monitor interface with
`ERROR_NOT_SUPPORTED`. The fallback reaches the same ACPI methods through the `root\wmi`
namespace. It reads `WmiMonitorBrightness` and invokes `WmiSetBrightness`. `windows-sys`
publishes Win32 functions and no COM interfaces, so the interfaces this needs are declared by
hand, vtable layout included.

Two constraints shape the design. **Neither path may require elevation.** The display adapter
interface would answer these IOCTLs directly, and it goes unused, because opening it without
administrator rights fails. A tool that demands elevation to change brightness does not get
used. **The laptop screen exposes brightness and nothing else**, on a fixed scale of 0 to 100.
Every other control is refused without being attempted.

---

## 6. Identifying a display

An id is what a script, a saved profile or a hotkey refers to later. It has to keep meaning
the same display across reboots and replugging. Where the evidence cannot support that, the id
carries a prefix saying how far it can be trusted.

**Identity is never positional.** Windows exposes no supported mapping from a monitor handle
to an EDID record. Pairing the two lists by enumeration order is the obvious approach, and it
is wrong on any laptop. The laptop screen occupies a slot in one list and not the other. That
shifts every match after it. The result is a monitor labelled with another display's serial
number.

Instead, the model name a monitor reports in its capability string is matched against the name
in each EDID record. A claimed record is removed from the pool, so a second monitor of the
same model degrades and cannot borrow the first one's serial. sideband uses the strongest form
the evidence supports:

```
<manufacturer>-<name>-<serial>   from EDID; survives reboots, cable swaps and port changes
  ↓ no EDID match or no serial in it
model:<model>                    stable per model; two of the same model share it
  ↓ no model name
index:<n>                        position in the enumeration, and nothing more
```

`list` prints a note whenever any display has fallen to one of the last two.

The laptop screen takes a different route. It publishes no capability string, so there is no
model name to pair on. Its driver names the device instance it belongs to, and that instance
is the registry key its EDID record already sits under. The lookup is direct, so the mismatch
above cannot arise. A screen whose EDID carries no serial is identified as `internal`. That is
durable for its own reason: a machine has one laptop screen, and no port order for it to drift
with.

---

## 7. Arranging windows

A zone is a rectangle a window can be given. A layout is a set of zones covering one monitor.
Both are stored as fractions of the monitor's working area, and converted to pixels only when
a window moves. That is what lets a layout drawn on one screen mean the same thing on a screen
of a different size or scaling.

It also puts all the geometry in `sideband-core`, where it is tested with no display attached.
The errors this feature is prone to are arithmetic: a layout whose parts do not quite tile, a
window matched to the zone beside the one it was dropped in, a zone that rounds away to
nothing. A test asserts those directly. Looking at the screen often will not catch them.

### 7.1 What the screen reports is not what you see

Two Windows behaviours decide whether an arranger looks right or looks broken. Neither is
visible until it is tried.

`GetWindowRect` has not returned the visible edges of a window since Windows Vista. A window
carries an invisible resize border outside what is drawn. A window placed flush against a zone
looks inset on three sides. `DwmGetWindowAttribute` reports the bounds a person
would draw around the window. The difference between the two is the padding to add back.

Not every top-level window is a window, either. The desktop, the shell and every suspended
Store application are all enumerated. Suspended Store applications report themselves visible
for as long as they exist. Only the compositor knows they are cloaked. Arranging without that
check moves windows nobody can see, and pushes the real ones out of the layout.

### 7.2 The projection decides how many monitors there are

How many displays answer over DDC/CI and how many monitors the desktop spans are different
questions. A laptop beside an external monitor reports two displays either way. It reports one
monitor or two depending on the Win+P projection: duplicated, extended, internal only,
external only. A layout means nothing except against the projection it was made for, so
anything saved has to record which one that was.

### 7.3 Snapping is asked for, and asks before it acts

Nothing about a drag changes until snapping is turned on. Once it is, zones appear on any
drag. That is how the tools people already use behave. Requiring a modifier as well made the
feature look broken to anyone who had not read the menu. Holding a key can be made the
requirement instead, for anyone who wants ordinary drags left alone.

Windows reports the start and end of a move through an accessibility event hook. That needs no
injection into other processes and no elevation. Nothing is reported in between, so the cursor
is polled on a timer. That is what moves the highlight from zone to zone.

Arranging from the command line is the one destructive case, because it moves every window at
once. Positions are recorded before anything moves, and `arrange undo` puts them back. Windows
are matched on handle and title together. A handle is reused once a window closes, and a title
alone cannot tell two documents of the same name apart.

## 8. Safety model

Two kinds of write can leave a display unusable. Both are restricted.

**Input source and power** can leave the user with no picture and no way back except the
buttons on the monitor. Both ask for confirmation before anything is sent. Both are exempt
from read-back, for the same reason they are dangerous: the monitor may be gone before it can
answer. Success is reported on the acknowledgement alone.

**Manufacturer registers `0xE0`–`0xFF`** are reserved by MCCS for vendor use, and mean
different things on different hardware. A register that selects USB upstream on one monitor
may drive a factory calibration mode on another. These are refused unless `--unsafe` is given.
None of them is ever given a friendly name, because a name is a claim about what a register
does.

Every other write is checked three ways: against the value list the monitor advertised,
against the maximum the monitor reports for continuous controls and against a read-back
afterwards.

sideband requires no administrator rights anywhere and installs no service. It writes one
registry value under `HKEY_CURRENT_USER`, and only when the user turns on **Start with
Windows**.

---

## 9. What monitors do

Each of these was observed on real hardware. They explain code that otherwise reads as
over-careful. Removing one reintroduces the failure it was written for.

1. **Monitors lie about their capabilities, in both directions.** They advertise codes the
   firmware does not implement and implement codes they never advertise. A missing capability
   string degrades to probing common controls.

2. **Capability strings arrive malformed.** They come truncated mid-token, with unbalanced
   parentheses and null padding. Whitespace lands wherever the firmware author put it: one
   display opens a value list as `60( 19 0F 11 12)`, space included. Parsing is a
   balanced-paren scanner that returns partial results. A regular expression mis-associates
   nested value lists, and abandons the whole string when it cannot match.

3. **The channel is slow and intolerant of haste.** Roughly 40 ms after a read and 50 ms after
   a write, enforced at the transport.

4. **Writes are ignored in silence, and the advertised range is not the accepted range.** A
   monitor will acknowledge a command on the wire and do nothing. One display reports a
   maximum of 100 for brightness, accepts every value up to 75 and discards every value above
   it while acknowledging each one. Reading the value back is the only way to tell the
   difference. That is why a write is not reported as successful until it has been.

5. **Enumerated replies echo into both bytes.** `0x19` arrives as `0x1919`.

6. **Input codes above `0x12` are not standardised.** Manufacturers use that range for USB-C
   and Thunderbolt. Those values are printed as raw hex. A label for them would be a guess.

7. **EDID names are not always under descriptor `0xFC`.** A laptop screen was observed
   publishing its name under `0xFE`. A parser reading only `0xFC` and `0xFF` reports an empty
   name, and the display becomes unidentifiable.

8. **The registry's device `Control` subkey needs elevation.** Filtering EDID records on
   whether a device is started discards every record when run as a normal user.
   Stale records are tolerated instead. Pairing on a model name that only a connected monitor
   can report keeps them harmless.

9. **Docks, KVM switches and MST hubs frequently do not pass DDC/CI through.** No software
   fixes this and sideband reports it plainly.

10. **Nothing owns these controls exclusively.** A monitor's own menu, a vendor utility, the
    operating system and the person sitting there all write the same registers. A laptop
    screen is the sharpest case. The Windows brightness slider writes the identical control
    this tool writes, confirmed in both directions on real hardware and Windows also moves it
    for battery saver and adaptive brightness. A value can be correct when it is set and
    different a minute later, with nothing having gone wrong. Anything built on top has to
    read the display, because what it last asked for may no longer be true. That is why the
    tray reads each display as it opens its menu.

---

## 10. Where the seams are

The crate split makes three kinds of change cheap. That is the test of whether it was drawn in
the right places.

**A front end** calls `discover`, `resolve` and `Display::apply` and reaches a display no
other way. What it owns is presentation: how a display is named for a reader, how a refusal is
reported and how a question is asked before a write that can black out a screen. The command
line and the tray are two answers to those questions over one implementation.

**A control** is one entry in the catalogue, carrying its code, name, kind and summary. Value
naming, write validation and `get` output all follow from that entry. A manufacturer-specific
control does not belong in the catalogue at all. A name is a claim about what a register does,
and the raw register syntax already reaches it.

**A transport** implements the read, write and capability calls for a platform and leaves
`core` untouched. DDC/CI is a VESA standard carried over the same I²C lines everywhere. A
Linux backend over `i2c-dev` would reuse every parser, the catalogue, all validation and the
safety model. Only the transport differs.

That portability covers monitor control and stops there. Topology watching and window
management are deeply platform-specific. A second platform would need genuine reimplementation.

**Testing** runs on any platform with `cargo test -p sideband-core`. The tests construct the
EDID blocks and capability strings they need, so each states the single property it is about,
and the crate depends on nothing outside its own tree. Exercising the FFI layer needs a
Windows target and a real display.
