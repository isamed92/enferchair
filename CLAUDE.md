# CLAUDE.md — Enferchair Project

## Project Overview

**Enferchair** is an Arduino-based smart wheelchair control system that uses electromyography (EMG) muscle signals from a Myo armband to generate movement commands. The system includes obstacle detection via an ultrasonic sensor and uses fuzzy logic to dynamically adjust motor speed based on proximity to obstacles.

The codebase is primarily in **Spanish** (variables, comments, functions). This is intentional — maintain Spanish naming conventions when modifying or extending existing code.

---

## Repository Structure

```
enferchair/
├── calibrar/
│   └── calibrar.ino          # Standalone EMG calibration utility (debugging tool)
├── myoband/
│   └── myoband.ino           # EMG sensor processing — reads muscle signals, sends commands
├── myochairled/
│   └── myochairled.ino       # Main wheelchair controller — receives commands, drives motors
└── lib/
    ├── EMGFilters/           # OYMotion EMG filtering library (BSD 2-Clause)
    │   ├── EMGFilters.h
    │   ├── EMGFilters.cpp
    │   └── examples/
    └── SoftwareSerial/       # Software UART library (LGPL 2.1, Arduino)
        └── SoftwareSerial.h
```

---

## Hardware Architecture

The system consists of **two physically separate Arduino boards** communicating via SoftwareSerial:

```
[Myo Armband]
      |
      | (analog EMG signals on A0, A5)
      v
[Arduino #1 — myoband]
      |
      | SoftwareSerial (pins 10=RX, 11=TX, 9450 baud)
      | sends single-byte command (1–4)
      v
[Arduino #2 — myochairled]
      |
      +---> [HC-SR04 Ultrasonic Sensor] (pins 2=echo, 3=trigger)
      +---> [Left Motor] (PWM pins 5=forward, 6=backward)
      +---> [Right Motor] (PWM pins 9=forward, 10=backward)
      +---> [4 Direction LEDs] (pins 8=up, 13=down, 12=left, 11=right)
```

---

## Module Descriptions

### `calibrar/calibrar.ino`
A debugging and calibration tool — **not deployed on either wheelchair Arduino**. Used to determine EMG threshold values before programming `myoband.ino`.

- Reads from pins A0 (selector signal) and A5 (action signal)
- Runs `calibrar()` to find max EMG values over 100 valid samples
- Outputs calibration results over USB serial (9450 baud)
- Note: `isCalibrado()` checks if limits are non-zero; `Limite_selector` and `Limite_accion` start at 0 in this sketch (unlike `myoband.ino` where they start at 600/800)

### `myoband/myoband.ino`
Runs on Arduino #1, connected to the Myo EMG armband.

**Startup sequence:**
1. Calibrates `Limite_selector` (A0 pin) via `calibrar()`
2. Calibrates `Limite_accion` (A5 pin) via `calibrar()`
3. Calculates baseline averages via `calcularSignalProm()`

**Main loop:**
- `controlar()`: Reads selector EMG signal; if above baseline, cycles `menu` 1→2→3→4→1
- `activar()`: Reads action EMG signal; if above baseline, sends `menu` value over SoftwareSerial

**Menu/command values:**
| Value | Direction |
|-------|-----------|
| 1 | ADELANTE (forward) |
| 2 | DERECHA (right) |
| 3 | IZQUIERDA (left) |
| 4 | ATRAS (backward) |

**Signal processing pipeline:** `analogRead()` → `filtro.update()` → `sq()` → threshold gate

### `myochairled/myochairled.ino`
Runs on Arduino #2, the main wheelchair controller.

**Startup sequence:**
1. Initializes fuzzy logic system (3 distance sets, 3 speed sets, 3 rules)
2. Sets up motor output pins (5, 6, 9, 10)
3. Sets up LED output pins (8, 11, 12, 13)
4. Sets up ultrasonic sensor (pin 2 input, pin 3 output)

**Main loop:**
- Reads command byte from `Serial.read()`
- Handles `switch(estado)` with cases 0–4
- For movement cases (1–4): checks distance, stops if ≤15cm, otherwise runs fuzzy logic

**Fuzzy logic system (distance → speed):**
| Input (distance cm) | Output (PWM speed 0–255) |
|---------------------|--------------------------|
| corta: 0–40 cm | baja: 0–120 |
| segura: 30–70 cm | promedio: 90–210 |
| grande: 60–250 cm | rapido: 180–240 |

**`movimiento()` function signature:**
```cpp
void movimiento(int velocidadMotorDerechoB, int velocidadMotorIzquierdoB,
                int velocidadMotorDerechoA, int velocidadMotorIzquierdoA)
```
- Parameters map directly to PWM on pins: `derB(10)`, `izqB(6)`, `derA(9)`, `izqA(5)`
- Each direction uses a specific pattern (e.g., forward: `movimiento(0,0,vel,vel)`)

---

## External Libraries (Not in Repo)

The following must be installed separately in the Arduino IDE:

- **Fuzzy Logic Library** — used in `myochairled.ino`
  - Headers: `Fuzzy.h`, `FuzzyComposition.h`, `FuzzyInput.h`, `FuzzyIO.h`, `FuzzyOutput.h`, `FuzzyRule.h`, `FuzzyRuleAntecedent.h`, `FuzzyRuleConsequent.h`, `FuzzySet.h`
  - Install via Arduino Library Manager: search "Fuzzy"

Libraries bundled in `lib/` are already available:
- `EMGFilters` — included via `#include "EMGFilters.h"` (relative path)
- `SoftwareSerial` — included via `#include <SoftwareSerial.h>`

---

## Development Workflow

### Building & Uploading

This project uses the **Arduino IDE** (no build automation). Steps:

1. Open the target `.ino` file in Arduino IDE
2. Select the correct board (Arduino Uno or compatible AVR board)
3. Select the correct COM port
4. Click Upload

There is no `Makefile`, `platformio.ini`, or CI/CD pipeline.

### Deployment Order

When setting up the full system:
1. Run `calibrar.ino` first to find EMG threshold values
2. Update `Limite_selector` and `Limite_accion` constants in `myoband.ino` if needed
3. Upload `myoband.ino` to Arduino #1
4. Upload `myochairled.ino` to Arduino #2

### Debugging

Both sketches use `Serial.print()` extensively for debugging via USB serial monitor:
- `myoband.ino`: 9450 baud — prints EMG values, calibration progress, direction selection
- `myochairled.ino`: 9600 baud — prints `Estado`, `distancia`, `Velocidad` via `message()` helper
- `calibrar.ino`: 9450 baud — prints threshold calibration steps

**Note on baud rate mismatch:** `myoband.ino` transmits at 9450 baud on SoftwareSerial, while `myochairled.ino` receives on hardware Serial at 9600 baud. This mismatch exists in the current code and may cause communication issues.

---

## Code Conventions

### Language
- **Variable and function names**: Spanish (e.g., `velocidad`, `distancia`, `cambiarDireccion`)
- **Comments**: Mix of Spanish and English; inline comments tend to be Spanish
- **String literals in Serial.print**: Spanish (e.g., `"ADELANTE"`, `"IZQUIERDA"`)

### Style
- Indentation: tabs
- No consistent formatting standard at the project level
- The `lib/EMGFilters/` directory has a `.clang-format` file (for the library only)
- `calibrar.ino` and `myoband.ino` are more verbose (debugging artifacts); `myochairled.ino` is the production sketch

### Known Patterns
- EMG signal threshold gating: `return (valorObtenido > limite) ? valorObtenido : 0;`
- Fuzzy defuzzification result can be negative — always apply `if (vel <= 0) vel *= -1;` before using
- The `cont` variable in `myochairled.ino` limits fuzzy loop iterations to 3 (`while (cont <= 3)`); it is reset to 0 after the switch block
- `estado = Serial.read()` returns -1 (no data) which falls through to `default:` — this is expected behavior

---

## Pin Reference

### myoband (Arduino #1)
| Pin | Role |
|-----|------|
| A0 | EMG selector signal input |
| A5 | EMG action signal input |
| 10 | SoftwareSerial RX |
| 11 | SoftwareSerial TX → connects to myochairled RX |

### myochairled (Arduino #2)
| Pin | Role |
|-----|------|
| 2 | HC-SR04 Echo (input) |
| 3 | HC-SR04 Trigger (output) |
| 5 | Left motor forward (PWM) |
| 6 | Left motor backward (PWM) |
| 8 | LED Up (forward indicator) |
| 9 | Right motor forward (PWM) |
| 10 | Right motor backward (PWM) |
| 11 | LED Right |
| 12 | LED Left |
| 13 | LED Down (backward indicator) |

---

## Serial Communication Protocol

Commands are single-byte integers sent from `myoband` to `myochairled` via SoftwareSerial:

| Byte value | Command |
|------------|---------|
| 0 | Stop (parar) |
| 1 | Forward (adelante) |
| 2 | Right (derecha) |
| 3 | Left (izquierda) |
| 4 | Backward (atras) |

---

## Important Notes for AI Assistants

1. **Two-board architecture**: Changes to `myoband.ino` affect the sensor side; changes to `myochairled.ino` affect the motor/actuator side. They are independent sketches uploaded separately.

2. **No unit tests**: There is no test framework. Testing is done by observing Serial output with a serial monitor while the hardware runs.

3. **Spanish naming is intentional**: Do not rename Spanish variables to English unless explicitly asked.

4. **The `lib/` directory contains vendored libraries**: Do not modify files under `lib/`. If library updates are needed, replace the entire directory.

5. **Fuzzy logic setup is in `setup()`**: The entire fuzzy system (inputs, outputs, rules) is constructed in `setup()` using heap allocation (`new`). This is standard for the Arduino Fuzzy library — do not move it to loop().

6. **`calibrar.ino` vs `myoband.ino`**: These share similar functions (`calibrar`, `getSignal`, `calcularSignalProm`) but are separate sketches with different initial threshold values. `calibrar.ino` starts thresholds at 0; `myoband.ino` starts at 600/800.

7. **Obstacle stop threshold**: Distance ≤15cm triggers an emergency stop (`movimiento(0,0,0,0)`). The lower bound of ≥2cm filters out sensor noise/invalid readings.
