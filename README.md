# Klipper Piston Lung Simulator

Klipper configuration for a 3 L piston lung simulator using a BIGTREETECH SKR Pico V1.0 and a stepper-driven GT2 belt mechanism.

The piston is controlled as a Klipper `manual_stepper`. After manual homing at the physical bottom, it parks at the configured top position. Each breathing cycle starts at the top, moves downward by the distance required for the requested tidal volume, and then returns to the same top position.

> [!CAUTION]
> This is open-loop test equipment, not a medical ventilator. It has no pressure, flow, obstruction, disconnect, or redundant position monitoring. Do not connect it to a person or animal. Use a physical emergency stop and mechanical travel limits.

## Current hardware configuration

- Controller: BIGTREETECH SKR Pico V1.0
- Motor connection: X motor socket and onboard TMC2209
- Transmission: direct-drive GT2, 16-tooth pulley
- Rotation distance: 32 mm per motor revolution
- Maximum mechanical travel: 450 mm
- Stepper current: 1.1 A RMS; active driver cooling is required
- Klipper host connection: SKR Pico dedicated Raspberry Pi UART connector
- Optional 24 V fan: hotend-heater output, GPIO23

The configuration assumes that positive motor movement goes from the physical bottom toward the top. If the motor moves in the wrong direction, invert `dir_pin` in `[manual_stepper piston]`; do not compensate by changing the coordinate convention.

## Installation

1. Build and flash Klipper for the SKR Pico using UART0 on GPIO0/GPIO1, not USB.
2. Connect the Raspberry Pi to the board's dedicated UART connector.
3. Copy [`printer.cfg`](printer.cfg) into the Klipper configuration directory.
4. Ensure `mainsail.cfg` exists because it is included by `printer.cfg`.
5. Run `FIRMWARE_RESTART` and confirm that the MCU connects through `/dev/serial0`.

## Calibration

The supplied volume calibration is nominal:

```text
3000 mL / 450 mm = 6.6667 mL/mm
```

Measure the actual displaced volume and update `variable_ml_per_mm` under `[gcode_macro _LUNG_STATE]`. Belt geometry, piston leakage, tubing compliance, and dead volume can make the delivered volume differ from the nominal value.

`SET_TRAVEL_LENGTH` changes the operational travel only until the next Klipper restart. To make another value permanent, update both:

```ini
[manual_stepper piston]
position_max: 450.0

[gcode_macro _LUNG_STATE]
variable_travel_length_mm: 450.0
```

The operational travel may be smaller than `position_max`, but never larger.

## Manual homing and operation

After every Klipper restart, power interruption, driver reset, skipped step, or manual movement, home the piston again:

```gcode
RELEASE_PISTON
```

Move the piston by hand to its maximum physical bottom position, then run:

```gcode
SET_PISTON_BOTTOM
```

This sets the current position to `0` and, by default, moves the piston upward to the configured top. To set zero without moving away from the bottom, use `SET_PISTON_BOTTOM PARK=0`, followed later by `PARK_PISTON_TOP`.

Start continuous breathing by providing tidal volume in millilitres and breaths per minute:

```gcode
BREATHE TV=500 BPM=12
```

Stop gracefully and return to the top:

```gcode
STOP_BREATHING
```

Use `M112` for an emergency stop. This does not replace a hard-wired emergency-stop circuit.

## Macro reference

| Command | Purpose |
| --- | --- |
| `SET_TRAVEL_LENGTH LENGTH=<mm>` | Temporarily set the operational bottom-to-top travel. |
| `RELEASE_PISTON` | Disable the motor and invalidate the manual home. |
| `SET_PISTON_BOTTOM` | Set the current physical bottom to zero and park at the top. |
| `SET_PISTON_BOTTOM PARK=0` | Set bottom zero without moving the piston. |
| `PARK_PISTON_TOP` | Move a homed, idle piston to the operational top. |
| `RETURN_TO_BOTTOM` | Move a homed, idle piston to zero. |
| `BREATHE TV=<mL> BPM=<rate>` | Start continuous breathing with a 1:1 motion-time ratio. |
| `STOP_BREATHING` | Finish the queued motion and return to the top. |
| `STOP_BREATHING RETURN=0` | Stop without commanding a return to the top. |
| `STOP_BREATHING DISABLE=1` | Stop, release the motor, and invalidate home. |
| `LUNG_STATUS` | Report the home, position-state, travel, TV, BPM, and capacity state. |

The `BREATHE` macro calculates stroke distance from `TV / ml_per_mm`. It accounts for the configured acceleration when calculating motion speed and rejects combinations that exceed the configured speed, acceleration, travel, volume, or BPM limits.

## Fan output warning

The optional fan is configured on the hotend-heater output:

```ini
[fan]
pin: gpio23
kick_start_time: 0.0
```

GPIO23 switches the HE connector through a low-side MOSFET, but this heater output does not have the flyback diode found on the SKR Pico's dedicated fan outputs. A 24 V brushless fan can produce startup-current and switching transients that reset the TMC2209, particularly while the stepper is active.

If GPIO23 must be used, provide correctly rated external transient suppression or a protected fan-driver module, verify power-supply headroom and wiring, and monitor the 24 V rail during fan startup. GPIO21 is also a heater output and does not solve this issue. Do not continue operation after a TMC reset without manually homing again.

## License

No license has been selected. All rights are reserved unless a license is added later.
