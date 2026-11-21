# Alpicool-compatible refrigerator BLE protocol

These unofficial notes cover the tested Alpicool P15 and CarTour WD45 units. Packet-level observations come from a P15 manufacturer-app capture. Independent-client hardware tests establish the control behavior listed in [Model observations](#model-observations). The [verification notes](../research/alpicool.md) list upstream sources and open questions.

## GATT profile

| Role | Short UUID | Operation |
| --- | --- | --- |
| Service | `0x1234` | Discover after connecting |
| Client to refrigerator | `0x1235` | Write commands. The P15 capture used ATT Write Request. |
| Refrigerator to client | `0x1236` | Subscribe to notifications |

The full UUIDs follow the Bluetooth base UUID form `0000xxxx-0000-1000-8000-00805f9b34fb`. Subscribe to notifications before issuing the first status query. An independent client used this profile to query and control the WD45. Its characteristic properties and ATT write procedure have yet to be checked against a WD45 capture.

## Frame and checksum

```text
FE FE  LEN  CMD  [DATA]  SUM_HI SUM_LO
```

`LEN` is at frame offset 2 and counts the command byte, data bytes and two checksum bytes. `CMD` is at offset 3, and any data starts at offset 4. The total frame size is `3 + LEN`. Sum the unsigned bytes from the header through the end of the data, keep the low 16 bits and append them in big-endian order. All 69 writes and 69 notifications in the P15 capture matched this length and checksum rule.

## QUERY (0x01)

The request has no data. The P15 replied with command code `0x01` and 30 data bytes, giving `LEN = 0x21` and a 36-byte frame. The capture also contained 36-byte status notifications with code `0x02` after SET.

All offsets in the following tables are zero-based and measured from the first data byte after `CMD`.

| Offset | Field |
| --- | --- |
| `0x00` | Panel lock (0 unlocked, 1 locked) |
| `0x01` | Overall cooling power (0 off, 1 on) |
| `0x02` | Cooling mode (0 MAX, 1 ECO) |
| `0x03` | Low-voltage protection level (0 L, 1 M, 2 H) |
| `0x04` | Primary target temperature |
| `0x05`, `0x06` | Maximum and minimum selectable temperatures |
| `0x09` | Temperature unit (0 °C, 1 °F) |
| `0x0E` | Primary current temperature |
| `0x0F` | Battery gauge percentage |
| `0x10`, `0x11` | Input voltage in volts and tenths of a volt |
| `0x12` | Secondary target temperature in the upstream extended layout. Used for WD45 right-side setpoint readback in the independent-client tests. |
| `0x1A` | Secondary current temperature candidate from the upstream layout. The WD45 right-side sensor mapping awaits raw-packet verification. |
| `0x1B` | Extended status byte. Bit meanings are unconfirmed. |

Temperature fields use signed 8-bit two's complement in the reported unit. The gauge and voltage fields alone are insufficient to identify a power source or charging state.

## Target temperature (0x05 and 0x06)

The tested client sends one signed temperature byte, using the same unit as the current QUERY result. These commands use `LEN = 0x04` and are seven bytes long. For example, −20 encodes as `EC`, −7 as `F9`, and −1 as `FF`.

On P15, `0x05` sets the single zone. On the tested WD45, `0x05` sets the left zone and `0x06` sets the right zone. All five P15 manufacturer-app `0x05` writes received the same notification, even when the requested temperatures differed. Read the target from a subsequent QUERY to verify the change.

## SET (0x02)

The observed P15 SET carries 14 settings bytes, with `LEN = 0x11` and 20 bytes total. For the captured power and battery-protection changes, copying QUERY offsets `0x00`–`0x0D` and changing the relevant field reproduced the manufacturer's write byte for byte. A fresh QUERY confirmed that the other base settings were preserved.

| SET data offset | Setting |
| --- | --- |
| `0x00` | Panel lock |
| `0x01` | Overall cooling power |
| `0x02` | MAX/ECO |
| `0x03` | L/M/H low-voltage protection |
| `0x09` | °C/°F |

In the P15 manufacturer-app capture, unit changes also converted temperature-related values, including the target and limits, and adjusted temperature differences and affected calibration values. The capture showed locking. The independent-client unlock path was not verified on the hardware. The capture held the mode at ECO, and separate hardware tests confirmed switching between MAX and ECO.

An upstream implementation builds a 25-byte extended SET with the 14 base settings, eight secondary settings from QUERY offsets `0x12`–`0x19`, and three trailing zero bytes. Using the framing above gives `LEN = 0x1C` and 31 bytes total. WD45 overall power, mode and battery protection worked in hardware tests. The complete extended write and preservation of all settings bytes still lack WD45 raw-packet verification. [Upstream SET builder](https://github.com/Gruni22/alpicool_ha_ble/blob/main/custom_components/alpicool_ble/api.py)

## Packet examples

Each row gives a complete characteristic value. Calculated examples reproduce the tested client's encoding.

| Operation | Frame | Evidence |
| --- | --- | --- |
| QUERY | `FE FE 03 01 02 00` | P15 manufacturer-app write |
| P15 set +3 °C | `FE FE 04 05 03 02 08` | P15 manufacturer-app write |
| P15 temperature command notification | `FE FE 04 05 00 02 05` | P15 manufacturer-app capture, identical across five temperature writes |
| P15 set −1 °C | `FE FE 04 05 FF 03 04` | Calculated frame, temperature change verified on hardware |
| P15 set −20 °C | `FE FE 04 05 EC 02 F1` | Calculated frame, temperature change verified on hardware |
| P15 set +20 °C | `FE FE 04 05 14 02 19` | Calculated frame, temperature change verified on hardware |
| WD45 set right to −7 °C | `FE FE 04 06 F9 02 FF` | Calculated frame, right-side change verified on hardware |
| P15 switch cooling on | `FE FE 11 02 00 01 01 00 01 14 EC 02 00 00 00 00 FD 00 04 11` | P15 manufacturer-app write using that session's existing settings |

When constructing a SET, use fresh settings from the connected refrigerator. The example preserves the settings from one observed session.

## Model observations

- **Alpicool P15, one zone.** All 56 QUERY responses in the manufacturer-app capture had 30 data bytes. The expanded settings block at `0x12`–`0x19` and the current-temperature candidate at `0x1A` were zero throughout. Response length and zero-valued extension fields are insufficient to establish a second zone. The refrigerator continued answering QUERY on the same BLE connection after cooling was switched off.
- **CarTour WD45, two zones.** The user repeatedly changed the left setpoint with `0x05` and the right setpoint with `0x06`, and verified overall power, MAX/ECO and L/M/H operation on the unit. Switching overall power off ended the BLE connection on the tested unit. The user observed the same result with the manufacturer's app. Firmware-wide behavior and independent power control for each zone remain unconfirmed.
