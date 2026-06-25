# Safety State Model

Status: Draft  
Version: 0.1

## 1. Scope / system boundary

This document describes the first version of the safety state model for the autonomous robot safety controller.

In scope for this version:

| Area | Description |
|---|---|
| Core temperature input | The safety controller receives validated core temperature readings. |
| Safety states | The controller classifies the robot into one safety state at a time. |
| Critical state locking | Critical states use hysteresis so the robot does not immediately restart at the activation threshold. |
| Sensor fault handling | Repeated invalid sensor readings can trigger `SensorFault`. |
| Controller output | The controller returns safety constraints, including safety state, speed limit and brake lock. |
| Simulator responsibility | The simulator applies the controller result and handles physical movement. |

Out of scope for this document:

| Area | Reason |
|---|---|
| Temperature simulation | How core temperature changes over time belongs in a separate simulation model. |
| Weather sensors | Planned for a later version. |
| Battery model | Planned for a later version. |
| Logging | Planned for a later version. |


## 2. Units

| Quantity | Unit |
|---|---|
| Position | meter, `m` |
| Speed | meter per second, `m/s` |
| Acceleration | meter per second squared, `m/s^2` |
| Time | seconds, `s` |
| Temperature | degrees Celsius, `°C` |
| Temperature trend | degrees Celsius per second, `°C/s` |

## 3. Safety states

| State | Description | Speed limit | Brake lock |
|---|---|---:|---|
| `Normal` | Temperature is within the safe operating range. | 35 m/s | false |
| `WarningLow` | Temperature is low, but not critical. | 40 m/s | false |
| `WarningHigh` | Temperature is high, but not critical. | 15 m/s | false |
| `CriticalLow` | Temperature is critically low. | 0 m/s | true |
| `CriticalHigh` | Temperature is critically high. | 0 m/s | true |
| `SensorFault` | Sensor readings cannot be trusted. | 0 m/s | true |

Only one safety state shall be active at a time.

## 4. State transitions and hysteresis

### 4.1 State activation

The following temperature transitions apply when the sensor reading is valid and no critical lock is already active.

| Condition | State |
|---|---|
| `temperature < 0 °C` | `CriticalLow` |
| `0 °C <= temperature < 20 °C` | `WarningLow` |
| `20 °C <= temperature <= 60 °C` | `Normal` |
| `60 °C < temperature < 80 °C` | `WarningHigh` |
| `temperature >= 80 °C` | `CriticalHigh` |

`SensorFault` is activated after three invalid sensor readings in a row.

### 4.2 Critical lock release

Critical states are locked states. After a critical state has been activated, the state remains active until its release condition is met.

| Active locked state | Remains active while | Lock may be released when |
|---|---|---|
| `CriticalLow` | `temperature < 10 °C` | `temperature >= 10 °C` and the sensor reading is valid |
| `CriticalHigh` | `temperature > 70 °C` | `temperature <= 70 °C` and the sensor reading is valid |
| `SensorFault` | Fewer than 9 valid readings in a row have been received | At least 9 valid readings in a row have been received |

After a critical lock is released, the new safety state shall be based on the latest valid temperature reading.

Examples:

| Previous state | Latest valid temperature | Result after release |
|---|---:|---|
| `CriticalHigh` | 70.0 °C | `WarningHigh` |
| `CriticalLow` | 10.0 °C | `WarningLow` |


## 5. Sensor fault rules

| Rule | Value |
|---|---|
| `SensorFault` activation threshold | 3 invalid readings in a row |
| Valid reading before `SensorFault` is active | Resets the invalid reading counter |
| `SensorFault` recovery threshold | 9 valid readings in a row |
| Invalid reading during `SensorFault` recovery | Resets the recovery counter |

Invalid reading number one and two shall not trigger `SensorFault`. While `SensorFault` is not active, invalid reading number one and two shall cause the controller to keep the previous safety result. The controller shall not increase the speed limit based on invalid data.

A sensor reading is invalid if:

- the value is missing,
- the value is not finite,
- the value is outside the accepted sensor input range.

The accepted sensor input range is -30 °C to 100 °C 

## 6. Controller output
The controller shall not control the robot's physical movement directly. The controller shall only evaluate the safety state and return safety constraints that the rest of the system must follow.

The controller output shall consist of:

| Field | Type / value | Meaning |
|---|---|---|
| `safetyState` | `Normal`, `WarningLow`, `WarningHigh`, `CriticalLow`, `CriticalHigh`, `SensorFault` | The system's assessment of the current safety state. |
| `speedLimit` | `m/s` | The maximum allowed speed based on the safety state. |
| `brakeLock` | `true` / `false` | Indicates whether the brake lock shall be active. |
| `decision` | `SetSpeedLimit`, `ApplyBrakeLock`, `MaintainBrakeLock`, `ReleaseBrakeLock`, `KeepPreviousResult` | High-level explanation of the safety decision. |

The controller returns a speed limit, not a direct target speed. This means that the controller decides the maximum safe speed, while the simulator decides how the actual speed changes over time.

`speedLimit` and `brakeLock` are authoritative. `decision` explains the reason for the controller result, but the simulator shall follow `speedLimit` and `brakeLock`.

| State | Speed limit | Brake lock | Typical decision |
|---|---:|---|---|
| `Normal` | 35 m/s | false | `SetSpeedLimit` |
| `WarningLow` | 40 m/s | false | `SetSpeedLimit` |
| `WarningHigh` | 15 m/s | false | `SetSpeedLimit` |
| `CriticalLow` | 0 m/s | true | `ApplyBrakeLock` / `MaintainBrakeLock` |
| `CriticalHigh` | 0 m/s | true | `ApplyBrakeLock` / `MaintainBrakeLock` |
| `SensorFault` | 0 m/s | true | `ApplyBrakeLock` / `MaintainBrakeLock` |

The decisions mean:

| Decision | Meaning |
|---|---|
| `SetSpeedLimit` | The controller sets or updates the maximum safe speed. |
| `ApplyBrakeLock` | The controller requires the brake lock to be activated. |
| `MaintainBrakeLock` | The controller requires the brake lock to remain active. |
| `ReleaseBrakeLock` | The controller allows the brake lock to be released. |
| `KeepPreviousResult` | The controller keeps the previous safety result because the current input is not reliable enough to update the result safely. |

ReleaseBrakeLock does not mean that the robot shall automatically start moving again. It only means that the safety lock is released, so movement may be allowed by other logic.

## 7. Simulator responsibility

The simulator is responsible for physical movement and shall follow the safety result returned by the controller.

The simulator shall handle:

- actual speed,
- acceleration,
- braking,
- position updates,
- gradual stopping,
- compliance with `speedLimit`,
- compliance with `brakeLock`.

The simulator shall not classify the safety state itself. It shall not decide whether the temperature is `Normal`, `WarningHigh`, `CriticalHigh`, or any other safety state. This is the controller's responsibility.

| Controller result | Simulator responsibility |
|---|---|
| `speedLimit = 35 m/s`, `brakeLock = false` | Allow movement up to 35 m/s. |
| `speedLimit = 40 m/s`, `brakeLock = false` | Allow movement up to 40 m/s. |
| `speedLimit = 15 m/s`, `brakeLock = false` | Brake if the actual speed is above 15 m/s. |
| `speedLimit = 0 m/s`, `brakeLock = true` | Disable propulsion and brake gradually toward 0 m/s. |
| `brakeLock = false` after a previous brake lock | Allow movement again, but do not accelerate automatically. |

Emergency stop shall not be understood as setting the speed directly to 0 in a single step. Emergency stop means:

- propulsion is disabled,
- braking is activated,
- the simulator reduces speed gradually,
- the stop is complete when actual speed is 0 m/s.

If the actual speed is higher than `speedLimit`, the simulator shall reduce the speed gradually until it is less than or equal to the limit.

If the actual speed is lower than `speedLimit`, the simulator shall not automatically accelerate just because the limit allows it. Acceleration must come from separate movement logic outside the safety controller.

## 8. Initial test cases

| Test ID | Initial state | Input | Expected state | Expected output |
|---|---|---|---|---|
| SST-001 | No critical lock | Valid temp 60.0 °C | `Normal` | `speedLimit = 35 m/s`, `brakeLock = false`, `decision = SetSpeedLimit` |
| SST-002 | No critical lock | Valid temp 60.1 °C | `WarningHigh` | `speedLimit = 15 m/s`, `brakeLock = false`, `decision = SetSpeedLimit` |
| SST-003 | No critical lock | Valid temp 80.0 °C | `CriticalHigh` | `speedLimit = 0 m/s`, `brakeLock = true`, `decision = ApplyBrakeLock` |
| SST-004 | `CriticalHigh` active | Valid temp 75.0 °C | `CriticalHigh` | `speedLimit = 0 m/s`, `brakeLock = true`, `decision = MaintainBrakeLock` |
| SST-005 | `CriticalHigh` active | Valid temp 70.0 °C | `WarningHigh` | `speedLimit = 15 m/s`, `brakeLock = false`, `decision = ReleaseBrakeLock` |
| SST-006 | No critical lock | Valid temp 0.0 °C | `WarningLow` | `speedLimit = 40 m/s`, `brakeLock = false`, `decision = SetSpeedLimit` |
| SST-007 | No critical lock | Valid temp 20.0 °C | `Normal` | `speedLimit = 35 m/s`, `brakeLock = false`, `decision = SetSpeedLimit` |
| SST-008 | No critical lock | Valid temp -0.1 °C | `CriticalLow` | `speedLimit = 0 m/s`, `brakeLock = true`, `decision = ApplyBrakeLock` |
| SST-009 | `CriticalLow` active | Valid temp 9.9 °C | `CriticalLow` | `speedLimit = 0 m/s`, `brakeLock = true`, `decision = MaintainBrakeLock` |
| SST-010 | `CriticalLow` active | Valid temp 10.0 °C | `WarningLow` | `speedLimit = 40 m/s`, `brakeLock = false`, `decision = ReleaseBrakeLock` |
| SST-011 | Not `SensorFault` | 1 invalid reading | Previous state is kept | Previous safety result is kept; the controller shall not increase the speed limit based on invalid data |
| SST-012 | Not `SensorFault` | 2 invalid readings in a row | Previous state is kept | Previous safety result is kept; the controller shall remain conservative |
| SST-013 | Not `SensorFault` | 3 invalid readings in a row | `SensorFault` | `speedLimit = 0 m/s`, `brakeLock = true`, `decision = ApplyBrakeLock` |
| SST-014 | `SensorFault` active | 8 valid readings in a row | `SensorFault` | `speedLimit = 0 m/s`, `brakeLock = true`, `decision = MaintainBrakeLock` |
| SST-015 | `SensorFault` active | 9 valid readings in a row and latest valid temp is 20.0 °C | `Normal` | `speedLimit = 35 m/s`, `brakeLock = false`, `decision = ReleaseBrakeLock` |
| SST-016 | `SensorFault` active recovery in progress | Invalid reading before 9 valid readings in a row | `SensorFault` | Recovery counter is reset, `brakeLock = true`, `decision = MaintainBrakeLock` |
