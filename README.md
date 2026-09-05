# Hydra

**A modern, cross-platform software KVM** — share one keyboard and mouse across Mac, Windows, and Linux by moving the cursor to the edge of the screen. A spiritual successor to Synergy and Barrier, with end to end encryption, support for online relays to bridge networks or VPN connections, and sending key input as pre-resolved Unicode characters to eliminate keyboard layout issues.

> [!IMPORTANT]
> This repository is a fork of the original [PacAnimal/hydra](https://github.com/PacAnimal/hydra) project. It intentionally carries fork-specific TUI management, relay diagnostics, remote configuration, and macOS behavior. Releases and support expectations for this fork may differ from upstream.

[![License: GPL v2](https://img.shields.io/badge/License-GPL_v2-blue.svg)](LICENSE)

_Demonstration artwork from the upstream project:_

![Hydra — cursor crossing from Windows to macOS](https://raw.githubusercontent.com/PacAnimal/hydra/assets/hero.gif)

---

## What this fork adds

- A local cross-platform terminal control center for status, logs, diagnostics, lifecycle controls, and safe configuration editing.
- Explicitly paired remote configuration over Hydra's encrypted relay, with redaction, revision checks, and automatic rollback of unconfirmed changes.
- Route and adapter diagnostics for understanding which interface and socket Hydra actually selected.
- Fork-specific macOS keyboard, media, audio, brightness, and service behavior.
- Continued support for upstream Hydra's encrypted relay, network-aware profiles, Unicode-aware keyboard forwarding, and headless Linux input forwarding.
- Optional OS-aware sleep coexistence that closes the relay before suspend and reconnects after wake on macOS, Windows, and systemd Linux.

---

## A day in the life

**The commuting laptop.** Walk into the office, dock your laptop, and Hydra activates your Office profile automatically — cursor flows between screens, files copy across with one hotkey. Unplug at 5pm: the dock-detected profile drops. Get home and join the home WiFi: Hydra silently switches to your Home profile, where the same laptop now controls a mini-PC plugged into the TV. At a coffee shop with neither network: Hydra idles silently — there's nothing to connect to.

**The Raspberry Pi as a wireless keyboard.** A headless Pi tucked behind the TV runs Hydra in remote-only mode. Plug any USB keyboard and mouse into it, and they instantly control your Mac across the room — no display server, no Xorg, just evdev and a network cable.

**Typing foreign characters across layouts.** Norwegian master, US slave — type `å` on the master and `å` arrives correctly on the slave, even though the slave's keyboard has no key for it. Hydra resolves characters to Unicode on the master before transmission; dead-key composition (`' + a` → `á`) works the same way. No "force all machines to use the same layout" workarounds needed.

**The shared office screen.** A 98-inch display on the conference room wall runs as a slave. Any of the five people around the table can slide their cursor onto it — whoever gets there first takes control. Put up a diagram, hand off to a colleague, pass it back — no cables, no HDMI switches, no "can you share your screen?" interruptions.

**The VPN problem, solved.** Your work laptop is on the corporate VPN; it can't see your personal machine sitting right next to it on the LAN. Drop a Styx container on a cheap VPS, paste the relay config into both machines' `hydra.conf`, and they connect through the relay as if they were on the same network — end-to-end encrypted, no port forwarding, no changes to the VPN.

---

## Install

This fork does not currently publish downloadable binary releases. Clone this repository and build it with the .NET 10 SDK to get the fork-specific behavior described here; [upstream releases](https://github.com/PacAnimal/hydra/releases) do not contain this fork's additional changes.

```bash
cd hydra
dotnet publish Hydra --configuration Release --runtime osx-arm64 --self-contained
```

Replace `osx-arm64` with the appropriate runtime identifier: `win-x64`, `linux-x64`, or `linux-arm64`. The executable is written to `Hydra/bin/Release/net10.0/<rid>/publish/`.

Run it directly first:

```bash
cd Hydra/bin/Release/net10.0/osx-arm64/publish
./Hydra
```

On macOS, `./Hydra --install` registers and starts a per-user LaunchAgent. Grant Accessibility permission under System Settings → Privacy & Security → Accessibility. `./Hydra --uninstall` removes it.

On Windows, run `.\Hydra.exe --install` from an elevated terminal to install the LocalSystem service; use `.\Hydra.exe --uninstall` to remove it. Service mode stays active at login and lock screens.

Linux has no Hydra-owned installer. Run `Hydra` directly or configure your preferred supervisor. Linux desktop use requires X11 with XInput2; Wayland is not supported. Headless Linux uses evdev and is documented under [Remote-only mode](docs/CONFIGURATION.md#headless-linux-no-display-server).

> **Fork update note:** the current in-app self-updater still checks the upstream `PacAnimal/hydra` release feed. Set `"autoUpdate": false` in `hydra.conf` to prevent a source-built fork binary from replacing itself with an upstream release.

### Terminal control center

Open Hydra's local cross-platform TUI in another terminal. The TUI is the fork's only interactive configuration editor:

```bash
./Hydra tui
./Hydra tui --config /path/to/hydra.conf
```

The control center shows the running process, active profile, relay, screens, peers, current routing state, exact relay network interface/socket, peer RTT/jitter, adapter traffic/error counters, embedded-relay peer interfaces, and a bounded live log. It can request a relay reconnect or Hydra restart, and can validate and atomically save `hydra.conf`; accepted actions show live progress in the bottom activity line without blocking refreshes behind a success dialog. The Configuration tab has a sectioned form for common settings and a complete JSON text editor; switching between them preserves advanced fields. Selected tabs and form sections use persistent colour independent of focus, and empty optional fields show their effective default or inherited value. Hovering an option or moving keyboard focus to it displays contextual help. Configuration editing remains available when Hydra is offline. Relay passwords and `networkConfig` values are hidden unless you explicitly reveal them in Text mode.

The Overview tab also provides a confirmed **Shutdown Hydra** action. On macOS, it unloads but preserves the current LaunchAgent so its `KeepAlive` setting does not immediately relaunch Hydra. Windows service-managed sessions must instead be stopped through Windows Services or an elevated terminal.

After the TUI confirms shutdown, **Start Hydra** becomes available. It starts the installed macOS LaunchAgent when available, or launches the current executable directly with the selected configuration. A generic management connection failure does not enable Start because Hydra may still be running.

The local management endpoint remains machine-local. The Remote tab can manage an explicitly paired online peer through Hydra's end-to-end encrypted relay. On the peer, run `./Hydra pair` to generate a single-use 10-minute code, then enter its host name and code in the controlling TUI. Remote configurations are redacted at the source. A remote apply keeps a restrictive last-known-good backup and rolls back automatically unless the restarted peer reconnects on the candidate revision and the controller confirms it within 90 seconds. Relay, hostname, and profile-activation changes remain local-only because they could remove the recovery path.

Closing the TUI does not stop Hydra. Use `Esc` to quit the TUI.

---

## Quickstart

Create `hydra.conf` next to the binary on **each machine**.

**Master** (the machine with the physical keyboard and mouse):

```json
{
  "name": "desktop",
  "profiles": [{
    "mode": "Master",
    "embeddedStyxServer": { "port": 5000, "password": "secret" },
    "hosts": [
      { "name": "desktop", "neighbours": [{ "direction": "right", "name": "laptop" }] },
      { "name": "laptop" }
    ]
  }]
}
```

**Slave** (the machine that receives input):

```json
{
  "name": "laptop",
  "profiles": [{
    "mode": "Slave",
    "embeddedStyx": { "server": "http://192.168.1.10:5000", "password": "secret" }
  }]
}
```

Replace `192.168.1.10` with the master's IP address. Run `./Hydra` on both machines. Move the cursor past the right edge of the master's screen — it appears on the slave.

### Hotkeys

All hotkeys use **Ctrl+Alt+Super** (Super = ⌘ on macOS, Win key on Windows):

| Hotkey | Action |
|--------|--------|
| `Ctrl+Alt+Super+L` | Toggle cursor lock — pin to current screen, or release to roam freely |
| `Ctrl+Alt+Super+M` | Toggle relative mouse mode on the current remote screen (useful for games) |
| `Ctrl+Alt+Super+C` | Copy selected files/folders to Hydra's cross-machine clipboard (macOS, Windows) |
| `Ctrl+Alt+Super+V` | Paste previously copied files to the current machine |

For cross-network setups (different LANs or over a VPN), see [Networking with Styx](docs/CONFIGURATION.md#networking-with-styx).

---

## Configure Hydra

Create the initial `hydra.conf` from the examples or the quickstart above, then use `Hydra tui` for guided edits or complete JSON editing. The TUI uses Hydra's canonical parser and validator and preserves fields that are not exposed by the guided form. See the [configuration reference](docs/CONFIGURATION.md) for every field and topology option.

---

## Features

- Seamless cursor transitions in any direction (left, right, up, down)
- **Multi-monitor support** — multiple local and remote monitors, auto-detected at startup and on connect/disconnect
- Flexible layout: L-shaped, grids, or any topology
- **Range-based neighbours** — split edges to route to different hosts by cursor position
- **Per-screen scale** — control cursor speed on each remote screen
- Full keyboard forwarding including dead keys and special characters — resolved on the master using its keyboard layout
- Mouse button and scroll forwarding
- **Clipboard sync** — text and images synced automatically when switching machines (all platforms)
- **File transfer** — cross-machine copy/paste of files and folders via hotkey (macOS and Windows)
- **Media key forwarding** — volume, playback, brightness keys forwarded to the active machine
- **Screensaver sync** — activating the screensaver on the master locks all connected slaves
- **Windows login screen support** — installed as a system service, Hydra stays active on the lock and login screens
- End-to-end encrypted relay via **Styx** for machines on different networks
- **Multiple masters per slave** — several machines can share a single slave display; whoever moves their cursor onto it takes control
- **Remote-only mode** — use a headless Linux machine (e.g. Raspberry Pi) as a dedicated input forwarder with no local screen
- **Terminal control center** — monitor, control, and configure the local Hydra instance from macOS, Windows, Linux, or SSH

---

## Full documentation

- [Configuration reference](docs/CONFIGURATION.md) — all config fields, screen layout options, network-aware profiles, hotkeys, Styx setup, and building from source
- [TUI architecture](docs/TUI_ARCHITECTURE.md) — current management boundaries, security invariants, platform lifecycle behavior, and validation expectations
- [Stability and simplification programme](docs/STABILITY_AND_SIMPLIFICATION_PLAN.md) — active reliability priorities and safe-change rules for this fork
- [Styx protocol](Styx.md) — the relay's wire protocol, for implementing your own client or server against it
