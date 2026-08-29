# Security

## Reporting a vulnerability

Report a suspected vulnerability through [private vulnerability reporting](https://github.com/runtimenoteslabs/sideband-public/security/advisories/new),
or by email to runtimenotes@proton.me. Expect an acknowledgement within seven days. Do not
open a public issue for a vulnerability.

Include the `sideband` version, your Windows version and the steps that reproduce the
problem. If a particular display is involved, include the output of `sideband caps <display>`.

Fixes go into the next release. Only the most recent release is supported.

## Network access

`sideband` makes no network connections. It never checks for updates, so you find new versions
on the releases page rather than through the program.

To confirm this on your own machine, open Resource Monitor, select the **Network** tab and
watch the `sideband.exe` row while you change a setting.

The release binary links two external libraries, both of them bindings to the Windows API:

| Crate | What it provides |
|---|---|
| `windows-sys` | Declarations for the Win32 functions `sideband` calls |
| `windows-link` | Resolution of those functions at load time |

The manifests request no networking feature from either. `sideband-core`, which holds the
parsers and the validation, takes no external dependency at all.

## Privileges

`sideband` runs as a standard user and never requests elevation. Reading and writing a monitor
over DDC/CI needs no special rights, and neither does setting the brightness of a built-in
laptop panel through the display driver.

One thing follows from this. A window belonging to a program running as administrator cannot
be moved by a program that is not, so `arrange` leaves those windows where they are and names
them.

## What sideband writes

| Path | Written when |
|---|---|
| `%APPDATA%\sideband\settings.txt` | You change a setting in the tray menu. |
| `%APPDATA%\sideband\layouts.txt` | You save a zone layout. |
| `%APPDATA%\sideband\last-arrangement.txt` | You run `arrange`, so that `arrange undo` can put the windows back. |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`, value `sideband` | You turn on **Start with Windows**. Turning it off deletes the value. |

Nothing else on the machine is modified. `sideband` installs no service and no driver.

Task Manager's **Startup** tab lists the Run entry alongside every other, so you can remove it
there without going through the tray.

## Checking a download

Each release publishes a `SHA256SUMS` file next to the binary. Compare it against the file you
downloaded:

```powershell
Get-FileHash sideband.exe -Algorithm SHA256
```

Releases are built by a GitHub Actions workflow from a tagged commit and published from that
run. The source repository is private; the design document describes what the code does, and
the checksum tells you that you have the file the workflow produced.

## Running an unsigned build

Releases are not code-signed yet. Two things follow, and the second one has no workaround.

**SmartScreen shows a warning.** The first time you run the file, Windows displays **Windows
protected your PC**. Select **More info**, then **Run anyway**. Check the SHA-256 first.

**Smart App Control blocks the file outright.** On a Windows 11 machine with Smart App Control
turned on, an unsigned executable is blocked and there is no per-file exception. Turning Smart
App Control off is permanent: it cannot be turned back on without resetting Windows. Leave it
on and wait for a signed release instead.

## Binary hardening

| Mitigation | State |
|---|---|
| Control Flow Guard | On |
| ASLR, including high-entropy ASLR | On |
| Data Execution Prevention | On |

Control Flow Guard is checked before each release by reading `GuardCFFunctionCount` from the
binary's load configuration. The `CF_INSTRUMENTED` bit in `DllCharacteristics` is not a
sufficient check: prebuilt standard-library objects set it even when the guard table is empty.

## Writing to a display

DDC/CI has no authentication. Any program running as you can send a display the same commands
`sideband` sends, and the display acts on them. What `sideband` adds is restraint about which
commands it will send:

- Manufacturer registers `0xE0` to `0xFF` are refused unless you pass `--unsafe`, and none of
  them is given a friendly name. MCCS reserves them for vendor use, so the same register means
  different things on different hardware.
- Changing input or power prompts for confirmation, because either can leave you with no
  picture and no way back except the monitor's own buttons.
- Every other write is checked against the range or the value list the display advertises, and
  read back afterwards to confirm the display acted on it.

Section 8 of the [design document](docs/DESIGN.md) covers the reasoning.
