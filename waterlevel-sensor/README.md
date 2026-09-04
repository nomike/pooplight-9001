# Tank level → WLED (ESPHome)

Standalone ESPHome firmware for a toilet-cistern water-level sensor. An insulated capacitive probe is read through the **classic ESP32 touch peripheral**. The fill fraction is POSTed to a stock WLED controller as the **Percent** effect intensity (`ix`) on one segment.

Home Assistant is not required. A local web UI is served at `http://tank-wled.local` for live raw values and calibration.

This directory is the working copy. Run every `esphome` command from here so `secrets.yaml` is found.

Active config: [`tank-wled-standalone.yaml`](tank-wled-standalone.yaml).

---

## How it works

1. An insulated aluminium sense plate on an acrylic strip, plus a bare stainless rod in the water (GND), form a capacitor. Water (εᵣ ≈ 80) is the dielectric; capacitance rises with submerged area.
2. The ESP32 touch peripheral converts capacitance to a raw count. On classic ESP32, **more water = lower raw**.
3. A piecewise `calibrate_linear` map turns that raw value into 0–100 %.
4. On each percent change the node POSTs `{"seg":[{"id":0,"ix":<0-255>}]}` to `http://<wled_host>/json/state`. That writes RAM only; frequent updates do not wear WLED’s flash.

The WLED segment must be **on** and running the **Percent** effect or nothing will show. Other segments and the running preset are left alone.

---

## What you need

- Classic **ESP32-WROOM / NodeMCU** (touch v1). ESP32-S3 touch is unreliable with a large wet electrode; ESP32-C3 has no touch peripheral.
- An existing WLED controller on the same LAN.
- The capacitive probe: acrylic strip, aluminium tape, stainless rod (see [Wiring](#wire-the-sensor)).
- USB cable for the first flash (CH340/CP2102 boards appear as `/dev/ttyUSB0`, not `/dev/ttyACM0`).

---

## Wire the sensor

### Sense plate

Use a thin piece of scrap acrylic. It must be at least as tall as the water column in the tank and should fit in a shrink tube.
Stick aluminium tape to one side over the whole length and width (the more surface area the more accurate the sensor will be).

Solder a Wire to one end of the tape (do that before you stick it on the acrylic to prevent melting it).

Then put a shrink tube around the whole acrylic piece, leaving 1-2 cm of empty tube on both sides and the wire sticking out of it.

***Note:** It is preferable to use shrink tube with glue inside.*

```
  acrylic strip
  ┌─────────────────────────────────────┐
  │ ████████ aluminium tape ██████████  │  full width, insulated
  └─────────────────────────────────────┘
                    │
                    └── GPIO32 (D32)  — substitution `touch_pin`
```

### Ground

The other electrode could be anything which doesn't corrode and needs to constantly stay in contact with the water. There the surface area doesn't matter.

You could use an insulated stranded copper wire, solder a stainless steel screw to one end and cover the solder-joint with a short shrink tube. Use hot-glue to glue the cable to the bottom of the tank (empty it first!). Make sure to use hot-glue on the insulated section and not on the screw to prevent accidentally insulating the screw.

### Measuring the water-tightness of the sense probe

Fully submerge the sense plate and the ground next to each other in a jug of water. Measure the resistance between those. It should be out of or at least in the Mega Ω range. If you measure Kilo Ωs, your sense rod is not completely isolated and measurements won't work.

### Wiring it up

The sense plate needs to be connected to one of these pins of the ESP: 4, 13, 14, 27, 32, 33. Avoid pins 0, 2, 12, and 15.
In the example, it is wired to pin 32.

You need to set the pin in tank-wled-standalone.yaml.

Ground need to be connected to one of the GND pins of the ESP.

## Install ESPHome

This project is flashed with the **ESPHome command-line tool**. The steps below are for Debian and Ubuntu. For Home Assistant, Windows, macOS, Docker, or the Device Builder desktop app, use the official guide:

**[Install ESPHome](https://esphome.io/guides/installing_esphome.html)**

### Debian / Ubuntu

You can install ESPHome with pipx:

```shell
pipx install esphome
```

## `secrets.yaml`

Wi-Fi and OTA credentials are **not** stored in the firmware YAML. They live in `secrets.yaml` in this directory (the same directory you invoke `esphome` from). That file is gitignored.

```bash
cp secrets.yaml.example secrets.yaml
```

Edit the three keys:

| Key | Purpose |
|-----|---------|
| `wifi_ssid` | Your 2.4 GHz Wi-Fi network name |
| `wifi_password` | That network’s password |
| `ota_password` | Password for later over-the-air flashes. Pick one and keep it; changing it later needs a USB reflash if the device is unreachable. |

`secrets.yaml.example` is a template only. Do not put real passwords in it.

The ESP32 joins Wi-Fi on first boot with these credentials — there is no separate pairing step. If the join fails, the board raises a fallback AP named **Tank WLED Fallback**; connect to that and use the captive portal.

***Note:** ESPs only support 2.4 GHz WiFi*

## Upload the firmware

Close anything else holding the serial port (`screen`, `picocom`, Thonny, `mpremote`, another `esphome logs`). Otherwise the port shows as busy.

### First flash (USB)

Plug the classic ESP32 in. It should appear as `/dev/ttyUSB0` (CH340/CP2102). Confirm with `ls /dev/ttyUSB*`.

```bash
cd "/path/to/pooplight 9001/esp-home"
esphome run tank-wled-standalone.yaml --device /dev/ttyUSB0
```

***Note:** If you ommit the `--device` parameter esphome will autodetect available devices and prompt you for it.*

If flashing stalls at `Connecting...`, hold **BOOT** while tapping **RESET**, then retry.

The first compile downloads the ESP-IDF toolchain and takes several minutes. Later compiles are incremental.

### Later flashes (OTA)

Once the node is on Wi-Fi:

```bash
~/esphome-venv/bin/esphome run tank-wled-standalone.yaml --device tank-wled.local
```

If both USB and OTA are visible and `esphome` asks which to use, pass `--device` explicitly as above.

### After it boots

1. Open **http://tank-wled.local** (or the board’s IP).
2. Confirm **Tank Raw** is a non-zero number that drops when you touch the sense strip or dip the probe.
3. Point `wled_host` in the YAML at your WLED box if it is not `pooplight-9001.local`. A raw IP is more reliable than `.local` if mDNS resolution fails (`couldn't get hostname for :…: getaddrinfo()` in the logs). Then reflash.

## Customise the YAML

In `tank-wled-standalone.yaml`, under `substitutions:`:

| Key | Meaning |
|-----|---------|
| `touch_pin` | Sense-strip GPIO (default `GPIO32`) |
| `wled_host` | WLED hostname or IP |
| `wled_seg` | Percent-effect segment id (default `0`) |
| `ix_empty` / `ix_full` | WLED intensity at 0 % and 100 %. Percent fills from one end for `0…100` and from the other for `100…200`. If the strip fills the wrong way, use e.g. `ix_empty: "200"` and `ix_full: "100"`. |

Calibration datapoints are in the `calibrate_linear` block on the **Tank Level** sensor (see below). Changing them needs a reflash (OTA is enough).

---

## Sensor calibration

The raw-vs-height curve is still somewhat **nonlinear**: the touch count is roughly 1/C, even though C now tracks submerged area much more cleanly. It is **not** logarithmic. A two-point line is often good enough; extra mid-points help if the strip does not track the real column.

ESPHome’s `calibrate_linear` with `method: exact` does piecewise-linear interpolation. **From-values must be listed ascending** — ESPHome does not sort. A descending curve (more water = lower raw) is fine: list the lowest raw first.

```yaml
- 103 -> 100     # full  (lowest raw)
- 187 -> 50
- 470 -> 0       # empty (highest raw)
```

### Procedure

1. Mount the probe where it will live. Numbers gathered using a jar of water on your workbench **do not transfer** — tank walls, water mass, and nearby plastic change the raw scale. Recollect after any change of chip, pin, probe geometry, or insulation.
2. Open http://tank-wled.local and watch **Tank Raw**.
3. Set the water to a known **height** (stick some masking tape to the outside of the tank and mark spots with a ruler). Be aware that the sensor measures water-level and not volume.
4. Hands off the jug/tank. Wait until the median filter settles (a few seconds), then write down the stable raw value.
5. Convert height to percent of the usable column (e.g. 0–16 cm → 8 cm = 50 %).
6. Repeat for empty, full, and a few points in between. Skip a height if the raw value has not moved enough to be distinct from the last point.
7. Paste the pairs into `datapoints:` (ascending raw) and run `esphome run` again.
8. Verify at a height you did **not** calibrate.

Never calibrate a bare or leaky probe. Confirm the seal with a resistance check first.

### Filters (for reading, not for resolution)

`median` / `window_size` keeps the last N raw samples and publishes the middle value. That throws away a drip or a nearby hand. Raising `window_size` makes the number calmer and adds lag; it does not add resolution.

If you want to reduce sensor lag, try lowering the `update_interval` value.

---

## Typical problems

| Symptom | What to check |
|---------|----------------|
| Raw stuck at **0** | `setup_mode` must be `true` (already set). Without it, ESPHome never polls the analog touch value on classic ESP32. |
| Raw barely moves when you add water | Sense plate leaking (resistance check), stainless rod not in the water / not on GND, or the wrong GPIO. |
| `couldn't get hostname for :pooplight-9001.local` | WLED mDNS failed. Put WLED’s IP in `wled_host` and reflash. The sensor itself is fine. |
| Port busy / permission denied | Close other serial programs. Add your user to `dialout` and re-login. Do not use `sudo` to flash. |
| `This chip is ESP32-S3, not ESP32` | Wrong `board:`. This config is `esp32dev` for classic ESP32 only. |
