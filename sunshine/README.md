# Declarative Sunshine host

Reusable Sunshine configuration for Fedora, KDE Plasma/Wayland, NVIDIA NVENC,
and a 4K HDMI dummy plug. It was validated on Fedora 44 with an RTX 5070 Ti,
but machine identity and hardware selectors are kept outside version control.

## Adapt it to your host

Create the ignored local configuration before running any installer:

```bash
cp sunshine/config/host.env.example sunshine/config/host.env
${EDITOR:-vi} sunshine/config/host.env
```

`SUNSHINE_HOST_NAME` controls Moonlight's host label. Set the EDID prefix from
KWin's `edidIdentifier`, or leave it empty to select the first enabled 4K
output. The connector and NIC can also be overridden; an empty NIC is detected
from the default route. Script filenames beginning with `hades-` are retained
only for compatibility with existing installations and are not host selectors.

## Fresh install / one command

Run this **from a terminal inside the logged-in Plasma session**:

```bash
./sunshine/bin/bootstrap
```

It installs required Fedora packages, enables KMS capture and virtual input,
opens only Sunshine's standard ports in the active firewalld zone, deploys the
managed configuration, enables SDDM autologin to Plasma Wayland on the dummy
plug, disables system sleep, and enables Sunshine in the graphical user session.
It also installs the hardware performance policy described below.
Reboot once, then validate over SSH with:

```bash
./sunshine/bin/doctor
```

For ordinary config updates, no sudo is needed:

```bash
./sunshine/bin/deploy
```

`deploy` backs up changed managed files and restarts an already-running service.
It does not copy or overwrite `sunshine_state.json`, certificates, pairings, web
credentials, or logs. Back up that state separately only if preserving paired
clients across a reinstall is desired.

## Unattended/headless boot

No SSH command is needed to create the desktop. At boot, SDDM logs the account
that ran `bootstrap` into Plasma Wayland on the connected dummy display; Plasma activates its user
graphical target, which starts Sunshine. The display, PipeWire audio session,
KWallet/session services, Steam, and Sunshine therefore share one proper desktop
login even when nobody is physically present.

An SSH shell remains a TTY and will show `XDG_SESSION_TYPE=tty`; that is expected.
`bin/doctor` separately discovers the active graphical seat session through
logind. Display commands should run as Sunshine application prep commands inside
Plasma, not directly from an unadorned SSH environment.

KWin can turn off a physically connected dummy output after idle time. The
declarative Plasma settings disable display dimming, DPMS turn-off, screen
locking, and automatic suspend. Runtime mode/HDR switching is deliberately not
used: it can temporarily remove NVIDIA's KMS scanout exactly when Sunshine opens
it. The source remains kernel-forced 4K60 SDR and NVENC scales it for the client.

Autologin means anyone with physical access to the host can reach the desktop; it
does not store the account password in the SDDM file. Full-disk encryption still
requires someone to unlock the disk after a cold boot unless network-bound disk
unlock is configured separately. Sleep targets are masked so the host remains
reachable; use shutdown when it should be off.

## Profiles

Moonlight controls stream resolution, FPS, bitrate, and codec. Select these in
the client; the host profiles control the physical dummy display and launcher:

| Sunshine app | Host source | Suggested Moonlight setting |
|---|---:|---|
| TV — 4K60 Gaming | True 3840×2160 60 Hz SDR, 100% scale | 4K, 60 FPS, HEVC/AV1, 80–120 Mbps |
| TV — 1080p60 Low Latency | Fixed 3840×2160 60 Hz SDR | 1080p, 60 FPS, 20–40 Mbps |
| Tablet — Desktop | Fixed 3840×2160 60 Hz SDR | native client resolution, 60 FPS, 20–50 Mbps |
| Mac — Desktop | Fixed 3840×2160 60 Hz SDR | native/client resolution, 60 FPS |

To make the dummy plug a true 4K capture desktop, run
`~/.local/bin/configure-kwin-4k`, then reboot. The reboot is intentional: live
KWin output reconfiguration is unreliable with the NVIDIA headless session.

The dummy EDID supports 4K60, 1080p60, HDR10 metadata, and HDMI 2.0 bandwidth,
but does not expose 1440p or high-resolution 16:10 modes. Sunshine/NVENC scales
the 4K source to a tablet's requested stream resolution. HDR should remain off
for desktop/tablet use and enabled only when the Moonlight client and final TV
path are both HDR-capable.

If the dummy plug moves to another connector, set it for the service environment
or edit the helper's default:

```bash
SUNSHINE_DISPLAY_CONNECTOR=DP-1 hades-display-profile status
```

## Why the copied setup failed

- The log shows `/dev/dri/card1: Permission denied` and denied virtual-input
  devices; the user was not in the `input` group.
- Sunshine was launched from SSH outside the graphical session
  (`WAYLAND_DISPLAY` absent). It now starts from Plasma's user target at boot.
- `sunshine.conf.custom` used unsupported INI sections and `kmsgrab` rather than
  Sunshine's flat `key = value` format and `kms` capture name.
- Every old app, including “4K HDR,” forced 1920×1080 at 60 Hz.
- `/dev/dri/card1` was hard-coded even though DRM card numbering is not stable.
- HEVC Main10 and AV1 Main10 were forced instead of being capability-probed.

Start troubleshooting with `bin/doctor`, then inspect:

```bash
journalctl --user -u sunshine.service -b --no-pager
```

The Web UI is `https://localhost:47990`; create credentials and pair Moonlight
there after the service passes diagnostics. Keep UPnP disabled for a LAN-only
host.

## Hardware notes

- RTX 5070 Ti NVENC is the correct low-latency encoder; P1 plus quarter-resolution
  two-pass is used. Client bitrate is intentionally not capped on the host.
- Sunshine explicitly targets Plasma's `wayland-0` and Xwayland `:0` sockets so
  graphical app launching still works after an SSH-initiated service restart.
- The Ryzen 9 5900X uses `amd-pstate-epp`, the performance governor, 100% minimum
  performance, and boost under TuneD's `latency-performance` profile. The reference host was
  audited with boost unexpectedly disabled, so a boot service verifies/enables
  it after TuneD.
- GameMode is held active for the lifetime of either gaming stream, providing
  game-process CPU/I/O priority and screen-inhibition without permanently
  applying its per-game behavior to desktop streams.
- NVIDIA persistence is enabled. No overclock, Coolbits, fixed power limit, or
  forced clock offset is applied; NVENC P1 and quarter-resolution two-pass are
  Sunshine's documented low-latency choices.
- The RTL8125 is a 2.5 GbE NIC using `r8169`, but the audited link partner only
  advertises 1 Gb/s. That comfortably carries a 4K60 stream, but a 2.5 GbE switch
  port and suitable cable are needed to negotiate 2.5 Gb/s. EEE is disabled to
  avoid link wake latency; hardware offloads remain enabled.
- 64 GiB is ample, but DDR4-3000 is below the common Zen 3 performance sweet
  spot. Memory frequency/timings and FCLK are firmware decisions and are not
  changed by this repository; stability testing is required before enabling a
  faster DOCP/XMP profile.
- The TV should use Game Mode and a wired client. A wired Steam Deck dock/client
  is preferable to Wi-Fi tablet streaming for the primary TV target.

## Publishing safely

Never add `config/host.env` or the legacy `sunshine/` runtime directory. The
latter contains live TLS keys, Web UI state, pairings, client identifiers, IP
addresses, and logs. Repository and nested ignore rules protect both locations.
Verify them immediately before committing or pushing:

```bash
./sunshine/bin/publication-check
git status --ignored
```

The deployment intentionally manages only `sunshine.conf` and `apps.json`; it
does not copy, delete, or publish Sunshine's runtime identity.

For games launched outside the two Sunshine gaming entries, use this Steam launch
option to request the same per-game scheduling optimizations:

```text
gamemoderun %command%
```

### Samsung Book Cover trackpad

KWin applies a device-scoped flat pointer profile to Sunshine's relative virtual
mouse (uinput vendor `0xBEEF`, product `0xDEAD`). Pointer speed is `-0.25` and the
scroll factor is `0.75`; this avoids double acceleration after Android has
already shaped the Tab S9 Ultra Book Cover trackpad signal. Touchscreen absolute
input and physical mice connected directly to the host are unaffected.
