# Self-Balancing Robot Calibration Procedure

This document explains the correct calibration procedure for the self-balancing robot firmware.

Firmware target: final self-balancing robot firmware with:

- MPU6050 calibration stored in EEPROM
- Balance point calibration stored in EEPROM
- PID values stored in EEPROM
- ENABLE / DISABLE state stored in EEPROM
- Bluetooth app control
- OLED eye display
- Battery telemetry on A6

---

## 1. What Calibration Does

The robot has two main calibration commands:

```text
CAL_GYRO
CAL_BALANCE
```

These two commands are different and both are required.

---

## 2. `CAL_GYRO`

```text
CAL_GYRO
```

This command calibrates the MPU6050 sensor.

It measures and saves:

```text
gyroX_offset
gyroY_offset
gyroZ_offset
angle_acc_offset
```

These values are saved to EEPROM.

### Purpose

`CAL_GYRO` removes gyro sensor drift and sets the accelerometer reference while the robot is upright and still.

### When to Use

Use `CAL_GYRO`:

- First time after uploading firmware
- After changing MPU6050 mounting
- After replacing MPU6050 sensor
- After changing robot frame geometry
- If the app angle slowly drifts while the robot is still
- If the robot does not detect upright angle correctly

### Important Condition

During `CAL_GYRO`, the robot must be:

```text
upright
still
not moving
on a flat surface or held steady
```

Do not touch, shake, or move the robot during this calibration.

---

## 3. `CAL_BALANCE`

```text
CAL_BALANCE
```

This command saves the real mechanical balance point.

It updates and saves:

```text
manual_balance_offset
```

### Purpose

A self-balancing robot is not always perfectly balanced at exactly 0 degrees. Battery position, 3D printed parts, wires, display, and chassis weight can make the real balance point slightly forward or backward.

`CAL_BALANCE` tells the robot:

```text
This current angle is my real balance center.
```

### When to Use

Use `CAL_BALANCE`:

- First time after `CAL_GYRO`
- After moving the battery
- After changing the OLED/display position
- After changing the frame/body
- After adding or removing weight
- If the robot stands but always leans forward/backward
- If the app angle does not look centered after calibration

### Important Condition

During `CAL_BALANCE`, hold the robot at the real mechanical balance position.

That means:

```text
The position where the robot can almost stand naturally
without strongly falling forward or backward.
```

Do not hold it at an artificial angle.

---

## 4. `ENABLE`

```text
ENABLE
```

This command allows balancing to start.

The ENABLE state is saved to EEPROM.

### Important

The robot does not start balancing immediately just because `ENABLE` is sent.

Balancing starts only when:

```text
ENABLE state is active
and
robot angle is inside the auto-arm window
```

Current auto-arm window:

```text
-1° < angle < +1°
```

This safety prevents the motors from activating while the robot is lying down or tilted too far.

---

## 5. `DISABLE`

```text
DISABLE
```

This command turns balancing off.

The DISABLE state is saved to EEPROM.

When disabled:

```text
motor drivers are disabled
D8 goes HIGH
movement commands are cleared
OLED eyes show disabled / closed-eye mode
```

Use `DISABLE` before carrying, adjusting, or working on the robot.

---

## 6. `CAL_RESET`

```text
CAL_RESET
```

Current firmware behavior:

```text
Robot replies OK only.
No calibration data is changed.
EEPROM is not cleared.
PID values are not reset.
```

This command is included only for app compatibility.

---

# First-Time Calibration Procedure

Use this procedure after uploading firmware for the first time.

---

## Step 1: Place Robot Safely

Put the robot on a stand or hold it carefully so the wheels can rotate freely.

Recommended:

```text
Wheels should not touch the floor during first calibration and first enable test.
```

---

## Step 2: Power On Robot

Power on the robot.

The firmware will load saved EEPROM data if valid data exists.

Important:

```text
The robot does not automatically run gyro calibration at every startup.
```

This is intentional because calibration values are saved to EEPROM.

---

## Step 3: Keep Robot Upright and Still

Hold the robot upright and completely still.

Do not move the robot during the next step.

---

## Step 4: Send `CAL_GYRO`

From the Android app or Bluetooth terminal, send:

```text
CAL_GYRO
```

Expected robot reply:

```text
OK
```

During this process:

```text
motor drivers are disabled
MPU6050 gyro offsets are measured
accelerometer angle reference is measured
calibration values are saved to EEPROM
```

Wait until the robot replies `OK`.

---

## Step 5: Hold Mechanical Balance Point

Now hold the robot at its real mechanical balance point.

This is usually close to vertical, but not always exactly vertical.

Correct position:

```text
Robot feels naturally balanced.
You do not need to force it strongly forward or backward.
```

---

## Step 6: Send `CAL_BALANCE`

Send:

```text
CAL_BALANCE
```

Expected robot reply:

```text
OK
```

This saves the current angle as:

```text
manual_balance_offset
```

This value is saved to EEPROM.

---

## Step 7: Send PID Values

Send your starting PID values.

Example:

```text
PID,KP=3.5,KI=0.000,KD=4.5
```

Expected robot reply:

```text
OK
```

These PID values are saved to EEPROM.

Recommended starting values:

```text
KP = 3.5
KI = 0.000
KD = 4.5
```

---

## Step 8: Enable Balancing

Send:

```text
ENABLE
```

Expected robot reply:

```text
OK
```

The ENABLE state is saved to EEPROM.

---

## Step 9: Bring Robot Inside Auto-Arm Angle

Hold the robot upright so the angle is inside:

```text
-1° to +1°
```

When the condition is correct:

```text
balancing becomes active
D8 goes LOW
motor drivers are enabled
ST telemetry becomes BALANCING
OLED eyes change from disabled/calibration mode to normal/blink mode
```

---

# Normal Startup Procedure After Calibration

After `CAL_GYRO`, `CAL_BALANCE`, PID, and `ENABLE` are saved once, normal startup is simple.

## Step 1: Power On

The robot loads saved EEPROM data.

Loaded values include:

```text
gyro offsets
accelerometer angle offset
manual balance offset
PID values
ENABLE / DISABLE state
```

---

## Step 2: Hold Robot Upright

Hold the robot upright near the saved balance point.

---

## Step 3: Wait for Auto-Arm

If saved state is ENABLE and angle is inside:

```text
-1° to +1°
```

then balancing activates automatically.

No new `CAL_GYRO` is required at every startup.

---

# Recalibration Procedure

Use recalibration when the robot behavior changes after hardware changes.

---

## Recalibrate After These Changes

Run full calibration again after:

- Changing MPU6050 mounting
- Replacing MPU6050 module
- Moving battery position
- Changing wheel size
- Changing motor position
- Adding OLED/display weight
- Changing frame or 3D printed parts
- Rewiring the robot
- Robot suffered a hard fall
- Robot angle reading looks wrong
- Robot always leans one direction

---

## Full Recalibration Steps

Send commands in this order:

```text
DISABLE
CAL_GYRO
CAL_BALANCE
ENABLE
```

Detailed flow:

```text
1. Send DISABLE
2. Hold robot upright and still
3. Send CAL_GYRO
4. Wait for OK
5. Hold robot at real mechanical balance point
6. Send CAL_BALANCE
7. Wait for OK
8. Send ENABLE
9. Hold robot inside -1° to +1°
10. Wait for BALANCING state
```

---

# PID Recalibration / Retuning

PID tuning is separate from gyro and balance calibration.

Use PID tuning when:

- Robot balances but oscillates
- Robot feels weak
- Robot falls slowly
- Robot vibrates
- Wheel size changed
- Robot weight changed

Request current PID values:

```text
PIDREQUEST
```

Robot reply example:

```text
PID,KP=3.5,KI=0.000,KD=4.5
```

Update PID:

```text
PID,KP=3.5,KI=0.000,KD=4.5
```

Expected reply:

```text
OK
```

PID values are saved to EEPROM automatically.

---

# Recommended PID Starting Values

Use these values after a fresh calibration:

```text
KP = 3.5
KI = 0.000
KD = 4.5
```

Tuning notes:

| Symptom | Adjustment |
|---|---|
| Robot is weak and falls slowly | Increase KP slightly |
| Robot oscillates forward/backward | Reduce KP or increase KD slightly |
| Motor vibrates/noisy | Reduce KD |
| Robot slowly drifts but is stable | Try KI = 0.001 |
| Robot falls after adding KI | Set KI back to 0.000 |

Important:

```text
KI = 0.1 is too high for this firmware.
Start with KI = 0.000.
Only test very small values such as 0.001 or 0.002.
```

---

# App Telemetry During Calibration

Telemetry format:

```text
A:<angle>,B:<battery_voltage>,LM:<left_motor>,RM:<right_motor>,ST:<state>
```

Example:

```text
A:0.24,B:12.18,LM:15,RM:15,ST:BALANCING
```

## Important Fields

### `A`

When balancing is active:

```text
A = balance error angle
```

Stable robot should show angle close to:

```text
0.00
```

When balancing is inactive:

```text
A = raw angle relative to saved CAL_BALANCE point
```

### `ST`

Possible states:

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

During calibration and startup, watch this field carefully.

---

# OLED Eye Behavior During Calibration

The OLED eye display helps show robot state.

| Robot state | OLED behavior |
|---|---|
| Calibration data missing | Calibration eyes |
| Disabled | Closed eyes |
| Normal balancing | Normal / blink eyes |
| Warning angle | Warning eyes |
| Fallen | Fall / X eyes |

If the robot is disabled, closed eyes are normal.

---

# Troubleshooting Calibration Problems

## Problem: Robot does not enable motors after `ENABLE`

Possible causes:

- Robot angle is not inside `-1° to +1°`
- Saved state is still DISABLE
- Calibration values are invalid
- Robot is tilted too far
- App did not send `ENABLE` correctly

Fix:

```text
1. Send ENABLE
2. Hold robot upright near 0°
3. Check telemetry ST field
4. If still not active, redo CAL_GYRO and CAL_BALANCE
```

---

## Problem: Robot falls immediately when balancing activates

Possible causes:

- Wrong motor direction
- Wrong MPU angle direction
- Wrong `MOUNT_MODE`
- Wrong `PID_OUTPUT_SIGN`
- Bad calibration

Fix order:

```text
1. Check motor directions
2. Check MOUNT_MODE
3. Check ANGLE_SIGN
4. Check PID_OUTPUT_SIGN
5. Run CAL_GYRO
6. Run CAL_BALANCE
```

---

## Problem: App angle starts around 4–8 degrees and slowly changes

Possible causes:

- Robot is not using the correct saved balance offset
- Calibration was done while robot was moving
- Gyro/accelerometer reference is settling
- Robot physical balance point is not calibrated correctly

Fix:

```text
1. DISABLE
2. Hold upright and still
3. CAL_GYRO
4. Hold true balance point
5. CAL_BALANCE
6. ENABLE
```

---

## Problem: Robot balances but always rolls forward/backward

Possible causes:

- Balance point not calibrated correctly
- Battery weight shifted
- Stop-drift trim direction needs tuning
- Floor is not level

Fix:

```text
1. Redo CAL_BALANCE
2. Check self-balance trim parameters
3. Retune PID if needed
```

---

## Problem: `CAL_GYRO` gives bad result

Possible causes:

- Robot moved during calibration
- MPU6050 vibration
- Robot was not upright
- Power supply noise

Fix:

```text
1. Put robot on stable surface
2. Hold completely still
3. Send CAL_GYRO again
```

---

## Problem: Calibration lost after power off

Possible causes:

- EEPROM save failed
- Firmware EEPROM version changed
- EEPROM data invalid
- Arduino board EEPROM issue

Fix:

```text
1. Send CAL_GYRO
2. Send CAL_BALANCE
3. Send PID command
4. Send ENABLE
5. Power cycle and check if values remain
```

---

# Recommended First-Time Full Setup

Use this complete command sequence:

```text
CAL_GYRO
CAL_BALANCE
PID,KP=3.5,KI=0.000,KD=4.5
ENABLE
```

Then hold the robot upright inside:

```text
-1° to +1°
```

Expected final telemetry:

```text
ST:BALANCING
A close to 0.00
LM and RM changing slightly
```

---

# Recommended Safe Shutdown

Before carrying or working on the robot, send:

```text
DISABLE
```

Expected result:

```text
ST:DISABLED
D8 HIGH
motor drivers disabled
OLED closed eyes
```

---

# Quick Calibration Command Summary

```text
CAL_GYRO      Calibrate MPU6050 gyro/accelerometer reference and save to EEPROM
CAL_BALANCE   Save current mechanical balance point to EEPROM
CAL_RESET     Reply OK only; no changes
ENABLE        Enable balancing permission and save state to EEPROM
DISABLE       Disable balancing and save state to EEPROM
PIDREQUEST    Request current saved PID values
PID,...       Update PID values and save to EEPROM
```

---

# Best Practice

Do not run `CAL_GYRO` every startup.

Only run it when:

```text
sensor position changed
robot hardware changed
angle reading is wrong
gyro drift is noticeable
```

Do run `CAL_BALANCE` whenever the physical balance point changes.

Examples:

```text
battery moved
OLED added
wires moved
new 3D printed part added
robot weight changed
```

For normal daily use:

```text
1. Power on
2. Hold upright
3. Wait for auto-balancing
4. Drive from app
```
