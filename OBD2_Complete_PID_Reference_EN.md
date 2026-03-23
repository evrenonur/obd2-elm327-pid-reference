# OBD-II / ELM327 Complete PID & Command Reference

**Source:** Car Scanner ELM OBD2 (com.ovz.carscanner) reverse engineering  
**Standard:** SAE J1979 / ISO 15031-5  
**Date:** March 23, 2026  
**Purpose:** Complete reference for building OBD-II diagnostic applications  

---

## TABLE OF CONTENTS

1. [ELM327 Adapter Communication](#1-elm327-adapter-communication)
2. [Connection Setup](#2-connection-setup)
3. [AT Commands Full List](#3-at-commands-full-list)
4. [OBD Protocols](#4-obd-protocols)
5. [Mode 01 — Live Data (150 PIDs)](#5-mode-01--live-data-150-pids)
6. [Mode 02 — Freeze Frame](#6-mode-02--freeze-frame)
7. [Mode 03 — Read DTCs](#7-mode-03--read-dtcs)
8. [Mode 04 — Clear DTCs](#8-mode-04--clear-dtcs)
9. [Mode 05 — O2 Sensor Monitor Test](#9-mode-05--o2-sensor-monitor-test)
10. [Mode 06 — On-Board Monitoring](#10-mode-06--on-board-monitoring)
11. [Mode 09 — Vehicle Information](#11-mode-09--vehicle-information)
12. [Mode 22 — Extended / UDS](#12-mode-22--extended-diagnostics-uds)
13. [DTC Code Structure](#13-dtc-code-structure)
14. [Manufacturer-Specific Protocols](#14-manufacturer-specific-protocols)
15. [Connection Profiles](#15-connection-profiles)
16. [Data Parse Formulas](#16-data-parse-formulas)
17. [Sample Code Structures](#17-sample-code-structures)

---

## 1. ELM327 Adapter Communication

### Communication Basics

- **Line ending:** Every command ends with `\r` (CR, 0x0D)
- **Response end:** `>` prompt character indicates response is complete
- **Multiple ECUs:** Multiple ECUs may respond — parse all responses
- **Errors:** `NO DATA`, `UNABLE TO CONNECT`, `?`, `ERROR`
- **Timeout:** Default ~200ms, configurable via `ATST`

### Connection Addresses

| Method | Address | Port | Description |
|--------|---------|------|-------------|
| **WiFi (Default)** | `192.168.0.10` | `35000` | Most WiFi ELM327 clones |
| **WiFi (PLX KiWi)** | `192.168.0.10` | `30000` | PLX KiWi WiFi |
| **WiFi (OBDLink)** | `192.168.0.74` | `23` | OBDLink WiFi |
| **WiFi (Alt. Subnet)** | `192.168.1.10` | `35000` | Alternative network |
| **Bluetooth SPP** | UUID: `00001101-0000-1000-8000-00805F9B34FB` | — | Serial Port Profile (RFCOMM) |
| **Bluetooth LE** | GATT services | — | BLE 4.0+ adapters |

WiFi network detection: Look for `WiFi_OBDII` or `Wi-Fi_OBDII` SSID.

### TCP Socket Connection (WiFi)

```
1. Connect to WiFi network (SSID: WiFi_OBDII)
2. Open TCP socket → 192.168.0.10:35000
3. Send ATZ\r → Wait for "ELM327 v1.5" response
4. Run init sequence
```

### Bluetooth SPP Connection

```
1. Scan for Bluetooth devices
2. Find device containing "OBD" or "ELM"
3. Establish RFCOMM connection with SPP UUID:
   UUID: 00001101-0000-1000-8000-00805F9B34FB
4. Send ATZ\r → Wait for response
5. Run init sequence
```

---

## 2. Connection Setup

### Init Sequence (Initialization)

The following commands are sent sequentially. Wait for `>` prompt after each command.

```
ATZ\r          → Reset adapter              → Response: "ELM327 v1.5" (version info)
ATE0\r         → Echo off                   → Response: "OK"
ATS0\r         → Remove spaces              → Response: "OK"
ATL0\r         → Linefeed off               → Response: "OK"
ATH0\r         → Hide headers               → Response: "OK"
ATAL\r         → Allow long messages        → Response: "OK"
ATSP0\r        → Auto-detect protocol       → Response: "OK"
```

### Voltage Reading

```
ATRV\r         → Response: "12.6V"   (battery voltage)
```

### Protocol Selection (Manual)

```
ATSP1\r   → SAE J1850 PWM (41.6 kbps)
ATSP2\r   → SAE J1850 VPW (10.4 kbps)
ATSP3\r   → ISO 9141-2 (5 baud init)
ATSP4\r   → ISO 14230 KWP (5 baud init)
ATSP5\r   → ISO 14230 KWP (fast init)
ATSP6\r   → ISO 15765 CAN 11-bit 500 kbps
ATSP7\r   → ISO 15765 CAN 29-bit 500 kbps
ATSP8\r   → ISO 15765 CAN 11-bit 250 kbps
ATSP9\r   → ISO 15765 CAN 29-bit 250 kbps
ATSPA\r   → SAE J1939 CAN 29-bit 250 kbps
ATSP0\r   → Auto-detect
```

---

## 3. AT Commands Full List

### General Commands

| Command | Description | Response |
|---------|-------------|----------|
| `ATZ` | Full adapter reset | `ELM327 v1.5` |
| `ATD` | Reset all settings to defaults | `OK` |
| `ATE0` | Echo off (don't echo sent commands) | `OK` |
| `ATE1` | Echo on | `OK` |
| `ATI` | ELM327 version info | `ELM327 v1.5` |
| `AT@1` | Device description string | Variable |
| `AT@2` | Device identifier | Variable |
| `ATRV` | Read vehicle battery voltage | `12.6V` |
| `ATWS` | Warm start | — |
| `ATLP` | Enter low power mode | — |

### Protocol Commands

| Command | Description |
|---------|-------------|
| `ATSP0-9,A` | Select protocol (0=auto, 1-9=manual, A=J1939) |
| `ATDP` | Display current protocol |
| `ATDPN` | Display current protocol number |
| `ATMA` | Monitor all CAN messages |
| `ATCM` | Set CAN ID mask |
| `ATCF` | Set CAN ID filter |
| `ATAR` | Set automatic receive |
| `ATAT0-2` | Adaptive Timing (0=off, 1=normal, 2=aggressive) |

### Format Commands

| Command | Description |
|---------|-------------|
| `ATS0` | Remove spaces from responses |
| `ATS1` | Show spaces in responses |
| `ATH0` | Hide CAN headers |
| `ATH1` | Show CAN headers |
| `ATL0` | Hide linefeed |
| `ATL1` | Show linefeed |
| `ATAL` | Allow long messages (>7 bytes) |

### CAN Bus Commands

| Command | Description |
|---------|-------------|
| `ATSH xxx` | Set CAN header/address (e.g., `ATSH 7E0`) |
| `ATST xx` | Set timeout (hex, ×4ms) |
| `ATFC` | Set Flow Control |
| `ATCAF0` | CAN Auto Formatting off |
| `ATCAF1` | CAN Auto Formatting on |
| `ATCEA` | CAN Extended Address |
| `ATCER` | CAN Error control (ELM327 v2.2+) |
| `ATTA xx` | Set Transmit Address (ELM327 v2.2+) |
| `ATCSM0` | CAN Silent Mode off |
| `ATCSM1` | CAN Silent Mode on |

### J1939 Commands

| Command | Description |
|---------|-------------|
| `ATDM1` | Monitor DM1 (active faults) |
| `ATMP xxxx` | Monitor J1939 PGN |
| `ATJE` | J1939 ELM format |
| `ATJS` | J1939 SAE format |
| `ATJHF0/1` | J1939 Header Formatting |

---

## 4. OBD Protocols

### Auto-Detection Order

| # | Protocol | AT Code | Baud | Year | Vehicle Type |
|---|----------|---------|------|------|-------------|
| 6 | ISO 15765-4 CAN 11-bit 500k | `ATSP6` | 500 kbps | 2008+ | **Most common (try first)** |
| 8 | ISO 15765-4 CAN 11-bit 250k | `ATSP8` | 250 kbps | 2004+ | Some EU vehicles |
| 7 | ISO 15765-4 CAN 29-bit 500k | `ATSP7` | 500 kbps | 2004+ | Some vehicles |
| 9 | ISO 15765-4 CAN 29-bit 250k | `ATSP9` | 250 kbps | 2004+ | Rare |
| 5 | ISO 14230-4 KWP (fast init) | `ATSP5` | 10.4 kbps | 1999-2008 | Older European/Asian |
| 4 | ISO 14230-4 KWP (5 baud) | `ATSP4` | 10.4 kbps | 1999-2008 | Older vehicles |
| 3 | ISO 9141-2 | `ATSP3` | 10.4 kbps | 1996-2004 | Older vehicles |
| 1 | SAE J1850 PWM | `ATSP1` | 41.6 kbps | 1996-2008 | Older Ford |
| 2 | SAE J1850 VPW | `ATSP2` | 10.4 kbps | 1996-2008 | Older GM |
| A | SAE J1939 | `ATSPA` | 250 kbps | All | Commercial vehicles |
| — | ISO 27145-2 WWH-OBD | Over CAN | CAN | 2025+ | Next-gen |

### CAN Addressing

| Address | Send → Receive | Description |
|---------|----------------|-------------|
| `7E0` → `7E8` | ECU #1 (Engine) | Main engine control unit |
| `7E1` → `7E9` | ECU #2 (Transmission) | Automatic transmission |
| `7E2` → `7EA` | ECU #3 | Additional control unit |
| `7E3` → `7EB` | ECU #4 | Additional control unit |
| `7E4` → `7EC` | ECU #5 | Additional control unit |
| `7E5` → `7ED` | ECU #6 | Additional control unit |
| `7E6` → `7EE` | ECU #7 | Additional control unit |
| `7E7` → `7EF` | ECU #8 | Additional control unit |
| `7DF` | — | Broadcast (all ECUs) |

---

## 5. Mode 01 — Live Data (150 PIDs)

### Send Format
```
01XX\r
```
XX = PID number (hex)

### Response Format
```
41 XX AA BB CC DD
```
- `41` = Mode 01 response (01 + 0x40)
- `XX` = PID number
- `AA BB CC DD` = Data bytes (count varies by PID)

### Supported PID Query

PIDs `00`, `20`, `40`, `60`, `80`, `A0`, `C0`, `E0` → return 32-bit bitmap showing which PIDs are supported.

**Bitmap reading:** Each bit represents the next 32 PIDs. MSB = first PID.

### Complete PID Table

| PID (Hex) | Command | Sensor Name | Bytes | Min | Max | Unit | Formula |
|-----------|---------|-------------|-------|-----|-----|------|---------|
| `00` | `0100` | PIDs supported [01-20] | 4 | — | — | Bitmap | Bit masking |
| `01` | `0101` | Monitor status since DTCs cleared (MIL status, DTC count) | 4 | — | — | — | A: bit 7=MIL on/off, bit 0-6=DTC count |
| `02` | `0102` | Freeze frame DTC | 2 | — | — | DTC | Standard DTC parse |
| `03` | `0103` | Fuel system status | 2 | — | — | Enum | 1=OL, 2=CL, 4=OL-drive, 8=OL-fault, 16=CL-fault |
| `04` | `0104` | Calculated engine load | 1 | 0 | 100 | % | `A × 100 / 255` |
| `05` | `0105` | Engine coolant temperature | 1 | −40 | 215 | °C | `A − 40` |
| `06` | `0106` | Short term fuel trim — Bank 1 | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `07` | `0107` | Long term fuel trim — Bank 1 | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `08` | `0108` | Short term fuel trim — Bank 2 | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `09` | `0109` | Long term fuel trim — Bank 2 | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `0A` | `010A` | Fuel pressure (gauge) | 1 | 0 | 765 | kPa | `A × 3` |
| `0B` | `010B` | Intake manifold absolute pressure (MAP) | 1 | 0 | 255 | kPa | `A` |
| `0C` | `010C` | Engine RPM | 2 | 0 | 16383.75 | rpm | `((A × 256) + B) / 4` |
| `0D` | `010D` | Vehicle speed | 1 | 0 | 255 | km/h | `A` |
| `0E` | `010E` | Timing advance | 1 | −64 | 63.5 | ° (BTDC) | `(A / 2) − 64` |
| `0F` | `010F` | Intake air temperature | 1 | −40 | 215 | °C | `A − 40` |
| `10` | `0110` | Mass air flow sensor (MAF) rate | 2 | 0 | 655.35 | g/s | `((A × 256) + B) / 100` |
| `11` | `0111` | Throttle position | 1 | 0 | 100 | % | `A × 100 / 255` |
| `12` | `0112` | Commanded secondary air status | 1 | — | — | Enum | Bit masking |
| `13` | `0113` | Oxygen sensors present (2 banks) | 1 | — | — | Bitmap | — |
| `14` | `0114` | O2 Sensor 1 — Voltage, STFT | 2 | 0-0 | 1.275-99.2 | V, % | `A/200`, `(B−128)×100/128` |
| `15` | `0115` | O2 Sensor 2 — Voltage, STFT | 2 | — | — | V, % | Same formula |
| `16` | `0116` | O2 Sensor 3 — Voltage, STFT | 2 | — | — | V, % | Same formula |
| `17` | `0117` | O2 Sensor 4 — Voltage, STFT | 2 | — | — | V, % | Same formula |
| `18` | `0118` | O2 Sensor 5 — Voltage, STFT | 2 | — | — | V, % | Same formula |
| `19` | `0119` | O2 Sensor 6 — Voltage, STFT | 2 | — | — | V, % | Same formula |
| `1A` | `011A` | O2 Sensor 7 — Voltage, STFT | 2 | — | — | V, % | Same formula |
| `1B` | `011B` | O2 Sensor 8 — Voltage, STFT | 2 | — | — | V, % | Same formula |
| `1C` | `011C` | OBD standards compliance | 1 | — | — | Enum | 1=OBD-II, 2=OBD, 3=OBD+OBD-II... |
| `1D` | `011D` | Oxygen sensors present (4 banks) | 1 | — | — | Bitmap | — |
| `1E` | `011E` | Auxiliary input status | 1 | — | — | Bit | Bit 0=PTO active |
| `1F` | `011F` | Run time since engine start | 2 | 0 | 65535 | seconds | `(A × 256) + B` |
| `20` | `0120` | PIDs supported [21-40] | 4 | — | — | Bitmap | — |
| `21` | `0121` | Distance traveled with MIL on | 2 | 0 | 65535 | km | `(A × 256) + B` |
| `22` | `0122` | Fuel rail pressure (relative to manifold vacuum) | 2 | 0 | 5177.27 | kPa | `((A × 256) + B) × 0.079` |
| `23` | `0123` | Fuel rail gauge pressure (diesel/GDI) | 2 | 0 | 655350 | kPa | `((A × 256) + B) × 10` |
| `24` | `0124` | O2 Sensor 1 — λ ratio, voltage | 4 | 0 | 2/8 | ratio/V | `λ=((A×256)+B)×2/65536`, `V=((C×256)+D)×8/65536` |
| `25` | `0125` | O2 Sensor 2 — λ ratio, voltage | 4 | — | — | ratio/V | Same formula |
| `26` | `0126` | O2 Sensor 3 — λ ratio, voltage | 4 | — | — | ratio/V | Same formula |
| `27` | `0127` | O2 Sensor 4 — λ ratio, voltage | 4 | — | — | ratio/V | Same formula |
| `28` | `0128` | O2 Sensor 5 — λ ratio, voltage | 4 | — | — | ratio/V | Same formula |
| `29` | `0129` | O2 Sensor 6 — λ ratio, voltage | 4 | — | — | ratio/V | Same formula |
| `2A` | `012A` | O2 Sensor 7 — λ ratio, voltage | 4 | — | — | ratio/V | Same formula |
| `2B` | `012B` | O2 Sensor 8 — λ ratio, voltage | 4 | — | — | ratio/V | Same formula |
| `2C` | `012C` | Commanded EGR | 1 | 0 | 100 | % | `A × 100 / 255` |
| `2D` | `012D` | EGR error | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `2E` | `012E` | Commanded evaporative purge | 1 | 0 | 100 | % | `A × 100 / 255` |
| `2F` | `012F` | Fuel tank level input | 1 | 0 | 100 | % | `A × 100 / 255` |
| `30` | `0130` | Warm-ups since DTCs cleared | 1 | 0 | 255 | count | `A` |
| `31` | `0131` | Distance traveled since DTCs cleared | 2 | 0 | 65535 | km | `(A × 256) + B` |
| `32` | `0132` | Evap system vapor pressure | 2 | −8192 | 8191.75 | Pa | `((A×256)+B) / 4` (signed) |
| `33` | `0133` | Absolute barometric pressure | 1 | 0 | 255 | kPa | `A` |
| `34` | `0134` | O2 Sensor 1 — λ ratio, current | 4 | 0 | 2/128 | ratio/mA | `λ=((A×256)+B)/32768`, `i=((C×256)+D)/256−128` |
| `35` | `0135` | O2 Sensor 2 — λ ratio, current | 4 | — | — | ratio/mA | Same formula |
| `36` | `0136` | O2 Sensor 3 — λ ratio, current | 4 | — | — | ratio/mA | Same formula |
| `37` | `0137` | O2 Sensor 4 — λ ratio, current | 4 | — | — | ratio/mA | Same formula |
| `38` | `0138` | O2 Sensor 5 — λ ratio, current | 4 | — | — | ratio/mA | Same formula |
| `39` | `0139` | O2 Sensor 6 — λ ratio, current | 4 | — | — | ratio/mA | Same formula |
| `3A` | `013A` | O2 Sensor 7 — λ ratio, current | 4 | — | — | ratio/mA | Same formula |
| `3B` | `013B` | O2 Sensor 8 — λ ratio, current | 4 | — | — | ratio/mA | Same formula |
| `3C` | `013C` | Catalyst temperature B1S1 | 2 | −40 | 6513.5 | °C | `((A×256)+B)/10 − 40` |
| `3D` | `013D` | Catalyst temperature B2S1 | 2 | −40 | 6513.5 | °C | `((A×256)+B)/10 − 40` |
| `3E` | `013E` | Catalyst temperature B1S2 | 2 | −40 | 6513.5 | °C | `((A×256)+B)/10 − 40` |
| `3F` | `013F` | Catalyst temperature B2S2 | 2 | −40 | 6513.5 | °C | `((A×256)+B)/10 − 40` |
| `40` | `0140` | PIDs supported [41-60] | 4 | — | — | Bitmap | — |
| `41` | `0141` | Monitor status this drive cycle | 4 | — | — | — | Same structure as PID 01 |
| `42` | `0142` | Control module voltage | 2 | 0 | 65.535 | V | `((A × 256) + B) / 1000` |
| `43` | `0143` | Absolute load value | 2 | 0 | 25700 | % | `((A × 256) + B) × 100 / 255` |
| `44` | `0144` | Commanded air-fuel equiv. ratio | 2 | 0 | 2 | λ | `((A × 256) + B) / 32768` |
| `45` | `0145` | Relative throttle position | 1 | 0 | 100 | % | `A × 100 / 255` |
| `46` | `0146` | Ambient air temperature | 1 | −40 | 215 | °C | `A − 40` |
| `47` | `0147` | Absolute throttle position B | 1 | 0 | 100 | % | `A × 100 / 255` |
| `48` | `0148` | Absolute throttle position C | 1 | 0 | 100 | % | `A × 100 / 255` |
| `49` | `0149` | Accelerator pedal position D | 1 | 0 | 100 | % | `A × 100 / 255` |
| `4A` | `014A` | Accelerator pedal position E | 1 | 0 | 100 | % | `A × 100 / 255` |
| `4B` | `014B` | Accelerator pedal position F | 1 | 0 | 100 | % | `A × 100 / 255` |
| `4C` | `014C` | Commanded throttle actuator | 1 | 0 | 100 | % | `A × 100 / 255` |
| `4D` | `014D` | Time run with MIL on | 2 | 0 | 65535 | minutes | `(A × 256) + B` |
| `4E` | `014E` | Time since DTCs cleared | 2 | 0 | 65535 | minutes | `(A × 256) + B` |
| `4F` | `014F` | Max values — λ, O2V, O2I, intake pressure | 4 | — | — | various | `A=ratio, B=V, C=mA, D=kPa×10` |
| `50` | `0150` | Max value — air flow rate from MAF | 4 | 0 | 2550 | g/s | `A × 10` |
| `51` | `0151` | Fuel type | 1 | — | — | Enum | 1=Gasoline, 4=Diesel... |
| `52` | `0152` | Ethanol fuel % | 1 | 0 | 100 | % | `A × 100 / 255` |
| `53` | `0153` | Absolute evap system vapor pressure | 2 | 0 | 327.675 | kPa | `((A×256)+B) / 200` |
| `54` | `0154` | Evap system vapor pressure (wide) | 2 | −32767 | 32767 | Pa | `((A×256)+B) − 32767` |
| `55` | `0155` | Short term secondary O2 trim B1+B3 | 2 | −100 | 99.2 | % | Each byte: `(X−128)×100/128` |
| `56` | `0156` | Long term secondary O2 trim B1+B3 | 2 | — | — | % | Same formula |
| `57` | `0157` | Short term secondary O2 trim B2+B4 | 2 | — | — | % | Same formula |
| `58` | `0158` | Long term secondary O2 trim B2+B4 | 2 | — | — | % | Same formula |
| `59` | `0159` | Fuel rail absolute pressure | 2 | 0 | 655350 | kPa | `((A×256)+B) × 10` |
| `5A` | `015A` | Relative accelerator pedal position | 1 | 0 | 100 | % | `A × 100 / 255` |
| `5B` | `015B` | Hybrid battery pack remaining life | 1 | 0 | 100 | % | `A × 100 / 255` |
| `5C` | `015C` | Engine oil temperature | 1 | −40 | 210 | °C | `A − 40` |
| `5D` | `015D` | Fuel injection timing | 2 | −210 | 301.99 | ° | `(((A×256)+B)−26880) / 128` |
| `5E` | `015E` | Engine fuel rate | 2 | 0 | 3212.75 | L/h | `((A×256)+B) / 20` |
| `5F` | `015F` | Emission requirements | 1 | — | — | Enum | — |
| `60` | `0160` | PIDs supported [61-80] | 4 | — | — | Bitmap | — |
| `61` | `0161` | Driver's demand engine torque | 1 | −125 | 130 | % | `A − 125` |
| `62` | `0162` | Actual engine torque | 1 | −125 | 130 | % | `A − 125` |
| `63` | `0163` | Engine reference torque | 2 | 0 | 65535 | Nm | `(A × 256) + B` |
| `64` | `0164` | Engine percent torque data | 5 | — | — | % | 5 bytes: idle, pt1, pt2, pt3, pt4 |
| `65` | `0165` | Auxiliary input / output supported | 2 | — | — | Bitmap | — |
| `66` | `0166` | Mass air flow sensor (advanced) | 5 | — | — | g/s | — |
| `67` | `0167` | Engine coolant temperature (dual) | 3 | — | — | °C | — |
| `68` | `0168` | Intake air temperature sensor | 7 | — | — | °C | — |
| `69` | `0169` | Commanded EGR and EGR error | 7 | — | — | %/% | — |
| `6A` | `016A` | Commanded Diesel intake air flow control | 5 | — | — | %/% | — |
| `6B` | `016B` | Exhaust gas recirculation temperature | 5 | — | — | °C | — |
| `6C` | `016C` | Commanded throttle actuator control | 5 | — | — | %/% | — |
| `6D` | `016D` | Fuel pressure control system | 6 | — | — | kPa | — |
| `6E` | `016E` | Injection pressure control system | 5 | — | — | kPa | — |
| `6F` | `016F` | Turbocharger compressor inlet pressure | 3 | — | — | kPa | — |
| `70` | `0170` | Boost pressure control | 9 | — | — | kPa | — |
| `71` | `0171` | Variable geometry turbo control | 5 | — | — | %/% | — |
| `72` | `0172` | Wastegate control | 5 | — | — | %/% | — |
| `73` | `0173` | Exhaust pressure | 5 | — | — | kPa | — |
| `74` | `0174` | Turbocharger RPM | 5 | — | — | rpm | — |
| `75` | `0175` | Turbocharger temperature A | 7 | — | — | °C | — |
| `76` | `0176` | Turbocharger temperature B | 7 | — | — | °C | — |
| `77` | `0177` | Charge air cooler temperature | 5 | — | — | °C | — |
| `78` | `0178` | Exhaust gas temperature Bank 1 | 9 | — | — | °C | `((A×256)+B)/10 − 40` |
| `79` | `0179` | Exhaust gas temperature Bank 2 | 9 | — | — | °C | `((A×256)+B)/10 − 40` |
| `7A` | `017A` | Diesel particulate filter (DPF) temperature | 9 | — | — | °C | — |
| `7B` | `017B` | Diesel particulate filter (DPF) pressure | 9 | — | — | kPa | — |
| `7C` | `017C` | Diesel particulate filter (DPF) differential pressure | 5 | — | — | kPa | — |
| `7D` | `017D` | NOx NTE control area status | 1 | — | — | Bitmap | — |
| `7E` | `017E` | PM NTE control area status | 1 | — | — | Bitmap | — |
| `7F` | `017F` | Engine run time (extended) | 13 | — | — | seconds | — |
| `80` | `0180` | PIDs supported [81-A0] | 4 | — | — | Bitmap | — |
| `81` | `0181` | Engine run time — AECD #1-#5 | 21 | — | — | seconds | — |
| `82` | `0182` | Engine run time — AECD #6-#10 | 21 | — | — | seconds | — |
| `83` | `0183` | NOx sensor | 5 | — | — | ppm | — |
| `84` | `0184` | Manifold surface temperature | 1 | — | — | °C | — |
| `85` | `0185` | NOx reagent system | 10 | — | — | — | — |
| `86` | `0186` | Particulate matter (PM) sensor | 5 | — | — | mg/m³ | — |
| `87` | `0187` | Intake manifold absolute pressure (B) | 5 | — | — | kPa | — |
| `88` | `0188` | SCR Induce System | 13 | — | — | — | — |
| `89` | `0189` | Run time — AECD #11-#15 | 21 | — | — | seconds | — |
| `8A` | `018A` | Run time — AECD #16-#20 | 21 | — | — | seconds | — |
| `8B` | `018B` | Diesel Aftertreatment | 7 | — | — | — | — |
| `8C` | `018C` | O2 Sensor (Wide Range) | 17 | — | — | µA | — |
| `8D` | `018D` | Throttle position G | 1 | — | — | % | — |
| `8E` | `018E` | Engine friction — percent torque | 1 | — | — | % | — |
| `90` | `0190` | PIDs supported [91-C0] | 4 | — | — | Bitmap | — |
| `91` | `0191` | PM sensor bank 1 & 2 | 5 | — | — | mg/m³ | — |
| `92` | `0192` | WWH-OBD vehicle OBD system info | 5 | — | — | — | — |
| `93` | `0193` | WWH-OBD vehicle OBD counters | 5 | — | — | — | — |
| `94` | `0194` | NOx warning and inducement | 12 | — | — | — | — |
| `98` | `0198` | Exhaust gas temperature sensor (wide) | 9 | — | — | °C | — |
| `99` | `0199` | Exhaust gas temperature sensor bank | 9 | — | — | °C | — |
| `9A` | `019A` | Hybrid/EV vehicle system data | 7 | — | — | — | — |
| `9B` | `019B` | Diesel exhaust fluid sensor data | 7 | — | — | — | — |
| `9C` | `019C` | O2 Sensor Data | 17 | — | — | — | — |
| `9D` | `019D` | Engine fuel rate (multi-fuel) | 9 | — | — | g/s | — |
| `9E` | `019E` | Engine exhaust flow rate | 5 | — | — | kg/h | — |
| `9F` | `019F` | Fuel system percentage use | 9 | — | — | % | — |
| `A0` | `01A0` | PIDs supported [A1-C0] | 4 | — | — | Bitmap | — |
| `A1` | `01A1` | NOx sensor corrected data | 5 | — | — | ppm | — |
| `A2` | `01A2` | Cylinder fuel rate | 1 | — | — | mg/stroke | — |
| `A3` | `01A3` | EVAP system vapor pressure (ultra) | 5 | — | — | Pa | — |
| `A4` | `01A4` | Transmission actual gear | 4 | — | — | ratio | — |
| `A5` | `01A5` | Diesel Exhaust Fluid Dosing | 4 | — | — | — | — |
| `A6` | `01A6` | Odometer | 4 | 0 | — | km | `((A×2^24)+(B×2^16)+(C×2^8)+D) / 10` |
| `A7` | `01A7` | NOx sensor concentration 3+4 | 9 | — | — | ppm | — |
| `A8` | `01A8` | NOx sensor corrected 3+4 | 9 | — | — | ppm | — |
| `A9` | `01A9` | ABS Disable State | 1 | — | — | Bitmap | — |

### Additional PIDs (Extracted from App — 0xAD → 0xFF)

| PID | Command | Description |
|-----|---------|-------------|
| `AD` | `01AD` | Exhaust pressure control |
| `B1` | `01B1` | Sensor calibration data |
| `B3` | `01B3` | Sensor calibration data 2 |
| `B8` | `01B8` | EV / Hybrid data |
| `B9` | `01B9` | EV / Hybrid data 2 |
| `BB` | `01BB` | EV / Hybrid data 3 |
| `BF` | `01BF` | Extended data |
| `C0-FF` | `01C0-01FF` | Manufacturer specific extended PIDs |

---

## 6. Mode 02 — Freeze Frame

**Command:** `02XX\r` (XX = PID number, same PIDs as Mode 01)  
**Response:** `42 XX FF AA BB...` (FF = freeze frame number, usually 00)

Freeze Frame is a snapshot of sensor values at the moment a DTC was triggered.  
The same PID numbers and formulas from Mode 01 apply.

**Found in application: 146 Freeze Frame PIDs**

---

## 7. Mode 03 — Read DTCs

### Command
```
03\r
```

### Response Format
```
43 XX YY ZZ WW ...
```
Every 2 bytes represent one DTC code.

### DTC Parse Algorithm

```
Byte 1 (XX): First nibble → Letter, Second nibble → First digit
Byte 2 (YY): Two more digits

Letter determination (Byte1 >> 6):
  0 = P (Powertrain)
  1 = C (Chassis)
  2 = B (Body)
  3 = U (Network/Communication)

Second character (Byte1 bits 4-5):
  0-3 range

Remaining: Byte1 bits 0-3 + Byte2

Example: Byte1=01, Byte2=33 → P0133
  01 = 0000 0001 → P + 0 + 1
  33 = 0011 0011 → 3 + 3
  Result: P0133 = O2 Sensor Circuit Slow Response
```

### If response is "NO DATA" → No stored DTCs (good news!)

---

## 8. Mode 04 — Clear DTCs

### Command
```
04\r
```

### Response
```
44    → Success (DTCs cleared, MIL turned off)
```

**WARNING:** This command:
- Clears all DTCs
- Turns off MIL (Check Engine) light
- Erases freeze frame data
- Resets readiness monitors
- Vehicle may not be "ready" until all monitors complete again

---

## 9. Mode 05 — O2 Sensor Monitor Test

**Command:** `05TT00\r` (TT = Test ID)  
**Response:** `45 TT 00 AA BB CC DD`

| Test ID | Description |
|---------|-------------|
| `01` | Rich to lean sensor threshold voltage |
| `02` | Lean to rich sensor threshold voltage |
| `03` | Low sensor voltage for switch time calculation |
| `04` | High sensor voltage for switch time calculation |
| `05` | Rich to lean sensor switch time |
| `06` | Lean to rich sensor switch time |
| `07` | Minimum sensor voltage for test cycle |
| `08` | Maximum sensor voltage for test cycle |
| `09` | Time between sensor transitions |

**NOTE:** Mode 05 is not supported on CAN protocols, use Mode 06 instead.

**Found in application: 142 PIDs**

---

## 10. Mode 06 — On-Board Monitoring

**Command:** `06TT\r` (TT = Test ID)  
**Response:** `46 TT ... `

This mode returns test results from on-board diagnostic monitors.  
On CAN vehicles, O2 sensor tests are also performed here.

| Test | Description |
|------|-------------|
| `00` | Supported test IDs |
| `01-FF` | Manufacturer and standard test results |

**Found in application: 141 PIDs**

---

## 11. Mode 09 — Vehicle Information

**Command:** `09XX\r`  
**Response:** `49 XX ...`

### Standard PIDs

| PID | Command | Info | Description |
|-----|---------|------|-------------|
| `00` | `0900` | Supported PIDs [01-20] | Supported PIDs |
| `01` | `0901` | VIN message count | Message count for VIN |
| `02` | `0902` | **VIN** | Vehicle Identification Number (17 char ASCII) |
| `03` | `0903` | Calibration ID message count | Calibration ID message count |
| `04` | `0904` | **Calibration ID** | ECU calibration identifier |
| `05` | `0905` | CVN message count | CVN message count |
| `06` | `0906` | **CVN** | Calibration Verification Number |
| `07` | `0907` | In-use performance tracking count | Performance tracking message count |
| `08` | `0908` | **In-use perf. tracking (spark)** | Spark ignition engine performance tracking |
| `09` | `0909` | ECU name message count | ECU name message count |
| `0A` | `090A` | **ECU name** | ECU name (ASCII) |
| `0B` | `090B` | **In-use perf. tracking (compression)** | Compression ignition engine perf. tracking |
| `0C` | `090C` | ESN message count | Engine serial number message count |
| `0D` | `090D` | **ESN** | Engine Serial Number |
| `0E` | `090E` | Exhaust regulation | Exhaust regulation |
| `0F` | `090F` | WWH-OBD ECU OBD system info | WWH-OBD ECU info |
| `10` | `0910` | WWH-OBD in-use monitoring | WWH-OBD usage monitoring |

### VIN Parse Example

```
Send: 0902\r
Response: 49 02 01 57 42 41 50 46 36 31 32 34 35 36 37 38 39 30
Parse: 49 02 = Mode 09 PID 02 response
       01 = message sequence
       57 42 41... = ASCII → "WBAPF612456789.0"
```

### Complete Mode 09 PID List (139 PIDs)

```
0901, 0902, 0903, 0904, 0906, 0907, 0908, 0909, 090A, 090B,
090D, 090E, 090F, 0910, 0914, 0915, 0916, 0917, 0918, 0919,
091A, 091C, 091F, 0920, 0921, 0923, 0924, 0925, 0926, 0927,
0928, 0929, 092E, 092F, 0930, 0931, 0932, 0934, 0935, 0936,
0937, 0939, 093B, 093D, 0943, 0944, 0947, 0948, 094C, 094E,
094F, 0950, 0951, 0952, 0953, 0954, 0955, 0956, 0957, 0959,
095A, 095F, 0961, 0963, 0964, 0965, 096A, 096B, 096C, 096E,
0970, 0977, 0978, 097B, 097D, 097E, 0980, 0983, 0985, 0987,
098A, 098B, 098F, 0990, 0991, 0992, 0993, 0996, 0997, 0998,
099D, 099E, 09A0, 09A1, 09A2, 09A5, 09A9, 09AB, 09AD, 09B1,
09B2, 09B3, 09B5, 09B8, 09B9, 09BA, 09BB, 09BD, 09BF, 09C1,
09C3, 09C4, 09C5, 09C8, 09C9, 09CD, 09CF, 09D0, 09D4, 09D6,
09D9, 09DA, 09DC, 09DE, 09E0, 09E4, 09E6, 09E7, 09EC, 09ED,
09F1, 09F2, 09F3, 09F9, 09FA, 09FB, 09FC, 09FE, 09FF
```

---

## 12. Mode 22 — Extended Diagnostics (UDS)

### UDS Service 0x22: Read Data By Identifier

**Command:** `22XXXX\r` (XXXX = Data Identifier / DID)  
**Response:** `62 XX XX AA BB CC...`

This mode is not part of the SAE J1979 standard. It uses the UDS (ISO 14229) protocol.  
Used to access manufacturer-specific data.

### Header Configuration Required

Extended PIDs may require setting the ECU header first:

```
ATSH 7E0\r       → Direct to Engine ECU
22F190\r         → Read VIN (UDS)
22F18C\r         → Read ECU serial number
22F187\r         → Read part number
```

### Known UDS DIDs (Standard)

| DID | Command | Description |
|-----|---------|-------------|
| `F186` | `22F186` | Active diagnostic session |
| `F187` | `22F187` | Spare part number |
| `F188` | `22F188` | ECU software version |
| `F189` | `22F189` | ECU calibration version |
| `F18A` | `22F18A` | System supplier ID |
| `F18B` | `22F18B` | ECU manufacturing date |
| `F18C` | `22F18C` | ECU serial number |
| `F190` | `22F190` | VIN (UDS method) |
| `F191` | `22F191` | ECU hardware version |
| `F194` | `22F194` | System name |
| `F195` | `22F195` | Software version |

### Manufacturer-Specific DIDs (Extracted from App)

| DID | Command | Likely Function |
|-----|---------|----------------|
| `2101` | `222101` | Hyundai/Kia — extended engine data |
| `2102` | `222102` | Hyundai/Kia — additional sensors |
| `2103` | `222103` | Hyundai/Kia — transmission data |
| `2104` | `222104` | Hyundai/Kia — A/C system |
| `2105` | `222105` | Hyundai/Kia — battery/charging |
| `2106` | `222106` | Hyundai/Kia — oxygen sensors |
| `2107` | `222107` | Hyundai/Kia — catalyst |
| `2108` | `222108` | Hyundai/Kia — EVAP system |
| `2200` | `222200` | VW/Audi — extended parameter 1 |
| `2201` | `222201` | VW/Audi — extended parameter 2 |
| `2202` | `222202` | VW/Audi — extended parameter 3 |
| `2204` | `222204` | VW/Audi — turbo boost pressure |
| `2205` | `222205` | VW/Audi — injection data |
| `F400` | `22F400` | Renault/Dacia — extended data |
| `F4xx` | `22F4xx` | WWH-OBD (ISO 27145-2) data |

### Hyundai/Kia 2101 Parse Structure

```
Send: 2101\r
Response: 6101 XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX

Byte  1-2:  Engine RPM                  (A×256+B) / 4
Byte  3:    Coolant Temperature         A − 40
Byte  4:    Intake Air Temperature      A − 40
Byte  5-6:  MAF Air Flow                (A×256+B) / 100
Byte  7:    Throttle Position           A × 100 / 255
Byte  8-9:  Vehicle Speed               (A×256+B)
Byte  10:   Battery Voltage             A / 10
Byte  11:   Engine Load                 A × 100 / 255
...
(total ~32 bytes of data)
```

### Nissan Consult II Specific PID Structure

```
Header: ATSH 7E0\r (or appropriate ECU address)

Nissan Consult II is a proprietary protocol.
Running it over ELM327 requires a special init sequence:
  - ATSH 7E0\r
  - Send Consult II init command
  - Request data

Nissan Consult 3 (newer vehicles, CAN 11-bit):
  - Runs over standard CAN protocol
  - Uses UDS Service 0x22
```

### VW TP 2.0 Protocol

```
VW TP 2.0 (Transport Protocol):
  - Proprietary transport protocol used in VW Group vehicles
  - Runs over CAN
  - Requires channel setup
  - Connection is opened to target ECU, then UDS commands are sent
  
Supported platforms:
  - MQB (Modular Querbaukasten) — 2012+
  - PQ26 — Polo, Ibiza etc.
  
Applicable vehicles:
  - VW Arteon, Atlas/Teramont, Tiguan II
  - Skoda Kodiaq, Karoq, Octavia A7, Superb MK3
  - Seat Arona, Ateca, Ibiza 5, Toledo 4
  - Audi A1 Mk2, A3 Mk3, TT Mk3, Q2, Q3 Mk2
```

---

## 13. DTC Code Structure

### DTC Format

```
XNNNN

X = Category letter:
  P = Powertrain (Engine/Transmission)
  C = Chassis
  B = Body
  U = Network/Communication

N = 4-digit number (hex)

First digit:
  0 = SAE/ISO standard (generic)
  1 = Manufacturer specific
  2 = SAE/ISO standard (extended)
  3 = Manufacturer specific + SAE
```

### Common DTC Code Ranges

| DTC | Description |
|-----|-------------|
| **P0100-P0199** | Air flow / MAF / MAP sensors |
| **P0200-P0299** | Fuel system / Injectors |
| **P0300-P0399** | Ignition system / Misfire |
| **P0400-P0499** | Emission control (EGR, EVAP) |
| **P0500-P0599** | Vehicle speed / Idle control |
| **P0600-P0699** | ECU internal faults |
| **P0700-P0799** | Transmission |
| **P0800-P0899** | Transmission (continued) |
| **P2000-P2999** | SAE extended codes |
| **P3000-P3499** | SAE reserved |
| **C0001-C0999** | Chassis — Generic |
| **B0001-B0999** | Body — Generic |
| **U0001-U0999** | Communication — Generic |
| **U0100-U0399** | CAN communication loss |

### DTC Parse Code (Pseudocode)

```python
def parse_dtc(byte1, byte2):
    # Determine letter
    letter_map = {0: 'P', 1: 'C', 2: 'B', 3: 'U'}
    letter = letter_map[(byte1 >> 6) & 0x03]
    
    # Second character
    second = (byte1 >> 4) & 0x03
    
    # Remaining digits
    third = byte1 & 0x0F
    fourth = (byte2 >> 4) & 0x0F
    fifth = byte2 & 0x0F
    
    return f"{letter}{second}{third:X}{fourth:X}{fifth:X}"

# Example: byte1=0x01, byte2=0x33 → P0133
```

---

## 14. Manufacturer-Specific Protocols

### Hyundai / Kia

| Profile | Year | Protocol | Specific PIDs |
|---------|------|----------|---------------|
| `HyundaiKia` | All | Auto | 2101-2108 |
| `HyundaiKia 2015+` | 2015+ | CAN 11-bit | Extended 2101+ |
| `HyundaiKiaCAN` | CAN vehicles | CAN | 2101-2108 |
| `HyundaiKia.CRDI` | Diesel | CAN | CRDI specific data |

### Nissan / Infiniti

| Profile | Year | Protocol | Specific |
|---------|------|----------|----------|
| `Nissan Consult II` | 1989-2010 | K-Line proprietary | Consult II specific |
| `Nissan Consult 3` | 2006+ | CAN 11-bit | UDS based |
| `Nissan_22` | — | UDS | Service 0x22 |
| `NissanCVT` | — | — | CVT transmission |
| `Nissan Leaf` | — | — | EV battery data |

### VW Group (VW, Audi, Skoda, Seat)

| Profile | Platform | Specific |
|---------|----------|----------|
| `VW TP 2.0` | MQB/PQ26 | Transport Protocol 2.0 |
| `VW Arteon` | MQB | 2017+ |
| `VW Tiguan II / FL` | MQB | 2016+ |
| `VW Polo 5 FL (6C)` | PQ26 | 2014-2018 |
| `Skoda Kodiaq/Karoq` | MQB | 2017+ |
| `Skoda Octavia A7` | MQB | 2013+ |
| `Skoda SuperB MK3 MY 2020+` | MQB | 2020+ |
| `Seat Arona/Ibiza 5 (KJ)` | MQB-A0 | 2017+ |
| `Seat Ateca` | MQB | 2016+ |
| `Audi A1 Mk2, A3 Mk3, TT Mk3, Q2, Q3 Mk2` | MQB | 2012+ |

### Toyota / Lexus

| Profile | Protocol |
|---------|----------|
| `TOYOTA` | CAN standard |
| `ToyotaJDM` | Japan domestic market specific |
| `ToyotaTPMS` | Tire pressure monitoring |

### Other Manufacturers

| Manufacturer | Profile | Specific Protocol |
|--------------|---------|-------------------|
| Renault/Dacia | `Renault/Nissan/Dacia` | Specific PIDs |
| Mitsubishi | `Mitsubishi MUT-2` | MUT-2 proprietary |
| Chevrolet/GM/Opel | `GM-Opel` | CAN + J1850 |
| Jeep/Chrysler/Dodge | `JeepChryslerDodge` | CAN UDS |
| Subaru | `SubaruDTC` | Custom DTC format |
| VAZ/Lada | `VAZ/Lada` | K-Line + custom |
| Alfa Romeo | `AlfaRomeo` | CAN |
| Jaguar/Land Rover | `JaguarLandRover` | CAN UDS |
| Porsche | `Porsche` | CAN UDS |

---

## 15. Connection Profiles

| # | Profile Name | Year | Protocol | Description |
|---|-------------|------|----------|-------------|
| 1 | `OBDII/EOBD` | All | Auto | General profile — try on any vehicle |
| 2 | `OBD-II / EOBD + CAN 11 bit` | 2004+ | CAN 500k | Modern vehicles |
| 3 | `OBD-II / EOBD Diesel + CAN 11 bit` | 2004+ | CAN 500k | Diesel vehicles |
| 4 | `OBD-II / EOBD ~1999-2008 K-Line/KWP + extra sensors` | 1999-2008 | KWP2000 | Older + extra sensors |
| 5 | `OBD-II / EOBD ~2006-2009 CAN + extra sensors` | 2006-2009 | CAN | Transition period + extra |
| 6 | `OBD-II / EOBD ~2010-2022 CAN + extra sensors` | 2010-2022 | CAN | Modern + extra sensors |
| 7 | `OBD-II / EOBD ~2016+ CAN + extra sensors` | 2016+ | CAN | Newest + extra sensors |
| 8 | `OBD-II / EOBD + CAN and extra PIDs` | 2006+ | CAN | CAN + extended PIDs |
| 9 | `OBD-II + SAE J1850` | <2004 | J1850 | Older Ford/GM |
| 10 | `OBDII/EOBD + SAE J1850 (Old Ford vehicles)` | Older | J1850 PWM | Older Ford trucks |
| 11 | `WWH-OBD + CAN and extra PIDs (2025-)` | 2025+ | ISO 27145-2 | Next generation |

---

## 16. Data Parse Formulas

### General Formula Types

```python
# TYPE 1: Direct value
value = A                              # Example: PID 0D (speed), PID 0B (MAP)

# TYPE 2: Offset (−40)
value = A - 40                         # Example: PID 05 (coolant), PID 0F (intake air)

# TYPE 3: Percentage (0-100%)
value = A * 100 / 255                  # Example: PID 04 (load), PID 11 (throttle)

# TYPE 4: Signed percentage (−100 ~ +99.2%)
value = (A - 128) * 100 / 128          # Example: PID 06-09 (fuel trim)

# TYPE 5: 16-bit direct
value = (A * 256) + B                  # Example: PID 1F (runtime)

# TYPE 6: 16-bit divided
value = ((A * 256) + B) / 4            # Example: PID 0C (RPM)
value = ((A * 256) + B) / 100          # Example: PID 10 (MAF)
value = ((A * 256) + B) / 1000         # Example: PID 42 (voltage)

# TYPE 7: 16-bit multiplied
value = ((A * 256) + B) * 0.079        # Example: PID 22 (fuel pressure)
value = ((A * 256) + B) * 10           # Example: PID 23 (diesel pressure)

# TYPE 8: Temperature (wide range)
value = ((A * 256) + B) / 10 - 40      # Example: PID 3C-3F (catalyst)

# TYPE 9: Lambda
value = ((A * 256) + B) * 2 / 65536    # Ratio
value = ((C * 256) + D) * 8 / 65536    # Voltage

# TYPE 10: Signed offset
value = (A / 2) - 64                    # Example: PID 0E (timing advance)
value = A - 125                          # Example: PID 61-62 (torque)

# TYPE 11: 32-bit (Odometer)
value = ((A<<24) + (B<<16) + (C<<8) + D) / 10   # PID A6
```

### Complete Parse Code (Python)

```python
class OBDParser:
    @staticmethod
    def parse_response(raw_response):
        """
        Parse ELM327 response.
        raw_response: string like "41 0C 1A F8"
        """
        # Clean spaces
        clean = raw_response.replace(" ", "").strip()
        
        # Remove ">" prompt
        clean = clean.replace(">", "")
        
        # Error check
        if clean in ("NODATA", "ERROR", "?", "UNABLETOCONNECT", "CANERROR"):
            return None
        
        # Split into bytes
        bytes_list = [int(clean[i:i+2], 16) for i in range(0, len(clean), 2)]
        
        if len(bytes_list) < 2:
            return None
        
        mode = bytes_list[0] - 0x40  # 41→01, 42→02 etc.
        pid = bytes_list[1]
        data = bytes_list[2:]
        
        return mode, pid, data
    
    @staticmethod
    def calculate_pid(pid, data):
        """Calculate PID value."""
        A = data[0] if len(data) > 0 else 0
        B = data[1] if len(data) > 1 else 0
        C = data[2] if len(data) > 2 else 0
        D = data[3] if len(data) > 3 else 0
        
        formulas = {
            0x04: lambda: A * 100.0 / 255,                    # Engine Load %
            0x05: lambda: A - 40,                               # Coolant Temp °C
            0x06: lambda: (A - 128) * 100.0 / 128,            # STFT Bank 1 %
            0x07: lambda: (A - 128) * 100.0 / 128,            # LTFT Bank 1 %
            0x08: lambda: (A - 128) * 100.0 / 128,            # STFT Bank 2 %
            0x09: lambda: (A - 128) * 100.0 / 128,            # LTFT Bank 2 %
            0x0A: lambda: A * 3,                                # Fuel Pressure kPa
            0x0B: lambda: A,                                    # MAP kPa
            0x0C: lambda: ((A * 256) + B) / 4.0,              # RPM
            0x0D: lambda: A,                                    # Speed km/h
            0x0E: lambda: (A / 2.0) - 64,                     # Timing ° BTDC
            0x0F: lambda: A - 40,                               # Intake Temp °C
            0x10: lambda: ((A * 256) + B) / 100.0,            # MAF g/s
            0x11: lambda: A * 100.0 / 255,                     # Throttle %
            0x1F: lambda: (A * 256) + B,                        # Runtime sec
            0x21: lambda: (A * 256) + B,                        # Distance w/MIL km
            0x22: lambda: ((A * 256) + B) * 0.079,            # Fuel Rail kPa
            0x23: lambda: ((A * 256) + B) * 10,               # Fuel Rail kPa (diesel)
            0x2C: lambda: A * 100.0 / 255,                     # EGR %
            0x2D: lambda: (A - 128) * 100.0 / 128,            # EGR Error %
            0x2E: lambda: A * 100.0 / 255,                     # EVAP Purge %
            0x2F: lambda: A * 100.0 / 255,                     # Fuel Level %
            0x30: lambda: A,                                    # Warm-ups count
            0x31: lambda: (A * 256) + B,                        # Distance since DTC km
            0x33: lambda: A,                                    # Baro Pressure kPa
            0x3C: lambda: ((A * 256) + B) / 10.0 - 40,        # Cat Temp B1S1 °C
            0x3D: lambda: ((A * 256) + B) / 10.0 - 40,        # Cat Temp B2S1 °C
            0x3E: lambda: ((A * 256) + B) / 10.0 - 40,        # Cat Temp B1S2 °C
            0x3F: lambda: ((A * 256) + B) / 10.0 - 40,        # Cat Temp B2S2 °C
            0x42: lambda: ((A * 256) + B) / 1000.0,           # Module Voltage V
            0x43: lambda: ((A * 256) + B) * 100.0 / 255,      # Absolute Load %
            0x44: lambda: ((A * 256) + B) / 32768.0,           # Lambda ratio
            0x45: lambda: A * 100.0 / 255,                     # Relative Throttle %
            0x46: lambda: A - 40,                               # Ambient Temp °C
            0x47: lambda: A * 100.0 / 255,                     # Throttle B %
            0x48: lambda: A * 100.0 / 255,                     # Throttle C %
            0x49: lambda: A * 100.0 / 255,                     # Accel Pedal D %
            0x4A: lambda: A * 100.0 / 255,                     # Accel Pedal E %
            0x4B: lambda: A * 100.0 / 255,                     # Accel Pedal F %
            0x4C: lambda: A * 100.0 / 255,                     # Cmd Throttle %
            0x4D: lambda: (A * 256) + B,                        # Time w/MIL min
            0x4E: lambda: (A * 256) + B,                        # Time since DTC min
            0x51: lambda: A,                                    # Fuel Type enum
            0x52: lambda: A * 100.0 / 255,                     # Ethanol %
            0x5A: lambda: A * 100.0 / 255,                     # Rel Accel Pedal %
            0x5B: lambda: A * 100.0 / 255,                     # Hybrid Battery %
            0x5C: lambda: A - 40,                               # Oil Temp °C
            0x5D: lambda: (((A * 256) + B) - 26880) / 128.0,  # Injection Timing °
            0x5E: lambda: ((A * 256) + B) / 20.0,             # Fuel Rate L/h
            0x61: lambda: A - 125,                              # Driver Demand %
            0x62: lambda: A - 125,                              # Actual Torque %
            0x63: lambda: (A * 256) + B,                        # Ref. Torque Nm
            0xA6: lambda: ((A<<24)+(B<<16)+(C<<8)+D) / 10.0,  # Odometer km
        }
        
        if pid in formulas:
            return formulas[pid]()
        return None
```

### DTC Parse Code

```python
def parse_dtcs(response):
    """
    Parse Mode 03 response.
    response: string like "43 01 33 02 34 00 00"
    """
    clean = response.replace(" ", "")
    if clean.startswith("43"):
        clean = clean[2:]  # Remove "43" prefix
    
    dtcs = []
    letter_map = {0: 'P', 1: 'C', 2: 'B', 3: 'U'}
    
    for i in range(0, len(clean), 4):
        if i + 4 > len(clean):
            break
        byte1 = int(clean[i:i+2], 16)
        byte2 = int(clean[i+2:i+4], 16)
        
        if byte1 == 0 and byte2 == 0:
            continue  # Empty slot
        
        letter = letter_map[(byte1 >> 6) & 0x03]
        second = (byte1 >> 4) & 0x03
        third = byte1 & 0x0F
        fourth = (byte2 >> 4) & 0x0F
        fifth = byte2 & 0x0F
        
        dtc = f"{letter}{second}{third:X}{fourth:X}{fifth:X}"
        dtcs.append(dtc)
    
    return dtcs
```

### ELM327 Connection Code

```python
import socket
import serial
import time

class ELM327Connection:
    """Connect to ELM327 adapter and send commands."""
    
    def __init__(self):
        self.conn = None
        self.protocol = None
    
    # ===== WiFi Connection =====
    def connect_wifi(self, ip="192.168.0.10", port=35000, timeout=10):
        self.conn = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.conn.settimeout(timeout)
        self.conn.connect((ip, port))
        self._init_elm()
    
    # ===== Bluetooth Connection =====
    def connect_bluetooth(self, port, baudrate=38400, timeout=10):
        """
        port: COM port (Windows) or /dev/rfcomm0 (Linux)
        """
        self.conn = serial.Serial(port, baudrate, timeout=timeout)
        self._init_elm()
    
    # ===== Init Sequence =====
    def _init_elm(self):
        """ELM327 initialization sequence."""
        responses = {}
        responses['reset'] = self.send_command("ATZ")       # Reset
        time.sleep(1)
        responses['echo'] = self.send_command("ATE0")       # Echo off
        responses['spaces'] = self.send_command("ATS0")      # Spaces off
        responses['linefeed'] = self.send_command("ATL0")    # Linefeed off
        responses['headers'] = self.send_command("ATH0")     # Headers off
        responses['longmsg'] = self.send_command("ATAL")     # Allow long msgs
        responses['protocol'] = self.send_command("ATSP0")   # Auto protocol
        responses['voltage'] = self.send_command("ATRV")     # Battery voltage
        return responses
    
    # ===== Send Command =====
    def send_command(self, cmd, timeout=5):
        """
        Send command to ELM327 and get response.
        """
        cmd_bytes = (cmd + "\r").encode('ascii')
        
        if isinstance(self.conn, socket.socket):
            self.conn.send(cmd_bytes)
            response = b""
            start = time.time()
            while time.time() - start < timeout:
                try:
                    chunk = self.conn.recv(1024)
                    response += chunk
                    if b">" in chunk:
                        break
                except socket.timeout:
                    break
        else:  # Serial
            self.conn.write(cmd_bytes)
            response = b""
            start = time.time()
            while time.time() - start < timeout:
                if self.conn.in_waiting:
                    chunk = self.conn.read(self.conn.in_waiting)
                    response += chunk
                    if b">" in chunk:
                        break
                time.sleep(0.01)
        
        return response.decode('ascii', errors='ignore').strip().replace(">","")
    
    # ===== Read PID =====
    def read_pid(self, mode, pid):
        """
        Read OBD PID.
        mode: 1-9 (int)
        pid: 0x00-0xFF (int)
        """
        cmd = f"{mode:02X}{pid:02X}"
        return self.send_command(cmd)
    
    # ===== Read Live Data =====
    def get_rpm(self):
        resp = self.read_pid(1, 0x0C)
        mode, pid, data = OBDParser.parse_response(resp)
        return ((data[0] * 256) + data[1]) / 4.0
    
    def get_speed(self):
        resp = self.read_pid(1, 0x0D)
        mode, pid, data = OBDParser.parse_response(resp)
        return data[0]
    
    def get_coolant_temp(self):
        resp = self.read_pid(1, 0x05)
        mode, pid, data = OBDParser.parse_response(resp)
        return data[0] - 40
    
    def get_fuel_level(self):
        resp = self.read_pid(1, 0x2F)
        mode, pid, data = OBDParser.parse_response(resp)
        return data[0] * 100.0 / 255
    
    def get_voltage(self):
        resp = self.send_command("ATRV")
        return float(resp.replace("V","").strip())
    
    def get_vin(self):
        resp = self.read_pid(9, 0x02)
        # ASCII parse
        mode, pid, data = OBDParser.parse_response(resp)
        return bytes(data[1:]).decode('ascii', errors='ignore')  # First byte = count
    
    def get_dtcs(self):
        resp = self.send_command("03")
        return parse_dtcs(resp)
    
    def clear_dtcs(self):
        return self.send_command("04")
    
    # ===== Find Supported PIDs =====
    def get_supported_pids(self):
        """Detect all PIDs supported by the vehicle."""
        supported = []
        
        for base_pid in [0x00, 0x20, 0x40, 0x60, 0x80, 0xA0, 0xC0, 0xE0]:
            resp = self.read_pid(1, base_pid)
            result = OBDParser.parse_response(resp)
            if result is None:
                break
            
            mode, pid, data = result
            bitmap = (data[0] << 24) | (data[1] << 16) | (data[2] << 8) | data[3]
            
            for i in range(32):
                if bitmap & (1 << (31 - i)):
                    supported.append(base_pid + i + 1)
        
        return supported
    
    # ===== Disconnect =====
    def disconnect(self):
        if self.conn:
            self.conn.close()
```

---

## 17. Sample Code Structures

### Command-Response Table

| # | Sent | Raw Response | Parse | Result |
|---|------|-------------|-------|--------|
| 1 | `0100\r` | `41 00 BE 3E B8 13` | Bitmap | PID 01-20 support |
| 2 | `0104\r` | `41 04 64` | 100×100/255 | **39.2% load** |
| 3 | `0105\r` | `41 05 7B` | 123−40 | **83°C coolant** |
| 4 | `010C\r` | `41 0C 1A F8` | (6912+248)/4 | **1790 RPM** |
| 5 | `010D\r` | `41 0D 3C` | 60 | **60 km/h** |
| 6 | `010F\r` | `41 0F 46` | 70−40 | **30°C intake** |
| 7 | `0110\r` | `41 10 01 FA` | 506/100 | **5.06 g/s MAF** |
| 8 | `0111\r` | `41 11 33` | 51×100/255 | **20% throttle** |
| 9 | `012F\r` | `41 2F 80` | 128×100/255 | **50.2% fuel** |
| 10 | `0142\r` | `41 42 30 D4` | 12500/1000 | **12.5V ECU** |
| 11 | `0146\r` | `41 46 37` | 55−40 | **15°C ambient** |
| 12 | `015C\r` | `41 5C 6E` | 110−40 | **70°C oil** |
| 13 | `01A6\r` | `41 A6 00 01 86 A0` | 100000/10 | **10000.0 km** |
| 14 | `ATRV` | `12.6V` | Direct | **12.6V battery** |
| 15 | `0902\r` | `49 02 01 57 42 41...` | ASCII | **VIN** |
| 16 | `03\r` | `43 01 33 00 00 00 00` | DTC parse | **P0133** |
| 17 | `04\r` | `44` | — | **DTCs cleared** |

### Fuel Type Enum Values (PID 51)

| Value | Fuel Type |
|-------|-----------|
| 0 | Not available |
| 1 | Gasoline |
| 2 | Methanol |
| 3 | Ethanol |
| 4 | Diesel |
| 5 | LPG |
| 6 | CNG |
| 7 | Propane |
| 8 | Electric |
| 9 | Bifuel — Gasoline |
| 10 | Bifuel — Methanol |
| 11 | Bifuel — Ethanol |
| 12 | Bifuel — LPG |
| 13 | Bifuel — CNG |
| 14 | Bifuel — Propane |
| 15 | Bifuel — Electric |
| 16 | Bifuel — Gasoline/Electric |
| 17 | Hybrid Gasoline |
| 18 | Hybrid Ethanol |
| 19 | Hybrid Diesel |
| 20 | Hybrid Electric |
| 21 | Hybrid Mixed |
| 22 | Hybrid Regenerative |
| 23 | Bifuel — Diesel |

### OBD Standards Enum (PID 1C)

| Value | Standard |
|-------|----------|
| 1 | OBD-II (CARB) |
| 2 | OBD (EPA) |
| 3 | OBD + OBD-II |
| 4 | OBD-I |
| 5 | Not OBD compliant |
| 6 | EOBD (Europe) |
| 7 | EOBD + OBD-II |
| 8 | EOBD + OBD |
| 9 | EOBD + OBD + OBD-II |
| 10 | JOBD (Japan) |
| 11 | JOBD + OBD-II |
| 12 | JOBD + EOBD |
| 13 | JOBD + EOBD + OBD-II |
| 17 | EMD (Engine Manufacturer Diagnostics) |
| 18 | EMD+ |
| 19 | HD OBD-C |
| 20 | HD OBD |
| 21 | WWH OBD |
| 23 | HD EOBD-I |
| 24 | HD EOBD-I N |
| 25 | HD EOBD-II |
| 26 | HD EOBD-II N |
| 28 | OBDBR-1 (Brazil) |
| 29 | OBDBR-2 |
| 30 | KOBD (Korea) |
| 31 | IOBD I (India) |
| 32 | IOBD II |
| 33 | HD EOBD-IV |

### Monitor Status Bit Masks (PID 01)

```
Byte A:
  Bit 7: MIL on/off (1=MIL on, Check Engine light is on)
  Bit 6-0: DTC count (0-127)

Byte B:
  Bit 0: Misfire test available
  Bit 1: Fuel system test available
  Bit 2: Components test available
  Bit 3: Reserved
  Bit 4: Misfire test incomplete
  Bit 5: Fuel system test incomplete
  Bit 6: Components test incomplete
  Bit 7: Reserved

Byte C (Gasoline):
  Bit 0: Catalyst test available
  Bit 1: Heated catalyst available
  Bit 2: Evap system available
  Bit 3: Secondary air available
  Bit 4: A/C refrigerant available
  Bit 5: O2 sensor available
  Bit 6: O2 sensor heater available
  Bit 7: EGR system available

Byte C (Diesel):
  Bit 0: NMHC catalyst available
  Bit 1: NOx/SCR monitor available
  Bit 2: Reserved
  Bit 3: Boost pressure available
  Bit 4: Reserved
  Bit 5: Exhaust gas sensor available
  Bit 6: PM filter monitoring available
  Bit 7: EGR/VVT system available

Byte D: Same structure — test incomplete bits
```

---

## Summary Statistics

| Category | Count |
|----------|-------|
| Mode 01 PIDs | **150** |
| Mode 02 PIDs (Freeze Frame) | **146** |
| Mode 03 PIDs (DTC) | **158** |
| Mode 05 PIDs (O2 Monitor) | **142** |
| Mode 06 PIDs (On-Board) | **141** |
| Mode 09 PIDs (Vehicle Info) | **139** |
| Mode 22 Extended PIDs (UDS) | **142** |
| **Total PIDs** | **~1018** |
| AT Commands | **30+** |
| Supported Protocols | **10** |
| Manufacturer Profiles | **25+** |
| Connection Profiles | **11** |

---

## Android Permissions (For Development)

Permissions required for your own application:

```xml
<!-- Bluetooth Classic -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />

<!-- WiFi -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

<!-- Location (required for BT scanning on Android 6+) -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- Background service -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />

<!-- Optional -->
<uses-feature android:name="android.hardware.bluetooth" android:required="false" />
<uses-feature android:name="android.hardware.location" android:required="false" />
```

---

*OBD-II / ELM327 Complete PID & Command Reference — Created: March 23, 2026*  
*Source: Car Scanner ELM OBD2 (com.ovz.carscanner) reverse engineering analysis*
