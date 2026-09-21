# HiveHub off-grid power telemetry

Off-grid builds add **power telemetry** to the normal HiveHub measurement
cycle: LiPo fuel-gauge monitoring with a **MAX17048** (the module used on the
ESP32-C6 Scale Module V0.4). It is an optional I2C module and disabled by
default. Enable it per device in `firmware/include/secrets.h`.

> **Connectivity has moved.** Earlier firmware supported a SIM7080G LTE/NB-IoT
> modem for cellular transport. That has been removed — the ESP32 firmware is now
> **Wi-Fi only**. LTE/NB-IoT, solar charging, and battery management are handled
> by a separate **Power Module** that connects to the Scale Module over
> I2C/ESP-NOW. The backend still accepts `network_transport` so that Power Module
> can report its transport later (on-device firmware reports
> `network_transport: "wifi"`). The old `cellular_ok` / `cellular_csq` fields are
> no longer accepted — they were dropped from the measurement schema; the unused
> database columns are left in place to avoid a destructive migration.

---

## Feature flags

Use numeric `0` / `1` values because the firmware uses preprocessor `#if` checks.

```cpp
#define ENABLE_MAX17048_BATTERY  1
```

| Flag | Default | Effect |
|---|---|---|
| `ENABLE_MAX17048_BATTERY` | `0` | Compiles in MAX17048 support and adds LiPo telemetry fields |

The matching library (`sparkfun/SparkFun MAX1704x Fuel Gauge Arduino Library`)
is listed in `firmware/platformio.ini`.

> **Legacy: INA219 solar telemetry.** Earlier builds could add an INA219 for
> solar/load voltage-current-power telemetry (`ENABLE_INA219_SOLAR`,
> `solar_*` measurement fields). The flag and the backend columns still exist
> for old devices, but the INA219 is no longer part of the recommended setup
> or the BOM — the MAX17048 covers battery state, which is what matters for
> an unattended deployment.

---

## LiPo telemetry with MAX17048

Enable:

```cpp
#define ENABLE_MAX17048_BATTERY 1
#define MAX17048_ALERT_PERCENT  20
#define MAX17048_RCOMP_DEFAULT  0x97   // see "Tuning MAX17048 accuracy" below
```

Wiring uses the shared I2C bus plus the battery sense connection required by the breakout board.

Measurement fields:

| Field | Description |
|---|---|
| `battery_monitor_ok` | MAX17048 detected and readable |
| `battery_voltage_v` | Battery voltage in volts |
| `battery_soc_percent` | Battery state-of-charge percentage |
| `battery_alert` | Alert flag from the fuel gauge |

The backend also returns `battery_voltage` for backwards compatibility, mapped from `battery_voltage_v` when available.

---

## Tuning MAX17048 accuracy with RCOMP

**RCOMP is not a battery capacity/mAh setting.** The MAX17048 has no capacity
input at all — its "ModelGauge" algorithm estimates state-of-charge purely
from the cell's open-circuit voltage curve, regardless of the pack's mAh
rating. If SOC readings look inaccurate for a large pack (e.g. a 10 Ah LiPo),
capacity is not the lever to pull. RCOMP is an 8-bit curve-fit constant
(`0x97` by default) that Maxim's own datasheet describes only vaguely
("contact Maxim for instructions for optimization"); the one documented,
reproducible use of it is **temperature compensation**.

This applies identically to any MAX17048 breakout — Adafruit's, SparkFun's,
or the generic AliExpress module referenced in the BOM. It's the same Maxim
IC and the same RCOMP register in every case; only the physical board differs.

### Temperature compensation

```cpp
#define ENABLE_MAX17048_TEMP_COMPENSATION 1
#define MAX17048_RCOMP_TEMPCO_UP   (-0.5f)
#define MAX17048_RCOMP_TEMPCO_DOWN (-5.0f)
```

When enabled, RCOMP is recomputed every measurement cycle from that cycle's
ambient temperature reading (requires `ENABLE_SHT4X_AMBIENT`,
`ENABLE_SHT3X_AMBIENT`, or `ENABLE_BME280_AMBIENT`), using Maxim's documented
formula:

```
T > 20C:  RCOMP = MAX17048_RCOMP_DEFAULT + (T - 20) * MAX17048_RCOMP_TEMPCO_UP
T <= 20C: RCOMP = MAX17048_RCOMP_DEFAULT + (T - 20) * MAX17048_RCOMP_TEMPCO_DOWN
```

If no ambient sensor is enabled, or the ambient read fails on a given cycle,
this silently no-ops and the static `MAX17048_RCOMP_DEFAULT` stays in effect —
no build-time dependency is enforced.

### Empirical starting points by pack capacity

There is no official Maxim table mapping capacity to RCOMP — capacity and
internal resistance are only loosely correlated, and RCOMP is fundamentally
an internal-resistance/voltage-sag compensation value, not a capacity value.
The table below is a **heuristic starting point** drawn from community
calibration experience, capped at the 10 Ah range HiveHub off-grid builds
typically use:

| Pack capacity | Starting `MAX17048_RCOMP_DEFAULT` | Notes |
|---|---|---|
| < 500 mAh | `0x97` (datasheet default) | Small high-ESR pouch cells; default is tuned for this range |
| 500 – 1500 mAh | `0x80` – `0x90` | Typical single 18650 or small pouch |
| 1500 – 3000 mAh | `0x70` – `0x80` | Common single-cell 18650/21700 |
| 3000 – 6000 mAh | `0x60` – `0x70` | Parallel packs, generally lower ESR |
| 6000 – 10000 mAh | `0x48` – `0x60` | Large parallel packs; lower end of the range typically tracks better |

**Do not treat this table as authoritative.** Use it only as a starting point,
then validate and adjust with the calibration procedure below.

### Calibration procedure

1. Fully charge the pack, then let it rest (no load) for at least 30 minutes
   so the gauge's initial SOC estimate isn't skewed by transient sag/surface
   charge.
2. Log `battery_voltage_v`, `battery_soc_percent`, and (if exposed) the raw
   `getChangeRate()` value across a full, realistic discharge cycle at the
   device's normal duty cycle — not a fast bench discharge, since RCOMP
   compensates for load-dependent sag.
3. Compare the logged SOC curve against the pack's actual known-capacity
   discharge curve (from its datasheet or a bench discharge at constant
   current, tracked independently, e.g. with a lab supply/electronic load or
   the INA219 if `ENABLE_INA219_SOLAR` is also fitted).
4. If the gauge's SOC drops faster than reality in the low-temperature range,
   or reports 0%/100% prematurely, adjust `MAX17048_RCOMP_DEFAULT` up or down
   in small steps (e.g. `0x08` at a time) and repeat the discharge test.
5. Once temperature compensation and RCOMP are both dialed in, expect the
   MAX17048 to still be a voltage-model gauge, not a coulomb counter — accept
   several-percent SOC error near the ends of the charge/discharge curve as
   inherent to the chip, not a config problem.

---

## Backend storage and API behavior

The backend stores power telemetry in dedicated columns as well as in `raw_json`.

The startup schema and `server/migrations/001_offgrid_telemetry.sql` include columns for:

- battery state-of-charge, voltage, monitor status, and alert
- solar voltage/current/power values
- transport and cellular status (reserved for the Power Module)
- calibration mode state
- boot count
- time source

These fields are returned by `/api/v1/measurements/latest` and the HivePal app measurement endpoints.

---

## Power-saving behavior

Normal operation is one wake cycle:

1. Wake from deep sleep or reset.
2. Power up sensors and scale ADCs.
3. Measure weights, temperatures, acoustic levels, and optional power telemetry.
4. Connect over Wi-Fi.
5. Upload the measurement and retry cached rows.
6. Poll config and commands.
7. Power down the scale ADCs and SD where supported.
8. Enter deep sleep until the next send interval.

The MAX17048 sits on the shared I2C bus and is read in step 3 when its flag is enabled.

---

## Practical notes

- Add bulk capacitance close to any LiPo/solar input; load peaks can brown out weak supplies.
- Keep load-cell analog wiring away from switching regulators.
- Set the MAX17048 low-battery alert (`MAX17048_ALERT_PERCENT`) to match your battery and duty cycle.
- `quickStart()` only fires on a genuine MAX17048 power-on-reset (detected via
  the gauge's own RI status bit), not on every ESP32 deep-sleep wake — the
  gauge stays powered continuously off the battery independent of the ESP32's
  sleep state, so re-running `quickStart()` on every wake would repeatedly
  re-seed (and degrade) the SOC estimate.
- Use the SD card cache as protection against temporary Wi-Fi outages, not as a replacement for regular connectivity checks.
- For cellular/solar charging in the field, use the separate Power Module rather than wiring a modem directly to the Scale Module.
