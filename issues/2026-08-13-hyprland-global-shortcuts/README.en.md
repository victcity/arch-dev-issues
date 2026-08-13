# Hyprland Global Shortcuts: macOS-style Command+C/V Copy/Paste and Super+A Select-All

## Summary

On Hyprland 0.56 (config migrated from hyprlang to Lua), a "Hyprland bind + focus-aware script" approach gives global `Super+C/V` copy/paste semantics (no `SIGINT` in terminals) and `Super+A` select-all, covering kitty, opencode TUI, Chromium, and other GTK/Qt apps. Injection tools such as `wtype` and `ydotool` were rejected along the way; the final solution uses Hyprland's zero-dependency native `send_shortcut` / `pass` dispatchers.

## Environment

- **OS**: Arch Linux (rolling)
- **Compositor**: Hyprland 0.56.2 (Wayland)
- **Terminal**: kitty
- **TUI app**: opencode (running inside kitty)
- **Browser**: Chromium 151 (native Wayland, `xwayland: false`)
- **Script deps**: `bash`, `jq`, `hyprctl`

## Background

Hyprland 0.56 deprecated the hyprlang `hyprland.conf` in favor of a Lua config (`hyprland.lua`); bindings migrated from `bind = ...` to `hl.bind(...)`, and dispatchers to `hl.dsp.*` (e.g. `hl.dsp.exec_cmd`, `hl.dsp.send_shortcut`, `hl.dsp.pass`).

macOS `Command+C/V` is a *semantic* copy/paste — the system dispatches it to the focused app, so terminal apps copy selected text instead of sending `SIGINT`. Wayland has no equivalent; it must be built by hand. The hard part is that "injecting key events" under Wayland has multiple protocols, and each tool behaves very differently with Electron/Chromium apps.

## Reproduction (original problem)

1. Binding `Super+C` → `Ctrl+C` in Hyprland: inside kitty this sends `SIGINT` (kills the foreground process) instead of copying.
2. Injecting `Ctrl+C/V` with `wtype`: modifiers are lost in Chromium/code-oss, so pasting inserts the raw characters `c`/`v`.
3. Injecting with `send_shortcut` plus `non_consuming` passthrough: Chromium's omnibox shows double-triggering and a leftover `Super` modifier.
4. Injecting with `ydotool` (kernel uinput): the event channel is unstable and needs a delay to avoid `Super` residue, causing lag.

## Investigation

| Attempt | Mechanism | Result | Verdict |
|---|---|---|---|
| `wtype` | wlr-virtual-keyboard protocol | Modifiers dropped in Electron/Chromium, raw chars inserted | Virtual-keyboard protocol incompatible with Electron |
| `send_shortcut` + `non_consuming=true` | Hyprland synthesized wl_keyboard events | Chromium double-triggers, `Super` residue | Passthrough `Super` interfered with injected `Ctrl` |
| `ydotool` | kernel uinput real events | Unstable event channel, needs 200ms delay → lag | uinput hotplug timing unreliable on Hyprland 0.56 |
| `send_shortcut` + focus-aware script | Hyprland synthesized events (single modifier) | **Works**: zero deps, clean single-modifier, no delay | Chosen |

Key verified facts (observed with `wev`):
- `send_shortcut` injecting a single-modifier `Ctrl+C` is clean (`utf8 '\x03'`, correct `Ctrl`, no `Super` residue, no delay needed).
- `send_shortcut` with **multi-modifier** `Ctrl+Shift` has wrong ordering (`Shift` presses after `C`), so kitty copy cannot rely on `Ctrl+Shift+C`.
- `pass`'s `window` argument must be a full selector string `"address:0x..."`; passing bare `"0x..."` returns `window not found` and silently fails (a key pitfall hit during this investigation).

## Root Cause Analysis

1. **Electron is incompatible with synthesized/virtual events**: `wtype` (virtual keyboard) and early `send_shortcut` attempts failed because Electron's Wayland input pipeline mishandles modifier state from synthesized/virtual keyboards. `send_shortcut` actually works because it emits the same `wl_keyboard` events as a real keyboard — the earlier failure was caused by `Super` residue from `non_consuming` passthrough, not by `send_shortcut` itself.
2. **kitty's `map` swallows child-process keys**: a kitty-level `map super+a` consumes `super+a`, so opencode TUI (running inside kitty) never receives it. Hence kitty's own select-all uses `alt+a` (translated by the script), while opencode's select-all uses `pass` passthrough — no conflict.
3. **`pass` selector format**: `hl.get_active_window().address` returns `"0x..."`, but `pass({ window = ... })` requires `"address:0x..."`; the missing prefix causes silent failure.

## Solution

### Final keymap

| Combo | Old action | New action |
|---|---|---|
| `Super+C` | launch kate | copy (global) |
| `Super+V` | toggle float | paste (global) |
| `Super+A` | — | select-all (global) |
| `Super+X` | — | toggle float (moved from `Super+V`) |
| `Super+Alt+C` | — | launch kate (moved from `Super+C`) |

### Script `~/.config/hypr/scripts/clipboard.sh`

```bash
#!/bin/bash

action="$1"

case "$action" in
  copy)
    KEY="C"
    ;;
  paste)
    KEY="V"
    ;;
  *)
    exit 1
    ;;
esac

info="$(hyprctl activewindow -j 2>/dev/null)"
class="$(echo "$info" | jq -r '.class // empty' 2>/dev/null)"

case "$class" in
  kitty)
    MODS="ALT"
    ;;
  *)
    MODS="CTRL"
    ;;
esac

hyprctl dispatch "hl.dsp.send_shortcut({ mods = '$MODS', key = '$KEY' })"
```

### Config `~/.config/hypr/hyprland.lua`

```lua
-- Copy / paste
hl.bind(mainMod .. " + C", hl.dsp.exec_cmd("~/.config/hypr/scripts/clipboard.sh copy"))
hl.bind(mainMod .. " + V", hl.dsp.exec_cmd("~/.config/hypr/scripts/clipboard.sh paste"))

-- Select-all
hl.bind(mainMod .. " + A", function()
  local w = hl.get_active_window()
  if w and w.class == "kitty" then
    if w.title and w.title:match("^OC") then
      -- opencode TUI: passthrough super+a, native select-all
      hl.dispatch(hl.dsp.pass({ window = "address:" .. w.address }))
    else
      -- plain kitty: translate to alt+a, triggering kitty map alt+a select_all
      hl.dispatch(hl.dsp.send_shortcut({ mods = "ALT", key = "A" }))
    end
  else
    -- other apps: standard Ctrl+A
    hl.dispatch(hl.dsp.send_shortcut({ mods = "CTRL", key = "A" }))
  end
end)

-- Keymap migration
hl.bind(mainMod .. " + X", hl.dsp.window.float())          -- float (was Super+V)
hl.bind(mainMod .. " + ALT + C", hl.dsp.exec_cmd("kate"))  -- kate (was Super+C)
```

### Config `~/.config/kitty/kitty.conf`

```
map alt+c copy_to_clipboard
map alt+v paste_from_clipboard
map super+c copy_to_clipboard
map super+v paste_from_clipboard
map alt+a select_all
```

### Config `~/.config/opencode/tui.json`

```json
{
  "$schema": "https://opencode.ai/tui.json",
  "keybinds": {
    "input_select_all": "super+a"
  }
}
```

### Per-app strategy

| Focused window | Copy/paste | Select-all |
|---|---|---|
| kitty plain shell | send `Alt+C/V` (kitty `map alt+c/v`) | send `Alt+A` (kitty `map alt+a select_all`) |
| opencode TUI (kitty + title prefix `OC`) | kitty branch | `pass` passthrough `Super+A` (opencode native) |
| others (Chromium, GTK/Qt) | send `Ctrl+C/V` | send `Ctrl+A` |

## References

- [Hyprland wiki - Expanding functionality](https://wiki.hyprland.org/Configuring/Advanced-and-Cool/Expanding-functionality/)
- [Hyprland wiki - Dispatchers](https://wiki.hyprland.org/Configuring/Dispatchers/)
- [kitty - Keyboard shortcuts](https://sw.kovidgoyal.net/kitty/conf/#shortcut-kitty)
- [opencode docs - Keybinds](https://opencode.ai/docs/keybinds/)
- [opencode docs - TUI](https://opencode.ai/docs/tui/)
