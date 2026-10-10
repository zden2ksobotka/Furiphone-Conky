# Conky for FuriLabs FLX1s (FuriOS / Phosh)

A full-screen Conky system monitor tuned for the **FuriLabs FLX1s** running **FuriOS** with the **Phosh** shell on Wayland. It renders as a background layer (not a normal window) and automatically switches layout when the phone is rotated.

**Portrait** – full system overview:

- Clock and date
- System: uptime, battery level and charging status
- CPU: usage and load average
- Memory: RAM and root filesystem usage
- Network: Wi-Fi SSID, IPv4, gateway, IPv6, public IP, traffic
- Logged-in users (`who`)
- Live `dmesg` tail filling the rest of the screen

**Landscape** – wide live `dmesg` view only.

## Requirements

- FuriLabs FLX1s (720×1600 display, Phosh scaling 1.75×)
- Conky built with Wayland support
- `iio-sensor-proxy` (provides `monitor-sensor` for rotation detection)
- `curl` (for the public IP line)

```bash
sudo apt install conky-all curl
```

Verify Wayland support:

```bash
conky -v | grep -i wayland
```

The output must contain `wayland`. Without it, Conky runs through XWayland and Phosh shows it as a normal window.

Verify rotation detection (rotate the phone, then `Ctrl+C`):

```bash
monitor-sensor --accel
```

You should see lines such as `Accelerometer orientation changed: left-up`.

## Which user?

**Run everything as the regular `furios` user, not as root.**

Conky has to connect to the Phosh session, which belongs to `furios`. Root does not have the session's `WAYLAND_DISPLAY` and `XDG_RUNTIME_DIR`, so Conky would fail to connect to the display. All configuration lives in `~/.config/` of the `furios` user.

`sudo` is only needed to install packages.

## Files

```
~/.config/conky/
├── conky.conf        # portrait layout
├── landscape.conf    # landscape layout (dmesg only)
├── conky-orient.sh   # watches rotation and switches layouts
├── bat.sh            # battery level / status (auto-detects the battery)
├── ipv6.sh           # global IPv6 address of wlan0
└── dmesg.sh          # filtered dmesg tail: dmesg.sh [lines] [columns]
~/.config/autostart/
└── conky.desktop
```

Helper commands are kept in separate scripts on purpose: Conky treats `{`, `}` and `$` as part of its own variable syntax, so commands using `awk '{...}'` or `xargs -I{}` break when written inline in `conky.text`.

## Installation

### 1. Helper scripts

```bash
mkdir -p ~/.config/conky

cat > ~/.config/conky/bat.sh <<'EOF'
#!/bin/sh
# Prints battery capacity (default) or status ("status" argument).
for b in /sys/class/power_supply/*; do
  [ "$(cat "$b/type" 2>/dev/null)" = Battery ] || continue
  [ "$1" = status ] && cat "$b/status" || cat "$b/capacity"
  exit 0
done
echo "?"
EOF

cat > ~/.config/conky/ipv6.sh <<'EOF'
#!/bin/sh
# Prints the first global IPv6 address of wlan0.
ip -6 -o addr show wlan0 scope global | awk '{print $4; exit}'
EOF

cat > ~/.config/conky/dmesg.sh <<'EOF'
#!/bin/sh
# usage: dmesg.sh [lines] [columns]
# Filtered dmesg tail; "[Sat Oct 10 21:30:42 2026]" is shortened to "[21:30:42]".
dmesg -T | grep -v -E "(binder: |sd 0:0:0:2: | opfunc_therm_ctrl: )" \
  | sed -E 's/^\[[A-Za-z]{3} [A-Za-z]{3} +[0-9]+ ([0-9:]+) [0-9]{4}\]/[\1]/' \
  | tail -n "${1:-36}" | cut -c1-"${2:-78}"
EOF

chmod +x ~/.config/conky/*.sh
```

### 2. Portrait configuration

```bash
cat > ~/.config/conky/conky.conf <<'EOF'
conky.config = {
    out_to_wayland = true,
    out_to_x = false,
    out_to_console = false,

    own_window = true,
    own_window_type = 'desktop',
    own_window_transparent = false,
    own_window_argb_visual = true,
    own_window_argb_value = 160,
    own_window_colour = '000000',
    own_window_hints = 'undecorated,below,sticky,skip_taskbar,skip_pager',

    -- Portrait: logical width ~411 px (720 px / 1.75 scaling)
    alignment = 'top_middle',
    gap_x = 0,
    gap_y = 13,
    minimum_width = 388,
    maximum_width = 388,
    minimum_height = 828,
    border_inner_margin = 4,

    use_xft = true,
    font = 'DejaVu Sans:size=9',
    override_utf8_locale = true,
    double_buffer = true,
    draw_shades = false,
    draw_outline = false,
    draw_borders = false,
    default_bar_height = 6,
    default_color = 'EEEEEE',
    color1 = '7FD4FF',
    color2 = '999999',
    color3 = 'B0B0B0',

    update_interval = 2,
    cpu_avg_samples = 2,
    net_avg_samples = 2,
    no_buffers = true,
    text_buffer_size = 8192,
};

conky.text = [[
${alignc}${font DejaVu Sans:size=32}${color1}${time %H:%M}${font}${color}
${alignc}${font DejaVu Sans:size=10}${time %A %d. %B %Y}${font}
${color1}SYSTEM ${color2}${hr 1}${color}
${color2}Uptime:${color}${alignr}${uptime_short}
${color2}Battery:${color}${alignr}${execi 30 ~/.config/conky/bat.sh} % ${color2}(${execi 30 ~/.config/conky/bat.sh status})${color}
${execibar 30 ~/.config/conky/bat.sh}
${color1}CPU ${color2}${hr 1}${color}
${color2}Usage:${color}${alignr}${cpu cpu0} %
${cpubar cpu0}
${color2}Load:${color}${alignr}${loadavg}
${color1}MEMORY ${color2}${hr 1}${color}
${color2}RAM:${color}${alignr}${mem} / ${memmax}
${membar}
${color2}Disk /:${color}${alignr}${fs_used /} / ${fs_size /}
${fs_bar /}
${color1}NETWORK ${color2}${hr 1}${color}
${color2}Wi-Fi:${color}${alignr}${wireless_essid wlan0}
${color2}IPv4:${color}${alignr}${addr wlan0}
${color2}Gateway:${color}${alignr}${gw_ip}
${color2}IPv6:${color}${alignr}${font DejaVu Sans Mono:size=7}${execi 60 ~/.config/conky/ipv6.sh}${font}
${color2}Public:${color}${alignr}${execi 900 curl -s -4 --max-time 5 ifconfig.me}
${color2}↓${color} ${downspeed wlan0}${alignr}${color2}↑${color} ${upspeed wlan0}
${color1}LOGGED IN ${color2}${hr 1}${color}
${font DejaVu Sans Mono:size=7}${execi 15 who | cut -c1-66}${font}
${color1}DMESG ${color2}${hr 1}${color3}
${font DejaVu Sans Mono:size=6}${execi 10 ~/.config/conky/dmesg.sh}${font}
]];
EOF
```

### 3. Landscape configuration

```bash
cat > ~/.config/conky/landscape.conf <<'EOF'
conky.config = {
    out_to_wayland = true, out_to_x = false, out_to_console = false,
    own_window = true, own_window_type = 'desktop',
    own_window_transparent = false, own_window_argb_visual = true,
    own_window_argb_value = 160, own_window_colour = '000000',
    own_window_hints = 'undecorated,below,sticky,skip_taskbar,skip_pager',

    -- Landscape: logical ~914 x 411, minus the top bar
    alignment = 'top_middle',
    gap_x = 0, gap_y = 6,
    minimum_width = 880, maximum_width = 880,
    minimum_height = 340,
    border_inner_margin = 4,

    use_xft = true, font = 'DejaVu Sans:size=9',
    override_utf8_locale = true, double_buffer = true,
    draw_shades = false, draw_outline = false, draw_borders = false,
    default_color = 'EEEEEE', color1 = '7FD4FF', color2 = '999999', color3 = 'B0B0B0',
    update_interval = 2, no_buffers = true, text_buffer_size = 32768,
};

conky.text = [[
${color1}DMESG ${color2}${hr 1}${color3}
${font DejaVu Sans Mono:size=6}${execi 10 ~/.config/conky/dmesg.sh 30 170}${font}
]];
EOF
```

### 4. Rotation watcher

```bash
cat > ~/.config/conky/conky-orient.sh <<'EOF'
#!/bin/sh
# Switches Conky layout on accelerometer orientation changes (iio-sensor-proxy).
DIR="$HOME/.config/conky"
last=""
stdbuf -oL monitor-sensor --accel | while read -r line; do
  case "$line" in
    *left-up*|*right-up*) mode=landscape ;;
    *normal*|*bottom-up*) mode=portrait ;;
    *) continue ;;
  esac
  [ "$mode" = "$last" ] && continue
  pkill -x conky; sleep 1
  if [ "$mode" = landscape ]; then
    conky -q -d -c "$DIR/landscape.conf"
  else
    conky -q -d -c "$DIR/conky.conf"
  fi
  last=$mode
done
EOF

chmod +x ~/.config/conky/conky-orient.sh
```

How it works:

- On the FLX1s the accelerometer is not exposed as a plain file in `/sys`; sensors go through the Android HAL. `iio-sensor-proxy` computes the orientation and publishes it over D-Bus, and `monitor-sensor` prints every change.
- The script reacts only to events, so it uses no CPU while idle.
- `stdbuf -oL` forces line-buffered output; without it `monitor-sensor` buffers its output when piped and rotation events arrive late.
- The initial `=== Has accelerometer (orientation: ...)` line selects the correct layout right at startup.

### 5. First run (foreground test)

```bash
conky -c ~/.config/conky/conky.conf
```

Lines such as `FOUND: wayland`, `drawing to single buffer`, `Fontconfig warning` or `'unknown' tty session running` are informational only. Stop with `Ctrl+C`.

### 6. Start in the background

```bash
nohup ~/.config/conky/conky-orient.sh >/dev/null 2>&1 &
```

Stop everything:

```bash
pkill -f conky-orient.sh; pkill -x conky
```

Reload a layout after editing it (the watcher keeps running):

```bash
pkill -x conky; sleep 1; conky -q -d -c ~/.config/conky/conky.conf >/dev/null 2>&1       # portrait
pkill -x conky; sleep 1; conky -q -d -c ~/.config/conky/landscape.conf >/dev/null 2>&1   # landscape
```

### 7. Autostart

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/conky.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Conky
Exec=sh -c "sleep 10 && ~/.config/conky/conky-orient.sh >/dev/null 2>&1"
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF
```
