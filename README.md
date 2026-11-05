# niri-display-manager

A Windows `Win+P`-style display menu for [niri](https://github.com/YaLTeR/niri), built as a keyboard-driven TUI with [gum](https://github.com/charmbracelet/gum) and [fzf](https://github.com/junegunn/fzf).

Press `Super+P`, pick an HDMI/DisplayPort display, then mirror it, extend it, change its resolution, or turn it off — without remembering `niri msg output` syntax.

```
╔══════════════════════════════════════════════════════════╗
║ 󰍹  DISPLAY MANAGER                                      ║
║              Mirror • Extend • Arrange                   ║
╚══════════════════════════════════════════════════════════╝

Choose an HDMI/DisplayPort display to configure:

Display >  󰍹  DP-2      │ Dell U2723QE │ 2560x1440 @ 180 Hz │ 0,0
           󰍹  HDMI-A-1  │ LG TV        │ 2560x1440 @ 100 Hz │ 2560,0
           󰈆  Quit
```

## Features

- Lists every connected HDMI/DP output, including displays that are currently off.
- **Mirror** another display (software mirror via `wl-mirror`).
- **Extend** right, left, above, or below another display, with automatic positioning.
- **Use only this display** (second-screen-only mode).
- **Turn off** a display (PC-screen-only mode); refuses to disable the last active output.
- **Display settings**: resolution/refresh rate, rotation, scale, VRR, reset position.
- Marks the currently active arrangement in the menu.
- Same visual style and W/S + arrow navigation as the companion `app-manager` TUI.
- Stops a running mirror from the same menu.

## Requirements

- Arch Linux or CachyOS (the script uses `pacman` for the optional mirror dependency).
- A running [niri](https://github.com/YaLTeR/niri) session on Wayland.
- `gum`, `fzf`, `jq`, and `niri` on `PATH`.

Developed and tested against niri `26.04` with alacritty on CachyOS.

## Installation

### 1. Install dependencies

```bash
sudo pacman -S --needed gum fzf jq
```

`niri` is assumed to be installed already. For mirroring, install `wl-mirror` now or let the script offer to install it on first use:

```bash
sudo pacman -S --needed wl-mirror
```

### 2. Install the script

Clone the repository and copy the script into your `PATH`:

```bash
git clone https://github.com/GuruOfTheMoon/niri-display-manager.git
cd niri-display-manager
install -Dm755 display-manager ~/.local/bin/display-manager
```

Or download just the script:

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/GuruOfTheMoon/niri-display-manager/main/display-manager \
  -o ~/.local/bin/display-manager
chmod +x ~/.local/bin/display-manager
```

Make sure `~/.local/bin` is in your `PATH`. If `command -v display-manager` prints nothing, add this to your shell config (`~/.bashrc`, `~/.zshrc`, …):

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 3. Add the niri keybind

Add this inside the `binds { … }` block of your niri config (usually `~/.config/niri/config.kdl` or an included file):

```kdl
Mod+P hotkey-overlay-title="Open Display Menu: Mirror / Extend" {
    spawn "alacritty" "--class" "display-manager" "-e" "sh" "-c" "$HOME/.local/bin/display-manager";
}
```

`Mod` is `Super` in niri's default configuration, so this gives you `Super+P`.

Using a different terminal? Keep the app-id exactly `display-manager`, because the script finds its own window by that value:

```kdl
// kitty
spawn "kitty" "--class" "display-manager" "sh" "-c" "$HOME/.local/bin/display-manager";

// foot
spawn "foot" "--app-id=display-manager" "sh" "-c" "$HOME/.local/bin/display-manager";
```

### 4. Add the floating-window rule

Add this to the window rules section of your niri config:

```kdl
window-rule {
    match app-id="display-manager"
    open-floating true
    default-column-width { fixed 820; }
    default-window-height { fixed 640; }
}
```

The floating window keeps the TUI from disturbing your tiling layout and gives it a stable size.

### 5. Reload niri

```bash
niri msg action load-config-file
```

Validate the config first if you like:

```bash
niri validate
```

Now press `Super+P`.

## Usage

| Key | Action |
| --- | --- |
| `W` / `S`, `↑` / `↓` | Move in menus |
| `Enter` | Select |
| `Esc` | Back / quit |
| Type | Search in the resolution list |
| `Ctrl+W` / `Ctrl+S` | Move in searchable lists (so `W`/`S` can be typed) |

After selecting a display you can:

- **Mirror another display** — choose which display is the source; it is duplicated fullscreen onto the selected display.
- **Extend to the right / left / above / below** — choose the display to extend from; the new display is positioned automatically next to the active layout.
- **Use only this display** — disable all other outputs.
- **Display settings** — resolution and refresh rate, rotation, scale, VRR (if supported), and reset position.
- **Turn off this display** — disable it, as long as at least one other display stays on.

## How mirroring works

niri does not provide native output mirroring, so this tool runs [`wl-mirror`](https://github.com/Ferdi265/wl-mirror) as a fullscreen overlay on the target display. It is a software mirror, not a hardware clone mode.

- Only one mirror is active at a time.
- The menu keeps track of the `wl-mirror` process in `$XDG_RUNTIME_DIR/display-manager-mirror.state`, so it can stop it cleanly.
- While mirroring, the menu moves its own window to the source display so the fullscreen mirror does not cover it.
- Stop a mirror with `Super+P` → **Stop mirroring**, or by turning off/extending either involved display.

If `wl-mirror` is missing, the script asks whether to install it with `pacman`. On non-Arch systems, install the equivalent package manually.

## Persisting display settings

Changes made by this menu go through `niri msg output` and are **temporary**: they last until you reload the niri config or log out.

To make a layout permanent, add `output` blocks to your niri config. Use `niri msg outputs` to find connector names and modes:

```kdl
output "DP-2" {
    mode "2560x1440@180.002"
    scale 1
    position x=0 y=0
}

output "HDMI-A-1" {
    mode "2560x1440@100.000"
    scale 1
    position x=2560 y=0
}
```

See the [niri output configuration docs](https://github.com/YaLTeR/niri/wiki/Configuration:-Outputs) for all options.

## Troubleshooting

**"Missing dependencies"**

Install the packages listed in [Requirements](#requirements) and make sure they are on `PATH`.

**"Cannot reach the running niri instance"**

The script needs `NIRI_SOCKET`. This is set automatically for processes launched by niri (including the `Super+P` keybind). If you run the script from an SSH session or another TTY, export `NIRI_SOCKET` first.

**"No HDMI/DP display detected"**

The display list matches outputs whose names start with `HDMI-` or `DP-` (for example `HDMI-A-1`, `DP-2`). Built-in laptop panels (`eDP-1`) are intentionally hidden. If your dock exposes a different connector prefix, adjust the `list_external_outputs` regex in the script.

**Mirroring fails or the screen stays black**

- Verify `wl-mirror` is installed: `wl-mirror --version`.
- niri must be able to capture the source output. Test manually:
  ```bash
  wl-mirror --fullscreen --fullscreen-output HDMI-A-1 DP-2
  ```
- Make sure source and target are different displays and both are turned on.

**"Cannot turn off the last active display"**

This is intentional. niri would otherwise leave you with no usable screen.

**The keybind opens nothing**

- Run `~/.local/bin/display-manager` from a terminal inside niri to see errors.
- Check that the terminal named in the keybind is installed.
- Check that the window rule uses `app-id="display-manager"` and the terminal is launched with that exact class/app-id.

## Uninstall

```bash
rm -f ~/.local/bin/display-manager
rm -f "${XDG_RUNTIME_DIR:-/tmp}/display-manager-mirror.state"
```

Then remove the `Mod+P` keybind and the `display-manager` window rule from your niri config and reload it.

## Related

- [niri](https://github.com/YaLTeR/niri)
- [gum](https://github.com/charmbracelet/gum)
- [fzf](https://github.com/junegunn/fzf)
- [wl-mirror](https://github.com/Ferdi265/wl-mirror)
