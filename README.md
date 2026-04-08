# CO2-Driven Ventilation Control for Zehnder ComfoAir Q600

> **Disclaimer:** This project is a personal hobbyist implementation and is not affiliated with or endorsed by Zehnder Group. Interfacing with the Zehnder Option Box involves low-voltage signal wiring connected to a mains-powered ventilation unit. Installation and commissioning of the Option Box should be carried out by a certified installer as stated in the official Zehnder documentation. Use this project at your own risk. The author provides no warranty and accepts no liability for incorrect ventilation behaviour, equipment damage, or any other consequences arising from its use.

---

Automates a Zehnder ComfoAir Q600 HRV unit from Home Assistant using CO2 readings from IKEA Alpstuga sensors. A Shelly 0-10V dimmer produces an analog voltage that mimics a real Zehnder CO2 sensor. The Zehnder Option Box receives this signal and uses its built-in PI controller to regulate airflow to the configured CO2 setpoint.

---

## System Architecture

```
IKEA Alpstuga sensors (CO2 ppm)
        │
        ▼
Home Assistant (smoothing → worst-room → synthetic sensor voltage %)
        │  brightness_pct maps CO2 ppm to 2–10V range
        ▼
Shelly 0-10V Dimmer PM Gen3 (0–10V analog output)
        │  emulates Zehnder CO2 sensor signal
        ▼
Zehnder Option Box (PI controller → regulates to CO2 setpoint)
        │
        ▼
ComfoAir Q600
```

---

## Hardware

| Component | Role |
|---|---|
| IKEA Alpstuga (×2) | CO2 source sensors |
| Shelly 0-10V Dimmer PM Gen3 | Analog voltage output (0–10 V) |
| Zehnder Option Box | Accepts 0–10 V control input |
| Zehnder ComfoAir Q600 | HRV unit |

> **Important — Shelly sourcing vs sinking:** The Shelly 0-10V Dimmer PM Gen3 is a **sourcing** device: it actively drives voltage on the signal wire. The Zehnder Option Box analog input expects a sourcing signal, so this combination is compatible. Do not use a sinking-type 0-10V controller without additional circuitry.

---

## Zehnder Option Box Settings

The Option Box is configured to receive a CO2 sensor voltage signal and regulate airflow using **PI CONTROL (METHOD: CONTROL)**. The following parameters match Zehnder's recommended settings for a CO2 sensor (0–2000 ppm):

| Parameter | Value |
|---|---|
| METHOD | **CONTROL** |
| INPUT AT 0% | **10.0 V** (= 2000 ppm) |
| INPUT AT 100% | **2.0 V** (= 400 ppm) |
| SETPOINT | **5.0 V** (= 1000 ppm) |
| PROPORTIONAL BAND | **50%** |
| INTEGRAL TIME | **300 s** |
| 0-10V FUNCTION | **FLOW-PROPORTIONAL** |
| 0-10V PRIORITY | **AUTO ONLY** |

**How PI control works here:** The Option Box continuously calculates an error as *setpoint minus measured value*. When CO2 is above 1000 ppm, the measured voltage is below 5 V (because the input is inverted: high ppm = low voltage per the INPUT AT 0%/100% settings), producing a positive error that drives the airflow output upward. The proportional band determines the sensitivity of the response, and the integral time determines how quickly the accumulated error is corrected toward zero. The controller keeps adjusting until CO2 stabilises at the setpoint — it does not produce a simple fixed mapping of voltage to flow.

The input inversion (INPUT AT 0% = 10V, INPUT AT 100% = 2V) is required by the Option Box's control logic: without it, higher CO2 would reduce flow rather than increase it.

At 0 V the Option Box receives no external request and the unit falls back to Preset 1 (normal ventilation), which is safe default behaviour if Home Assistant is unavailable.

---

## Design Decisions

**Emulating a CO2 sensor voltage, not a fan speed.** The Shelly outputs a voltage between 2V (400 ppm) and 10V (2000 ppm), mimicking what a real Zehnder CO2 sensor would produce. The Option Box's PI controller then takes over and regulates airflow to hold CO2 at the configured setpoint. Home Assistant's role is purely to produce the right voltage — the closed-loop control happens inside the Option Box.

**Why linear interpolation in HA is still correct.** Home Assistant maps CO2 ppm to a `brightness_pct` (0–100%) which the Shelly converts to 0–10V. The formula linearly maps the CO2 range to the voltage range a real Zehnder sensor would produce, making our synthetic signal indistinguishable to the Option Box from an actual sensor.

**Worst-room CO2.** The sensor voltage is derived from the highest CO2 reading across all rooms, not an average. This ensures the signal reflects the room with the most urgent ventilation need.

**Smoothing per sensor.** Each Alpstuga reading is smoothed with a 2-minute rolling mean before being compared. This filters out momentary spikes without adding meaningful lag.

**Linear voltage mapping.** The HA template maps CO2 ppm linearly to a 0–100% brightness value (0% = low CO2, 100% = 2000 ppm), which the Shelly converts to 0–10V. This produces a voltage curve identical to what a real Zehnder CO2 sensor would output across the same range.

**0 V = no external request.** The Zehnder Option Box treats 0 V as "no sensor active" and falls back to Preset 1 (normal ventilation). This is consistent with how Zehnder's own CO2 sensors behave and means the system is safe to deploy: if Home Assistant is unavailable, the unit continues ventilating normally.

---

## Files

### `configuration.yaml`

Contains all sensors. Add these blocks to your existing `configuration.yaml` (do not duplicate the top-level `sensor:` or `template:` keys if they already exist).

**What to change:**
- Replace `sensor.alpstuga_air_quality_moni_co2` and `sensor.alpstuga_air_quality_2_co2` with your actual Alpstuga entity IDs.
- Replace `light.shelly0110dimg3_REPLACE_ME` (in the *Ventilation signal* template sensor) with your Shelly entity ID.

Find entity IDs in Home Assistant via **Developer Tools → States**.

### `automations.yaml`

Contains one automation that fires whenever the `fan_speed_target` sensor changes (this sensor holds the 0–100% voltage mapping) and calls `light.turn_on` with the corresponding `brightness_pct` on the Shelly entity, producing the equivalent CO2 sensor voltage.

**What to change:**
- Replace `light.shelly0110dimg3_REPLACE_ME` with your Shelly entity ID (same ID as above).

---

## Deployment Checklist

1. Add sensor blocks from `configuration.yaml` to your HA `configuration.yaml`.
2. Add the automation from `automations.yaml` to your HA `automations.yaml`.
3. Substitute all `REPLACE_ME` placeholders with real entity IDs from **Developer Tools → States**.
4. Validate config: **Developer Tools → YAML → Check Configuration**.
5. Restart Home Assistant.
6. Verify sensor chain in **Developer Tools → States**: Alpstuga → smooth → worst room → sensor voltage % → automation firing → Shelly brightness → correct voltage output.

---

## Future Considerations

- Add more Alpstuga sensors by extending the `max` list in the *CO2 worst room* template.
- Humidity-based boost could be layered in as an additional control signal.
