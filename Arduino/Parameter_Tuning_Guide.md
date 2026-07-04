# Self-Balancing Robot Firmware Parameter Tuning Guide

Firmware target: `Phase03X_OLED_DisabledClosedEyes_StableFinal_D5D6.ino`

This guide explains the important parameters in the final firmware, what each parameter does, what happens when it is increased or decreased, and how to tune the robot safely.

The firmware is designed for:

- Arduino Nano / ATmega328P
- DRV8825 stepper drivers
- MPU6050 IMU
- HC-05 Bluetooth module on hardware serial
- 0.97 inch SSD1306 I2C OLED eye display
- Battery voltage input on A6 through a resistor voltage divider

---

## Important Safety Notes

Self-balancing robots can fall, suddenly move, or overdrive the motors during tuning. Always tune in this order:

1. Put the robot on a stand so the wheels can rotate freely.
2. Test motor direction first.
3. Test MPU6050 angle direction.
4. Calibrate using `CAL_GYRO`.
5. Calibrate balance point using `CAL_BALANCE`.
6. Tune PID with small changes.
7. Test on the floor only after the robot is stable on a stand.

Never connect battery voltage directly to A6. Always use a voltage divider.

---

## EEPROM Behavior

Some values are saved to EEPROM and survive power-off:

- PID values: `pid_p_gain`, `pid_i_gain`, `pid_d_gain`
- Gyro offsets
- Accelerometer angle offset
- Manual balance offset
- ENABLE / DISABLE state

Important:

```cpp
float pid_p_gain = 3.5;
float pid_i_gain = 0.0;
float pid_d_gain = 4.5;
```

These are only default values. If valid EEPROM data exists, the EEPROM PID values will replace these defaults at startup.

If you change PID defaults in the code and the robot still uses the old PID values, it means EEPROM is loading the previously saved PID values. Send a new PID command from the app or change the EEPROM version for development testing.

---

# 1. Serial / Bluetooth Parameters

## `SERIAL_BAUD_RATE`

```cpp
#define SERIAL_BAUD_RATE 9600
```

This is the Bluetooth serial baud rate.

Current value:

```text
9600
```

Used for:

- HC-05 communication
- Android app commands
- Telemetry output

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase to `115200` | Works only if HC-05 is also configured to 115200. Otherwise app communication fails. |
| Keep at `9600` | Recommended for HC-05 default/app compatibility. |

Recommendation:

```cpp
#define SERIAL_BAUD_RATE 9600
```

Do not change this unless the HC-05 module baud rate is also changed.

---

## `RX_COMMAND_TIMEOUT_MS`

```cpp
const unsigned long RX_COMMAND_TIMEOUT_MS = 35;
```

This accepts commands even if the app does not send `\n`, `\r`, or `;`.

Current value:

```text
35 ms
```

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Firmware waits longer before accepting a command without newline. Can make button response slightly slower. |
| Decrease | Commands without newline are accepted faster, but incomplete commands may be processed too early. |

Recommended range:

```text
25 ms to 60 ms
```

Best value:

```cpp
const unsigned long RX_COMMAND_TIMEOUT_MS = 35;
```

---

# 2. Hardware Pin Parameters

## Stepper Motor Pins

```cpp
#define LEFT_STEP_PIN    5
#define LEFT_DIR_PIN     2
#define RIGHT_STEP_PIN   6
#define RIGHT_DIR_PIN    3
#define ENABLE_PIN       8
#define LED_PIN          13
#define BATTERY_PIN      A6
```

Meaning:

| Pin | Purpose |
|---|---|
| D5 | Left motor STEP |
| D2 | Left motor DIR |
| D6 | Right motor STEP |
| D3 | Right motor DIR |
| D8 | DRV8825 ENABLE, active LOW |
| D13 | Status LED |
| A6 | Battery voltage input |

Important:

- D8 HIGH = drivers disabled
- D8 LOW = drivers enabled
- A6 must receive scaled voltage through a voltage divider

Do not change motor pins unless you also update the direct `PORTD` masks.

---

## Direct PORTD Masks

```cpp
#define LEFT_STEP_MASK   0b00100000
#define LEFT_DIR_MASK    0b00000100
#define RIGHT_STEP_MASK  0b01000000
#define RIGHT_DIR_MASK   0b00001000
```

These are used inside the Timer2 interrupt for fast step pulse generation.

Do not change these unless you understand AVR port registers and have changed the physical pins.

---

# 3. Motor Direction Parameters

## `LEFT_FORWARD_DIR_HIGH`

```cpp
#define LEFT_FORWARD_DIR_HIGH 1
```

Controls the left motor direction logic.

If only the left motor rotates the wrong way during forward movement, flip this value:

```cpp
#define LEFT_FORWARD_DIR_HIGH 0
```

---

## `RIGHT_FORWARD_DIR_HIGH`

```cpp
#define RIGHT_FORWARD_DIR_HIGH 0
```

Controls the right motor direction logic.

If only the right motor rotates the wrong way during forward movement, flip this value:

```cpp
#define RIGHT_FORWARD_DIR_HIGH 1
```

---

## Motor Direction Tuning Rule

Use these only when one motor is mechanically reversed.

| Symptom | What to change |
|---|---|
| Left wheel only is reversed | Flip `LEFT_FORWARD_DIR_HIGH` |
| Right wheel only is reversed | Flip `RIGHT_FORWARD_DIR_HIGH` |
| Both wheels correct the wrong way during balancing | Flip `PID_OUTPUT_SIGN` |
| Forward/backward app command reversed | Flip `FORWARD_COMMAND_SIGN` or `DRIVE_COMMAND_SIGN` depending on behavior |
| Left/right app turn reversed | Flip `TURN_COMMAND_SIGN` |

---

# 4. MPU6050 Mounting Parameters

## `MPU_ADDRESS`

```cpp
#define MPU_ADDRESS 0x68
```

This is the I2C address of the MPU6050.

Most MPU6050 modules use:

```text
0x68
```

Some modules can use:

```text
0x69
```

Change only if your MPU6050 address is different.

---

## `MOUNT_MODE`

```cpp
#define MOUNT_MODE 2
```

This selects which accelerometer and gyro axes are used for balance angle.

Current tested mode:

```text
MOUNT_MODE 2 = AccY + GyroX
```

Available modes in this firmware:

| Mode | Accelerometer axis | Gyro axis | Use when |
|---|---|---|---|
| `1` | AccX | GyroY | Axis finder shows AccX/GyroY follows forward/back tilt |
| `2` | AccY | GyroX | Axis finder shows AccY/GyroX follows forward/back tilt |

Current robot:

```cpp
#define MOUNT_MODE 2
```

---

## How to Change MPU6050 Mounting

If someone mounts the MPU6050 differently:

1. Upload the Axis Finder sketch.
2. Slowly tilt the robot forward and backward.
3. Check which candidate angle changes smoothly from negative to positive.
4. Set `MOUNT_MODE` according to the result.

Use:

```cpp
#define MOUNT_MODE 1
```

if AccX + GyroY is correct.

Use:

```cpp
#define MOUNT_MODE 2
```

if AccY + GyroX is correct.

After changing `MOUNT_MODE`:

1. Upload the firmware.
2. Hold robot upright and still.
3. Send `CAL_GYRO`.
4. Hold the real mechanical balance point.
5. Send `CAL_BALANCE`.
6. Send `ENABLE`.

---

## `ANGLE_SIGN`

```cpp
#define ANGLE_SIGN 1
```

This flips the measured balance angle direction.

Increase/decrease is not relevant here. It is a sign switch.

| Value | Effect |
|---|---|
| `1` | Normal angle direction |
| `-1` | Reversed angle direction |

Change this if:

- The app angle increases when it should decrease
- The robot angle direction is reversed after changing MPU mount orientation

Example:

```cpp
#define ANGLE_SIGN -1
```

---

## `PID_OUTPUT_SIGN`

```cpp
#define PID_OUTPUT_SIGN 1
```

This flips the balance correction direction sent to the motors.

| Value | Effect |
|---|---|
| `1` | Normal PID motor correction |
| `-1` | Reverse PID motor correction |

Change this if:

- The robot leans forward and the wheels move the wrong way
- The robot falls faster when balancing activates

Important:

Only change `PID_OUTPUT_SIGN` after motor wiring direction and MPU angle direction are correct.

---

# 5. Battery Telemetry Parameters

Battery telemetry is sent as:

```text
B:<battery_voltage>
```

Example:

```text
B:12.18
```

## Battery Wiring

Use this voltage divider:

```text
Battery +  -> R1 -> A6 -> R2 -> GND
Battery -  -> Arduino GND
```

Default firmware values:

```cpp
const float BATTERY_ADC_REF_VOLTAGE = 5.00;
const float BATTERY_R1_OHMS = 2200.0;
const float BATTERY_R2_OHMS = 1000.0;
const float BATTERY_CALIBRATION = 1.000;
const float BATTERY_FILTER_ALPHA = 0.15;
```

---

## `BATTERY_ADC_REF_VOLTAGE`

```cpp
const float BATTERY_ADC_REF_VOLTAGE = 5.00;
```

This is the Arduino ADC reference voltage.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | App battery reading becomes higher |
| Decrease | App battery reading becomes lower |

How to tune:

Measure Arduino 5V pin with a multimeter.

Example:

If Arduino 5V is actually 4.82V:

```cpp
const float BATTERY_ADC_REF_VOLTAGE = 4.82;
```

---

## `BATTERY_R1_OHMS`

```cpp
const float BATTERY_R1_OHMS = 2200.0;
```

This is the top resistor from battery positive to A6.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Allows higher battery voltage before A6 reaches 5V, but ADC reading may be noisier |
| Decrease | Better ADC drive strength, but lower maximum measurable battery voltage |

Default:

```text
2.2k
```

---

## `BATTERY_R2_OHMS`

```cpp
const float BATTERY_R2_OHMS = 1000.0;
```

This is the bottom resistor from A6 to GND.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | A6 voltage becomes higher for the same battery voltage |
| Decrease | A6 voltage becomes lower and safer for higher battery voltage |

Default:

```text
1k
```

---

## Voltage Divider Maximum

With 100k / 33k:

```text
A6 voltage = battery voltage × 1k / (2.2k + 1k)
```

Approximate safe maximum battery voltage with 5V ADC reference:

```text
5.0 × (2.2k + 1k) / 1k = about 16.00V
```

This is suitable for common 2S and 3S Li-ion/LiPo packs.

Never connect battery directly to A6.

---

## `BATTERY_CALIBRATION`

```cpp
const float BATTERY_CALIBRATION = 1.000;
```

This is a fine adjustment multiplier.

Use it when the app voltage does not exactly match the multimeter voltage.

Formula:

```text
BATTERY_CALIBRATION = multimeter voltage / app voltage
```

Example:

Multimeter shows:

```text
12.30V
```

App shows:

```text
12.00V
```

Then:

```text
12.30 / 12.00 = 1.025
```

Set:

```cpp
const float BATTERY_CALIBRATION = 1.025;
```

---

## `BATTERY_FILTER_ALPHA`

```cpp
const float BATTERY_FILTER_ALPHA = 0.15;
```

This smooths the battery voltage reading.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Battery reading reacts faster but becomes noisier |
| Decrease | Battery reading is smoother but slower |

Recommended range:

```text
0.05 to 0.30
```

Default:

```cpp
const float BATTERY_FILTER_ALPHA = 0.15;
```

---

# 6. Main Loop Timing

## `LOOP_TIME_US`

```cpp
const unsigned long LOOP_TIME_US = 4000;
```

This is the main control loop period.

Current value:

```text
4000 us = 4 ms = 250 Hz
```

Do not change this unless you retune:

- Gyro integration constant
- PID values
- Output ramp
- Movement ramp values

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Slower control loop, less CPU load, weaker balance response |
| Decrease | Faster loop, more CPU load, PID/gyro math may need retuning |

Recommendation:

```cpp
const unsigned long LOOP_TIME_US = 4000;
```

---

# 7. PID Parameters

PID controls the self-balancing correction.

```cpp
float pid_p_gain = 3.5;
float pid_i_gain = 0.0;
float pid_d_gain = 4.5;
```

These defaults are overwritten by EEPROM if PID values were saved from the app.

---

## `pid_p_gain`

```cpp
float pid_p_gain = 3.5;
```

P gain reacts to angle error.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Stronger correction, faster response |
| Too high | Oscillation, vibration, aggressive movement, falling |
| Decrease | Softer, smoother response |
| Too low | Robot feels weak and falls slowly |

Recommended tuning range for this robot:

```text
2.5 to 4.5
```

Suggested app slider:

```text
0.0 to 8.0, step 0.1
```

Tuning method:

1. Keep I at `0.000`.
2. Keep D around `4.0` to `4.5`.
3. Increase P until the robot stands strongly.
4. If it oscillates, reduce P.

---

## `pid_i_gain`

```cpp
float pid_i_gain = 0.0;
```

I gain corrects long-term accumulated error.

Important:

This firmware updates PID every 4 ms. Therefore even small I values can be strong.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Helps remove long-term offset or drift |
| Too high | Runaway, slow lean buildup, falling |
| Decrease | Safer and more stable |
| `0.000` | Recommended starting value |

Recommended values:

```text
0.000 = safest
0.001 = very small correction
0.002 = small correction
0.005 = already strong
0.010+ = risky
0.100 = too high for this firmware
```

Suggested app slider:

```text
0.000 to 0.020, step 0.001
```

Tuning method:

1. Tune P and D first.
2. Leave I at `0.000`.
3. If the robot balances but slowly leans/drifts after several seconds, try `0.001`.
4. If instability appears, return to `0.000`.

Because this firmware already has stop-drift trim, I gain is usually not required.

---

## `pid_d_gain`

```cpp
float pid_d_gain = 4.5;
```

D gain reacts to change in angle error. It provides damping and braking.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | More damping, better braking, less overshoot |
| Too high | Noise, vibration, harsh motor response |
| Decrease | Smoother motors |
| Too low | Robot overshoots and feels loose |

Recommended tuning range:

```text
3.0 to 6.0
```

Suggested app slider:

```text
0.0 to 10.0, step 0.1
```

Tuning method:

1. Start around `4.0` to `4.5`.
2. If robot oscillates slowly, increase D slightly.
3. If motor noise/vibration becomes high, reduce D.

---

## PID Tuning Order

Recommended order:

1. Set I = `0.000`.
2. Tune P until the robot can stand.
3. Tune D to reduce oscillation.
4. Add tiny I only if needed.
5. Save final PID through the app.

Example stable starting point:

```text
KP = 3.5
KI = 0.000
KD = 4.5
```

---

# 8. Balance Offset Parameter

## `manual_balance_offset`

```cpp
float manual_balance_offset = 0.0;
```

This is the saved mechanical balance point.

Do not normally edit this manually.

It is set by:

```text
CAL_BALANCE
```

When you send `CAL_BALANCE`, the firmware saves the current robot angle as the real balance point.

Use `CAL_BALANCE` when:

- Robot balances but always wants to lean forward/backward
- Mechanical center changed
- Battery position changed
- Robot body weight distribution changed

---

# 9. PID Output Safety Parameters

## `MAX_PID_OUTPUT_SAFE`

```cpp
const float MAX_PID_OUTPUT_SAFE = 45.0;
```

This limits only the balance PID output before movement/turn mixing.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Stronger balance correction |
| Too high | Stepper stall, sudden jerks, noise |
| Decrease | Softer and safer |
| Too low | Robot cannot catch itself |

Recommended range:

```text
35 to 80
```

Default:

```cpp
const float MAX_PID_OUTPUT_SAFE = 45.0;
```

---

## `OUTPUT_RAMP_STEP`

```cpp
const float OUTPUT_RAMP_STEP = 0.8;
```

Limits how quickly PID output can change every 4 ms.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Faster correction, more aggressive |
| Too high | Jerky motor response |
| Decrease | Smoother response |
| Too low | Robot reacts too late and falls |

Recommended range:

```text
0.5 to 2.0
```

Default:

```cpp
const float OUTPUT_RAMP_STEP = 0.8;
```

---

## `MOTOR_OUTPUT_DEADBAND`

```cpp
const int MOTOR_OUTPUT_DEADBAND = 5;
```

Small motor outputs between `-5` and `+5` are treated as zero.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Less jitter and small direction flipping |
| Too high | Small corrections disappear and balance becomes weak |
| Decrease | More sensitive small corrections |
| Too low | Motor jitter, unstable direction changes |

Recommended range:

```text
3 to 8
```

Default:

```cpp
const int MOTOR_OUTPUT_DEADBAND = 5;
```

---

# 10. Angle Filter Parameters

## `ACC_FILTER_ALPHA`

```cpp
const float ACC_FILTER_ALPHA = 0.04;
```

This filters the accelerometer angle before it is mixed with gyro angle.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Accelerometer follows faster |
| Too high | Vibration noise enters angle calculation |
| Decrease | Smoother angle |
| Too low | Accelerometer correction becomes slow |

Recommended range:

```text
0.02 to 0.08
```

Default:

```cpp
const float ACC_FILTER_ALPHA = 0.04;
```

---

## `COMPLEMENTARY_GYRO_WEIGHT`

```cpp
const float COMPLEMENTARY_GYRO_WEIGHT = 0.99985;
```

## `COMPLEMENTARY_ACC_WEIGHT`

```cpp
const float COMPLEMENTARY_ACC_WEIGHT = 0.00015;
```

These combine gyro angle and accelerometer angle.

The two values should normally add up to:

```text
1.0
```

Current:

```text
0.99985 + 0.00015 = 1.00000
```

Increase/decrease effects:

| Change | Effect |
|---|---|
| More gyro weight | Smoother angle, less vibration, but slower drift correction |
| More accelerometer weight | Faster drift correction, but more vibration/noise |
| Too much accelerometer | Robot may shake or oscillate |
| Too much gyro | Angle may drift slowly |

Recommended stable values:

```cpp
const float COMPLEMENTARY_GYRO_WEIGHT = 0.99985;
const float COMPLEMENTARY_ACC_WEIGHT  = 0.00015;
```

If startup angle settles too slowly, try:

```cpp
const float COMPLEMENTARY_GYRO_WEIGHT = 0.9997;
const float COMPLEMENTARY_ACC_WEIGHT  = 0.0003;
```

If vibration increases, go back to the default.

---

# 11. Startup / Fall Safety Parameters

## `TIP_OVER_ANGLE_LIMIT`

```cpp
const float TIP_OVER_ANGLE_LIMIT = 22.0;
```

If the robot angle exceeds this value, balancing is deactivated and D8 goes HIGH.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Robot keeps trying at larger tilt angles |
| Too high | Motors may keep driving after robot has fallen |
| Decrease | Safer, motors disable sooner |
| Too low | Balancing may disable during normal movement |

Recommended range:

```text
18 to 30 degrees
```

Default:

```cpp
const float TIP_OVER_ANGLE_LIMIT = 22.0;
```

---

## `AUTO_BALANCE_ARM_ANGLE_LIMIT`

```cpp
const float AUTO_BALANCE_ARM_ANGLE_LIMIT = 1.0;
```

Balancing starts only when ENABLE is active and the robot angle is inside this window.

Current condition:

```text
-1° < angle < +1°
```

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Easier to start balancing |
| Too high | Robot may start aggressively while not upright |
| Decrease | Safer start |
| Too low | Harder to arm balancing |

Recommended range:

```text
0.5 to 2.0 degrees
```

Default:

```cpp
const float AUTO_BALANCE_ARM_ANGLE_LIMIT = 1.0;
```

---

## Legacy Startup Values

These values exist in the firmware but are not the main active auto-arm logic in Phase03X:

```cpp
const float ARM_ANGLE_LIMIT = 2.5;
const float ARM_GYRO_RAW_LIMIT = 700.0;
const int ARM_STABLE_COUNT_REQUIRED = 80;
```

Current Phase03X mainly uses:

```cpp
AUTO_BALANCE_ARM_ANGLE_LIMIT
```

For normal tuning, adjust `AUTO_BALANCE_ARM_ANGLE_LIMIT`, not the legacy values.

---

# 12. Stop-Drift Trim Parameters

When the robot is balancing with no movement command, the firmware watches PID output. If the robot still needs motor power in one direction, it slowly adjusts the internal balance point to reduce rolling drift.

## `SELF_BALANCE_STEP`

```cpp
const float SELF_BALANCE_STEP = 0.25;
```

This is how fast the internal balance point is adjusted while stopped.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Drift correction reacts faster |
| Too high | Over-trim, hunting, rocking |
| Decrease | Smoother |
| Too low | Robot takes too long to stop drift |

Recommended range:

```text
0.05 to 0.35
```

Default:

```cpp
const float SELF_BALANCE_STEP = 0.25;
```

---

## `STOP_DRIFT_OUTPUT_DEADBAND`

```cpp
const float STOP_DRIFT_OUTPUT_DEADBAND = 5.0;
```

The stop-drift trim activates only when PID output is outside this deadband.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Trim activates less often |
| Too high | Drift may not correct |
| Decrease | Trim activates sooner |
| Too low | Trim may hunt or oscillate |

Recommended range:

```text
4 to 12
```

Default:

```cpp
const float STOP_DRIFT_OUTPUT_DEADBAND = 5.0;
```

---

## `SELF_BALANCE_MAX_TRIM`

```cpp
const float SELF_BALANCE_MAX_TRIM = 8.0;
```

Maximum internal balance trim range around `manual_balance_offset`.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Can correct stronger weight imbalance |
| Too high | Robot can develop large lean bias |
| Decrease | Safer |
| Too low | Drift correction may not be enough |

Recommended range:

```text
2.0 to 8.0
```

Default:

```cpp
const float SELF_BALANCE_MAX_TRIM = 8.0;
```

---

## `SELF_BALANCE_CORRECTION_SIGN`

```cpp
#define SELF_BALANCE_CORRECTION_SIGN 1
```

This controls trim correction direction.

| Value | Effect |
|---|---|
| `1` | Current correction direction |
| `-1` | Reversed correction direction |

If stop-drift correction makes drift worse, flip this value.

Example:

```cpp
#define SELF_BALANCE_CORRECTION_SIGN -1
```

---

# 13. App Movement Direction Parameters

## `FORWARD_COMMAND_SIGN`

```cpp
#define FORWARD_COMMAND_SIGN 1
```

This controls the lean direction for forward/backward movement.

If FORWARD command makes the robot lean backward, change to:

```cpp
#define FORWARD_COMMAND_SIGN -1
```

---

## `DRIVE_COMMAND_SIGN`

```cpp
#define DRIVE_COMMAND_SIGN 1
```

This controls direct wheel drive direction for forward/backward commands.

If FORWARD command drives wheels backward, change to:

```cpp
#define DRIVE_COMMAND_SIGN -1
```

---

## `TURN_COMMAND_SIGN`

```cpp
#define TURN_COMMAND_SIGN 1
```

This controls left/right turn direction.

If LEFT and RIGHT are reversed, change to:

```cpp
#define TURN_COMMAND_SIGN -1
```

---

# 14. Left / Right Rotation Parameters

## `TURN_SPEED`

```cpp
const float TURN_SPEED = 10.0;
```

This controls rotation strength.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Faster rotation |
| Too high | Robot may jerk or lose balance while turning |
| Decrease | Slower, smoother rotation |
| Too low | Turn may be weak |

Recommended range:

```text
8 to 25
```

Default:

```cpp
const float TURN_SPEED = 10.0;
```

---

## `TURN_RAMP_STEP`

```cpp
const float TURN_RAMP_STEP = 0.25;
```

This controls how quickly turn output ramps up/down every 4 ms.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Turn starts/stops faster |
| Too high | Sudden jerk |
| Decrease | Smoother |
| Too low | Turn feels delayed |

Recommended range:

```text
0.15 to 1.0
```

Default:

```cpp
const float TURN_RAMP_STEP = 0.25;
```

---

# 15. Forward / Backward Movement Parameters

Forward/backward movement uses two effects:

1. Small lean target through `MOVE_LEAN_ANGLE`
2. Direct drive assist through `MOVE_DRIVE_SPEED`

## `MOVE_DRIVE_SPEED`

```cpp
const float MOVE_DRIVE_SPEED = 8.0;
```

This is the cruise direct-drive output for FORWARD/BACKWARD.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Faster forward/back movement |
| Too high | Balance may become aggressive or unstable |
| Decrease | Slower movement |
| Too low | Robot may lean but not move |

Recommended test range:

```text
8 to 35
```

Default:

```cpp
const float MOVE_DRIVE_SPEED = 8.0;
```

Tuning tip:

If the robot leans but does not move, increase this value gradually.

---

## `MOVE_DRIVE_START_BOOST`

```cpp
const float MOVE_DRIVE_START_BOOST = 5.0;
```

This is an immediate start kick when FORWARD/BACKWARD begins.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Movement starts faster |
| Too high | Sudden kick/jerk |
| Decrease | Smoother start |
| Too low | Robot may need time to start moving |

Recommended test range:

```text
5 to 30
```

Default:

```cpp
const float MOVE_DRIVE_START_BOOST = 5.0;
```

Tuning tip:

If forward/backward movement starts late, increase this first.

---

## `MOVE_DRIVE_RAMP_STEP`

```cpp
const float MOVE_DRIVE_RAMP_STEP = 0.25;
```

This controls how quickly direct drive reaches `MOVE_DRIVE_SPEED`.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Faster acceleration |
| Too high | Jerky movement |
| Decrease | Smoother acceleration |
| Too low | Movement feels delayed |

Recommended range:

```text
0.20 to 1.50
```

Default:

```cpp
const float MOVE_DRIVE_RAMP_STEP = 0.25;
```

---

## `MOVE_LEAN_ANGLE`

```cpp
const float MOVE_LEAN_ANGLE = 0.8;
```

This sets the fixed lean angle used while moving forward/backward.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Stronger travel force |
| Too high | Robot may fall or run away |
| Decrease | Safer, smoother |
| Too low | Robot may lean only slightly and not travel |

Recommended range:

```text
0.3 to 2.0 degrees
```

Default:

```cpp
const float MOVE_LEAN_ANGLE = 0.8;
```

---

## `STOP_LEAN_RETURN_STEP`

```cpp
const float STOP_LEAN_RETURN_STEP = 0.08;
```

This controls how fast the movement lean target returns to zero after STOP.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Faster stop response |
| Too high | Abrupt stopping |
| Decrease | Smoother stop |
| Too low | Robot may continue leaning/coasting longer |

Recommended range:

```text
0.05 to 0.20
```

Default:

```cpp
const float STOP_LEAN_RETURN_STEP = 0.08;
```

---

## `MAX_MOTOR_OUTPUT_SAFE`

```cpp
const float MAX_MOTOR_OUTPUT_SAFE = 95.0;
```

This limits final motor command after combining:

```text
balance PID + forward/back drive + turn output
```

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Stronger total motor output |
| Too high | Stepper noise, stall, harsh correction |
| Decrease | Safer and smoother |
| Too low | Robot may not balance or move properly |

Recommended range:

```text
70 to 130
```

Default:

```cpp
const float MAX_MOTOR_OUTPUT_SAFE = 95.0;
```

---

# 16. Movement Timeout Parameter

## `MOTION_COMMAND_TIMEOUT_MS`

```cpp
const unsigned long MOTION_COMMAND_TIMEOUT_MS = 2000;
```

If no movement command arrives for this time, the robot clears:

- FORWARD
- BACKWARD
- LEFT
- RIGHT

Balancing remains active.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Robot keeps moving longer after app stops sending commands |
| Too high | Less safe if Bluetooth/app freezes |
| Decrease | Safer communication loss behavior |
| Too low | Movement may stop while holding app button if app repeat rate is slow |

Recommended value:

```cpp
const unsigned long MOTION_COMMAND_TIMEOUT_MS = 2000;
```

Recommended app behavior:

```text
Send movement command every 100 ms to 300 ms while button is held.
Send STOP when button is released.
```

---

# 17. Telemetry Parameter

## `TELEMETRY_INTERVAL_MS`

```cpp
const unsigned long TELEMETRY_INTERVAL_MS = 200;
```

Robot sends telemetry every 200 ms.

Telemetry format:

```text
A:<angle>,B:<battery>,LM:<left_motor>,RM:<right_motor>,ST:<state>
```

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Less Bluetooth traffic |
| Too high | App updates slowly |
| Decrease | Faster app updates |
| Too low | More serial traffic, possible timing load |

Recommended range:

```text
100 to 500 ms
```

Default:

```cpp
const unsigned long TELEMETRY_INTERVAL_MS = 200;
```

---

# 18. OLED Eye Display Parameters

## `OLED_ADDR`

```cpp
#define OLED_ADDR 0x3C
```

Most SSD1306 OLED displays use:

```text
0x3C
```

If the OLED is blank and wiring is correct, try:

```cpp
#define OLED_ADDR 0x3D
```

---

## `EYE_UPDATE_MS`

```cpp
#define EYE_UPDATE_MS 180
```

OLED refresh interval.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Less I2C/CPU load, slower animation |
| Decrease | Smoother animation |
| Too low | OLED updates may disturb balancing timing |

Recommended range:

```text
180 to 300 ms
```

Default:

```cpp
#define EYE_UPDATE_MS 180
```

If balancing becomes less stable after OLED integration, increase this:

```cpp
#define EYE_UPDATE_MS 250
```

---

## `WARNING_ANGLE`

```cpp
#define WARNING_ANGLE 10
```

If angle is above this value, OLED shows warning eyes.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Warning appears later |
| Decrease | Warning appears sooner |

This affects OLED only. It does not change balancing safety.

Recommended range:

```text
8 to 15 degrees
```

Default:

```cpp
#define WARNING_ANGLE 10
```

---

## `FALL_ANGLE`

```cpp
#define FALL_ANGLE 28
```

If angle is above this value, OLED shows fallen/X eyes.

Increase/decrease effects:

| Change | Effect |
|---|---|
| Increase | Fallen eyes appear later |
| Decrease | Fallen eyes appear sooner |

This affects OLED only. It does not disable motors.

Motor disable safety is controlled by:

```cpp
TIP_OVER_ANGLE_LIMIT
```

Recommended:

```text
FALL_ANGLE should be close to or slightly above TIP_OVER_ANGLE_LIMIT.
```

Default:

```cpp
#define FALL_ANGLE 28
```

---

# 19. OLED Eye Modes

Firmware eye modes:

```cpp
EYE_CALIB
EYE_DISABLED
EYE_NORMAL
EYE_BLINK
EYE_WARN
EYE_FALL
```

Meaning:

| Mode | When it appears |
|---|---|
| `EYE_CALIB` | EEPROM calibration is not ready |
| `EYE_DISABLED` | Robot balancing is disabled |
| `EYE_NORMAL` | Robot is enabled and stable |
| `EYE_BLINK` | Periodic blink in normal state |
| `EYE_WARN` | Angle is above `WARNING_ANGLE` |
| `EYE_FALL` | Fallen state or angle above `FALL_ANGLE` |

Priority order:

```text
Calibration not ready
Disabled
Fallen
Warning
Blink
Normal
```

---

# 20. EEPROM Constants

## `SETTINGS_EEPROM_MAGIC`

```cpp
#define SETTINGS_EEPROM_MAGIC 0x52423331UL
```

This identifies valid robot EEPROM data.

Do not change this unless you want the firmware to ignore old saved settings.

---

## `SETTINGS_EEPROM_VERSION`

```cpp
#define SETTINGS_EEPROM_VERSION 2
```

This identifies the EEPROM data format version.

Change this only if the `RobotEEPROMData` struct layout changes.

Development tip:

If you want to force the firmware to ignore old EEPROM settings during testing, increase this value:

```cpp
#define SETTINGS_EEPROM_VERSION 3
```

Then recalibrate and resend PID values.

---

# 21. Recommended Tuning Workflow

## Step 1: Confirm Motor Direction

Lift the robot on a stand.

If balancing correction goes the wrong way:

```cpp
#define PID_OUTPUT_SIGN -1
```

If only one motor is reversed:

```cpp
#define LEFT_FORWARD_DIR_HIGH 0
```

or

```cpp
#define RIGHT_FORWARD_DIR_HIGH 1
```

depending on which motor is wrong.

---

## Step 2: Confirm MPU Direction

If angle direction is reversed:

```cpp
#define ANGLE_SIGN -1
```

If MPU mounting changed:

```cpp
#define MOUNT_MODE 1
```

or:

```cpp
#define MOUNT_MODE 2
```

Then always run:

```text
CAL_GYRO
CAL_BALANCE
```

---

## Step 3: Calibrate

Send:

```text
CAL_GYRO
```

while robot is upright and completely still.

Then hold the real mechanical balance point and send:

```text
CAL_BALANCE
```

Then send:

```text
ENABLE
```

---

## Step 4: Tune PID

Start with:

```text
KP = 3.5
KI = 0.000
KD = 4.5
```

If robot is too weak:

```text
Increase KP slightly
```

If robot oscillates:

```text
Reduce KP or increase KD slightly
```

If motor vibrates/noisy:

```text
Reduce KD
```

If robot slowly drifts but is otherwise stable:

```text
Try KI = 0.001
```

If KI causes falling:

```text
Return KI to 0.000
```

---

## Step 5: Tune Stop Drift

If robot slowly rolls while balancing with no command:

1. Check `SELF_BALANCE_CORRECTION_SIGN`.
2. If correction makes drift worse, flip it.
3. Increase `SELF_BALANCE_STEP` slowly if correction is too slow.
4. Reduce `SELF_BALANCE_STEP` if it hunts/rocks.

---

## Step 6: Tune Forward / Backward Movement

If robot leans but does not move:

1. Increase `MOVE_DRIVE_START_BOOST`.
2. Increase `MOVE_DRIVE_SPEED`.
3. Increase `MOVE_LEAN_ANGLE` slightly.

If robot jerks when moving:

1. Reduce `MOVE_DRIVE_START_BOOST`.
2. Reduce `MOVE_DRIVE_RAMP_STEP`.
3. Reduce `MOVE_LEAN_ANGLE`.

If robot keeps moving too long after STOP:

1. Increase `STOP_LEAN_RETURN_STEP`.
2. Reduce `MOVE_DRIVE_SPEED`.

---

## Step 7: Tune Left / Right Rotation

If turning is too slow:

```cpp
const float TURN_SPEED = 15.0;
```

If turning is too jerky:

```cpp
const float TURN_RAMP_STEP = 0.15;
```

If left/right is reversed:

```cpp
#define TURN_COMMAND_SIGN -1
```

---

## Step 8: Tune Battery Reading

Measure battery with a multimeter.

If app voltage is wrong:

```text
BATTERY_CALIBRATION = multimeter voltage / app voltage
```

Example:

```text
12.30 / 12.00 = 1.025
```

Set:

```cpp
const float BATTERY_CALIBRATION = 1.025;
```

---

## Step 9: Tune OLED Load

If balancing becomes worse after OLED integration:

```cpp
#define EYE_UPDATE_MS 250
```

If OLED feels too slow:

```cpp
#define EYE_UPDATE_MS 180
```

Do not set OLED update too fast.

---

# 22. App Command Summary

## App to Robot

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

## Robot to App

```text
READY
OK
ERR
RX_OVERFLOW
PID,KP=<value>,KI=<value>,KD=<value>
A:<angle>,B:<battery>,LM:<left>,RM:<right>,ST:<state>
```

---

# 23. Best Known Starting Values

These are the current final firmware values:

```cpp
float pid_p_gain = 3.5;
float pid_i_gain = 0.0;
float pid_d_gain = 4.5;

const float MAX_PID_OUTPUT_SAFE = 45.0;
const float OUTPUT_RAMP_STEP = 0.8;

const float ACC_FILTER_ALPHA = 0.04;
const float COMPLEMENTARY_GYRO_WEIGHT = 0.99985;
const float COMPLEMENTARY_ACC_WEIGHT  = 0.00015;

const float TIP_OVER_ANGLE_LIMIT = 22.0;
const float AUTO_BALANCE_ARM_ANGLE_LIMIT = 1.0;

const float SELF_BALANCE_STEP = 0.25;
const float SELF_BALANCE_MAX_TRIM = 8.0;

const float TURN_SPEED = 10.0;
const float TURN_RAMP_STEP = 0.25;

const float MOVE_DRIVE_SPEED = 8.0;
const float MOVE_DRIVE_START_BOOST = 5.0;
const float MOVE_DRIVE_RAMP_STEP = 0.25;
const float MOVE_LEAN_ANGLE = 0.8;
const float MAX_MOTOR_OUTPUT_SAFE = 95.0;

const unsigned long MOTION_COMMAND_TIMEOUT_MS = 2000;
const unsigned long TELEMETRY_INTERVAL_MS = 200;

#define EYE_UPDATE_MS 180
#define WARNING_ANGLE 10
#define FALL_ANGLE 28
```

---

# 24. Quick Troubleshooting Table

| Problem | Likely parameter |
|---|---|
| Robot falls immediately when enabled | `PID_OUTPUT_SIGN`, `ANGLE_SIGN`, `MOUNT_MODE` |
| One wheel runs backward | `LEFT_FORWARD_DIR_HIGH` or `RIGHT_FORWARD_DIR_HIGH` |
| Forward/backward reversed | `FORWARD_COMMAND_SIGN` or `DRIVE_COMMAND_SIGN` |
| Left/right reversed | `TURN_COMMAND_SIGN` |
| Robot oscillates | Lower `pid_p_gain`, lower `pid_d_gain`, or reduce `MAX_PID_OUTPUT_SAFE` |
| Robot is weak | Increase `pid_p_gain` or `MAX_PID_OUTPUT_SAFE` |
| Motor noise/vibration | Reduce `pid_d_gain`, reduce `OUTPUT_RAMP_STEP`, increase `MOTOR_OUTPUT_DEADBAND` |
| Robot drifts while stopped | Tune `SELF_BALANCE_STEP`, `SELF_BALANCE_CORRECTION_SIGN`, `SELF_BALANCE_MAX_TRIM` |
| Robot leans but does not move forward/back | Increase `MOVE_DRIVE_START_BOOST`, `MOVE_DRIVE_SPEED`, or `MOVE_LEAN_ANGLE` |
| Robot starts movement with jerk | Reduce `MOVE_DRIVE_START_BOOST` or `MOVE_DRIVE_RAMP_STEP` |
| Robot disables too easily | Increase `TIP_OVER_ANGLE_LIMIT` slightly |
| Robot starts balancing too aggressively | Reduce `AUTO_BALANCE_ARM_ANGLE_LIMIT` |
| App telemetry is slow | Reduce `TELEMETRY_INTERVAL_MS` |
| Bluetooth movement stops while button held | App must repeat movement command faster than 2 seconds |
| Battery voltage wrong | Tune `BATTERY_ADC_REF_VOLTAGE`, resistor values, or `BATTERY_CALIBRATION` |
| OLED blank | Try `OLED_ADDR 0x3D`, check SDA/SCL wiring |
| OLED affects balancing | Increase `EYE_UPDATE_MS` |

---

## Final Notes

Tune one parameter at a time. After every change, test on a stand before testing on the floor.

For most builds, the safest tuning order is:

```text
MOUNT_MODE / ANGLE_SIGN
PID_OUTPUT_SIGN
CAL_GYRO
CAL_BALANCE
PID
stop-drift trim
forward/back movement
turn movement
battery calibration
OLED timing
```

The firmware is designed so app PID tuning can be done without recompiling, while hardware-specific values such as motor direction, MPU mount, battery divider, and OLED address remain code parameters.
