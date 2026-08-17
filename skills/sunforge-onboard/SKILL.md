---
name: sunforge-onboard
description: Configure or adapt a Sunforge Sunshine host safely and deterministically. Use when setting up Sunforge on a new Fedora/KDE host, creating or repairing the ignored sunshine/config/host.env, detecting the 4K display connector or EDID, selecting the streaming NIC, preparing a reinstall, or guiding a newcomer through bootstrap without exposing Sunshine credentials.
---

# Sunforge onboarding

Use the repository's deterministic configurator. Do not hand-write `host.env`
unless the script cannot run.

## Workflow

1. Confirm the current directory is a Sunforge checkout by locating
   `sunshine/bin/configure-host` and `sunshine/config/host.env.example`.
2. Run `skills/sunforge-onboard/scripts/configure --dry-run` for read-only
   detection. Summarize the detected host label, connector, whether an EDID was
   found, and NIC. Do not inspect or print Sunshine runtime credentials.
3. If detection is wrong, rerun with explicit `--name`, `--connector`,
   `--edid-prefix`, or `--nic` arguments. Prefer stable names from sysfs, KWin,
   and the default route; never guess `/dev/dri/cardN`.
4. Once values are defensible, run the same command with `--yes` and the chosen
   overrides. This atomically creates ignored `sunshine/config/host.env` with
   mode `0600`.
5. Run `./sunshine/bin/publication-check`. Confirm `git status --ignored` shows
   `sunshine/config/host.env` as ignored.
6. For a new installation, explain the security effects of autologin, disabled
   sleep/locking, and LAN firewall access before running `bootstrap`. Do not run
   bootstrap unless the user requested installation or system changes.
7. After deployment or reboot, run `doctor` and verify the live Sunshine logs
   show the intended physical and logical resolution and an NVENC encoder.

## Guardrails

- Never stage or commit `host.env`, `sunshine_state.json`, `credentials/`,
  `portal_token`, logs, backups, or `sunshine/sunshine/`.
- Treat EDID strings, interface names, and host labels as private local data even
  though they are not authentication secrets.
- Keep one stable 4K SDR host source unless the user has independently proven a
  reliable HDR/runtime modeset path.
- Use `backup-private-state` before reinstalling or replacing Sunshine state.
- Stop if deterministic detection returns the wrong physical target; request the
  intended connector rather than selecting an arbitrary display.
