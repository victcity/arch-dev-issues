# Hyprland 全局快捷键：复刻 macOS Command+C/V 复制粘贴与 Super+A 全选

## 概述

在 Hyprland 0.56（配置由 hyprlang 迁移到 Lua）下，通过「Hyprland 绑定 + 脚本判断焦点窗口」的方案，用 `Super+C/V` 实现全局复制/粘贴语义（终端内不触发 `SIGINT`），用 `Super+A` 实现全选，覆盖 kitty、opencode TUI、Chromium 及其他 GTK/Qt 应用。过程中否决了 `wtype`、`ydotool` 等注入方案，最终选定零依赖的 Hyprland 原生 `send_shortcut` / `pass` dispatcher。

## 环境

- **系统**: Arch Linux (rolling)
- **合成器**: Hyprland 0.56.2 (Wayland)
- **终端**: kitty
- **TUI 应用**: opencode（运行于 kitty 内）
- **浏览器**: Chromium 151（原生 Wayland，`xwayland: false`）
- **脚本依赖**: `bash`、`jq`、`hyprctl`

## 背景

Hyprland 0.56 弃用了 hyprlang 格式的 `hyprland.conf`，改用 Lua 配置（`hyprland.lua`），配置语法从 `bind = ...` 迁移为 `hl.bind(...)`，dispatcher 也迁移为 `hl.dsp.*`（如 `hl.dsp.exec_cmd`、`hl.dsp.send_shortcut`、`hl.dsp.pass`）。

macOS 的 `Command+C/V` 是「语义化」的复制/粘贴——由系统分发给前台应用，终端应用也能复制选中的文字而非发送 `SIGINT`。Wayland 下没有等价机制，需要自行实现。难点在于「向应用注入按键」在 Wayland 下存在多种协议，且不同工具对 Electron/Chromium 系应用的兼容性差异巨大。

## 复现步骤（原始问题）

1. 在 Hyprland 中直接绑定 `Super+C` → `Ctrl+C`：在 kitty 里触发 `SIGINT`（中断前台进程），而非复制。
2. 用 `wtype` 注入 `Ctrl+C/V`：Chromium/code-oss 等 Electron 应用中修饰键丢失，粘贴时直接插入裸字符 `c`/`v`。
3. 用 `send_shortcut` 注入并配合 `non_consuming` 透传：Chromium 地址栏出现「双触发」及 `Super` 键残留。
4. 用 `ydotool`（内核 uinput）注入：事件通道时好时坏，且必须加延迟避免 `Super` 残留，导致「拖沓不跟手」。

## 排查过程

| 尝试 | 机制 | 结果 | 结论 |
|---|---|---|---|
| `wtype` | wlr-virtual-keyboard 协议 | Electron/Chromium 修饰键丢失，插入裸字符 | 虚拟键盘协议对 Electron 不兼容 |
| `send_shortcut` + `non_consuming=true` | Hyprland 合成 wl_keyboard 事件 | Chromium 双触发、`Super` 残留干扰 | 透传 `Super` 干扰了注入的 `Ctrl` 修饰 |
| `ydotool` | 内核 uinput 真实事件 | 事件通道不稳定，需 200ms 延迟→拖沓 | Hyprland 0.56 下 uinput 热插拔时序不可靠 |
| `send_shortcut` + 脚本分流 | Hyprland 合成事件（单修饰） | **成功**：零依赖、单修饰干净、无需延迟 | 选定方案 |

关键验证事实（用 `wev` 观察）：
- `send_shortcut` 注入单修饰 `Ctrl+C` 是干净的（`utf8 '\x03'`，`Ctrl` 正确、无 `Super` 残留、无需延迟）。
- `send_shortcut` 的**多修饰** `Ctrl+Shift` 时序异常（`Shift` 在 `C` 之后才按下），因此 kitty 的复制不能靠 `Ctrl+Shift+C`。
- `pass` 的 `window` 参数必须是完整 selector 字符串 `"address:0x..."`，传裸 `"0x..."` 会报 `window not found`，透传静默失效（本次排查中踩过的关键坑）。

## 根因分析

1. **Electron 对合成事件不兼容**：`wtype`（虚拟键盘）与早期 `send_shortcut` 方案失效，本质是 Electron 的 Wayland 输入管线对「合成/虚拟键盘」产生的修饰键状态处理异常；而 `send_shortcut` 走的是与真实键盘一致的 `wl_keyboard` 事件，单修饰场景下可被 Electron 正确识别。之前的失败其实是 `non_consuming` 透传的 `Super` 残留干扰，而非 `send_shortcut` 本身。
2. **kitty 的 `map` 会拦截子进程按键**：kitty 层 `map super+a` 会吞掉 `super+a`，导致运行于 kitty 内的 opencode TUI 收不到该组合键。因此「kitty 自身全选」改用 `alt+a`（由脚本翻译），「opencode 全选」走 `pass` 透传，二者互不冲突。
3. **`pass` 的 selector 格式**：`hl.get_active_window().address` 返回 `"0x..."`，但 `pass({ window = ... })` 需要 `"address:0x..."`，缺前缀导致静默失败。

## 解决方案

### 最终键位布局

| 组合键 | 原功能 | 新功能 |
|---|---|---|
| `Super+C` | 启动 kate | 复制（全局） |
| `Super+V` | 切换浮动 | 粘贴（全局） |
| `Super+A` | — | 全选（全局） |
| `Super+X` | — | 切换浮动（自 `Super+V` 迁移） |
| `Super+Alt+C` | — | 启动 kate（自 `Super+C` 迁移） |

### 脚本 `~/.config/hypr/scripts/clipboard.sh`

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

### 配置 `~/.config/hypr/hyprland.lua`

```lua
-- 复制 / 粘贴
hl.bind(mainMod .. " + C", hl.dsp.exec_cmd("~/.config/hypr/scripts/clipboard.sh copy"))
hl.bind(mainMod .. " + V", hl.dsp.exec_cmd("~/.config/hypr/scripts/clipboard.sh paste"))

-- 全选
hl.bind(mainMod .. " + A", function()
  local w = hl.get_active_window()
  if w and w.class == "kitty" then
    if w.title and w.title:match("^OC") then
      -- opencode TUI：透传 super+a，opencode 原生全选
      hl.dispatch(hl.dsp.pass({ window = "address:" .. w.address }))
    else
      -- 普通 kitty：翻译为 alt+a，触发 kitty map alt+a select_all
      hl.dispatch(hl.dsp.send_shortcut({ mods = "ALT", key = "A" }))
    end
  else
    -- 其他应用：标准 Ctrl+A
    hl.dispatch(hl.dsp.send_shortcut({ mods = "CTRL", key = "A" }))
  end
end)

-- 键位迁移
hl.bind(mainMod .. " + X", hl.dsp.window.float())          -- 浮动（原 Super+V）
hl.bind(mainMod .. " + ALT + C", hl.dsp.exec_cmd("kate"))  -- kate（原 Super+C）
```

### 配置 `~/.config/kitty/kitty.conf`

```
map alt+c copy_to_clipboard
map alt+v paste_from_clipboard
map super+c copy_to_clipboard
map super+v paste_from_clipboard
map alt+a select_all
```

### 配置 `~/.config/opencode/tui.json`

```json
{
  "$schema": "https://opencode.ai/tui.json",
  "keybinds": {
    "input_select_all": "super+a"
  }
}
```

### 分应用处理策略

| 焦点窗口 | 复制/粘贴 | 全选 |
|---|---|---|
| kitty 普通 shell | 发 `Alt+C/V`（kitty `map alt+c/v`） | 发 `Alt+A`（kitty `map alt+a select_all`） |
| opencode TUI（kitty + title 前缀 `OC`） | 走 kitty 分支 | `pass` 透传 `Super+A`（opencode 原生） |
| 其他（Chromium、GTK/Qt） | 发 `Ctrl+C/V` | 发 `Ctrl+A` |

## 相关链接

- [Hyprland wiki - Expanding functionality](https://wiki.hyprland.org/Configuring/Advanced-and-Cool/Expanding-functionality/)
- [Hyprland wiki - Dispatchers](https://wiki.hyprland.org/Configuring/Dispatchers/)
- [kitty - Keyboard shortcuts](https://sw.kovidgoyal.net/kitty/conf/#shortcut-kitty)
- [opencode docs - Keybinds](https://opencode.ai/docs/keybinds/)
- [opencode docs - TUI](https://opencode.ai/docs/tui/)
