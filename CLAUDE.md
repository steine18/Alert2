# CR350 ALERT2 Datalogger — CLAUDE.md

## Project Overview

CRBasic program for a **Campbell Scientific CR350** datalogger deployed at a flood/stream monitoring station. The logger reads a tipping bucket rain gauge and two gage-height sensors (radar and bubbler), logs data locally, and forwards readings to a **Sutron AL200 ALERT2 Encoder/Modulator** operating in **data logger peripheral mode** via RS-232.

The AL200 handles all RF encoding, TDMA timing, and radio transmission. The CR350 only constructs and sends the application-layer payload in the **ALERT2 IND API v2 binary format**.

Transmissions occur:
- **Hourly** (timed, unconditional)
- **On threshold** — when rain tips are counted or gage height changes by more than 0.015 ft

---

## System Architecture

```
CR350 (APD)  ──RS-232──►  AL200 (IND)  ──RF──►  ALERT2 Network
```

In ALERT2 terminology:
- **APD** (Application Protocol Device): the CR350. Constructs IND API packets and sends them serially.
- **IND** (Intelligent Network Device): the AL200. Receives IND API packets, handles MANT/AirLink encoding, GPS timing, TDMA slot management, and drives the radio transmitter.

The CR350 sends pre-formatted binary packets to the AL200 using the **ALERT2 IND API v2 binary protocol** (prefix `AL22b`). The AL200 retransmits them as proper ALERT2 RF frames.

---

## Site / Station Info

| Constant      | Value           |
|---------------|-----------------|
| Station ID    | `30235`         |
| Test ID       | `30299`         |
| Latitude      | 36.1596948°N    |
| Longitude     | 115.0452761°W   |
| Elevation     | 527 m (1730 ft) |

Station is in the Las Vegas, NV area.

---

## Hardware

| Component | Detail |
|-----------|--------|
| Datalogger | Campbell Scientific CR350 |
| Rain gauge | Tipping bucket, `P_SW` pulse input, 0.01 in/tip |
| Radar gauge | SDI-12 address `"0"`, reports gage height (ft) |
| Bubbler gauge | SDI-12 address `"1"`, reports gage height (ft) |
| AL200 | Peripheral mode, CR350 `Com2` (RS-232), 57600 baud |
| Modem | Controlled via `SW12_1` switched 12V |

**Wiring note:** The AL200 RS-232 port is DCE (DB9 female). Connect to CR350 using a **DB9 male-to-male null modem cable**.

---

## AL200 Configuration (Peripheral Mode)

Configure via **Device Configuration Utility (DevConfig)** over USB at 57600 baud. OS version: `AL200.ALERT2.Std.06.00`.

| Tab | Setting | Value |
|-----|---------|-------|
| Main | Operation Mode | `ALERT2 on RS-232 (CS I/O and Sensors Disabled)` |
| ComPort | RS-232 Baud Rate | 57600 |
| ComPort | RS-232 Parity | None |
| ComPort | RS-232 Stop Bits | One |
| ComPort | RS-232 HW Flow Control | Off |
| ALERT2 | Station Source Address | **30235** (must match `StationID` constant) |
| ALERT2 | Enable TDMA | Yes |
| ALERT2 | TDMA Frame Length | Per network design (ms) |
| ALERT2 | TDMA Slot Length | Per network design (ms) |
| ALERT2 | TDMA Slot Start Offset | Per network design (ms) |

> **Important:** If the data logger sends IND commands, they overwrite values set in DevConfig. Do not use DevConfig and IND-API commands simultaneously. The current program does NOT send IND commands for TDMA/radio config — those must be set in DevConfig directly.

> **Important:** Station Source Address is a network-unique integer (1–65,501). Register at www.alert2.org. Never duplicate an address in the network.

---

## Program Structure

### Scan Sequences

**Fast scan — every 1 second (`P_SW` pulse count)**
- Counts tipping bucket pulses via `PulseCount`
- Sets `Precip_Flag = True` if tips > 0; accumulates `Precip_Tips` and `Precip_in`; calls `Record_Tip_Time` once per tip
- Handles `Force_Measurement` and `Calibrate_SDI` triggers

**Slow sequence — every 60 seconds**
- Reads battery voltage (`BattV`)
- Reads radar via `Read_Radar` sub (SDI-12 `M!` at address `"0"`)
- Controls modem power (`SW12_1`) based on `Modem_On` flag
- Compares radar reading to `Previous_Radar_GH`; sets `Radar_Flag` if delta > 0.015 ft
- Reads bubbler; compares to `Previous_Bubbler_GH`; sets `Bubbler_Flag` if delta > 0.015 ft
- Reads bubbler via `SDI12Recorder` at address `"1"` → `SDI2_Values()`
- Calls `OneMinute` data table
- If any flag set → sends ALERT2 via `Send_Event_IND` and clears all flags
- If `TimeIntoInterval(0,60,Min)` → sends hourly `Send_Timed_IND`
- If `TimeIntoInterval(0,5,Min)` → calls `FiveMinuteData` table (includes MQTT publish)
- Handles test mode auto-timeout (60 minutes)

---

## Data Tables

| Table | Interval | Records | Notes |
|-------|----------|---------|-------|
| `FiveMinuteData` | 5 min | 129600 (~450 days) | Includes `MQTTPublishTable` |
| `OneMinute` | 1 min | 129600 (~90 days) | |
| `ALERT2_Table` | On-call | 129600 | Logs raw IND hex string + last sent type |

All tables store: precipitation total, radar GH, bubbler GH (`SDI2_Values`), battery voltage.

---

## ALERT2 IND API Packet Construction

The CR350 constructs binary packets as hex strings and transmits them byte-by-byte to the AL200 via `SerialOutBlock` on `Com2`.

### Packet Format

The format is **ALERT2 IND API v2 binary** (`AL22b`). All fields are TLV (Type-Length-Value):

```
[Prefix: AL22b] [Total Length: 1 byte]
  [Set Parameters (0x0A)] [Length]
    [IND Address (0x18)] [Length] [StationID bytes]
  [Self-Report Protocol (0x00)] [Application PDU Length]
    [Control byte: 0x70]
    [Report TLV(s)...]
```

- **Prefix** `414C323262` = ASCII `AL22b`, where `2` = IND API version 2, `b` = binary format
- **Set Parameters** (type `0x0A`): wraps the source address
- **IND Address** (type `0x18`): the station source address (StationID)
- **Self-Report** (type `0x00`): wraps the Application PDU
- **Control byte** `0x70`: APDU ID disabled, not test, no timestamp, version 0

### Report Types Used

- **Type `01`** — General Sensor Report (GSR): one or more Sensor-ID / F/L / Value triplets
- **Type `02`** — Tipping Bucket Rain Gauge Report (TBRG): includes per-tip time offsets

### Sensor IDs

Per the ALERT2 Application Layer Protocol Specification v1.3 (as implemented by the AL200, Appendix G of the AL200 manual):

| Sensor | Code Constant | ID (hex) | Recommended per spec |
|--------|--------------|----------|---------------------|
| Rain / Precip | `PrecipID` | `00` | Rain ✓ |
| Radar gage height | `RadarID` | `07` | Stage ✓ |
| Battery voltage | `BatteryID` | `08` | Battery voltage ✓ |
| Bubbler gage height | `BubblerID` | `09` | Custom (9 not reserved) |
| GPS / Clock status | `GPSID` | `10` | Custom (10 not reserved) |

> **Note on sensor ID conventions:** The 2011 ALERT2 Protocol Description document (also in References/) lists a *different* convention where 1=Rain, 2=Stage, 3=Battery. That document predates the Application Layer Protocol Spec v1.3. The AL200 and this code follow the v1.3 convention (0=Rain, 7=Stage, 8=Battery), which is the correct one for this hardware.

### F/L Byte in `SensorReport`

The `SensorReport` function builds: `{id}{format}{sensor_length}{value_hex}`. The `format` ("1" = unsigned integer) and `sensor_length` (decimal byte count of value) concatenate to form the correct F/L byte:
- e.g., value = 1 byte → F/L = `"1"+"1"` = `0x11` = unsigned, 1 byte
- e.g., value = 2 bytes → F/L = `"1"+"2"` = `0x12` = unsigned, 2 bytes

### Value Scaling

| Sensor | Scaling |
|--------|---------|
| Radar GH | `Round(Radar_GH * 100, 0)` → integer hundredths of a foot |
| Bubbler GH | `Round(SDI2_Values(1) * 100, 0)` → integer hundredths of a foot |
| Battery | `Round(BattV * 100, 0)` → integer hundredths of a volt |
| Precip (timed) | `Round(Precip_Tips, 0)` → integer tip count |

### Two Transmission Subroutines

- **`Send_Timed_IND`** — hourly unconditional transmission. Sends all sensors as a GSR (Type 01). Sets `Last_Sent_IND = "Timed"`. Calls `Clear_Tip_Time` after sending.
- **`Send_Event_IND`** — threshold-triggered transmission. Sends TBRG (Type 02) with 2-byte accumulator and 1-byte-per-tip time offsets if `Precip_Flag` is set; sends GSR (Type 01) with stage sensor(s) and battery if `Radar_Flag` or `Bubbler_Flag` is set. Both report types can appear in the same APDU. Sets `Last_Sent_IND = "Event"`. Calls `Clear_Tip_Time` after sending.

---

## Public Variables / Flags

| Variable | Purpose |
|----------|---------|
| `Modem_On` | Controls `SW12_1` modem power |
| `Precip_Flag` | Set when tips counted; triggers ALERT2 |
| `Radar_Flag` | Set when radar GH change > 0.015 ft |
| `Bubbler_Flag` | Reserved; not yet set by any logic |
| `Precip_Tips` | Cumulative tip count (preserves across power cycle) |
| `Precip_in` | Rain in inches (display only — `Precip_Tips * TB_Tip_in`); data tables use `Sample(Precip_Tips)` directly |
| `Radar_GH` | Current radar gage height (ft), offset-corrected |
| `Radar_Offset` | Applied during `Calibrate_SDI` |
| `Radar_Raw` | Raw SDI-12 reading from radar |
| `Previous_Radar_GH` | Previous reading for delta threshold check |
| `BattV` | Battery voltage |
| `AL2_IND` | Last full ALERT2 IND packet as hex string |
| `Last_Sent_IND` | `"Event"` or `"Timed"` |
| `Calibrate_SDI` | Trigger: offset-calibrate radar against current GH |
| `Force_Measurement` | Trigger: force a radar read outside slow sequence |
| `Force_ALERT2` | Trigger: force an immediate ALERT2 transmission |
| `Test_RS232` | Debug: sends hex chars as ASCII text instead of binary |
| `Test_Alert` | Uses `TestStationID`; auto-disables after 60 minutes |
| `Test_Precip_Tips` | Tip count used in test mode |
| `Scanning_On` | Status flag (not currently used programmatically) |

All `Public` variables are readable/writable from LoggerNet/RTMC. `PreserveVariables` is set — most state survives power cycles.

---

## Known Issues & Incomplete Areas

### Bugs

1. ~~**Bubbler sensor report sends radar value**~~ — **Fixed.** `Send_Timed_IND` and `Send_Event_IND` now correctly pass `SDI2_Values(1)` for the bubbler sensor report.

2. ~~**`precip_value_hex` in `Send_Event_IND` ignores `Test_Alert` flag**~~ — **Fixed.** `precip_value_hex` now uses `AL2_Precip_Val` (which respects the `Test_Alert` flag) instead of always using `Precip_Tips`.

3. ~~**`Read_Radar` always resets `Previous_Radar_GH`**~~ — **Fixed.** Removed the `Previous_Radar_GH = Radar_GH` assignment from the end of `Read_Radar`; the slow sequence updates it after the delta check.

4. ~~**`Send_Timed_IND` drops all sensor data when `Precip_Flag = True`**~~ — **Fixed.** `Send_Timed_IND` rewritten as a clean linear packet builder. Removed all dead TBRG branch code and the broken dual-path assembly. All transmissions now send a full GSR (Type 01) with precip, radar, bubbler, and battery.

### Not Yet Implemented

- GPS clock status report (sensor ID `GPSID = "10"`) — requires querying AL200 Clock Status parameter (type `0x7F`) over `Com2`. Add to both `Send_Timed_IND` and `Send_Event_IND`. See IND API spec section on Parameter TLVs.
- **`Read_Radar` calibration uses `RadarSDI(2)` but normal reads use `RadarSDI(1)`** (`Sensors.CRB`) — both use the same `M!` command. If index 1 is the correct gage height field, the calibration offset is computed against the wrong value. Verify which SDI-12 response index the radar sensor returns gage height on.
- **NTP time sync** — CR350 clock can be synced via `NetworkTimeProtocol()` over PPP. A commented-out stub exists in the fast scan (`Cr350_Alert.CRB`). Determine COM port, sync interval, and whether to tie it to the hourly transmission or a separate schedule.

---

## File Structure

The program is split into four files. All files must be present on the datalogger's `CPU:` drive. `Include` performs inline text substitution, so declaration order matters — the includes must appear before `BeginProg`.

| File | Contents |
|------|----------|
| `Cr350_Alert.CRB` | Main file: constants, `Public`/`Dim` declarations, data tables, `BeginProg` with both scan sequences |
| `Helpers.CRB` | Pure utility functions: `IntToHexStr`, `SensorReport`, `ByteLength`, `HexCharToNibble`, `RealTimeSS90` |
| `Sensors.CRB` | Sensor subroutines: `Read_Radar`. Future bubbler calibration subs belong here too. |
| `ALERT2.CRB` | Packet construction and transmission: `Send_to_AL200`, `Send_Timed_IND`, `Send_Event_IND`, `Record_Tip_Time`, `Clear_Tip_Time` |

Include order in `Cr350_Alert.CRB` (after variable/table declarations, before `BeginProg`):

```vb
Include "CPU:Helpers.CRB"
Include "CPU:Sensors.CRB"
Include "CPU:ALERT2.CRB"
```

Helpers must come before ALERT2 since `Send_Timed_IND` and `Send_Event_IND` call `SensorReport` and `IntToHexStr`.

---

## Development Notes

- **Language**: CRBasic (Campbell Scientific proprietary BASIC dialect)
- **IDE**: CRBasic Editor (part of LoggerNet or PC400). Use `.CRB` extension. Note: the editor may show false "undefined symbol" errors for names defined in included files — this is a known editor limitation and does not affect compilation or execution on the logger.
- **Include syntax**: CRBasic uses `Include` (no `#` prefix). `#Include` will produce an "Unknown instruction" compile error.
- **Deployment**: Deploy all four `.CRB` files to the CR350 via USB or LoggerNet. Set `Cr350_Alert.CRB` as the running program.
- **Serial comms**: ALERT2 uses `Com2` in binary mode (`SerialOutBlock`). `Test_RS232 = True` switches to ASCII hex output for debugging with a terminal emulator.
- **`PreserveVariables`** means tip counts and offsets persist across power cycles — reset `Precip_Tips` and `Radar_Offset` intentionally when needed.
- **Test mode** (`Test_Alert = True`) uses `TestStationID = 30299` and auto-expires after 60 scan cycles (~60 minutes). Prevents test transmissions from polluting live network data.
- **AL200 GPS lock**: The AL200 needs a GPS fix for accurate TDMA timing. GPS lock may take up to 14 minutes after power-up. Without GPS, the AL200 transmits at random times (ALOHA mode). Monitor GPS LED status during commissioning.
- **Do not run DevConfig and IND-API commands simultaneously** — they will overwrite each other's settings on the AL200.

---

## Reference Documents

| File | Description |
|------|-------------|
| `References/ALERT2_Description_102511.pdf` | Overview of ALERT2 protocol layers (AirLink, MANT, Application). Note: uses an older sensor ID convention that differs from v1.3 spec. |
| `References/Alert2_IND_API_Ver2.0_FINAL_2020-6.pdf` | **Primary reference** for IND API packet format. Version 2.0, June 2020. Defines the binary TLV format that CR350 sends to AL200. |
| `References/al200.pdf` | AL200 product manual (Nov 2023). Covers peripheral mode setup, DevConfig settings, wiring, and recommended sensor IDs (Appendix G). |
