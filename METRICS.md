# Metrics Reference

Every metric below is sourced from a real `snmpwalk` against the actual device (HP R/T3000 HV INTL UPS, part number J2R04A, "G4", Eaton-manufactured, HP UPS Network Module AF465A, firmware `1.13.001`, UPS microcontroller firmware `01.02.0004`), cross-checked against two independent MIB trees where possible. Nothing here is guessed. Where a value could not be verified against the real device, it is not implemented (see "Deliberately excluded" at the end).

## Why two MIB trees

This UPS answers on both:
- **Standard UPS-MIB** (RFC 1628), OID root `1.3.6.1.2.1.33`
- **HP/Compaq CPQPOWER-MIB** (proprietary, vendor-recommended), OID root `1.3.6.1.4.1.232.165`

CPQPOWER-MIB is used as the primary source — it has HP's own richer data model (Advanced Battery Management state, outlet segment control, real device identity/MAC, self-test control, true Watts). Standard UPS-MIB supplements it for exactly three things CPQPOWER-MIB cannot provide or provides with less precision:
1. `upsAlarmsPresent` — CPQPOWER-MIB has no alarm-count OID at all.
2. `upsSecondsOnBattery` — CPQPOWER-MIB only reports remaining runtime, never elapsed time already spent on battery.
3. Output apparent power (VA) and 0.1A-resolution current — CPQPOWER-MIB's own current/apparent-power fields on this unit are whole-number resolution only; standard UPS-MIB gives one more decimal digit, and its VA value paired with CPQPOWER's real-Watts value lets the dashboard show power factor without fabricating anything.

## Verification method

Both MIB trees were walked live against the device (`10.50.20.3`, SNMPv3, user `readuser`, `authNoPriv`, MD5). Cross-checks that confirmed the OID map and scaling:
- Nameplate rating fields (`upsConfigOutputVA`=3000, `upsConfigOutputPower`=2700, `upsConfigOutputVoltage`=230, `upsConfigOutputFreq`=500→50.0Hz) match the device's own published spec sheet exactly, in both MIB trees independently.
- `upsIdentOemCode` = 12 (0x0C), matching the CPQPOWER-MIB's own documented statement that "this should be 0x0c for HP."
- Input/output voltage (235V) and line-quality counter (`Counter32: 5`) are identical between the two trees.
- Battery voltage: standard UPS-MIB reports `770` at 0.1V resolution = 77.0V; CPQPOWER-MIB reports `77` unscaled = 77V. Identical value, confirming CPQPOWER's battery voltage OID is whole-volt resolution (not 0.1V as might be assumed by analogy with other fields) and that 77V is plausible for the reported 6x 12V VRLA series-string battery configuration.
- Low/high voltage transfer points (160V/294V) match exactly between `upsConfigLowVoltageTransferPoint`/`HighVoltageTransferPoint` (standard MIB) and `upsConfigLowOutputVoltageLimit`/`HighOutputVoltageLimit` (CPQPOWER-MIB).
- The device's own real-time clock (CPQPOWER `upsConfigDateAndTime`) read back the correct current date at walk time, confirming the OID's identity.
- `upsTopoUnitNumber` = 0, matching the MIB's documented meaning "standalone units use a value of 0."

## Field reference

### Measurement: `snmp_ups_status` (polled every 10s)

| Field | OID | Source MIB | Raw type / scaling | Meaning |
|---|---|---|---|---|
| `output_source_code` | `1.3.6.1.4.1.232.165.3.4.5.0` | CPQPOWER `upsOutputSource` | INTEGER, no scaling | Operating mode enum (see below) |
| `load_percent` | `1.3.6.1.4.1.232.165.3.4.1.0` | CPQPOWER `upsOutputLoad` | INTEGER percent, no scaling | Output load, % of rated capacity (2700W) |
| `battery_charge_percent` | `1.3.6.1.4.1.232.165.3.2.4.0` | CPQPOWER `upsBatCapacity` | INTEGER percent, no scaling | Battery state of charge |
| `battery_abm_status_code` | `1.3.6.1.4.1.232.165.3.2.5.0` | CPQPOWER `upsBatteryAbmStatus` | INTEGER, no scaling | ABM charge-cycle state enum (see below) |
| `battery_runtime_seconds` | `1.3.6.1.4.1.232.165.3.2.1.0` | CPQPOWER `upsBatTimeRemaining` | INTEGER seconds, no scaling | Estimated runtime remaining at present load |
| `battery_voltage_volts` | `1.3.6.1.2.1.33.1.2.5.0` | Standard `upsBatteryVoltage` | INTEGER, **0.1V DC** (`conversion = "float(1)"`) | Battery string voltage |
| `battery_current_amps` | `1.3.6.1.2.1.33.1.2.6.0` | Standard `upsBatteryCurrent` | INTEGER, **0.1A DC** (`conversion = "float(1)"`) | Positive = discharging, negative = recharging |
| `alarms_present` | `1.3.6.1.2.1.33.1.6.1.0` | Standard `upsAlarmsPresent` | Gauge32, no scaling | Count of currently active device alarms |
| `seconds_on_battery` | `1.3.6.1.2.1.33.1.2.2.0` | Standard `upsSecondsOnBattery` | INTEGER seconds, no scaling | Elapsed time since the current on-battery event began (0 = on mains) |

**`output_source_code` enum** (CPQPOWER `upsOutputSource` / NUT's `cpqpower_pwr_info`): `1`=other, `2`=off/none, `3`=normal (online, on mains), `4`=bypass, `5`=battery, `6`=booster (online, boosting a sagging input), `7`=reducer (online, trimming a high input), `8`=parallelCapacity, `9`=parallelRedundant, `10`=highEfficiencyMode.

**`battery_abm_status_code` enum** (CPQPOWER `upsBatteryAbmStatus`): `1`=charging, `2`=discharging, `3`=floating (topping off to float voltage), `4`=resting (fully charged, idle), `5`=unknown.

### Measurement: `snmp_ups_electrical` (polled every 20s)

| Field | OID | Source MIB | Raw type / scaling | Meaning |
|---|---|---|---|---|
| `input_voltage_volts` | `1.3.6.1.4.1.232.165.3.3.4.1.2.1` | CPQPOWER `upsInputVoltage` (table, phase 1) | INTEGER RMS volts, no scaling | Mains input voltage |
| `input_current_amps` | `1.3.6.1.2.1.33.1.3.3.1.4.1` | Standard `upsInputCurrent` (table, line 1) | INTEGER, **0.1A** (`conversion = "float(1)"`) | Mains input current |
| `input_frequency_hz` | `1.3.6.1.4.1.232.165.3.3.1.0` | CPQPOWER `upsInputFrequency` | INTEGER, **0.1Hz** (`conversion = "float(1)"`) | Mains input frequency |
| `input_realpower_watts` | `1.3.6.1.4.1.232.165.3.3.4.1.4.1` | CPQPOWER `upsInputWatts` (table, phase 1) | INTEGER Watts, no scaling | Mains real input power |
| `input_source_code` | `1.3.6.1.4.1.232.165.3.3.5.0` | CPQPOWER `upsInputSource` | INTEGER, no scaling | Present input source enum (see below) |
| `output_voltage_volts` | `1.3.6.1.4.1.232.165.3.4.4.1.2.1` | CPQPOWER `upsOutputVoltage` (table, phase 1) | INTEGER RMS volts, no scaling | Output voltage |
| `output_current_amps` | `1.3.6.1.2.1.33.1.4.4.1.3.1` | Standard `upsOutputCurrent` (table, line 1) | INTEGER, **0.1A** (`conversion = "float(1)"`) | Output current |
| `output_frequency_hz` | `1.3.6.1.4.1.232.165.3.4.2.0` | CPQPOWER `upsOutputFrequency` | INTEGER, **0.1Hz** (`conversion = "float(1)"`) | Output frequency |
| `output_realpower_watts` | `1.3.6.1.4.1.232.165.3.4.4.1.4.1` | CPQPOWER `upsOutputWatts` (table, phase 1) | INTEGER Watts, no scaling | Output **real** power |
| `output_apparentpower_va` | `1.3.6.1.2.1.33.1.4.4.1.4.1` | Standard `upsOutputPower` (table, line 1) | INTEGER VA, no scaling | Output **apparent** power |

**`input_source_code` enum** (CPQPOWER `upsInputSource`): `1`=other, `2`=none, `3`=primaryUtility, `4`=bypassFeed, `5`=secondaryUtility, `6`=generator, `7`=flywheel, `8`=fuelcell.

This UPS is single-phase (`upsInputNumPhases`/`upsOutputNumPhases` = 1 in both MIB trees), so every table OID above is read directly at table index 1 rather than walked as a multi-row table.

**Real vs. apparent power**: `output_realpower_watts` (Watts) and `output_apparentpower_va` (VA) are deliberately pulled from two different MIB trees because CPQPOWER-MIB only exposes real power and standard UPS-MIB only exposes apparent power for the output side. Their ratio (W/VA) is the power factor, computed in the Grafana panel rather than in Telegraf. Neither MIB tree exposes an apparent-power value for the *input* side — both only expose real (true) input power — so no `input_apparentpower` field exists; it was not fabricated to match the output side.

### Measurement: `snmp_ups_outlet` (polled every 30s, tagged by `index` = outlet segment 1 or 2)

| Field | OID (column) | Source MIB | Raw type / scaling | Meaning |
|---|---|---|---|---|
| `status_code` | `1.3.6.1.4.1.232.165.3.10.2.1.2` | CPQPOWER `upsRecepStatus` | INTEGER, no scaling | Segment power state enum (see below) |
| `off_delay_seconds` | `1.3.6.1.4.1.232.165.3.10.2.1.3` | CPQPOWER `upsRecepOffDelaySecs` | INTEGER seconds, `-1`=no shutdown pending | Countdown to a commanded shutdown of this segment |
| `on_delay_seconds` | `1.3.6.1.4.1.232.165.3.10.2.1.4` | CPQPOWER `upsRecepOnDelaySecs` | INTEGER seconds, `-1`=no startup pending | Countdown to a commanded startup of this segment |
| `auto_off_delay_seconds` | `1.3.6.1.4.1.232.165.3.10.2.1.5` | CPQPOWER `upsRecepAutoOffDelay` | INTEGER seconds, `-1`=never | Configured delay before this segment auto-sheds after going on battery (load-shedding priority) |
| `auto_on_delay_seconds` | `1.3.6.1.4.1.232.165.3.10.2.1.6` | CPQPOWER `upsRecepAutoOnDelay` | INTEGER seconds, `-1`=never | Configured delay before this segment auto-restores after mains returns |

This UPS has exactly 2 independently controllable outlet segments (`upsNumReceptacles` = 2, matching the device's published "2 independent load segments" spec), each hardcoded as its own dashboard panel rather than a repeated/templated panel, since the count is fixed hardware.

**`status_code` enum** (CPQPOWER `upsRecepStatus`): `1`=on, `2`=off, `3`=pendingOff, `4`=pendingOn, `5`=unknown.

### Measurement: `snmp_ups_identity` (polled every 300s — static/rarely-changing data)

| Field | OID | Source MIB | Meaning |
|---|---|---|---|
| `manufacturer` | `1.3.6.1.4.1.232.165.3.1.1.0` | CPQPOWER `upsIdentManufacturer` | Real UPS manufacturer ("EATON" — HP/HPE rebadges Eaton-built hardware for this model) |
| `model` | `1.3.6.1.4.1.232.165.3.1.2.0` | CPQPOWER `upsIdentModel` | Model string |
| `ups_firmware_version` | `1.3.6.1.4.1.232.165.3.1.3.0` | CPQPOWER `upsIdentSoftwareVersions` | UPS microcontroller firmware |
| `oem_code` | `1.3.6.1.4.1.232.165.3.1.4.0` | CPQPOWER `upsIdentOemCode` | Vendor code (`12`/`0x0C` = HP, per MIB documentation) |
| `serial_number` | `1.3.6.1.4.1.232.165.1.2.7.0` | CPQPOWER `deviceSerialNumber` | UPS serial number |
| `nmc_manufacturer` | `1.3.6.1.4.1.232.165.1.2.1.0` | CPQPOWER `deviceManufacturer` | Network management card manufacturer |
| `nmc_model` | `1.3.6.1.4.1.232.165.1.2.2.0` | CPQPOWER `deviceModel` | Network management card model name |
| `nmc_firmware_version` | `1.3.6.1.4.1.232.165.1.2.3.0` | CPQPOWER `deviceFirmwareVersion` | Network management card firmware |
| `nmc_hardware_version` | `1.3.6.1.4.1.232.165.1.2.4.0` | CPQPOWER `deviceHardwareVersion` | Network management card hardware revision |
| `device_ident_name` | `1.3.6.1.4.1.232.165.1.2.5.0` | CPQPOWER `deviceIdentName` | Internal product identifier (e.g. `USVHPRT3000G4`) |
| `nmc_part_number` | `1.3.6.1.4.1.232.165.1.2.6.0` | CPQPOWER `devicePartNumber` | Network management card part number (AF465A) |
| `nmc_mac_address` | `1.3.6.1.4.1.232.165.1.2.8.0` | CPQPOWER `deviceMACAddress` | Network management card MAC address |
| `test_result_code` | `1.3.6.1.4.1.232.165.3.7.2.0` | CPQPOWER `upsTestBatteryStatus` | Last self-test result enum (see below) |
| `nominal_output_voltage_volts` | `1.3.6.1.4.1.232.165.3.9.1.0` | CPQPOWER `upsConfigOutputVoltage` | Nameplate output voltage |
| `nominal_input_voltage_volts` | `1.3.6.1.4.1.232.165.3.9.2.0` | CPQPOWER `upsConfigInputVoltage` | Nameplate input voltage |
| `nominal_output_power_watts` | `1.3.6.1.4.1.232.165.3.9.3.0` | CPQPOWER `upsConfigOutputWatts` | Nameplate real power rating (2700W) |
| `nominal_output_frequency_hz` | `1.3.6.1.4.1.232.165.3.9.4.0` | CPQPOWER `upsConfigOutputFreq` | Nameplate frequency, **0.1Hz** (`conversion = "float(1)"`) |
| `low_voltage_transfer_volts` | `1.3.6.1.4.1.232.165.3.9.6.0` | CPQPOWER `upsConfigLowOutputVoltageLimit` | Input voltage below which the UPS transfers to battery |
| `high_voltage_transfer_volts` | `1.3.6.1.4.1.232.165.3.9.7.0` | CPQPOWER `upsConfigHighOutputVoltageLimit` | Input voltage above which the UPS transfers to battery |
| `low_battery_runtime_minutes` | `1.3.6.1.2.1.33.1.9.7.0` | Standard `upsConfigLowBattTime` | Device's own "low battery" runtime warning threshold (minutes) — drives the Runtime Remaining gauge's red threshold |
| `outlet_count` | `1.3.6.1.4.1.232.165.3.10.1.0` | CPQPOWER `upsNumReceptacles` | Number of independently controllable outlet segments |

**`test_result_code` enum** (CPQPOWER `upsTestBatteryStatus`): `1`=unknown, `2`=passed, `3`=failed, `4`=inProgress, `5`=notSupported, `6`=inhibited, `7`=scheduled.

## Deliberately excluded

- **Ambient/environmental temperature and humidity** (`upsEnvAmbientTemp` and related OIDs under CPQPOWER `1.3.6.1.4.1.232.165.3.6`). The device answers these OIDs, but the temperature reading and its configured lower limit both read back a flat `0`, while the upper limit reads a plausible default (`40`) — the classic signature of an optional temperature/humidity probe accessory that is not physically connected to the network management card. Rather than build a panel around a phantom `0°C` reading, these were left out. If a probe is later connected, the OIDs are already known (see the CPQPOWER-MIB `upsEnvironment` group) and can be added.
- **`upsBatteryTemperature`** (standard UPS-MIB, `1.3.6.1.2.1.33.1.2.7.0`) — also reads a flat `0`, same reasoning.
- **UPS topology group** (`upsTopologyType`, `upsTopoMachineCode`, `upsTopoUnitNumber`, `upsTopoPowerStrategy`) — exists and was read (`upsTopoUnitNumber` = 0, correctly indicating a standalone, non-parallel unit), but has no practical monitoring value for a single standalone UPS; it only matters for parallel/modular UPS installations.
- **Bypass voltage/frequency table** (`upsBypassFrequency`/`upsBypassNumPhases`, both read as `0`) — this line-interactive unit has no separately-metered bypass line, so there is nothing to graph.
- **Control-group delay OIDs at the system level** (`upsControlOutputOffDelay` etc.) and the **battery/trap self-test command OIDs** (`upsTestBattery`, `upsTestTrap`) — these are write-only command triggers, not metrics; polling them just reads back whatever was last set (usually `0`/disabled) and provides no monitoring value. The equivalent per-outlet delay values are polled instead, since those are genuinely informative.

## Thresholds: device-reported vs. engineering defaults

| Threshold | Value | Source |
|---|---|---|
| Voltage transfer limits (input/output gauges) | 160V / 294V | **Device-reported** — `upsConfigLowOutputVoltageLimit`/`HighOutputVoltageLimit`, confirmed identical in both MIB trees |
| Runtime "critical" threshold | 180s (3 min) | **Device-reported** — standard UPS-MIB `upsConfigLowBattTime` = 3 minutes |
| Load "warning"/"critical" | 80% / 100% | Engineering default — 100% = nameplate-rated capacity (2700W), a standard UPS monitoring convention |
| Battery charge "warning"/"critical" | 50% / 20% | Engineering default — the device does not expose a percent-based low-battery threshold (only the time-based one above) |
| Frequency acceptable band | 50Hz ± 3% (48.5–51.5Hz "green", 47–53Hz "yellow") | Engineering default — neither MIB tree exposes a frequency transfer-limit OID for this model |
| Voltage nominal band | 230V ± 10% (207–253V "green") | Engineering default overlay inside the device-reported transfer limits |

## Polling intervals

| Block | Interval | Rationale |
|---|---|---|
| `snmp_ups_status` | 10s | Everything needed to reconstruct a power event minute-by-minute: operating mode, battery state/voltage/current/charge/runtime, alarms, elapsed time on battery |
| `snmp_ups_electrical` | 20s | Input/output voltage, current, frequency, power — changes more gradually than battery state under normal conditions |
| `snmp_ups_outlet` | 30s | Outlet segment status changes only during a shutdown/restart sequence |
| `snmp_ups_identity` | 300s | Static nameplate/identity data that only changes on a firmware or hardware change |
