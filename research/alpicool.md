# Alpicool-compatible refrigerators

## Model verification

| Model | BLE protocol verification | Outstanding checks |
| --- | --- | --- |
| Alpicool P15 | Hardware test pending | GATT profile, status query, and single-zone status layout |
| CarTour WD45 | Hardware test pending | GATT profile, status query, zone count, and meaning of any second-zone data |

The public references describe related refrigerators. Each model above needs its own hardware check before any compatibility or control behavior can be confirmed.

## Public references

| Upstream project | License | Relevant observations |
| --- | --- | --- |
| Gruni22, `alpicool_ha_ble` | MIT, as identified in the [upstream repository](https://github.com/Gruni22/alpicool_ha_ble) | Status payloads, split and concatenated notifications, and an absent-zone sentinel on a related single-zone unit. [Upstream implementation](https://github.com/Gruni22/alpicool_ha_ble/blob/main/custom_components/alpicool_ble/api.py) |
| klightspeed, `BrassMonkeyFridgeMonitor` | MIT, © 2023 Ben Peddell. [Upstream license](https://github.com/klightspeed/BrassMonkeyFridgeMonitor/blob/main/LICENSE) | GATT profile, query and response framing, and signed temperature fields. [Protocol notes](https://github.com/klightspeed/BrassMonkeyFridgeMonitor#technical) |

The upstream work provides a starting point for read-only tests. It documents service `0x1234`, command characteristic `0x1235`, notification characteristic `0x1236`, and status query `0x01`. Check the connected GATT profile before querying, then validate framing and checksum before reading temperatures. Confirm the temperature unit and the meaning of any second-zone field against the unit's display. Gruni22 documents a long response from a single-zone fridge, so payload length alone is insufficient to establish a second zone.

These notes summarize public protocol findings and link to the upstream implementations. They contain original prose. If upstream code is included in a later revision, retain its copyright notice and complete MIT permission notice.
