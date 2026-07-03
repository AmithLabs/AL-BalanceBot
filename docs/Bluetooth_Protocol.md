# Self-Balancing Robot Bluetooth Protocol

This document describes the Bluetooth communication protocol for the self-balancing robot firmware.

---

## Bluetooth Settings

```text
Module: HC-05
Baud rate: 9600
Arduino Serial: Hardware Serial D0 / D1
Data format: ASCII text command
```

Commands can be ended with:

```text
\n
\r
;
```

The firmware also accepts commands after a short receive timeout, so most app buttons can send plain text such as:

```text
FORWARD
```

Recommended delimiter:

```text
FORWARD\n
```

---

# App → Robot Commands

## 1. Movement Commands

These commands control only movement. They do not turn balancing on/off.

### Forward

```text
FORWARD
```

Effect:

```text
Robot moves forward.
Robot replies OK.
ST telemetry becomes FORWARD.
```

Important:

```text
The app should keep sending FORWARD repeatedly while the button is held.
If no movement command is received for 2 seconds, the robot automatically stops movement.
Balancing remains active.
```

---

### Backward

```text
BACKWARD
```

Effect:

```text
Robot moves backward.
Robot replies OK.
ST telemetry becomes BACKWARD.
```

---

### Rotate Left

```text
LEFT
```

Effect:

```text
Robot rotates left.
Robot replies OK.
ST telemetry becomes LEFT.
```

---

### Rotate Right

```text
RIGHT
```

Effect:

```text
Robot rotates right.
Robot replies OK.
ST telemetry becomes RIGHT.
```

---

### Stop Movement

```text
STOP
```

Effect:

```text
Robot stops forward/backward/left/right movement.
Robot replies OK.
Balancing remains active if already enabled and upright.
ST telemetry returns to BALANCING.
```

---

## 2. Enable / Disable Commands

### Enable Balancing

```text
ENABLE
```

Effect:

```text
Balancing permission is enabled.
This ENABLE state is saved to EEPROM.
Robot replies OK.
```

Balancing starts only when:

```text
ENABLE state is active
and robot angle is between -1° and +1° from saved balance point
```

When balancing becomes active:

```text
DRV8825 ENABLE pin D8 goes LOW
Motors are enabled
```

---

### Disable Balancing

```text
DISABLE
```

Effect:

```text
Balancing is disabled.
This DISABLE state is saved to EEPROM.
Robot replies OK.
Movement commands are cleared.
Motors are disabled.
```

When disabled:

```text
DRV8825 ENABLE pin D8 goes HIGH
ST telemetry becomes DISABLED
OLED eyes show disabled / closed-eye mode
```

Correct command:

```text
DISABLE
```

Wrong old spelling is not supported:

```text
DESABLE
```

---

## 3. Calibration Commands

### Gyro / Accelerometer Calibration

```text
CAL_GYRO
```

Use when:

```text
Robot is upright
Robot is completely still
Wheels are not moving
```

Effect:

```text
Stops movement
Disables motor drivers during calibration
Measures MPU6050 gyro offsets
Measures accelerometer balance reference
Saves calibration data to EEPROM
Robot replies OK
```

Saved to EEPROM:

```text
gyroX_offset
gyroY_offset
gyroZ_offset
angle_acc_offset
current PID values
current ENABLE/DISABLE state
```

---

### Balance Point Calibration

```text
CAL_BALANCE
```

Use when:

```text
Robot is held at the real mechanical balance point
```

Effect:

```text
Current angle becomes the robot balance center.
manual_balance_offset is updated.
Calibration is saved to EEPROM.
Movement and PID memory are cleared.
Robot replies OK.
```

Saved to EEPROM:

```text
manual_balance_offset
current PID values
gyro / accelerometer calibration values
current ENABLE/DISABLE state
```

---

### Calibration Reset

```text
CAL_RESET
```

Current firmware behavior:

```text
Robot replies OK.
No values are changed.
EEPROM is not cleared.
PID is not reset.
Calibration is not reset.
```

This command exists only for app compatibility.

---

## 4. PID Commands

### Request Current PID Values

```text
PIDREQUEST
```

Robot reply format:

```text
PID,KP=3.5,KI=0.000,KD=4.5
```

Exact format:

```text
PID,KP=<1 decimal>,KI=<3 decimals>,KD=<1 decimal>
```

Example replies:

```text
PID,KP=3.5,KI=0.000,KD=4.5
PID,KP=4.2,KI=0.002,KD=5.0
PID,KP=18.0,KI=0.500,KD=1.2
```

Important:

```text
PIDREQUEST does not reply OK.
It replies only the PID data line.
```

---

### Update PID Values

```text
PID,KP=3.5,KI=0.000,KD=4.5
```

Effect:

```text
KP, KI, KD are updated immediately.
PID integral memory is cleared.
New PID values are saved to EEPROM.
Robot replies OK.
```

Invalid PID command:

```text
PID,KP=3.5,KD=4.5
```

Reply:

```text
ERR
```

Because `KI=` is missing.

Firmware-side PID limits:

```text
No PID min/max limits are applied in firmware.
The app may send any float values.
PID tuning must be done carefully by the user.
```

Recommended app slider ranges:

```text
KP: 0.0 to 8.0
KI: 0.000 to 0.020
KD: 0.0 to 10.0
```

Firmware does not block values outside these ranges.

---

## 5. Config Command

The app may send:

```text
CFG,MAX_SPEED=50.3,MAX_TILT=12.0
```

Current firmware behavior:

```text
Robot replies OK.
No configuration values are changed.
Nothing is saved to EEPROM.
```

This command exists only for app compatibility.

Any command starting with:

```text
CFG,
```

will reply:

```text
OK
```

---

# Robot → App Messages

## 1. Startup Messages

After boot, robot sends:

```text
READY
```

Then initial telemetry:

```text
A:0.00,B:0.00,LM:0,RM:0,ST:READY
```

---

## 2. Command Replies

Most valid commands reply:

```text
OK
```

Examples:

```text
FORWARD  -> OK
BACKWARD -> OK
LEFT     -> OK
RIGHT    -> OK
STOP     -> OK
ENABLE   -> OK
DISABLE  -> OK
CAL_GYRO -> OK
CAL_BALANCE -> OK
CAL_RESET -> OK
PID,KP=3.5,KI=0.000,KD=4.5 -> OK
CFG,MAX_SPEED=50.3,MAX_TILT=12.0 -> OK
```

Invalid command reply:

```text
ERR
```

Example:

```text
HELLO
```

Reply:

```text
ERR
```

If the receive buffer overflows:

```text
RX_OVERFLOW
```

---

## 3. PID Response

When app sends:

```text
PIDREQUEST
```

Robot replies:

```text
PID,KP=3.5,KI=0.000,KD=4.5
```

No `OK` is sent after this.

---

# Telemetry Packet

Robot sends telemetry every:

```text
200 ms
```

Format:

```text
A:<angle>,B:<battery_voltage>,LM:<left_motor>,RM:<right_motor>,ST:<state>
```

Example:

```text
A:0.24,B:12.18,LM:15,RM:15,ST:BALANCING
```

---

## Telemetry Fields

### `A`

```text
A:<angle>
```

Example:

```text
A:0.24
```

Meaning:

When balancing is active:

```text
A = balance error angle
```

So when the robot is stable, `A` should be close to:

```text
0.00
```

When balancing is inactive:

```text
A = raw angle relative to saved CAL_BALANCE point
```

Decimal places:

```text
2 decimals
```

---

### `B`

```text
B:<battery_voltage>
```

Example:

```text
B:12.18
```

Meaning:

```text
Battery voltage measured from A6 through voltage divider.
```

Decimal places:

```text
2 decimals
```

Battery input wiring expected by firmware:

```text
Battery +  -> R1 -> A6 -> R2 -> GND
Battery -  -> Arduino GND
```

Default resistor values in firmware:

```text
R1 = 100k
R2 = 33k
```

Never connect battery voltage directly to A6.

---

### `LM`

```text
LM:<left_motor_output>
```

Example:

```text
LM:15
```

Meaning:

```text
Left motor target output value.
This is not RPM.
This is an internal motor command value used by the firmware.
```

---

### `RM`

```text
RM:<right_motor_output>
```

Example:

```text
RM:15
```

Meaning:

```text
Right motor target output value.
This is not RPM.
This is an internal motor command value used by the firmware.
```

---

### `ST`

```text
ST:<state>
```

Possible values:

```text
DISABLED
READY
FALLEN
FORWARD
BACKWARD
LEFT
RIGHT
BALANCING
```

---

# State Meaning

## `DISABLED`

```text
Balancing is disabled.
D8 is HIGH.
Motor drivers are disabled.
```

---

## `READY`

```text
ENABLE state is active, but robot has not armed balancing yet.
Usually angle is not inside -1° to +1° arming window.
```

---

## `FALLEN`

```text
Robot angle is greater than TIP_OVER_ANGLE_LIMIT.
Balancing is deactivated.
D8 is HIGH.
```

---

## `BALANCING`

```text
Robot is actively self-balancing.
No movement command is currently active.
```

---

## `FORWARD`

```text
Robot is balancing and forward command is active.
```

---

## `BACKWARD`

```text
Robot is balancing and backward command is active.
```

---

## `LEFT`

```text
Robot is balancing and left rotation command is active.
```

---

## `RIGHT`

```text
Robot is balancing and right rotation command is active.
```

---

# Recommended App Button Behavior

For movement buttons:

```text
When button pressed:
  send FORWARD / BACKWARD / LEFT / RIGHT immediately

While button is held:
  repeat the same command every 100 ms to 300 ms

When button released:
  send STOP
```

Reason:

```text
Firmware has a 2 second movement timeout.
If movement commands stop arriving, robot stops movement automatically.
Balancing continues.
```

Recommended repeat interval:

```text
100 ms
```

---

# Full Command Summary

## App → Robot

```text
FORWARD
BACKWARD
LEFT
RIGHT
STOP
ENABLE
DISABLE
CAL_GYRO
CAL_BALANCE
CAL_RESET
PIDREQUEST
PID,KP=<value>,KI=<value>,KD=<value>
CFG,<anything>
```

## Robot → App

```text
READY
OK
ERR
RX_OVERFLOW
PID,KP=<value>,KI=<value>,KD=<value>
A:<angle>,B:<battery>,LM:<left>,RM:<right>,ST:<state>
```

---

# Example Communication Flow

## First-time setup

```text
App -> Robot: CAL_GYRO
Robot -> App: OK

App -> Robot: CAL_BALANCE
Robot -> App: OK

App -> Robot: PID,KP=3.5,KI=0.000,KD=4.5
Robot -> App: OK

App -> Robot: ENABLE
Robot -> App: OK
```

## Request PID

```text
App -> Robot: PIDREQUEST
Robot -> App: PID,KP=3.5,KI=0.000,KD=4.5
```

## Move forward

```text
App -> Robot: FORWARD
Robot -> App: OK

Robot -> App: A:0.12,B:12.18,LM:14,RM:14,ST:FORWARD

App -> Robot: STOP
Robot -> App: OK
```

## Disable robot

```text
App -> Robot: DISABLE
Robot -> App: OK

Robot -> App: A:2.45,B:12.10,LM:0,RM:0,ST:DISABLED
```
