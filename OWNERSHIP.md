# Ownership Model

## Principle

This fork follows the **upstream-first Caelestia migration path**.

- **Nix owns the substrate**: platform, hardware, security, PAM,
  services, session wiring, package builds, version pinning.
- **This fork (CLI) owns the shell side-effect layer**: scheme
  switching, wallpaper management, recording, clipboard, emoji
  picker, toggle dispatch, resizer, shell IPC bridge.
- **Runtime owns shell live state**: CLI settings JSON, current
  scheme, wallpaper selection/history, screenshot/recording paths.

CLI runtime state is **not** required to live entirely inside Nix.
The CLI may read and write JSON/config state under
`~/.config/caelestia/` and `~/.local/state/caelestia/` at runtime.

## Relationship to shell fork

This CLI fork is functionally coupled with `ponkcore/shell`:
- wallpaper/scheme/record side effects live in the CLI
- the shell calls into the CLI for desktop side effects
- both share the `caelestia` config/state directory layout

## Removed subsystems

The shell fork has removed `VPN` and `GameMode` services. This CLI
fork does not need corresponding changes because the CLI does not
have VPN or GameMode subcommands — those were shell-only services.
