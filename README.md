# Conky for FuriLabs FLX1s (FuriOS / Phosh)

A full-screen Conky system monitor tuned for the **FuriLabs FLX1s** running **FuriOS** with the **Phosh** shell on Wayland. It renders as a background layer (not a normal window) and shows:

- Clock and date
- System: uptime, battery level and charging status
- CPU: usage and load average
- Memory: RAM and root filesystem usage
- Network: Wi-Fi SSID, IPv4, gateway, IPv6, public IP, traffic
- Logged-in users (`who`)
- Live `dmesg` tail (filtered, time only) filling the rest of the screen

## Requirements

- FuriLabs FLX1s (720 px wide display, Phosh scaling 1.75×, logical width ≈ 411 px)
- Conky built with Wayland support
- `curl` (for the public IP line)

```bash
sudo apt install conky-all curl
```

Verify Wayland support:

```bash
conky -v | grep -i wayland
```

The output must contain `wayland`. Without it, Conky runs through XWayland and Phosh shows it as a normal window.

## Which user?

**Run everything as the regular `furios` user, not as root.**

Conky has to connect to the Phosh session, which belongs to `furios`. Root does not have the session's `WAYLAND_DISPLAY` and `XDG_RUNTIME_DIR`, so Conky would fail to connect to the display. All configuration lives in `~/.config/` of the `furios` user.

`sudo` is only needed to install packages.

## Files

```
~/.config/conky/
├── conky.conf   # main configuration
├── bat.sh       # battery level / status (auto-detects the battery)
├── ipv6.sh      # global IPv6 address of wlan0
└── dmesg.sh     # filtered dmesg tail with time-only timestamps
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
# Filtered dmesg tail; "[Sat Oct 10 21:30:42 2026]" is shortened to "[21:30:42]".
dmesg -T | grep -v -E "(binder: |sd 0:0:0:2: | opfunc_therm_ctrl: )" \
  | sed -E 's/^\[[A-Za-z]{3} [A-Za-z]{3} +[0-9]+ ([0-9:]+) [0-9]{4}\]/[\1]/' \
  | tail -36 | cut -c1-78
EOF

chmod +x ~/.config/conky/*.sh
```

### 2. Conky configuration

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

    -- Logical width ~411 px (720 px / 1.75 scaling)
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

### 3. First run (foreground test)

```bash
conky -c ~/.config/conky/conky.conf
```

Check the screen. Lines such as `FOUND: wayland`, `drawing to single buffer`, `Fontconfig warning` or `'unknown' tty session running` are informational only. Stop with `Ctrl+C`.

### 4. Run in the background

```bash
conky -q -d -c ~/.config/conky/conky.conf >/dev/null 2>&1
```

- `-d` detaches Conky from the terminal
- `-q` and the redirect keep it from printing into the terminal

Stop it with:

```bash
pkill -x conky
```

Restart after editing the config:

```bash
pkill -x conky; sleep 1; conky -q -d -c ~/.config/conky/conky.conf >/dev/null 2>&1
```

### 5. Autostart

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/conky.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Conky
Exec=sh -c "sleep 10 && conky -q -d -c ~/.config/conky/conky.conf >/dev/null 2>&1"
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF
```
