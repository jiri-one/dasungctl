# dasungctl

Control the front light and display settings of **Dasung Paperlike** e-ink monitors from Linux, in single
brightness steps and below the darkest level the monitor's own buttons reach.

The buttons on the tested 25.3" monitor move the front light in big jumps, and the dimmest setting they reach is
10 %. Over USB the monitor accepts every value from 0 to 100, which is all it takes for a "just a little less"
setting.

## Tested on

dasungctl has been tested on exactly these two monitors, both at the same time on one computer:

| Monitor | Made | Reports itself as | Firmware (`dasungctl info`) | Connected by |
|---|---|---|---|---|
| Dasung Paperlike Color, 25.3" | 2022 | EDID `DSC Paperlike253C`; no touch controller on USB | MCU 31.11 | DisplayPort + USB-B |
| Dasung Paperlike 13.3" with front light and touch | 2024 | EDID `DSC Paperlike H D`; touch controller `27c0:0818` | MCU 30.10 | HDMI + USB |

Brightness, light colour, `tune`, and key bindings with held keys and notifications were used on both. Contrast,
refresh speed and clearing ghosting were tried on the 13.3" only; image mode and colour temperature are sent the
way the official clients send them but have not been tried. The computer ran Manjaro Linux with kernel 6.18,
niri 26.04, mako 1.11 and Python 3.14; nothing in dasungctl is specific to that setup.

Dasung's clients drive their other current models (Paperlike 253, 103, 13K) through the same code path, so they
will probably work too, but nobody has tried yet. If you have one, `dasungctl info` and an issue with the output
are very welcome.

## Requirements

- Linux with the `ch341` kernel module (part of every common distribution kernel)
- Python 3.8 or newer, standard library only
- for `--notify`: `notify-send` (libnotify) and a notification daemon such as mako or dunst
- the monitor's USB cable connected to the computer: on the 25.3" this is the **USB-B** port, which the official
  client needs too; its USB-C port alone does not expose the control interface

## Install

```sh
git clone https://github.com/jiri-one/dasungctl.git
cd dasungctl
install -Dm755 dasungctl ~/.local/bin/dasungctl
sudo install -m644 70-dasung.rules /etc/udev/rules.d/
sudo udevadm control --reload && sudo udevadm trigger --subsystem-match=tty
```

The udev rule lets the user at the seat open CH340 serial adapters, which the monitors use internally. udev cannot
tell a Dasung's CH340 from any other, so the rule covers all of them. If you prefer, skip it and add yourself to
the serial group instead (`uucp` on Arch, `dialout` on Debian and Ubuntu), then log in again.

## Usage

```sh
dasungctl list                  # detected monitors and their serial ports
dasungctl info                  # firmware version and current settings
dasungctl brightness            # current front light brightness, 0 when the light is off
dasungctl brightness 7          # set it; 0 switches the light off, anything else back on
dasungctl brightness -1         # one step darker; +1 one step brighter, +5 five steps
dasungctl light warm            # off, cold, warm or mixed
dasungctl temperature 30        # colour of the mixed light, 0 warm to 100 cold
dasungctl tune                  # adjust interactively with the arrow keys
dasungctl contrast 5            # 1-9; also: speed 1-5, mode 2-7, refresh
```

Without `-m`, dasungctl uses the only Dasung connected, or the 25.3" one when there are two; pick the other with
`-m 13` before the command, as in `dasungctl -m 13 brightness -1`. Two monitors of the same model have to be told
apart with `--port`; the links in `/dev/serial/by-path/` stay put across reboots, unlike `/dev/ttyUSB*`.

`dasungctl tune` also shows the values the monitor's buttons set, so you can see their steps.

## Key bindings

[niri](https://github.com/niri-wm/niri):

```kdl
binds {
    Mod+Insert { spawn "dasungctl" "--notify" "brightness" "-1"; }
    Mod+Backslash { spawn "dasungctl" "--notify" "brightness" "+1"; }
}
```

Sway:

```
bindsym $mod+Insert exec dasungctl --notify brightness -1
bindsym $mod+backslash exec dasungctl --notify brightness +1
```

A press is one step, and a held key keeps stepping at the keyboard's repeat rate. A relative step that starts
while another dasungctl is still talking to the monitor is skipped, so nothing keeps changing after you let go.

With `--notify`, each step shows the new level in a desktop notification. It replaces the previous one instead of
stacking up, so a held key leaves a single notification that ends on the real value. Errors such as an unplugged
monitor appear there too, since nothing a key binding prints reaches the screen.

Use the full path if `~/.local/bin` is not in your compositor's `PATH`.

## How it works

Dasung's official clients (Windows, macOS, and a small Linux tool from 2021 that their site has offered since 2023)
control the monitors through a CH340 USB-serial converter on the monitor's internal USB hub. They send ASCII hex
frames at 115200 baud, 8N1:

```
5FF5 CC VV 000000000000 A0FA      CC = command, VV = value
```

| Command | Meaning | Values |
|---|---|---|
| `01` | contrast | 1-9 |
| `02` | image mode | 2-7, the meaning differs by model |
| `03` | clear ghosting | 0 |
| `04` | refresh speed | 1-5 |
| `07` | front light | 0 off, 1 cold, 2 warm, 3 mixed |
| `08` | colour temperature of the mixed light | 0 warm to 100 cold |
| `09` | front light brightness | 0-100 |
| `0A` | query parameter `VV` (`10` is the firmware version) | reply `5FF5 F0 0A VV value …` |

The monitor acknowledges a setting with `F0 CC` about 20 ms later and drops any frame that arrives before that, so
dasungctl waits for each acknowledgement. After a front light change it reports the new state for about 200 ms and
drops frames meanwhile too. It also pushes a `CC value` frame whenever a value changes, including changes made with
its buttons. For exploring other models, `dasungctl watch` prints what the monitor sends and `dasungctl raw` sends
any command, both numbers in hex as in the table: `dasungctl raw 0A 10` asks for the firmware version.

Autodetection only considers a CH340 (`1a86:7523`) that shares its hub with a CH341 (`1a86:5512`, the flash
programmer both tested monitors carry), so frames do not go to unrelated serial adapters unless you name one with
`--port`. The monitor whose hub also holds the touch controller (`27c0:0818`) is the 13.3" one.

## Firmware

Dasung publishes no firmware for its monitors. The Windows client 2.0 can update the MCU and the FPGA (through the
internal CH341), but it ships no firmware files, so new versions have to come from Dasung support. dasungctl never
writes firmware.

## Credits

The frame format comes from Dasung's own Linux tool (`DS253.c` on [dasung.com](https://www.dasung.com/h-col-112.html)),
[cpmetz/dasung253](https://github.com/cpmetz/dasung253) and
[codefastdieyoung/paperlike-rs](https://github.com/codefastdieyoung/paperlike-rs). The front light commands, their
ranges and the light modes were read from Dasung's Paperlike Client 2.0 for Windows and 2.0.3 for macOS, then
checked on the monitors above.

dasungctl is not affiliated with Dasung.

## License

[GPL-3.0-or-later](LICENSE)
