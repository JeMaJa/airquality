# CO2-Driven Ventilation Control for Zehnder ComfoAir Q600

> **Disclaimer:** This project is a personal hobbyist implementation and is not affiliated with or endorsed by Zehnder Group. Interfacing with the Zehnder Option Box involves low-voltage signal wiring connected to a mains-powered ventilation unit. Installation and commissioning of the Option Box should be carried out by a certified installer as stated in the official Zehnder documentation. Use this project at your own risk. The author provides no warranty and accepts no liability for incorrect ventilation behaviour, equipment damage, or any other consequences arising from its use.

---

## What This Project Does

This project uses Home Assistant to drive a Zehnder ComfoAir Q600 heat recovery ventilation unit based on real-time CO2 readings from IKEA Alpstuga sensors. A Shelly 0-10V Dimmer PM Gen3 acts as the bridge between Home Assistant and the Zehnder Option Box, producing an analog voltage signal on the 0–10V input.

Two control modes are supported, selectable at runtime via a toggle in Home Assistant:

- **CONTROL mode** — The Shelly emulates a Zehnder CO2 sensor. The Option Box's own built-in PI controller regulates airflow to hold CO2 at the configured setpoint. Home Assistant's only job is to output the right voltage.
- **STEER (PID) mode** — Home Assistant runs its own PID controller and sends a direct airflow demand to the Shelly. The Option Box translates that voltage linearly into a flow request without applying any additional control logic of its own.

---

## System Architecture

```
IKEA Alpstuga sensors (CO2 ppm)
        │
        ▼
Home Assistant
  ├─ Smoothing (4-min rolling mean per sensor)
  ├─ Worst-room selector (highest CO2 across rooms)
  └─ PID automation (runs every 30 s)
        │
        │  Selects output based on input_boolean.pid_mode:
        │
        ├─── CONTROL mode (pid_mode OFF):
        │      CO2 ppm mapped to voltage (emulates Zehnder CO2 sensor)
        │      Option Box PI controller regulates airflow to setpoint
        │
        └─── STEER / PID mode (pid_mode ON):
               HA PID calculates airflow demand %
               Option Box passes voltage through as direct flow request
        │
        ▼
Shelly 0-10V Dimmer PM Gen3
  brightness_pct (0–100%) → analog voltage (0–10V)
        │
        ▼
Zehnder Option Box (0–10V input)
        │
        ▼
ComfoAir Q600
```

---

## Hardware

| Component | Role |
|---|---|
| IKEA Alpstuga (×2) | CO2 source sensors via Matter over Thread |
| Shelly 0-10V Dimmer PM Gen3 | Converts HA brightness_pct to 0–10V analog signal |
| Zehnder Option Box | Receives 0–10V signal, controls ComfoAir Q600 |
| Zehnder ComfoAir Q600 | Heat recovery ventilation unit |

> **Sourcing vs sinking:** The Shelly 0-10V Dimmer PM Gen3 is a **sourcing** device — it actively drives voltage on the signal wire. The Zehnder Option Box analog input expects a sourcing signal. These are directly compatible. Do not substitute a sinking-type 0-10V device without additional circuitry.

---

## Control Modes Explained

### Option 1 — CONTROL mode (Zehnder PI controller)

In this mode, Home Assistant outputs a voltage that mimics what a real Zehnder CO2 sensor would produce: **2V at 400 ppm, rising to 10V at 2000 ppm**. The Option Box is configured with METHOD: CONTROL and a setpoint of 5V (1000 ppm).

The Option Box's built-in PI controller continuously compares the measured voltage against the setpoint and adjusts airflow until CO2 stabilises at the target. The input is inverted (INPUT AT 0% = 10V, INPUT AT 100% = 2V) because the control loop needs to increase flow when CO2 is high, which corresponds to a low voltage in the Zehnder sensor convention.

Home Assistant does no flow regulation — it simply converts the CO2 reading to the correct voltage. The closed-loop control lives entirely inside the Option Box.

**When to use:** Simpler setup. Tuning is done on the Option Box hardware. Suitable if you are comfortable with the Zehnder's built-in PI parameters and do not need HA-side tuning or logging of the control loop.

---

### Option 2 — STEER mode (Home Assistant PID controller)

In this mode the Option Box is configured with METHOD: STEER. It receives a voltage and translates it directly and linearly into a proportional airflow request between Preset 1 and Preset 3, with no PI control of its own.

Home Assistant runs a PID controller implemented as an automation that fires every 30 seconds. It calculates an airflow demand based on the CO2 error:

- **Error** = measured CO2 − setpoint
- **Output** = Kp × error + Ki × integral + Kd × derivative, clamped to 0–100%
- A **deadband of ±25 ppm** suppresses integral accumulation near the setpoint to prevent hunting
- A **rate limiter of ±5% per 30s cycle** prevents sudden voltage steps to the Shelly
- When CO2 is below the setpoint, Kp is halved to reduce overshoot on the way down
- Kd is 0 by default — ventilation systems respond too slowly for derivative action to be useful

The calculated output percentage is sent to the Shelly as `brightness_pct`, producing a proportional voltage that the Option Box converts directly to a flow request.

**When to use:** You want full visibility and tuning control inside Home Assistant. All PID parameters (Kp, Ki, Kd, setpoint) are adjustable at runtime via input_number helpers without touching the Zehnder hardware. Easier to log and graph the full control loop.

---

## Zehnder Option Box Configuration

Access the Option Box settings via the ComfoAir Q600 installer menu:
**INSTALLER SETTINGS → OPTION BOX SETTINGS → 0-10V INPUT 1**

### For CONTROL mode (Zehnder PI controller active)

| Menu item | Setting |
|---|---|
| INPUT AT 0% | **10.0 V** (= 2000 ppm) |
| INPUT AT 100% | **2.0 V** (= 400 ppm) |
| METHOD | **CONTROL** |
| SETPOINT | **5.0 V** (= 1000 ppm) |
| PROPORTIONAL BAND | **50%** |
| INTEGRAL TIME | **300 s** |
| 0-10V FUNCTION | **FLOW-PROPORTIONAL** |
| 0-10V PRIORITY | **AUTO ONLY** |

The input inversion (INPUT AT 0% = 10V, INPUT AT 100% = 2V) is required by the Option Box's control loop sign convention. A high CO2 reading produces a low voltage in the Zehnder sensor convention, so the controller must be configured to interpret a low voltage as a demand for more ventilation.

### For STEER mode (Home Assistant PID controller active)

| Menu item | Setting |
|---|---|
| INPUT AT 0% | **0.0 V** |
| INPUT AT 100% | **10.0 V** |
| METHOD | **STEER** |
| CONTROL SETTINGS | n/a |
| 0-10V FUNCTION | **FLOW-PROPORTIONAL** |
| 0-10V PRIORITY | **AUTO ONLY** |

In STEER mode the input is not inverted — 0V means minimum flow and 10V means maximum flow. The Option Box simply scales the incoming voltage linearly to airflow between Preset 1 and Preset 3. Home Assistant's PID output maps directly to this range.

> **Important:** The `input_boolean.pid_mode` toggle in Home Assistant switches HA's output calculation between the two modes. It does **not** change the Option Box settings — those must be configured on the hardware to match whichever mode you are running.

---

## Home Assistant Configuration

### `configuration.yaml`

Defines all sensors, template sensors, and input helpers. Add the blocks below to your existing file — do not duplicate top-level keys (`sensor:`, `template:`, `input_number:`).

**Template sensors:**

| Sensor | Description |
|---|---|
| `sensor.co2_worst_room` | Highest smoothed CO2 reading across all rooms (ppm) |
| `sensor.fan_speed_target` | Linear voltage mapping: CO2 ppm / 2000 × 100% |
| `sensor.ventilatie_signaal` | Feedback: actual Shelly brightness as % |
| `sensor.pid_error` | Current CO2 error: measured − setpoint (ppm) |
| `sensor.pid_output` | Current PID output % |
| `sensor.pid_integral` | Current PID integral accumulator |
| `sensor.debiet_vraag` | Estimated airflow demand in m³/h (200–450 range) |

**Statistics sensors** (smoothing):

| Sensor | Source | Window |
|---|---|---|
| `sensor.co2_alpstuga_1_smooth` | `sensor.alpstuga_air_quality_moni_co2` | 4 min, 40 samples |
| `sensor.co2_alpstuga_2_smooth` | `sensor.alpstuga_air_quality_2_co2` | 4 min, 40 samples |

**Input helpers (all adjustable at runtime in HA):**

| Helper | Purpose | Default |
|---|---|---|
| `input_boolean.pid_mode` | ON = STEER/PID mode, OFF = CONTROL/sensor emulation | OFF |
| `input_number.pid_setpoint` | CO2 target (ppm) | 950 |
| `input_number.pid_kp` | Proportional gain | 0.15 |
| `input_number.pid_ki` | Integral gain | 0.02 |
| `input_number.pid_kd` | Derivative gain | 0.0 |
| `input_number.pid_integral` | Integral accumulator — managed by automation, do not edit manually | 0 |
| `input_number.pid_previous_error` | Previous error state — managed by automation, do not edit manually | 0 |
| `input_number.pid_previous_output` | Previous output for rate limiting — managed by automation, do not edit manually | 0 |

**What to change:**
- Replace `sensor.alpstuga_air_quality_moni_co2` and `sensor.alpstuga_air_quality_2_co2` with your actual Alpstuga entity IDs.
- Replace `light.shelly0110dimg3_REPLACE_ME` with your own Shelly entity ID throughout both files.

Find entity IDs via **Developer Tools → States**.

### `automations.yaml`

Contains one active automation: **PID CO2 Controller** (`pid_co2_controller`).

It triggers every 30 seconds and performs the following steps:
1. Reads CO2, setpoint, PID gains, and stored state from input_number helpers
2. Calculates error, integral (with deadband), and derivative
3. Computes PID output (STEER mode) or linear voltage mapping (CONTROL mode)
4. Selects the correct output based on `input_boolean.pid_mode`
5. Applies the ±5%/cycle rate limiter
6. Writes updated state back to the integral, previous error, and previous output helpers
7. Sends `brightness_pct` to the Shelly

A legacy automation (`increase ventilation`) is present in the file but disabled.

---

## Deployment Checklist

1. Add sensor and input_number blocks from `configuration.yaml` to your HA config.
2. Add the `pid_co2_controller` automation to your `automations.yaml`.
3. Replace the Shelly entity ID (`light.shelly0110dimg3_REPLACE_ME`) with your own.
4. Replace both Alpstuga entity IDs with your own.
5. Choose your control mode and configure the Option Box hardware accordingly (see above).
6. Set `input_boolean.pid_mode` in HA to match: **OFF** for CONTROL mode, **ON** for STEER mode.
7. Validate config: **Developer Tools → YAML → Check Configuration**.
8. Restart Home Assistant.
9. Verify the sensor chain in **Developer Tools → States**: Alpstuga raw → smooth → worst room → automation firing every 30s → Shelly brightness updating.

---

## Future Considerations

- Add more Alpstuga sensors by extending the `max` list in the *CO2 worst room* template.
- Bedroom-specific setpoint (~900 ppm) with its own smoothing window and PID parameters.
- HA dashboard with CO2 trend, PID error, output %, and mode toggle for live monitoring and tuning.
