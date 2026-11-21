# Alpicool-compatible refrigerators

The [protocol document](../protocols/alpicool.md) describes the frames, commands and fields supported by the evidence below.

## Model verification

| Model | Verified on hardware | Outstanding checks |
| --- | --- | --- |
| Alpicool P15 | One zone. Manufacturer-app HCI shows GATT `0x1234`/`0x1235`/`0x1236`, QUERY, the 14-byte SET and `0x05` temperature writes. Independent-client tests covered −1 °C, −20 °C and +20 °C, overall power, MAX/ECO and L/M/H. QUERY continued on the existing BLE link with cooling off. | Actual L/M/H cutoff voltages, power-source or charging indication, and independent-client unlock and unit changes on hardware. |
| CarTour WD45 | Two zones with status queries and independent target changes verified by the user. `0x05` changed the left setpoint and `0x06` changed the right. Overall power, MAX/ECO and L/M/H worked. Switching overall power off ended the BLE link on the tested unit, including with the manufacturer's app. | Raw-packet confirmation of the `0x1A` right current-temperature mapping and all-field preservation for the 25-byte SET. Per-zone ON/OFF, the meaning of `0x1B`, and behavior across firmware versions are unconfirmed. |

### Packet evidence

The P15 manufacturer-app session contained 69 ATT Write Requests and 69 notifications. In each direction there were 56 QUERY frames, eight SET frames and five target-temperature frames. Every frame matched the stated length and unsigned 16-bit sum. The captured temperature writes covered 0–3 °C. The negative and boundary temperature results come from separate independent-client hardware tests. The manufacturer capture showed lock activation and both unit conversions. It contained no MAX/ECO transition.

The WD45 results come from physical observations and independent-client readback. The left side was tested at 1, 2 and 3 °C and the right side at −7, −6 and −5 °C. A separate raw WD45 manufacturer-app capture was unavailable for packet-level comparison.

## Public references

| Upstream project | License | Relevant observations |
| --- | --- | --- |
| Gruni22, `alpicool_ha_ble` | MIT, as identified in the [upstream repository](https://github.com/Gruni22/alpicool_ha_ble) | Frame construction, status parsing, split and concatenated notifications, the extended SET builder, and an absent-zone sentinel on a related single-zone unit. [Upstream implementation](https://github.com/Gruni22/alpicool_ha_ble/blob/main/custom_components/alpicool_ble/api.py) |
| klightspeed, `BrassMonkeyFridgeMonitor` | MIT, © 2023 Ben Peddell. [Upstream license](https://github.com/klightspeed/BrassMonkeyFridgeMonitor/blob/main/LICENSE) | GATT profile, QUERY and SET layouts, and signed temperature fields. [Protocol notes](https://github.com/klightspeed/BrassMonkeyFridgeMonitor#technical) |

Upstream observations describe other test units and firmware. The BrassMonkey temperature-write example uses a length byte that differs from the P15 rule and its shown checksum is inconsistent with the bytes as listed. The packet examples in this repository use the P15-verified framing rule. Gruni22 also documents a long response from a single-zone unit. Any proposed zone-detection rule should account for such responses and for the zero-filled extension observed on P15.

These notes summarize public protocol findings in original prose. If upstream code is included in a later revision, retain its copyright notice and complete MIT permission notice.
