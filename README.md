# Robotic Hand

A 3D-printed, servo-driven robotic hand with five independently controlled fingers. It can be driven in two ways:

- **Manual mode:** five potentiometers wired directly to an Arduino Mega 2560 (for example on a glove or control board).
- **Remote mode:** five sliders in a [Blynk](https://blynk.io/) mobile app. An ESP32 receives them over Wi-Fi and forwards the values to the Mega over I²C.

Made as a final thesis project (*Završni rad*).

## How it works

```
 Blynk app ──Wi-Fi──▶ ESP32 ──I²C (address 10)──▶ Arduino Mega 2560 ──▶ 5 servos (fingers)
                                                          ▲
                                     5 potentiometers (A0–A4, manual mode)
```

1. The ESP32 connects to Wi-Fi and Blynk and reads six virtual pins:

   | Virtual pin | Meaning |
   | --- | --- |
   | `V0` | Mode switch (`0` = control from the Blynk app, any other value = manual potentiometers) |
   | `V1` | Thumb (*palac*) |
   | `V2` | Index finger (*kažiprst*) |
   | `V3` | Middle finger (*srednjak*) |
   | `V4` | Ring finger (*prstenjak*) |
   | `V5` | Little finger (*mali prst*) |

2. Each slider value (0–1023) is split into five bytes of at most 255 so it fits in I²C bytes. The ESP32 sends a **26-byte packet** to I²C address `10`: one mode byte followed by five bytes per finger, once per second.
3. The Mega receives the packet, adds each finger's five bytes back into a 0–1023 value and maps it to a servo angle.
4. On power-up the Mega is in manual mode (`buttonState = 1`) and reads the potentiometers directly.

### Servo configuration

Input values 0–1023 are mapped to a calibrated angle range for each finger:

| Finger | Servo pin | Potentiometer | Angle range |
| --- | --- | --- | --- |
| Thumb | 53 | A0 | 180 → 145 |
| Index | 51 | A1 | 180 → 145 |
| Middle | 49 | A2 | 175 → 141 |
| Ring | 47 | A3 | 170 → 135 |
| Little | 45 | A4 | 180 → 145 |

## Repository structure

| Path | Contents |
| --- | --- |
| `Zavrsni_ESP32/` | ESP32 sketch: Wi-Fi and Blynk input, I²C master |
| `Zavrsni_Mega2560_/` | Arduino Mega 2560 sketch: I²C slave, servo and potentiometer control |
| `stepper_test/` | Standalone stepper motor test sketch (STEP pin 33, DIR pin 34) |
| `dijelovi ruke/` | 3D models: `.3mf` / `.stl` print files and Fusion 360 (`.f3d`) sources for the fingers, knuckles, servo–finger joint, palm extension and forearm |
| `ruka_/` | Photos and renders of the hand |
| `LICENSE` | GNU GPL v3 |

## Hardware

- Arduino Mega 2560
- ESP32 development board
- 5 hobby servos and an external power supply that can handle their stall current
- 5 potentiometers (manual control)
- 3D-printed parts from `dijelovi ruke/`
- Optional: stepper motor and driver for `stepper_test`

### Wiring

- **I²C:** ESP32 SDA/SCL to Mega SDA (20) / SCL (21), with a common ground. The ESP32 is 3.3 V and the Mega is 5 V, so use a level shifter (or otherwise make sure the bus is safe for the ESP32).
- **Servos:** signal pins as in the table above. Power them from a separate supply and share ground with the Mega.
- **Potentiometers:** wiper to A0–A4, outer pins to 5 V and GND.

## Setup

### Requirements

- [Arduino IDE](https://www.arduino.cc/en/software)
- ESP32 board support package
- **Blynk** library (for the ESP32 sketch). `Wire` and `Servo` come with the Arduino core.

### 1. Blynk

1. In Blynk.Console, create a template with a switch on `V0` and five sliders on `V1`–`V5` (range 0–1023).
2. Create a device from the template and note its template ID, template name and auth token.

### 2. Credentials (`arduino_secrets.h`)

Credentials are not stored in the repository. `arduino_secrets.h` is listed in `.gitignore`, so you have to create it yourself.

Create `arduino_secrets.h` in the same folder as `Zavrsni_ESP32.ino`:

```cpp
#define SECRET_SSID "your-wifi-name"
#define SECRET_PASS "your-wifi-password"

#define SECRET_TEMPLATE_ID   "your-blynk-template-id"
#define SECRET_TEMPLATE_NAME "your-blynk-template-name"
#define SECRET_BLYNK_TOKEN   "your-blynk-auth-token"
```

The Mega sketch also includes `arduino_secrets.h` but does not use any values from it. Either create an empty file with that name in the Mega sketch folder, or remove the `#include "arduino_secrets.h"` line from the Mega sketch.

> **Never commit `arduino_secrets.h`.** If a token or password has been exposed, regenerate the Blynk token and change the Wi-Fi password.

### 3. Upload

1. **ESP32:** open `Zavrsni_ESP32/Zavrsni_ESP32/Zavrsni_ESP32.ino`, select your ESP32 board and upload.
2. **Mega:** open `Zavrsni_Mega2560_/Zavrsni_Mega2560_/Zavrsni_Mega2560_.ino`, select *Arduino Mega or Mega 2560* and upload.

## Usage

1. Power the Mega and the servos. The hand starts in manual mode and follows the potentiometers.
2. Power the ESP32 and wait until it connects to Blynk.
3. Set `V0` to `0` in the app to control the hand with the sliders. Set it to any other value to go back to the potentiometers.

## Known limitations

- The ESP32 sends data once per second (`delay(1000)`, marked in the code as a test interval), so app control is slow. Lower the delay for smoother movement.
- The Mega prints every received byte to Serial in the I²C handler. This is useful for debugging but slows the handler, so remove it for normal use.

## License

Released under the GNU General Public License v3.0. See [LICENSE](LICENSE).
