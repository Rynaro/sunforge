# Sunforge

**Turn a Fedora workstation into a reproducible, headless, low-latency game-streaming host.**

Sunforge is an opinionated, declarative setup for [Sunshine](https://github.com/LizardByte/Sunshine),
KDE Plasma Wayland, NVIDIA NVENC, and Moonlight clients. It is designed for the
awkward real-world case: the host boots without a physical monitor, creates a
proper graphical session on an HDMI dummy plug, launches games remotely, and
remains recoverable over SSH.

The configuration was built for 4K60 living-room gaming, but the host keeps one
stable 4K desktop while each Moonlight client chooses its own resolution,
bitrate, frame rate, and codec. A TV, tablet, handheld, and laptop can therefore
share the same host without fragile runtime display switching.

> [!IMPORTANT]
> Sunforge enables desktop autologin and disables automatic sleep and screen
> locking. This is appropriate for a trusted, dedicated streaming host—not a
> shared or physically exposed computer. Full-disk encryption may still require
> local or network-bound unlocking after a cold boot.

## Why Sunforge?

- **Headless by design:** SDDM creates a real Plasma Wayland session at boot,
  even when the only display is a 4K HDMI dummy plug.
- **Reinstall-friendly:** system state is derived from scripts and templates;
  machine identity and Sunshine pairings remain separate.
- **Low-latency NVIDIA capture:** KMS capture and NVENC are configured without
  unstable DRM card numbers or forced codec claims.
- **One stable source:** the host stays at 3840×2160 SDR and lets NVENC scale for
  each client, avoiding NVIDIA/KWin modeset races.
- **Remote app launching:** Sunshine discovers KWin's live Wayland, Xwayland,
  and Xauthority values so Steam works after boot or an SSH-driven restart.
- **Gaming-aware tuning:** TuneD, GameMode, IRQ balancing, NVIDIA persistence,
  CPU boost checks, and optional Ethernet EEE tuning are integrated.
- **Safe to publish:** credentials, TLS keys, client pairings, logs, host names,
  local EDIDs, and network choices are excluded and audited before a push.

## Architecture

```text
Boot
 ├─ systemd → performance and network policy
 └─ SDDM autologin → Plasma Wayland on HDMI dummy plug
                     ├─ KWin / PipeWire / input session
                     └─ Sunshine → KMS capture → NVENC → Moonlight
                                                   ├─ TV / Steam Deck
                                                   ├─ Android tablet
                                                   └─ macOS desktop
```

Sunshine's credentials and paired-client state remain runtime data. Sunforge
manages only reproducible configuration, launchers, diagnostics, and service
definitions.

## Target platform

The reference deployment uses:

- Fedora 44
- KDE Plasma on Wayland
- An NVIDIA GPU with NVENC
- A 4K60-capable HDMI dummy plug
- A wired LAN for the primary gaming path

The scripts intentionally detect users, DRM devices, graphical sessions, and
the default network route where practical. Fedora package names, Plasma tools,
systemd, firewalld, and NVIDIA-specific encoder settings make this project a
starting point—not a universal installer—for other distributions or GPUs.

## Quick start

### 1. Clone and personalize

```bash
git clone https://github.com/Rynaro/sunforge.git
cd sunforge
cp sunshine/config/host.env.example sunshine/config/host.env
${EDITOR:-vi} sunshine/config/host.env
```

The ignored `host.env` controls the Moonlight host label, optional EDID prefix,
display connector, and streaming NIC. It is never meant to be committed.

### 2. Bootstrap the host

Run from the Linux account that should own the unattended Plasma session:

```bash
./sunshine/bin/bootstrap
sudo systemctl reboot
```

Bootstrap installs the Fedora packages, capabilities, group membership,
firewall rules, autologin configuration, system performance policy, user
service, and managed Sunshine files.

### 3. Verify over SSH

After the reboot:

```bash
cd sunforge
./sunshine/bin/doctor
```

Then open Sunshine's Web UI at `https://HOST-IP:47990`, create credentials, and
pair a Moonlight client. Keep the service on a trusted LAN; UPnP is disabled.

### 4. Activate a true 4K source

Once Plasma has recorded the dummy display in KWin's output configuration:

```bash
~/.local/bin/configure-kwin-4k
sudo systemctl reboot
```

The reboot is deliberate. Live KWin output changes can remove NVIDIA's KMS
scanout while Sunshine is opening it, producing failed captures or HTTP 502
errors.

## Recommended Moonlight profiles

| Use case | Resolution | FPS | Codec | Bitrate |
|---|---:|---:|---|---:|
| 4K TV gaming | 3840×2160 | 60 | AV1 or HEVC | 80–120 Mbps |
| Low-latency TV | 1920×1080 | 60 | HEVC or H.264 | 20–40 Mbps |
| High-resolution tablet | Client native | 60 | HEVC or AV1 | 20–50 Mbps |
| Desktop access | Client native | 60 | HEVC | 20–50 Mbps |

Start with SDR. Enable HDR only after confirming that the dummy EDID, client,
display output, and TV all support the same HDR path. On a TV, enable Game Mode
and prefer a wired Moonlight client.

## Day-two operations

Deploy configuration changes without reinstalling system packages:

```bash
./sunshine/bin/deploy
```

Run diagnostics:

```bash
./sunshine/bin/doctor
journalctl --user -u sunshine.service -b --no-pager
```

Launch non-Sunshine Steam games with GameMode:

```text
gamemoderun %command%
```

## Statelessness and recovery

Sunforge separates two kinds of state:

| Managed and reproducible | Private runtime state |
|---|---|
| Sunshine settings and app catalog | Web UI password hash |
| systemd service definitions | TLS private keys |
| Plasma power/input policy | Moonlight client certificates and pairings |
| launch and diagnostic scripts | Logs and portal tokens |
| hardware-selection templates | Local `host.env` |

For a clean reinstall, clone the repository, recreate `host.env`, run
`bootstrap`, and pair clients again. If retaining pairings matters, back up the
live Sunshine state separately and protect it like a credential archive.

## Security and publication safety

The repository ignores known Sunshine secrets and legacy runtime directories at
both the project and component levels. Before every public push, run:

```bash
./sunshine/bin/publication-check
git status --ignored
```

Never force-add any of these:

- `sunshine/config/host.env`
- `sunshine/sunshine/`
- `sunshine_state.json`
- `credentials/`, `portal_token`, logs, or backups

The Web UI is allowed from the LAN, while WAN encryption remains enabled and
automatic router exposure remains disabled. Review firewall rules and network
trust before adapting this project to a different environment.

## Known boundaries

- Runtime resolution/HDR switching helpers remain for compatibility but are not
  used by the default profiles; fixed 4K SDR is the reliable NVIDIA path.
- The installer changes system power policy, enables autologin, and opens
  Sunshine's standard ports in the active firewalld zone. Read `bootstrap`
  before running it.
- Firmware may prevent Linux from enabling CPU boost. In that case, enable Core
  Performance Boost in UEFI/BIOS and validate thermals and stability.
- A 1 GbE link is sufficient for a 4K60 stream; 2.5 GbE is useful but not
  required.

## Project layout

```text
sunshine/
├── bin/       bootstrap, deployment, launch, and diagnostic tools
├── config/    public templates and the ignored per-host configuration
└── system/    system and user service definitions
```

Detailed implementation notes and hardware rationale live in
[`sunshine/README.md`](sunshine/README.md).

## Contributing

Issues and focused pull requests are welcome. Please avoid committing generated
Sunshine state or hardware-specific identifiers, keep scripts compatible with
Bash strict mode, and run the publication check before submitting changes.

## License

Sunforge is available under the [MIT License](LICENSE).
