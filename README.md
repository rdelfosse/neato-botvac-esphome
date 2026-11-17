# neato-botvac-esphome

Bring an old, offline **Neato Botvac D85** robot vacuum into **Home Assistant** with an **ESP32-S3** running **ESPHome**, soldered to the robot's internal serial port.

The robot keeps cleaning exactly as before. The ESP reads everything the robot's own serial console exposes (motors, battery, errors, bumpers, wheel counters, the laser scanner) and adds remote control, live telemetry, an Xbox controller for manual driving, and a 360° laser feed.

> **Status:** personal project, tested on one Botvac D85 (robot software 1.2.1) with ESPHome 2026.10.2. Other Botvac models with the same serial console will probably work, but they are untested. Opening the robot and soldering to its mainboard is at your own risk.

> **Language:** entity names and code comments are in French (the project was built for a French household). Names are easy to change in `neato.yaml`.

## How it works

Neato Botvac robots have a text console on their USB port: at 115200 baud you can type `Help`, `GetVersion`, `GetMotors`, `GetLDSScan`, `Clean House`… and the robot answers in plain text. Every reply ends with a `0x1A` byte.

The same console is available inside the robot on a 3-pin serial header on the mainboard. The ESP talks to it once per second and publishes the results to Home Assistant.

```
Neato mainboard (P7, 3.3 V)          ESP32-S3
  TX  ────────────────────────────── GPIO17 (RX)
  GND ────────────────────────────── GND
  RX  ────────────────────────────── GPIO18 (TX)

Robot battery ── (after the power switch) ── fuse ── 5 V step-down converter ── ESP 5V / GND
```

- **Serial header:** on our D85 it is marked **P7**, with ground in the middle. The two signal pins go to GPIO17 and GPIO18. If the robot does not answer, swap them: we found the right way round with a test build that swapped TX and RX every 10 seconds and logged what came back.
- **Power:** a small 5 V step-down converter fed by the robot's battery (NiMH, 10 cells; about 14.6 V measured when charged), after the robot's power switch, with a fuse on its input. Use a good fuse holder: a loose one made our ESP reboot at random.

### Alternative: USB instead of the internal port

The first version used the robot's USB port, with the ESP32-S3 acting as a USB host (`usb_uart`). It works, with two catches: on many ESP32-S3 boards the USB port gets no 5 V until you bridge the "USB-OTG" pads on the back, and the receive buffer must be raised to 8 KB or long replies (`GetMotors`, `GetVersion`) are cut off. A cable coming out of the robot also tends to trigger the "dustbin removed" error once it drives. The internal port avoids all of this.

## What you get in Home Assistant

| Kind | Entities (French names) |
|---|---|
| Buttons | Clean (`Nettoyer`), Spot clean (`Nettoyer autour`), Stop (`Arrêter`), Pause / Resume, Play sound, Set clock, Exit test mode, Laser scan, Stop driving, Forget controller, Restart the ESP |
| Spot size | Width and height of the spot clean area, 100 to 500 cm |
| Status | Activity (cleaning / moving / idle), Error, Cleaning, On the dock, Charging, Dustbin in place, Stuck, Test mode, Robot connected |
| Battery | Level, voltage, temperature, fault, overheating, low, empty, gauge reliable |
| Motors | Vacuum, brush and wheel speeds, wheel loads, brush, side brush and vacuum currents, vacuum power |
| Odometry | Trip distance, heading, total distance |
| Laser | Valid points, rotation speed |
| Robot info | Model, serial number, software version, cleaning hours |
| ESP health | Wi-Fi signal, uptime, reason for last reboot |

`Retour au départ` (back to start) only simulates button presses on the robot: the serial console has **no "return to dock" command**. Treat it as experimental.

### Live data for dashboards

- **`Direct` text sensor**, updated every second, a compact JSON for live dashboards: `a` vacuum rpm, `b` brush rpm, `g` / `d` left / right wheel rpm, `cg` / `cd` wheel loads (%), `ib` / `il` / `ia` brush, side brush and vacuum currents (mA), `p` bumper and wheel-drop bits, `t` trip (cm), `h` heading (°), `x` / `y` position (cm). Exclude it from the recorder, it changes every second:

  ```yaml
  recorder:
    exclude:
      entities:
        - sensor.neato_direct
  ```

  The individual motor sensors are published every 30 s and are fine to keep in history.

- **`esphome.neato_lidar` event**, one per laser turn (every second while cleaning or driving):

  | Field | Meaning |
  |---|---|
  | `d` | 720 characters: 2 base64url characters per degree, distance in cm on 12 bits, 0 = no reading. 0° is the front of the robot, angles increase to the left. |
  | `x`, `y`, `t` | Position and trip distance in mm, from the start point (x forward, y left) |
  | `h` | Heading in tenths of a degree, positive to the left |
  | `l`, `r` | Raw left and right wheel counters (mm) |
  | `n` | Number of valid points |
  | `m` | 1 = cleaning, 2 = controller driving, 0 = other |

### Odometry

The robot reports how far each wheel has turned. With the 242 mm wheel spacing, the ESP integrates distance, heading and position from the dock. The trip restarts when the wheels start moving while the robot is on its dock. It drifts with wheel slip: use it as an estimate.

## Driving with an Xbox controller

An Xbox Wireless controller (Bluetooth Low Energy, firmware 5.x; tested with an Elite Series 2) connects directly to the ESP.

| Control | Action |
|---|---|
| A | Start driving (puts the robot in test mode) |
| Left stick | Drive and turn |
| RT / LT | Turbo / slow |
| X | Brushes on/off |
| RB | Vacuum on/off |
| D-pad up / down | Vacuum power ±20 % |
| Y | Horn |
| B | Stop everything |

Safety: the robot will not drive forward while a bumper is pressed, and everything stops if it is lifted, if the controller disconnects, or after 5 minutes without input. Test mode is always switched off when the ESP boots.

Pairing: put the controller in pairing mode near the robot; the first Xbox controller seen is adopted and remembered. Use **Forget controller** to pair another one. After an ESP reboot the controller may need pairing mode again.

Notes for developers: Bluepad32 cannot be used here because it brings its own Bluetooth stack, which conflicts with ESPHome's. The controller is read as a plain BLE HID device (`ble_client`), and it sends nothing until the link is encrypted, so the config requests encryption on connect and re-subscribes to the HID reports afterwards.

## Installation

1. Copy `neato.yaml` into your ESPHome configuration directory.
2. Add the three secrets to your ESPHome `secrets.yaml` (see `secrets.yaml.example`). Generate the API key with ESPHome's dashboard or `openssl rand -base64 32`.
3. Check the board settings at the top of `neato.yaml` (ESP32-S3, 8 MB flash declared) and the UART pins.
4. Flash the ESP once over USB, then over the air.
5. Open the robot, solder the three wires to the serial header, wire the power supply, and close it up. Check the ESP's **reason for last reboot** sensor after reassembly: repeated power-on reboots point to a power or fuse problem.

## Known limitations

- No "return to dock" command in the serial console.
- No map or room selection on this model: cleaning is whole-home or a rectangular spot around the robot.
- A cleaning run takes about 15 seconds to start (laser, vacuum, brush) before the wheels move.
- After a run stopped with the robot's button, `Clean House` is ignored until `Clean Stop` is sent; the **Clean** button does this for you.
- Odometry drifts; the laser feed is raw (no mapping is done on the ESP).

## License

MIT, see [LICENSE](LICENSE).
