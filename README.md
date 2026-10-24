# Wayfire + Noctalia desktop — setup guide

This is the "engine" of my desktop with the Arknights **Endfield** theming
stripped out: **Wayfire** (wlroots compositor) running the **Noctalia** shell
(bar / dock / launcher / control-centre / notifications), plus the fixes that
make screen-sharing and Discord actually work on wlroots.

You get to pick your own colours, wallpaper and fonts — none of the Endfield
palette, fonts, window-button diamonds or `noctalia` fork are here.

Written for an **Arch-based** distro (CachyOS/Arch/EndeavourOS). On anything else
the package names differ but the config files are identical.

---

## 0. What's in the box

| Piece | Package | Role |
|---|---|---|
| Wayfire 0.11 | `wayfire wf-config wf-shell wcm` | the compositor |
| extra plugins | `wayfire-plugins-extra-git`, `wayfire-plugin-pixdecor-git` (AUR) | animations, window decorations |
| `wswitch` | `wswitch` (AUR) or build, see §4 | Alt+Tab window switcher (external, can't crash the compositor) |
| Noctalia | `noctalia` (AUR) | the shell / panel / launcher |
| portals | `xdg-desktop-portal xdg-desktop-portal-wlr xdg-desktop-portal-gtk` | screen-share / file pickers |
| polkit agent | `hyprpolkitagent` (or `lxqt-policykit` / `polkit-gnome`) | auth prompts |
| Discord | `vesktop` | Discord client that can screen-share on wlroots |
| screenshots | `grim slurp satty wl-clipboard` | the `shot` helper script |
| key remap (optional) | `keyd` | tap-Super to open the launcher |

Install the base now:

```sh
sudo pacman -S wayfire wf-config wf-shell wcm \
  xdg-desktop-portal xdg-desktop-portal-wlr xdg-desktop-portal-gtk \
  hyprpolkitagent grim slurp satty wl-clipboard vesktop keyd

# AUR (paru/yay):
paru -S wayfire-plugins-extra-git wayfire-plugin-pixdecor-git noctalia wswitch
```

> `wayfire-plugins-extra-git` / `pixdecor-git` build against `wayfire-git`, but
> the stable `wayfire` 0.11 satisfies the dep. If makepkg complains, build with
> `makepkg -si --nodeps` and install with `pacman -U --nodeps`.

---

## 1. The session

Wayfire is started by a small wrapper that sets the Wayland/Qt/Electron
environment **before** `wayfire` execs. Create it:

**`~/.local/bin/start-wayfire`** (`chmod +x`)

```sh
#!/usr/bin/env bash
export PATH="$HOME/.local/bin:$PATH"
export XDG_SESSION_TYPE=wayland
export XDG_SESSION_DESKTOP=wayfire
export XDG_CURRENT_DESKTOP=Wayfire
export QT_QPA_PLATFORM="wayland;xcb"
export QT_QPA_PLATFORMTHEME=qt6ct
export QT_WAYLAND_DISABLE_WINDOWDECORATION=1
export MOZ_ENABLE_WAYLAND=1
export NIXOS_OZONE_WL=1
export ELECTRON_OZONE_PLATFORM_HINT=auto
export _JAVA_AWT_WM_NONREPARENTING=1
export GDK_BACKEND="wayland,x11"
export SDL_VIDEODRIVER=wayland
export XCURSOR_THEME=Adwaita
export XCURSOR_SIZE=24
exec wayfire
```

Register it as a session so your display manager (SDDM, greetd, ly, GDM) lists
**"Wayfire"**:

**`/usr/share/wayland-sessions/wayfire.desktop`**

```ini
[Desktop Entry]
Name=Wayfire
Exec=/home/YOURNAME/.local/bin/start-wayfire
TryExec=/home/YOURNAME/.local/bin/start-wayfire
Type=Application
DesktopNames=Wayfire;wlroots
```

(If you don't use a display manager, just run `start-wayfire` from a TTY.)

---

## 2. `~/.config/wayfire.ini`

Drop this in whole. Wayfire fills in defaults for any section you omit, and
`auto_reload_config = true` means edits apply on save — no restart.

**Change every `/home/YOURNAME/` path** and the `[output:*]` block (run `wlr-randr`
to get your monitor names/modes).

```ini
[core]
plugins = \
  animate extra-animations autostart blur command vswitch expo \
  foreign-toplevel grid ipc ipc-rules move place resize scale \
  stipc window-rules wm-actions zoom pixdecor cube switcher vswipe \
  force-fullscreen alpha annotate view-shot
background_color = \#1A1A1AFF
close_top_view = <super> KEY_Q | <alt> KEY_F4
xwayland = true
preferred_decoration_mode = server
vwidth = 2
vheight = 2
xkb_layout = us
exit = <alt> <ctrl> KEY_BACKSPACE

# --- monitors: edit to match `wlr-randr` ---
[output:DP-1]
mode = 1920x1080@60000
position = 0,0

[input]
xkb_layout = us
natural_scroll = true
tap_to_click = true
cursor_theme = Adwaita
cursor_size = 24

[autostart]
autostart_wf_shell = false
0_env = dbus-update-activation-environment --systemd WAYLAND_DISPLAY DISPLAY WAYFIRE_SOCKET XDG_CURRENT_DESKTOP=Wayfire XCURSOR_THEME=Adwaita XCURSOR_SIZE=24
0a_polkit = systemctl --user start hyprpolkitagent.service
# Restart BOTH portals after the Wayland env is on the bus — this is what makes
# screen-sharing work (see §5).
1_portal = bash -c 'sleep 2; dbus-update-activation-environment --systemd WAYLAND_DISPLAY DISPLAY XDG_CURRENT_DESKTOP; systemctl --user restart xdg-desktop-portal.service; systemctl --user restart xdg-desktop-portal-wlr.service'
2_shell = noctalia -d
3_wswitch = wswitch --daemon

[command]
# screenshots (the `shot` script from §9)
binding_shot_region = <ctrl> KEY_SYSRQ
command_shot_region = /home/YOURNAME/.local/bin/shot region
binding_shot_full = <super> <shift> KEY_F
command_shot_full = /home/YOURNAME/.local/bin/shot full

# --- Noctalia shell ---
binding_launcher = <super> KEY_D
command_launcher = noctalia msg panel-toggle launcher
# tap Left-Super -> launcher (needs keyd, §8). Remove if you skip keyd.
binding_launcher_tap = KEY_F13
command_launcher_tap = noctalia msg panel-toggle launcher

binding_control = <super> KEY_A
command_control = noctalia msg panel-toggle control-center
binding_clipboard = <super> KEY_V
command_clipboard = noctalia msg panel-toggle clipboard
binding_session = <super> KEY_L
command_session = noctalia msg panel-toggle session
binding_wallpaper = <super> <shift> KEY_W
command_wallpaper = noctalia msg panel-toggle wallpaper
binding_reload_shell = <super> KEY_F5
command_reload_shell = noctalia msg config-reload

# --- utilities ---
binding_term = <super> KEY_ENTER | <super> KEY_T
command_term = kitty --directory ~
binding_files = <super> KEY_E
command_files = xdg-open ~

# --- Alt+Tab: wswitch (external daemon, see §4) ---
binding_alttab = <alt> KEY_TAB
command_alttab = wswitch next
binding_alttab_prev = <alt> <shift> KEY_TAB
command_alttab_prev = wswitch prev

# --- media / volume / brightness (routed through Noctalia for the OSD) ---
binding_media_toggle = KEY_PLAYPAUSE | KEY_PLAYCD | KEY_PLAY
command_media_toggle = noctalia msg media toggle
binding_media_next = KEY_NEXTSONG
command_media_next = noctalia msg media next
binding_media_prev = KEY_PREVIOUSSONG
command_media_prev = noctalia msg media previous
binding_volume_up = KEY_VOLUMEUP
command_volume_up = noctalia msg volume-up
binding_volume_down = KEY_VOLUMEDOWN
command_volume_down = noctalia msg volume-down
binding_mute = KEY_MUTE
command_mute = wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle
binding_brightness_up = KEY_BRIGHTNESSUP
command_brightness_up = noctalia msg brightness-up
binding_brightness_down = KEY_BRIGHTNESSDOWN
command_brightness_down = noctalia msg brightness-down

[wm-actions]
minimize = <super> KEY_M
toggle_fullscreen = <super> KEY_F
toggle_maximize = <super> KEY_UP
toggle_always_on_top = <super> <shift> KEY_Y

[grid]
# Super+arrows = snap the focused window
slot_l = <super> KEY_LEFT
slot_r = <super> KEY_RIGHT
slot_t = <super> KEY_UP
slot_b = <super> KEY_DOWN
duration = 300ms circle

[vswitch]
# Alt+Super+arrows = switch workspace; add Shift to drag the window along
binding_left = <alt> <super> KEY_LEFT
binding_right = <alt> <super> KEY_RIGHT
binding_up = <alt> <super> KEY_UP
binding_down = <alt> <super> KEY_DOWN
with_win_left = <alt> <shift> <super> KEY_LEFT
with_win_right = <alt> <shift> <super> KEY_RIGHT
duration = 300ms circle

[move]
activate = <super> BTN_LEFT | <alt> BTN_LEFT
enable_snap = true

[resize]
activate = <alt> BTN_RIGHT

[expo]
toggle = <super> KEY_W | hotspot left-top 5x5 400

[scale]
# full exposé of the current workspace
toggle = <super> KEY_GRAVE | hotspot left-top 5x5 400
toggle_all = <super> <shift> KEY_GRAVE
hover_focus = true
include_minimized = true
duration = 220ms circle
bg_color = \#12121280
text_color = \#F0F0F2FF
title_overlay = mouse
title_position = bottom
outer_margin = 24
spacing = 36
# leave this false — stock scale's middle-click-close crashes the whole session.
middle_click_close = false

[switcher]
# 3D coverflow on Super+Tab (secondary switcher; Alt+Tab is wswitch, see §4)
next_view = <super> KEY_TAB
prev_view = <super> <shift> KEY_TAB
speed = 250ms circle

# NOTE: Alt+Tab itself is wswitch — an external daemon, not a Wayfire plugin.
# It's started from [autostart] and driven by the binding_alttab commands above.
# See §4. There is no [peek]/switcher plugin section for it.

[animate]
open_animation = carpet
close_animation = carpet
minimize_animation = carpet
duration = 400ms circle

[extra-animations]
carpet_roll_duration = 300ms circle
carpet_roll_direction = 2
# "dodge" rotates an overlapped window aside when you raise another — distracting.
dodge_toggle = false
dodge_rotate = false
dodge_zoom = false

[blur]
method = kawase
kawase_iterations = 3
kawase_offset = 4
saturation = 1.2
blur_by_default = type is "toplevel" | type is "layer"

[pixdecor]
# server-side window decorations. Neutral defaults — restyle to taste in `wcm`.
border_size = 1
titlebar = always
csd_titlebar_height = 26
title_font = sans-serif 10
button_layout = :minimize,maximize,close
effect_type = none
overlay_engine = rounded_corners
rounded_corner_radius = 8

[workarounds]
auto_reload_config = true
# push server-side decorations onto Electron/Chromium apps that force CSD
force_preferred_decoration_mode = true
all_dialogs_modal = true

[expo]
toggle = <super> KEY_W

[view-shot]
capture = KEY_SYSRQ
command = notify-send "Captured %f"
filename = /tmp/snapshot-%F-%T.png
```

### The keybinds you get

| Key | Action |
|---|---|
| `Super+Enter` / `Super+T` | terminal (kitty — swap for your terminal) |
| `Super+D` / tap `Super` | launcher |
| `Super+A` | control centre |
| `Super+V` | clipboard history |
| `Super+L` | session / power menu |
| `Super+E` | file manager |
| `Super+M` | minimise · `Super+F` fullscreen · `Super+Up` maximise |
| `Super+Q` / `Alt+F4` | close window |
| `Super`+drag / `Alt`+drag | move window · `Alt`+right-drag resize |
| `Super`+arrows | snap window to half/edge |
| `Alt+Super`+arrows | switch workspace (`+Shift` drags the window) |
| `Alt+Tab` / `Alt+Shift+Tab` | window switcher (wswitch) — hold Alt, tap Tab, release to pick |
| `Super+Tab` | 3D coverflow switcher |
| `Super+\`` | exposé (all windows on this workspace) |
| `Super+W` | expo (all workspaces) |
| `Ctrl+PrtSc` | region screenshot → annotate |
| `Super+Shift+F` | fullscreen screenshot → annotate |
| `Ctrl+Alt+Backspace` | force-quit the session (escape hatch) |

---

## 3. Noctalia shell

`noctalia -d` is already in `[autostart]`, so it comes up with the session.

**First run:**

1. Log into the Wayfire session. You should get a bar at the top and a dock at
   the bottom.
2. Open the launcher (`Super+D`) → search "Noctalia Settings", or run
   `noctalia msg panel-toggle settings`.
3. In **Appearance** pick a colour scheme (built-in, wallpaper-generated, or a
   community palette), a wallpaper, and fonts. This drives the bar, launcher,
   notifications **and** — via Noctalia's templates — GTK/Qt/kitty/btop colours.
4. In **Bar** / **Dock** arrange the widgets you want.

**Wayfire caveats** (Noctalia is Hyprland-first):

- The **workspace widget** is degraded on Wayfire (no live workspace-occupancy
  feed). The **taskbar** widget works fine — use that for window switching from
  the bar. Minimised windows restore with a click.
- `[dock] toggle_minimize` (click a running dock icon to minimise/restore) is a
  patch in Mario's private Noctalia fork, **not** in the stock package. Stock
  dock icons just focus/raise. Not a big loss.
- Everything else — control centre (WiFi/BT/audio/brightness/night-light/DND/
  power profiles/media), notifications, launcher, clipboard, session menu,
  lock screen, OSDs — works.

> **Do not copy Mario's `~/.local/state/noctalia/settings.toml`.** It's full of
> his monitor layout, his plugins, and a Home-Assistant API token. Configure
> yours from scratch in the GUI.

---

## 4. `wswitch` — the Alt+Tab switcher

Wayfire's own switchers are a 3D coverflow (`switcher`) or a full-screen exposé
(`scale`). For a plain Alt+Tab, **wswitch** draws an icon+title card grid. It's
an **external layer-shell client** (not a compositor plugin), so a bug in it
can't take the session down — which is why it's used here instead of an
in-process plugin.

No live thumbnails — Wayfire 0.11 doesn't expose per-window capture to external
clients, so nothing outside the compositor can show them.

**Install** — AUR is easiest:

```sh
paru -S wswitch          # or: yay -S wswitch
```

Or build from source (deps: `wayland cairo pango json-c libxkbcommon glib2
librsvg` + `wayland-protocols pkgconf gcc`):

```sh
git clone https://github.com/DreamMaoMao/wswitch.git
cd wswitch
make
make install-user        # -> ~/.local/bin/wswitch + ~/.config/wswitch/
#  (or: sudo make install PREFIX=/usr  for a system install)
```

**Configure** `~/.config/wswitch/config.ini`:

```ini
[general]
mode = overview           # flat window list (not "context" workspace grouping)

[theme]
name = snappy-slate.ini   # a file that EXISTS in ~/.config/wswitch/themes/
                          # (the shipped example names a theme that doesn't —
                          #  check `ls ~/.config/wswitch/themes/`)

[icons]
theme = Papirus-Dark
fallback = Papirus
```

**Wiring** is already in the §2 config: the `3_wswitch` autostart line starts
the daemon, and `binding_alttab` / `binding_alttab_prev` drive it. If you did a
**system** install, drop the `~/.local/bin/` path — just `wswitch --daemon` /
`wswitch next`.

**Use:** hold `Alt`, tap `Tab` (or `Shift+Tab`) to cycle, release `Alt` to
select, `Esc` to cancel. CLI verbs: `--daemon`, `next`, `prev`, `toggle`,
`hide`, `quit`. If it ever stops responding: `pkill wswitch; wswitch --daemon &`.

**Zero-dependency fallback:** if wswitch misbehaves, add `fast-switcher` to the
`[core] plugins` line, delete the `binding_alttab*` command lines, and add
`[fast-switcher]` / `activate = <alt> KEY_TAB` — that's Wayfire's built-in plain
switcher, no external process.

---

## 5. Screen-sharing fix (the important one)

On wlroots, screen-share goes `app → xdg-desktop-portal → xdg-desktop-portal-wlr
→ wlr-screencopy`. Two things break it out of the box:

1. **`xdg-desktop-portal-wlr` isn't running in the session.** It's a `static`
   systemd unit with no `[Install]` and D-Bus activation is flaky. → the
   `1_portal` autostart line in §2 explicitly `systemctl --user restart`s both
   `xdg-desktop-portal` and `xdg-desktop-portal-wlr` after the Wayland env is on
   the bus.

2. **The portal doesn't know to use the `wlr` backend.** → create:

**`~/.config/xdg-desktop-portal/wayfire-portals.conf`**

```ini
[preferred]
default=gtk
org.freedesktop.impl.portal.ScreenCast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```

Wayfire's `wlr.portal` file already lists `Wayfire` in `UseIn`, so the config
name must be `wayfire-portals.conf` (matches `XDG_CURRENT_DESKTOP=Wayfire`).

**Verify:** after logging in,
`systemctl --user status xdg-desktop-portal-wlr` should be `active`, and
`grim - | wc -c` should spit out a non-zero byte count.

---

## 6. Discord — Vesktop

The Flatpak Discord and the official client **cannot screen-share on wlroots**.
Use **Vesktop** (already installed in §0). It's Vencord-patched Discord with a
working PipeWire screen-capture path.

**`~/.config/vesktop-flags.conf`**

```
--ozone-platform-hint=auto
--enable-features=WebRTCPipeWireCapturer,UseOzonePlatform,WaylandWindowDecorations
```

In Vesktop, make sure **Settings → Vencord → Plugins → `WebScreenShareFixes`**
is enabled (it is by default).

`~/.config/vesktop/settings.json` — the useful bits:

```json
{
  "discordBranch": "stable",
  "minimizeToTray": true,
  "arRPC": true
}
```

Optional cosmetic: to make it show up as plain **"Discord"** in the launcher,
drop a user override at
`~/.local/share/applications/vesktop.desktop` with `Name=Discord` and
`Icon=com.discordapp.Discord` (needs an icon theme that has the Discord icon,
e.g. `papirus-icon-theme`).

---

## 7. Firefox — video-freeze-while-sharing fix

Separate bug: while you're screen-sharing, a **fullscreen video in Firefox**
freezes (last frame sticks) even though audio keeps going. Fix — in Firefox
`about:config`:

```
widget.wayland.opaque-region.enabled = false
```

Then **fully quit and restart Firefox** (not just close the window).

---

## 8. `keyd` — tap-Super opens the launcher (optional)

Lets a *tap* of Left-Super fire `F13` (bound to the launcher in §2) while
*holding* Super still works as the modifier.

**`/etc/keyd/default.conf`**

```
[ids]
*

[main]
leftmeta = overload(meta, f13)
```

```sh
sudo systemctl enable --now keyd
```

If you skip this, delete the `binding_launcher_tap` / `command_launcher_tap`
lines from `wayfire.ini`. `Super+D` still opens the launcher.

---

## 9. `shot` — screenshot helper (optional)

ShareX-style: capture → annotate in `satty` → save + copy to clipboard.
Bound to `Ctrl+PrtSc` (region) and `Super+Shift+F` (fullscreen) in §2.

**`~/.local/bin/shot`** (`chmod +x`)

```sh
#!/usr/bin/env bash
set -uo pipefail
DIR="$HOME/Pictures/Screenshots"; mkdir -p "$DIR"
OUT="$DIR/ss_$(date +%Y%m%d_%H%M%S).png"
mode="${1:-region}"

case "$mode" in
  region) geo="$(slurp -d)" || exit 0; grim -g "$geo" "$OUT" ;;
  full)   grim "$OUT" ;;
  window) geo="$(slurp -r)" || exit 0; grim -g "$geo" "$OUT" ;;
esac
[ -s "$OUT" ] || { notify-send "shot" "capture failed"; exit 1; }

wl-copy < "$OUT"
satty --filename "$OUT" --output-filename "$OUT" --copy-command wl-copy \
      --initial-tool brush --early-exit --actions-on-enter save-to-clipboard \
      --disable-notifications
notify-send -t 2000 "shot" "saved ${OUT##*/} · copied"
```

---

## 10. Troubleshooting

| Symptom | Fix |
|---|---|
| No bar/dock after login | `noctalia -d &` from a terminal; check `noctalia` is on `PATH` (`which noctalia`). Look at `~/.local/share/noctalia/` logs. |
| Screen-share picker is empty / errors | `systemctl --user restart xdg-desktop-portal xdg-desktop-portal-wlr`; confirm `wayfire-portals.conf` exists and `XDG_CURRENT_DESKTOP=Wayfire`. |
| Alt+Tab does nothing | Is the daemon up? `pgrep wswitch` — if not, `wswitch --daemon &` and check `~/.config/wayfire.ini` has the `binding_alttab` lines. Theme name in `~/.config/wswitch/config.ini` must match a real file in `themes/`. |
| Config change ignored | `[workarounds] auto_reload_config = true` must be set; check the Wayfire log for a parse error (a bad line disables the reload). |
| Electron apps have no titlebar / can't be moved | `[workarounds] force_preferred_decoration_mode = true`; move any window with `Super`+drag regardless. |
| Auth dialogs never appear (mounting drives, etc.) | polkit agent not running: `systemctl --user start hyprpolkitagent`. |
| Qt apps render light/unthemed | `QT_QPA_PLATFORMTHEME=qt6ct` (set in `start-wayfire`); install `qt6ct` and pick a dark style. |
| Session instantly drops back to greeter | usually a crashing plugin — remove the last plugin you added from the `plugins` line, or boot with a minimal `plugins = autostart command ipc ipc-rules move resize`. |

---

## 11. What was deliberately left out

Mario's machine also has, and you don't need for a working desktop:

- **The Endfield theme** — custom palette, Chakra Petch / Jost / JetBrains Mono
  fonts, sharp-everything shape chassis, window-button diamonds, and the private
  `Benylin/noctalia` fork that adds chamfer/skew/bracket motifs. All of that is
  cosmetic and fork-specific.
- **`clockmost-theme`** CLI, the `noctalia-clockmost` dashboard plugin, the
  Home-Assistant widget — tied to Mario's home server.
- **NBDE disk auto-unlock**, `keepassxc-autounlock`, restic backups, Zabbix,
  the PIA VPN generator — his infra, not desktop-shell related.
- `easyeffects` autostart, spicetify, the Steam CEF launch flags — per-app
  tweaks unrelated to Wayfire/Noctalia.
