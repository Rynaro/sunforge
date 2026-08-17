# AGENTS.md

## Purpose

Sunforge turns a Fedora KDE Plasma workstation into a reproducible, headless
Sunshine game-streaming host. Preserve its declarative, reinstall-friendly
design: commit public configuration, scripts, and service definitions; keep
machine identity and Sunshine runtime state local.

## Repository map

- `sunshine/bin/`: bootstrap, deployment, backup/restore, diagnostics, and
  session helpers.
- `sunshine/config/`: public Sunshine configuration plus the ignored
  per-machine `host.env`.
- `sunshine/system/`: systemd, SDDM, and headless-display definitions.
- `sunshine/tests/`: the local and CI validation entry points.
- `skills/sunforge-onboard/`: the agent workflow and deterministic wrapper for
  creating `host.env`.
- `README.md`: user-facing setup, operations, security, and recovery guidance.
- `sunshine/README.md`: detailed implementation and hardware rationale.

## Working rules

- Read the relevant scripts and documentation before changing behavior. Keep
  the two READMEs aligned with user-visible changes.
- Keep Bash scripts compatible with strict mode (`set -euo pipefail`) and quote
  expansions. Preserve executable permissions on files with Bash shebangs.
- Detect users, sessions, DRM devices, display connectors, and the default-route
  NIC. Never introduce fixed `/dev/dri/cardN` assumptions or machine-specific
  identifiers.
- Keep the stable 3840x2160 60 Hz SDR host source unless a change explicitly
  targets and validates another display path. Moonlight clients choose their
  own stream resolution, bitrate, frame rate, and codec.
- Do not run `bootstrap`, `deploy`, restore operations, service restarts, or
  other host-changing commands unless the user requested system changes.
- Do not overwrite unrelated work in a dirty tree. Make focused edits and stage
  only files that belong to the task.

## Private state and publication safety

Never read, print, stage, or commit Sunshine credentials or machine-local state,
including:

- `sunshine/config/host.env`
- `sunshine/sunshine/`
- `sunshine_state.json`
- TLS private keys, credentials, pairing data, `portal_token`, logs, or backups

Treat host labels, EDIDs, connector names, and interface names as private local
data. Use `sunshine/bin/configure-host` or the `sunforge-onboard` skill to create
`host.env`; do not hand-write it when deterministic detection is available.
Run `./sunshine/bin/publication-check` immediately before every commit or push.

## Validation

Run the repository suite for every change:

```bash
./sunshine/tests/ci
```

The suite performs Bash syntax and ShellCheck validation, JSON parsing,
host-configuration tests, executable-bit checks, and publication safeguards.
Hardware-dependent behavior must be checked on the Fedora host with
`./sunshine/bin/doctor`; do not claim hardware validation from CI alone.

## Releases

Use Conventional Commit subjects because Release Please derives versions and
changelog entries from them. Do not manually edit `version.txt`,
`.release-please-manifest.json`, or generated changelog sections for an ordinary
feature or fix. Merge the Release Please PR to publish the GitHub release and
tag, then verify the resulting release and CI checks.
