# Conky for FuriLabs FLX1s (FuriOS / Phosh)

A full-screen Conky system monitor tuned for the **FuriLabs FLX1s** running **FuriOS** with the **Phosh** shell on Wayland. It renders as a transparent background layer (not a normal window), is shown only in portrait orientation and pauses itself whenever it cannot be seen, to save battery.

What it shows (portrait):

- Clock and date
- System: uptime, battery level and charging status (bar turns red below 20 %)
- CPU: usage, load average and usage bar
- Memory: RAM and root filesystem usage
- Network: Wi-Fi SSID, IPv4, gateway, IPv6, public IP
- Network graph: download / upload speed with history graphs
- Logged-in users (`who`, max. 10 lines)
- Syslog: kernel log (`dmesg`) and system journal merged by time, filling the rest of the screen

Behaviour:

| Situation | Conky |
|---|---|
| Portrait, home screen | running |
| Portrait, an app in the foreground | paused (`SIGSTOP`), no CPU |
| Screen locked / display off | paused (`SIGSTOP`), no CPU |
| Landscape | stopped (hidden before Phosh finishes the rotation) |
| Rotation lock enabled | tilting the phone changes nothing |

## Requirements

- FuriLabs FLX1s (720×1600 display, Phosh scaling 1.75×)
- The packages below (most of them are already part of FuriOS)

```bash
sudo apt install conky-std curl wlr-randr wlrctl iio-sensor-proxy \
  libglib2.0-bin fonts-dejavu-core fonts-noto-mono
```

| Package | Used for |
|---|---|
| `conky-std` (or `conky-all`) | Conky with Wayland output |
| `curl` | public IP line |
| `wlr-randr` | real display rotation and on/off state |
| `wlrctl` | foreground app detection |
| `iio-sensor-proxy` | `monitor-sensor` – accelerometer, early hide on rotation |
| `libglib2.0-bin` | `gdbus`, `gsettings` – Phosh D-Bus signals, rotation lock |
| `fonts-dejavu-core`, `fonts-noto-mono` | DejaVu Sans / Sans Mono, Noto Mono |
| `util-linux`, `procps`, `coreutils`, `mawk`, `bash` | `flock`, `dmesg`, `pgrep`/`pkill`, `stdbuf`, `sort`, `awk` (base system) |

Verify Wayland support:

```bash
conky -v | grep -i wayland
```

The output must contain `wayland`. Without it, Conky runs through XWayland and Phosh shows it as a normal window.

## Which user?

**Run everything as the regular `furios` user, not as root.**

Conky has to connect to the Phosh session, which belongs to `furios`. Root does not have the session's `WAYLAND_DISPLAY` and `XDG_RUNTIME_DIR`, so Conky would fail to connect to the display. All configuration lives in `~/.config/` of the `furios` user.

`sudo` is only needed to install packages and for the permission setup below.

## Permissions (journal and kernel log)

The syslog block reads the system journal and the kernel ring buffer as the `furios` user.

**System journal** – only root and the groups `systemd-journal` / `adm` can read it:

```bash
sudo usermod -aG systemd-journal furios
```

The group applies after a new login, so **reboot the phone** afterwards (the Phosh session that starts Conky must get the group too). Check:

```bash
id | grep -o systemd-journal
journalctl -b -n 5 --no-pager
```

**Kernel log** – FuriOS sets `ReadKMsg=no` in journald, so the journal contains no kernel messages. Instead of changing that (it would duplicate the kernel log), the script reads the kernel ring buffer directly with `dmesg`. This works as a normal user when `kernel.dmesg_restrict` is `0`:

```bash
sysctl kernel.dmesg_restrict
dmesg | tail -3
```

If it is `1`, allow it:

```bash
echo 'kernel.dmesg_restrict = 0' | sudo tee /etc/sysctl.d/90-dmesg.conf
sudo sysctl --system
```

No `rsyslog` is needed. The journal on FuriOS is kept in RAM (`10-journald-volatile.conf`), so nothing extra is written to flash.

## Files

```
~/.config/conky/
├── conky.conf        # layout (portrait)
├── conky-orient.sh   # controller: rotation, screen lock, foreground app, single instance
├── syslog.sh         # dmesg + journal merged by time
├── bat.sh            # battery level / status (auto-detects the battery)
└── ipv6.sh           # global IPv6 address of wlan0
~/.config/autostart/
└── conky.desktop
```

Helper commands are kept in separate scripts on purpose: Conky treats `{`, `}` and `$` as part of its own variable syntax, so commands using `awk '{...}'` break when written inline in `conky.text`.

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

chmod +x ~/.config/conky/*.sh
```

### 2. Syslog script

```bash
cat > ~/.config/conky/syslog.sh <<'EOF'
#!/bin/bash
# syslog.sh - kernel log (dmesg) + system journal merged by time, for Conky
#   no extra logging: journald keeps ReadKMsg=no (FuriOS default), the kernel
#   messages are read directly from the kernel ring buffer
#
# Requires:
#   systemd    - journalctl
#   group systemd-journal (or adm) - read the system journal:
#                sudo usermod -aG systemd-journal furios   (then re-login / reboot)
#   util-linux - dmesg (kernel ring buffer must be readable: kernel.dmesg_restrict=0)
#   coreutils  - sort, tail, who, wc
#   mawk/gawk  - awk
#   bash       - process substitution
#
# usage: syslog.sh [lines] [columns] [who_base]
#   lines     number of syslog lines when LOGGED IN shows who_base lines
#   who_base  optional: adapt [lines] to the current number of `who` lines, so the
#             syslog block always ends at the same place on screen
#   "2026-10-12T10:52:03+02:00 proc[1]: msg"   -> "[10:52:03] proc[1]: msg"
#   "2026-10-12T10:52:03,123456+02:00 msg"     -> "[10:52:03] kernel: msg"
#   lines longer than MAXLEN are cut before processing, output is cut to [columns]

MAXLEN=500
WHO_MAX=10           # must match `who | head -n 10` in conky.conf
WHO_NUM=7; WHO_DEN=6 # one `who` line (font size 7) = 7/6 syslog lines (font size 6)

N="${1:-36}"
C="${2:-78}"

if [ -n "${3:-}" ]; then
  W=$(who | wc -l)
  [ "$W" -gt "$WHO_MAX" ] && W=$WHO_MAX
  E=$((W - $3))                                          # extra who lines vs. base
  if [ "$E" -gt 0 ]; then
    N=$((N - (E * WHO_NUM + WHO_DEN - 1) / WHO_DEN))     # round up: never overflow
  else
    N=$((N - (E * WHO_NUM) / WHO_DEN))                   # fewer users: more syslog lines
  fi
  [ "$N" -lt 1 ] && N=1
fi

sort -m -s -k1.1,1.19 \
  <(dmesg --time-format=iso 2>/dev/null | tail -n 200) \
  <(journalctl -b -q -n 200 --no-pager --no-hostname -o short-iso 2>/dev/null) \
| awk -v n="$N" -v c="$C" -v max="$MAXLEN" '
  { $0 = substr($0, 1, max) }
  !/^[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]T/ { next }
  /binder: |sd 0:0:0:2: | opfunc_therm_ctrl: |callbacks suppressed/ { next }
  {
    t   = substr($0, 12, 8)
    src = (substr($0, 20, 1) == ",") ? "kernel: " : ""
    sub(/^[^ ]+ /, "")
    $0 = "[" t "] " src $0
    i++; buf[i % n] = substr($0, 1, c)
  }
  END { for (j = (i > n ? i - n + 1 : 1); j <= i; j++) print buf[j % n] }
'
EOF

chmod +x ~/.config/conky/syslog.sh
```

How it works:

- Kernel messages come from `dmesg`, service messages from `journalctl`. Both streams are already time-ordered, so `sort -m` only merges them (no real sorting).
- Kernel lines are recognised by the fractional seconds in their timestamp and get the `kernel:` prefix, like a classic syslog.
- Everything else (filtering, timestamp shortening, keeping the last N lines) is done in a single `awk`, so one refresh costs only a few short processes every 10 s.
- Noise is filtered by the `/binder: |sd 0:0:0:2: |.../` pattern – extend it as needed.
- With the third argument the number of syslog lines follows the number of logged-in users, so the block always ends above the bottom edge of the screen.

Test (after the reboot from the permissions step):

```bash
~/.config/conky/syslog.sh 15 120
```

### 3. Conky configuration

```bash
cat > ~/.config/conky/conky.conf <<'EOF'
conky.config = {
    out_to_wayland = true,
    out_to_x = false,
    out_to_console = false,

    own_window = true,
    own_window_type = 'desktop',
    own_window_transparent = true,
    own_window_argb_visual = true,
    own_window_argb_value = 0,
    own_window_colour = '000000',
    own_window_hints = 'undecorated,below,sticky,skip_taskbar,skip_pager',

    -- logical width ~411 px (720 / 1.75)
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
    draw_shades = true,              -- subtle shadow for readability on the wallpaper
    default_shade_color = '000000',
    draw_outline = false,
    draw_borders = false,
    default_bar_height = 6,

    -- palette
    default_color = 'EEEEEE',        -- values
    color1 = '7FD4FF',               -- headings, CPU
    color2 = '8E9AA6',               -- labels
    color3 = 'B8C4CE',               -- log text
    color4 = '3D6F8C',               -- separator lines
    color5 = '5FE08A',               -- battery, download
    color6 = 'FFB347',               -- upload
    color7 = 'B58CFF',               -- memory, disk
    color8 = 'FF5F5F',               -- warning (low battery)

    update_interval = 1,
    cpu_avg_samples = 1,
    net_avg_samples = 1,
    no_buffers = true,
    text_buffer_size = 16386,
};

conky.text = [[
${alignc}${font DejaVu Sans:size=32}${color1}${time %H:%M}${font}${color}
${alignc}${font DejaVu Sans:size=10}${time %A %d. %B %Y}${font}
${color1}SYSTEM ${color4}${hr 1}${color}
${color2}Uptime:${color}${alignr}${uptime_short}
${color2}Battery:${color}${alignr}${execi 30 ~/.config/conky/bat.sh} % ${color2}(${execi 30 ~/.config/conky/bat.sh status})${color}
${if_match ${execi 30 ~/.config/conky/bat.sh} < 20}${color8}${else}${color5}${endif}${execibar 30 ~/.config/conky/bat.sh}${color}
${color1}CPU ${color4}${hr 1}${color}
${color2}Usage:${color} ${cpu cpu0} %${alignr}${color2}Load:${color} ${loadavg}
${color1}${cpubar cpu0}${color}
${color1}MEMORY ${color4}${hr 1}${color}
${color2}RAM:${color}${alignr}${mem} / ${memmax}
${color7}${membar}${color}
${color2}Disk /:${color}${alignr}${fs_used /} / ${fs_size /}
${color7}${fs_bar /}${color}
${color1}NETWORK ${color4}${hr 1}${color}
${color2}Wi-Fi:${color}${alignr}${wireless_essid wlan0}
${color2}IPv4:${color}${alignr}${addr wlan0}
${color2}Gateway:${color}${alignr}${gw_ip}
${color2}IPv6:${color}${alignr}${font DejaVu Sans Mono:size=7}${execi 60 ~/.config/conky/ipv6.sh}${font}
${color2}Public:${color}${alignr}${execi 900 curl -s -4 --max-time 5 ifconfig.me}${voffset 3}
${color1}NETWORK GRAPH ${color4}${hr 1}${color}
${color2}Download: ${color5}${downspeed wlan0}${alignr}${color2}Upload: ${color6}${upspeed wlan0}${color}
${downspeedgraph wlan0 25,180 5FE08A 5FE08A}${alignr}${upspeedgraph wlan0 25,180 FFB347 FFB347}
${voffset -5}${color1}LOGGED IN ${color4}${hr 1}${color}
${font DejaVu Sans Mono:size=7}${execi 10 who | head -n 10 | cut -c1-66}${font}
${color1}SYSLOG ${color4}${hr 1}${color3}
${font Noto Mono:size=6}${execi 10 ~/.config/conky/syslog.sh 35 77 3}${font}${color}
]];
EOF
```

`syslog.sh 35 400 3` means: 35 syslog lines when 3 users are logged in, lines cut to 400 characters (Conky clips them at the screen edge). Raise or lower `35` until the last line ends just above the bottom edge.

### 4. Controller

```bash
cat > ~/.config/conky/conky-orient.sh <<'EOF'
#!/bin/bash
# Conky controller for FuriPhone FLX1s:
#  - early hide: accelerometer (monitor-sensor) reports landscape -> Conky is stopped
#    before Phosh finishes the rotation
#  - the real display rotation (org.gnome.Mutter.DisplayConfig + wlr-randr) decides
#  - rotation lock: read from gsettings at startup and on change; additionally learned
#    when the display does not follow the accelerometer
#  - portrait: shows conky.conf; any other orientation: no Conky
#  - pauses Conky while the screen is locked / display off / an app is in the foreground
#  - single instance only (flock), Conky is never started twice
#
# Requires (Debian packages):
#   conky-std / conky-all - Conky built with Wayland output
#   wlr-randr             - output transform (rotation) and enabled state
#   wlrctl                - foreground app detection (optional, APP_WATCH=1)
#   iio-sensor-proxy      - monitor-sensor (accelerometer, early hide)
#   libglib2.0-bin        - gdbus, gsettings (Phosh D-Bus signals, rotation lock)
#   util-linux            - flock
#   procps                - pgrep, pkill
#   coreutils             - stdbuf
#   bash
DIR="$HOME/.config/conky"
APP_WATCH=1    # 0 = disable foreground-app detection
POLL=0.4       # foreground-app check interval (seconds), only while Conky is visible
RUN="${XDG_RUNTIME_DIR:-/tmp}"
GATE="$RUN/conky-orient.poll"
LOCKKEY="org.gnome.settings-daemon.peripherals.touchscreen orientation-lock"

exec 9>"$RUN/conky-orient.lock"
if ! flock -n 9; then
  echo "conky-orient.sh is already running" >&2
  exit 1
fi

exec {SNOOZE_FD}<> <(:)
snooze() { read -rt "$1" -u "$SNOOZE_FD" _ || :; }

echo 0 > "$GATE"

start_conky() {
  local n=0
  while pgrep -x conky >/dev/null && [ "$n" -lt 50 ]; do
    snooze 0.1; n=$((n + 1))
  done
  pgrep -x conky >/dev/null || conky -q -d -c "$1" 9>&-
}

display_state() {  # sets ST to off / portrait / landscape
  local k v en="" tr=""
  while read -r k v _; do
    case "$k" in
      Enabled:)   en=$v ;;
      Transform:) tr=$v ;;
    esac
  done < <(wlr-randr 2>/dev/null 9>&-)
  if [ "$en" != yes ]; then ST=off
  else
    case "$tr" in
      90|270|flipped-90|flipped-270) ST=landscape ;;
      *) ST=portrait ;;
    esac
  fi
}

{
  echo "DISPLAY_CHANGED"
  stdbuf -oL monitor-sensor --accel 9>&- &
  stdbuf -oL gsettings monitor $LOCKKEY 9>&- &
  stdbuf -oL gdbus monitor --session --dest org.gnome.Mutter.DisplayConfig \
    --object-path /org/gnome/Mutter/DisplayConfig 9>&- &
  stdbuf -oL gdbus monitor --session --dest org.gnome.ScreenSaver \
    --object-path /org/gnome/ScreenSaver 9>&- &
  if [ "$APP_WATCH" = 1 ] && command -v wlrctl >/dev/null; then
    (
      last=""
      while :; do
        g=0; read -r g < "$GATE" 2>/dev/null
        if [ "$g" = 1 ]; then
          if wlrctl toplevel find state:active >/dev/null 2>&1 9>&-; then s=1; else s=0; fi
          [ "$s" != "$last" ] && { echo "APP $s"; last=$s; }
          snooze "$POLL"
        else
          snooze 1
        fi
      done
    ) &
  fi
  wait
} | {
  mode=""; locked=0; app=0; screenoff=0; paused=""; gate=0
  # rotation lock: from gsettings, or learned when the display does not follow
  rotlock=0; [ "$(gsettings get $LOCKKEY 2>/dev/null)" = true ] && rotlock=1
  update() {
    local want=0 g=0
    if [ "$locked" = 1 ] || [ "$screenoff" = 1 ] || [ "$app" = 1 ]; then want=1; fi
    if [ "$want" != "$paused" ]; then
      if [ "$want" = 1 ]; then pkill -STOP -x conky; else pkill -CONT -x conky; fi
      paused=$want
    fi
    if [ "$mode" = portrait ] && [ "$locked" = 0 ] && [ "$screenoff" = 0 ]; then g=1; fi
    if [ "$g" != "$gate" ]; then echo "$g" > "$GATE"; gate=$g; fi
  }
  while read -r line; do
    case "$line" in
      *orientation-lock*true*)  rotlock=1; continue ;;
      *orientation-lock*false*) rotlock=0; continue ;;
      *ActiveChanged*true*)  locked=1; update; continue ;;
      *ActiveChanged*false*) locked=0; update; continue ;;
      "APP 1")               app=1;    update; continue ;;
      "APP 0")               app=0;    update; continue ;;
      *left-up*|*right-up*)
        # early hide: the display is about to rotate to landscape
        if [ "$rotlock" = 0 ] && [ "$mode" = portrait ] && [ "$locked" = 0 ] && [ "$screenoff" = 0 ]; then
          pkill -CONT -x conky; pkill -x conky; paused=""
          i=0
          while [ "$i" -lt 10 ]; do        # wait max 2 s for the display to follow
            snooze 0.2; display_state
            [ "$ST" = landscape ] && break
            i=$((i + 1))
          done
          if [ "$ST" = landscape ]; then
            mode=landscape
          else                             # display did not rotate: rotation lock is on
            rotlock=1
            start_conky "$DIR/conky.conf"; paused=0; snooze 0.3
          fi
          update
        fi
        continue ;;
      *normal*|*bottom-up*) continue ;;  # back to portrait: wait for the display
      DISPLAY_CHANGED|*MonitorsChanged*|*PowerSaveMode*)
        display_state
        if [ "$ST" = off ]; then screenoff=1; update; continue; fi
        screenoff=0; new=$ST
        [ "$ST" = landscape ] && rotlock=0  # display rotated: rotation lock is off
        ;;
      *) continue ;;
    esac
    if [ "$new" = "$mode" ]; then update; continue; fi
    mode=$new
    pkill -CONT -x conky; pkill -x conky; paused=""
    if [ "$mode" = portrait ]; then
      start_conky "$DIR/conky.conf"
      paused=0; snooze 0.3
    fi
    update
  done
}
EOF

chmod +x ~/.config/conky/conky-orient.sh
```

How it works:

- **Events, not polling.** Several monitors are merged into one pipe and handled by a single loop:
  - `monitor-sensor --accel` – accelerometer orientation (via `iio-sensor-proxy`; on the FLX1s the sensor is not available in `/sys`, it goes through the Android HAL),
  - `gdbus monitor … org.gnome.Mutter.DisplayConfig` – `MonitorsChanged` when Phosh really rotates the display or switches it off,
  - `gdbus monitor … org.gnome.ScreenSaver` – `ActiveChanged (true/false)` on lock / unlock,
  - `gsettings monitor … orientation-lock` – the rotation lock toggle in the Phosh menu,
  - `wlrctl toplevel find state:active` – an app in the foreground, polled every 0.4 s, but **only while Conky could be visible** (portrait, unlocked, display on).
- **No flicker on rotation.** The display rotation is a compositor state (Phoc); the kernel never sees it, so there is nothing to read in `/sys` or `/proc`. The accelerometer event arrives *before* Phosh rotates the display, so Conky is stopped early; `wlr-randr` then confirms the real rotation. If the display does not follow (rotation lock), Conky comes back.
- **Pausing instead of killing.** Hidden Conky is paused with `SIGSTOP` and resumed with `SIGCONT`, so it continues instantly. A stopped process ignores `SIGTERM`, so it is always woken up (`SIGCONT`) before being terminated.
- **Single instance.** `flock` on `$XDG_RUNTIME_DIR/conky-orient.lock` refuses a second controller; child processes do not inherit the lock (`9>&-`). Conky itself is started only when no instance is running.
- **Cheap waiting.** `snooze` uses a bash `read` timeout instead of spawning `sleep`.

### 5. First run (foreground test)

```bash
conky -c ~/.config/conky/conky.conf
```

Lines such as `FOUND: wayland`, `drawing to single buffer`, `Fontconfig warning` or `'unknown' tty session running` are informational only. Stop with `Ctrl+C`.

### 6. Start in the background

```bash
nohup ~/.config/conky/conky-orient.sh >/dev/null 2>&1 &
```

A second start is refused (`conky-orient.sh is already running`).

Stop everything:

```bash
pkill -f conky-orient.sh; pkill -f "monitor-sensor --accel"; pkill -f "gsettings monitor"; \
pkill -f "gdbus monitor --session --dest org.gnome"; pkill -CONT -x conky; pkill -x conky
```

Changes in `conky.conf` are picked up by Conky automatically; changes in `conky-orient.sh` need a stop and a new start.

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

The 10-second delay gives Phosh time to start.

### 8. Verify (optional)

Detection sources:

```bash
monitor-sensor --accel                       # rotate the phone: left-up / normal ...
wlr-randr | grep -E 'Enabled|Transform'      # Transform: normal / 90 / 270 / 180
gdbus monitor --session --dest org.gnome.ScreenSaver --object-path /org/gnome/ScreenSaver
gsettings get org.gnome.settings-daemon.peripherals.touchscreen orientation-lock
wlrctl toplevel find state:active; echo $?   # 0 = an app is in the foreground
```

Power saving – lock the phone (or open an app) and run over SSH:

```bash
P=$(pgrep -x conky)
awk '{print "CPU ticks:", $14+$15}' /proc/$P/stat; sleep 10
awk '{print "CPU ticks:", $14+$15}' /proc/$P/stat
grep State /proc/$P/status
```

While hidden, both numbers are identical (no CPU time used in 10 s) and the state is `T (stopped)`. The `%CPU` column of `ps` is the average over the whole process lifetime, so it does not drop to zero while Conky is paused.

## Customisation

| What | Where |
|---|---|
| Colours | `color1` … `color8` in `conky.conf` (separator lines: `color4`) |
| Number of syslog lines | first number in `syslog.sh 35 400 3` |
| Max. logged-in lines | `head -n 10` in `conky.conf` **and** `WHO_MAX` in `syslog.sh` |
| Syslog noise filter | `/binder: \|sd 0:0:0:2: \|…/` pattern in `syslog.sh` |
| Low battery warning | `< 20` in the battery bar line |
| Foreground app detection | `APP_WATCH` / `POLL` in `conky-orient.sh` |
| Box size and margins | `minimum_width`, `maximum_width`, `gap_y`, `minimum_height` |
