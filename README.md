# caelestia-cli (ponkcore fork)

Fork of [`caelestia-dots/cli`](https://github.com/caelestia-dots/cli) — the
side-effect layer for the Caelestia desktop shell: scheme switching, wallpaper
management, screenshots, recording, clipboard, emoji picker, window resizer,
and the `caelestia shell ...` IPC bridge.

Deployed together with [`ponkcore/shell`](https://github.com/ponkcore/shell)
through `nix-config`. See [`OWNERSHIP.md`](OWNERSHIP.md) for the split of
responsibilities between the two forks.

<details><summary id="dependencies">External dependencies</summary>

Provided by the Nix package; listed here for non-Nix builds.

- [`libnotify`](https://gitlab.gnome.org/GNOME/libnotify) - sending notifications
- [`swappy`](https://github.com/jtheoof/swappy) - screenshot editor
- [`grim`](https://gitlab.freedesktop.org/emersion/grim) - taking screenshots
- [`dart-sass`](https://github.com/sass/dart-sass) - discord theming
- [`wl-clipboard`](https://github.com/bugaevc/wl-clipboard) - copying to clipboard
- [`slurp`](https://github.com/emersion/slurp) - selecting an area
- [`gpu-screen-recorder`](https://git.dec05eba.com/gpu-screen-recorder/about) - screen recording
- `glib2` - closing notifications
- [`cliphist`](https://github.com/sentriz/cliphist) - clipboard history
- [`fuzzel`](https://codeberg.org/dnkl/fuzzel) - clipboard history/emoji picker

</details>

## Installation

### Nix

This is the supported path for the fork.

```sh
nix run github:ponkcore/cli
```

Or add it to your configuration:

```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

    caelestia-cli = {
      url = "github:ponkcore/cli";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };
}
```

Packages:

- `caelestia-cli.packages.<system>.default` — CLI alone
- `caelestia-cli.packages.<system>.with-shell` — CLI plus
  [`ponkcore/shell`](https://github.com/ponkcore/shell), which is what the
  shell's Home Manager module uses. The `shell` subcommand and full
  wallpaper/scheme functionality need the shell package present.

The `caelestia-shell` input in this repo's `flake.nix` points at
`github:ponkcore/shell`. In `nix-config` the two follow each other, so the
Quickshell and m3shapes nodes are shared instead of duplicated.

There is no AUR package for this fork, and the `install` / `update`
subcommands are upstream's dotfiles deployer — they are not used here and
default to `caelestia-dots/caelestia`.

### Manual installation

Install all [dependencies](#dependencies), then
[`python-build`](https://github.com/pypa/build),
[`python-installer`](https://github.com/pypa/installer),
[`python-hatch`](https://github.com/pypa/hatch) and
[`python-hatch-vcs`](https://github.com/ofek/hatch-vcs).

On NixOS use `nix develop` rather than `pip install` — the interpreter lives
in the read-only store and PEP 668 blocks both `pip install` and
`pip install --user`.

```sh
git clone https://github.com/ponkcore/cli.git
cd cli
python -m build --wheel
python -m installer dist/*.whl
cp completions/caelestia.fish ~/.local/share/fish/vendor_completions.d/caelestia.fish
```

### Optional integrations

Both shell out to `sudo -n` for narrow commands (`papirus-folders`, and
`mkdir`/`tee` into `/etc`). On this host `nix-config` grants the primary user
`NOPASSWD: ALL` (`modules/nixos/security.nix`), so neither needs a per-command
sudoers entry — a deliberate trade-off for a personal laptop, not something to
copy elsewhere. The snippets below are the narrower alternative for a host that
does not grant blanket NOPASSWD.

Neither integration does anything here, for two different reasons:

- **Papirus folder recolouring silently no-ops — and this is an upstream bug on
  NixOS.** `apply_gtk()` calls `sync_papirus_colors()` unconditionally and
  `enableGtk` is `true`, so the function runs; `papirus-folders` is installed and
  `sudo -n` would succeed. But it early-returns before the sudo call because it
  probes hardcoded FHS paths (`theme.py`: `/usr/share/icons/Papirus*`,
  `~/.local/share/icons/Papirus`, `~/.icons/Papirus`). On NixOS none exist — the
  icons live in the store and are exposed at
  `/etc/profiles/per-user/<user>/share/icons/Papirus*` and
  `/run/current-system/sw/share/icons/Papirus*`. So the guard
  `if not any(p.exists() ...): return` trips and folder colours never sync.
  Fixing it means teaching that probe about the NixOS icon paths; it is not a
  permissions problem.
- **Chromium theming is disabled outright:** `enableChromium` is `false` in
  `~/.config/caelestia/cli.json`, so `apply_chromium()` never runs.

Both failures are silent — `sync_papirus_colors()` returns without logging when
the binary is missing *or* the icon dir probe fails, so a broken setup looks
identical to a working one.

#### Papirus folder colour theming

Requires [`papirus-folders`](https://github.com/PapirusDevelopmentTeam/papirus-folders)
runnable under `sudo` without a prompt. Narrower than blanket NOPASSWD:

```sh
echo "$USER ALL=(ALL) NOPASSWD: $(which papirus-folders)" | sudo tee /etc/sudoers.d/papirus-folders
sudo chmod 440 /etc/sudoers.d/papirus-folders
```

#### Chromium-based browser theming

The CLI must be able to create and write policy directories under `/etc`
without a prompt:

```sh
for dir in /etc/chromium/policies/managed /etc/brave/policies/managed /etc/opt/chrome/policies/managed; do
    echo "$USER ALL=(ALL) NOPASSWD: $(which mkdir) -p $dir" | sudo tee -a /etc/sudoers.d/caelestia-chromium
    echo "$USER ALL=(ALL) NOPASSWD: $(which tee) $dir/caelestia.json" | sudo tee -a /etc/sudoers.d/caelestia-chromium
done
sudo chmod 440 /etc/sudoers.d/caelestia-chromium
```

On NixOS these directories belong to the corresponding package modules —
prefer declaring the policy files in `nix-config` over granting `sudo` to
`tee`.

## Usage

All subcommands/options can be explored via the help flag.

```
$ caelestia -h
usage: caelestia [-h] [-v] COMMAND ...

Main control script for the Caelestia dotfiles

options:
  -h, --help     show this help message and exit
  -v, --version  print the current version

subcommands:
  valid subcommands

  COMMAND        the subcommand to run
    shell        start or message the shell
    toggle       toggle a special workspace
    scheme       manage the colour scheme
    screenshot   take a screenshot
    record       start a screen recording
    clipboard    open clipboard history
    emoji        emoji/glyph utilities
    wallpaper    manage the wallpaper
    resizer      window resizer daemon
    install      install the Caelestia dotfiles
    update       update the Caelestia dotfiles
```

> [!NOTE]
> `install` and `update` are upstream's dotfiles deployer — they clone and
> manage `caelestia-dots/caelestia` (see the `dots` key in
> [Configuring](#configuring)). This fork is deployed by `nix-config`, so both
> subcommands are inert here; do not run them against a NixOS system.

### User templates

Custom user templates can be defined in `~/.config/caelestia/templates/`.

#### Template syntax

`{{ <color>.<format> }}`

- `<color>` is a theme color role derived from the Material You color system (e.g. `primary`, `secondary`, `background`)
- `<format>` is the output format: `hex` or `rgb`

#### Examples

- `{{ primary.hex }}` outputs `3f4ba2`
- `{{ primary.rgb }}` outputs `rgb(193, 132, 207)`

Output files are written to `~/.local/state/caelestia/theme/`. You can symlink them to your desired locations.

## Configuring

All configuration options are in `~/.config/caelestia/cli.json`.

<details><summary>Example configuration</summary>

```json
{
    "record": {
        "extraArgs": []
    },
    "wallpaper": {
        "postHook": "echo $WALLPAPER_PATH $SCHEME_NAME $SCHEME_FLAVOUR $SCHEME_MODE $SCHEME_VARIANT $SCHEME_COLOURS"
    },
    "theme": {
        "enableTerm": true,
        "enableHypr": true,
        "enableDiscord": true,
        "enableSpicetify": true,
        "enablePandora": true,
        "enableFuzzel": true,
        "enableBtop": true,
        "enableNvtop": true,
        "enableHtop": true,
        "enableGtk": true,
        "enableQt": true,
        "enableWarp": true,
        "enableChromium": true,
        "enableZed": true,
        "enableCava": true,
        "iconTheme": "Papirus-Dark",
        "iconThemeLight": "Papirus-Light",
        "iconThemeDark": "Papirus-Dark",
        "postHook": "echo $SCHEME_NAME $SCHEME_FLAVOUR $SCHEME_MODE $SCHEME_VARIANT $SCHEME_COLOURS"
    },
    "toggles": {
        "communication": {
            "discord": {
                "enable": true,
                "match": [{ "class": "discord" }],
                "command": ["discord"],
                "move": true
            },
            "whatsapp": {
                "enable": true,
                "match": [{ "class": "whatsapp" }],
                "move": true
            }
        },
        "music": {
            "spotify": {
                "enable": true,
                "match": [{ "class": "Spotify" }, { "initialTitle": "Spotify" }, { "initialTitle": "Spotify Free" }],
                "command": ["spicetify", "watch", "-s"],
                "move": true
            },
            "feishin": {
                "enable": true,
                "match": [{ "class": "feishin" }],
                "move": true
            }
        },
        "sysmon": {
            "btop": {
                "enable": true,
                "match": [{ "class": "btop", "title": "btop", "workspace": { "name": "special:sysmon" } }],
                "command": ["foot", "-a", "btop", "-T", "btop", "fish", "-C", "exec btop"]
            }
        },
        "todo": {
            "todoist": {
                "enable": true,
                "match": [{ "class": "Todoist" }],
                "command": ["todoist"],
                "move": true
            }
        }
    },
    "dots": {
        "url": "https://github.com/caelestia-dots/caelestia.git",
        "branch": "main"
    }
}
```

</details>
