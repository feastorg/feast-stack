# BREAD Device Catalog

BREAD (Bread Relay-Extruder Automation Devices) is the hardware family. Each device type is
identified by a numeric type ID assigned in bread-crumbs-contracts and communicated during
capability discovery. Device firmware runs on AVR/ARM microcontrollers and speaks the CRUMBS I2C
framing protocol.

---

## Common Protocol (all BREAD devices)

All BREAD devices share two universal opcodes defined in bread-crumbs-contracts:

| Opcode | Name                          | Request payload | Reply payload                                                                              |
| ------ | ----------------------------- | --------------- | ------------------------------------------------------------------------------------------ |
| `0x00` | Version query                 | empty           | `[crumbs_version : u16_le]  [module_major : u8]  [module_minor : u8]  [module_patch : u8]` |
| `0x7F` | Capability query (`GET_CAPS`) | empty           | `[caps_schema : u8 = 0x01]  [caps_level : u8]  [caps_flags : u32_le]`                      |

Version query is always run during discovery. Capability query is attempted and, on failure, a
per-type baseline fallback is used instead (see `docs/stack.md` — Baseline fallback).

---

## RLHT — Relay-Heater with Thermocouples

**Type ID:** `0x01`  
**ADPP type string:** `bread.rlht`

A dual-channel closed-loop heater controller with thermocouple temperature sensing and relay
switching outputs. Supports open-loop PWM duty control and closed-loop PID temperature control.

### Signals

All signals are read from a single `GET_STATE` RPC call (opcode `0x01`). Payload is 19 bytes.

| Signal        | Type   | Description                                                             |
| ------------- | ------ | ----------------------------------------------------------------------- |
| `mode`        | STRING | Current control mode: `"open_loop"` or `"closed_loop"`                  |
| `t1_c`        | DOUBLE | Channel 1 thermocouple temperature, °C (encoded as deci-°C i16_le ÷ 10) |
| `t2_c`        | DOUBLE | Channel 2 thermocouple temperature, °C (encoded as deci-°C i16_le ÷ 10) |
| `setpoint1_c` | DOUBLE | Channel 1 closed-loop temperature setpoint, °C                          |
| `setpoint2_c` | DOUBLE | Channel 2 closed-loop temperature setpoint, °C                          |
| `period1_ms`  | UINT64 | Channel 1 PWM period, milliseconds                                      |
| `period2_ms`  | UINT64 | Channel 2 PWM period, milliseconds                                      |
| `relay1_on`   | BOOL   | `true` if relay 1 is currently energised (flag `RLHT_FLAG_RELAY1_ON`)   |
| `relay2_on`   | BOOL   | `true` if relay 2 is currently energised (flag `RLHT_FLAG_RELAY2_ON`)   |
| `estop`       | BOOL   | `true` if emergency stop is active (flag `RLHT_FLAG_ESTOP`)             |

**Wire payload layout (GET_STATE, 19 bytes minimum):**

```
[0]     mode       u8
[1]     flags      u8   (bit 0 = relay1_on, bit 1 = relay2_on, bit 2 = estop)
[2-3]   t1_deci_c  i16 LE
[4-5]   t2_deci_c  i16 LE
[6-7]   sp1_deci_c i16 LE
[8-9]   sp2_deci_c i16 LE
[10-11] on1_ms     u16 LE  (relay 1 on-time; not exposed as ADPP signal)
[12-13] on2_ms     u16 LE  (relay 2 on-time; not exposed as ADPP signal)
[14-15] period1_ms u16 LE
[16-17] period2_ms u16 LE
[18]    tc_select  u8      (thermocouple index; not exposed as ADPP signal)
```

### Functions

| ID  | Name                | Category | Args                                                                                 | Description                                                      |
| --- | ------------------- | -------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| 1   | `set_mode`          | CONFIG   | `mode: STRING` (`"open_loop"` or `"closed_loop"`)                                    | Switch control mode                                              |
| 2   | `set_setpoints`     | CONFIG   | `setpoint1_c: DOUBLE`, `setpoint2_c: DOUBLE`                                         | Set closed-loop temperature targets (°C)                         |
| 3   | `set_pid_x10`       | CONFIG   | `kp1_x10`, `ki1_x10`, `kd1_x10`, `kp2_x10`, `ki2_x10`, `kd2_x10`: UINT64 in [0, 255] | Set PID gains × 10 for both channels                             |
| 4   | `set_periods_ms`    | CONFIG   | `period1_ms`, `period2_ms`: UINT64 in [0, 65535]                                     | Set PWM relay periods                                            |
| 5   | `set_tc_select`     | CONFIG   | `tc1_index`, `tc2_index`: UINT64 in [0, 255]                                         | Select which physical thermocouple input to use for each channel |
| 6   | `set_open_duty_pct` | ACTUATE  | `duty1_pct`, `duty2_pct`: UINT64 in [0, 100]                                         | Set open-loop relay duty cycle (%) for both channels             |

### Capability Flags

Capability flags gate which functions appear in the ADPP `DescribeDevice` response. RLHT baseline
fallback (when GET_CAPS fails) sets all currently supported flags — closed-loop functions appear
by default.

| Capability level | Includes                                                 |
| ---------------- | -------------------------------------------------------- |
| Level 1          | All currently supported RLHT capability flags (baseline) |

### Hardware Variants

| Generation | Controller                      | Capability level |
| ---------- | ------------------------------- | ---------------- |
| Gen 1      | ATmega328P (Arduino Nano)       | Level 1          |
| Gen 2      | ATmega4809 (Arduino Nano Every) | Level 1          |

---

## DCMT — Dual-Channel Motor with Thermistors

**Type ID:** `0x02`  
**ADPP type string:** `bread.dcmt`

A dual-channel DC motor controller with thermistor temperature sensing. Supports open-loop PWM
(`open_loop`), closed-loop position control (`closed_position`), and closed-loop speed control
(`closed_speed`). Closed-loop modes require encoder hardware.

### Signals

All signals are read from a single `GET_STATE` RPC call (opcode `0x01`). Payload size depends
on mode.

| Signal          | Type   | Description                                                                      |
| --------------- | ------ | -------------------------------------------------------------------------------- |
| `mode`          | STRING | Current control mode: `"open_loop"`, `"closed_position"`, or `"closed_speed"`    |
| `motor1_target` | INT64  | Channel 1 target value (open-loop: PWM, closed-loop: position or speed setpoint) |
| `motor2_target` | INT64  | Channel 2 target value                                                           |
| `motor1_value`  | INT64  | Channel 1 measured value; equals `motor1_target` in open-loop mode               |
| `motor2_value`  | INT64  | Channel 2 measured value; equals `motor2_target` in open-loop mode               |
| `motor1_brake`  | BOOL   | `true` if channel 1 brake is active (brakes byte bit 0)                          |
| `motor2_brake`  | BOOL   | `true` if channel 2 brake is active (brakes byte bit 1)                          |
| `estop`         | BOOL   | `true` if emergency stop is active                                               |

**Wire payload layout (GET_STATE) — mode-conditional:**

Open-loop (`mode = 0x00`, 7 bytes):

```
[0]   mode     u8  = 0x00
[1-2] target1  i16 LE
[3-4] target2  i16 LE
[5]   brakes   u8   (bit 0 = motor1_brake, bit 1 = motor2_brake)
[6]   estop    u8
```

Closed-loop (`mode = 0x01` or `0x02`, 11 bytes):

```
[0]   mode     u8  = 0x01 (closed_position) or 0x02 (closed_speed)
[1-2] target1  i16 LE
[3-4] target2  i16 LE
[5-6] value1   i16 LE  (measured encoder position or speed)
[7-8] value2   i16 LE
[9]   brakes   u8
[10]  estop    u8
```

### Functions

| ID  | Name            | Category | Args                                                                                 | Description                              |
| --- | --------------- | -------- | ------------------------------------------------------------------------------------ | ---------------------------------------- |
| 1   | `set_open_loop` | ACTUATE  | `motor1_pwm: INT64 [-32768, 32767]`, `motor2_pwm: INT64`                             | Set open-loop PWM for both channels      |
| 2   | `set_brake`     | ACTUATE  | `motor1_brake: BOOL`, `motor2_brake: BOOL`                                           | Apply or release brake on each channel   |
| 3   | `set_mode`      | CONFIG   | `mode: STRING` (`"open_loop"`, `"closed_position"`, or `"closed_speed"`)             | Switch control mode                      |
| 4   | `set_setpoint`  | CONFIG   | `motor1_target: INT64 [-32768, 32767]`, `motor2_target: INT64`                       | Set closed-loop position or speed target |
| 5   | `set_pid_x10`   | CONFIG   | `kp1_x10`, `ki1_x10`, `kd1_x10`, `kp2_x10`, `ki2_x10`, `kd2_x10`: UINT64 in [0, 255] | Set PID gains × 10 for both channels     |

### Capability Flags and Levels

DCMT capabilities distinguish open-loop from closed-loop hardware. The baseline fallback on
GET_CAPS failure is conservative: only `OPEN_LOOP_CONTROL | BRAKE_CONTROL`.

| Level   | Capabilities                                            |
| ------- | ------------------------------------------------------- |
| Level 1 | Open-loop PWM + brake control                           |
| Level 2 | + Closed-loop position control + PID gain configuration |
| Level 3 | + Closed-loop speed control                             |

### Hardware Variants

| Generation | Controller                      | Capability level |
| ---------- | ------------------------------- | ---------------- |
| Gen 1      | ATmega328P (Arduino Nano)       | Level 2          |
| Gen 2      | ATmega328P (Arduino Nano)       | Level 2          |
| Gen 2      | ATmega4809 (Arduino Nano Every) | Level 3          |
