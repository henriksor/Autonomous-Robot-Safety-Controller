# Safety State Evaluator Tests

| Test ID | Requirement ID | Given | When | Then |
|---|---|---|---|---|
| SST-001 | REQ-SAFE-001 | No critical lock | temperature is 60.0 °C | state is Normal |
| SST-002 | REQ-SAFE-002 | No critical lock | temperature is 60.1 °C | state is WarningHigh |
| SST-003 | REQ-SAFE-003 | No critical lock | temperature is 80.0 °C | state is CriticalHigh |
| SST-004 | REQ-SAFE-004 | CriticalHigh is active | temperature is 75.0 °C | state remains CriticalHigh |
| SST-005 | REQ-SAFE-005 | CriticalHigh is active | temperature falls to 70.0 °C | state changes to WarningHigh |
| SST-006 | REQ-SAFE-006 | No critical lock | temperature is 0.0 °C | state is WarningLow |
| SST-007 | REQ-SAFE-007 | No critical lock | temperature is 20.0 °C | state is Normal |
| SST-008 | REQ-SAFE-008 | No critical lock | temperature is -0.1 °C | state is CriticalLow |
| SST-009 | REQ-SAFE-009 | CriticalLow is active | temperature is 9.9 °C | state remains CriticalLow |
| SST-010 | REQ-SAFE-010 | CriticalLow is active | temperature rises to 10.0 °C | state changes to WarningLow |
| SST-011 | REQ-SAFE-011 | No SensorFault is active | 3 invalid readings are received in a row | state is SensorFault |
| SST-012 | REQ-SAFE-012 | SensorFault is active | 9 valid readings are received in a row and latest valid temperature is 20.0 °C | state changes to Normal |