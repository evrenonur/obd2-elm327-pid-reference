# OBD-II / ELM327 Complete PID & Command Reference

> **Language / Dil:**  [English](OBD2_Complete_PID_Reference_EN.md) | [Türkçe](OBD2_Tam_PID_Referans.md)

Complete OBD-II PID reference for building your own car diagnostic application.

## What's Inside

| Section | Content |
|---------|---------|
| **ELM327 Communication** | WiFi/Bluetooth connection addresses, ports, UUID |
| **Init Sequence** | ATZ → ATE0 → ATS0 → ATL0 → ATH0 → ATAL → ATSP0 |
| **30+ AT Commands** | General, protocol, format, CAN bus, J1939 commands |
| **10 OBD Protocols** | CAN 11/29-bit, K-Line, KWP, J1850, J1939 |
| **150 Mode 01 PIDs** | Hex code, byte count, min/max, unit, **full formula** for each |
| **Mode 02-09** | Freeze Frame, DTC read/clear, O2 monitor, vehicle info |
| **Mode 22 (UDS)** | Extended DIDs, manufacturer-specific PIDs |
| **DTC Structure** | P/C/B/U codes, parse algorithm, common code ranges |
| **25+ Manufacturer Profiles** | Hyundai/Kia, VW Group, Nissan, Toyota, Renault, etc. |
| **Python Source Code** | Parser, DTC reader, connection handler, all PID calculations |
| **Command-Response Examples** | 17 real examples with raw data + calculated results |
| **Android Permissions** | Complete manifest XML for BT/WiFi OBD apps |

## Stats

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

## Quick Start

```python
from elm327 import ELM327Connection, OBDParser

conn = ELM327Connection()
conn.connect_wifi("192.168.0.10", 35000)

rpm = conn.get_rpm()        # 1790.0
speed = conn.get_speed()    # 60
temp = conn.get_coolant_temp()  # 83
vin = conn.get_vin()        # "WBAPF612456789"
dtcs = conn.get_dtcs()      # ["P0133"]

conn.disconnect()
```

## Standards

- SAE J1979 / ISO 15031-5
- ISO 14229 (UDS)
- ISO 15765-4 (CAN)
- ISO 27145-2 (WWH-OBD)

## License

MIT
