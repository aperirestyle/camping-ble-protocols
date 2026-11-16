# Khonsu E3500 BLE protocol

This unofficial specification describes observations from one physical E3500. The protocol was deduced from manufacturer-controller traffic and cross-checked by direct BLE writes and independent-client hardware tests. The hardware validation records are dated 2026-10-18. Each section identifies the limits of the observations.

## GATT profile

| Role | UUID | Operation |
| --- | --- | --- |
| Service | `6f6e7375-0001-0000-8b61-4d983bc456e1` | Discover after connecting |
| Client to lantern | `6f6e7375-0001-1000-bbc5-8ef45d9982fd` | Write Without Response |
| Lantern to client | `6f6e7375-0001-2000-aab2-77f2cb228ef9` | Subscribe to Notify. The characteristic also exposed Read in the observed profile. |
| Client Characteristic Configuration descriptor (CCCD) | `0x2902` | Enable notifications with `01 00` when managing the descriptor directly |

Discover services and characteristics by UUID. Use the BLE library's notification subscription API where available and wait for subscription setup before writing commands. Service advertisements are a discovery hint. The complete advertisement signature is unverified.

To start a session, send the `0xC0` status query after subscribing. Validate incoming frames and dispatch them by **response code**. For a setting change, use the corresponding application-layer notification and returned value to confirm the new state. Commands use Write Without Response at the ATT layer. Keep at most one setting change pending per connection so its confirmation is unambiguous.

## Frame format

The following byte sequences are hexadecimal. `N` is one unsigned byte that counts only the data bytes.

```text
Client write      A0  CMD  N  [DATA: N bytes]  12 34 56 78  CRC  AF
Lantern notify    CC  RSP  N  [DATA: N bytes]               CRC  BA
```

| Field | Client write | Lantern notification |
| --- | --- | --- |
| Header, offset 0 | `A0` | `CC` |
| Code, offset 1 | Command code | Response code |
| Data length, offset 2 | `N` | `N` |
| Data, starting at offset 3 | `N` bytes | `N` bytes |
| Constant after data | `12 34 56 78` | Absent |
| Checksum | One CRC-8 byte after the constant | One CRC-8 byte after the data |
| Footer | `AF` | `BA` |
| Total size | `N + 9` bytes | `N + 5` bytes |

The four constant bytes were present in every observed command. Their purpose is unknown. The client should write them in the listed order. The observed receive path parses complete frames from notifications and validates the header, footer, declared length, and CRC before interpreting the payload.

### CRC-8

| Parameter | Value |
| --- | --- |
| Width | 8 bits |
| Polynomial | `0x07` |
| Initial value | `0x00` |
| Bit order | Most significant bit first |
| Input/output reflection | Both false |
| Final XOR | `0x00` |
| Check value for ASCII `123456789` | `0xF4` |

The CRC covers every byte from the header through the last byte before the CRC. This includes the command constant on writes. To compute it, XOR each input byte into the accumulator, then perform eight left shifts, applying polynomial `0x07` whenever the accumulator's high bit was set before the shift. Keep the low eight bits after each step. The CRC byte and footer are outside the covered range.

## Commands

The payload column lists the bytes between the length field and the four-byte constant.

| Operation | Command | Payload | Response |
| --- | --- | --- | --- |
| Set power | `0x03` | `00` off or `01` on | `0x02` |
| Toggle flashing | `0x04` | `00` disable or `01` enable | `0x03` |
| Toggle breathing | `0x05` | `00` disable or `01` enable | `0x04` |
| Set brightness | `0x11` | One unsigned percentage byte, decimal 10–100 | `0x06` |
| Set color temperature | `0x12` | Unsigned 16-bit big-endian kelvin, decimal 1800–6500 | `0x07` |
| Set auto-off timer | `0x20` | `01 HH MM 00` | `0x08` |
| Cancel auto-off timer | `0x21` | Empty | `0x08` |
| Query status | `0xC0` | Empty | `0x00` status snapshot and `0x08` timer report observed. See the battery notes for `0x09`. |

### Complete write examples

| Operation | Frame |
| --- | --- |
| Power on | `A0 03 01 01 12 34 56 78 E0 AF` |
| Power off | `A0 03 01 00 12 34 56 78 82 AF` |
| Flashing on | `A0 04 01 01 12 34 56 78 F3 AF` |
| Flashing off | `A0 04 01 00 12 34 56 78 91 AF` |
| Breathing on | `A0 05 01 01 12 34 56 78 2C AF` |
| Breathing off | `A0 05 01 00 12 34 56 78 4E AF` |
| Brightness 50% | `A0 11 01 32 12 34 56 78 7A AF` |
| Color temperature 6500 K | `A0 12 02 19 64 12 34 56 78 56 AF` |
| Timer set to 1 minute | `A0 20 04 01 00 01 00 12 34 56 78 A2 AF` |
| Cancel timer | `A0 21 00 12 34 56 78 25 AF` |
| Query status | `A0 C0 00 12 34 56 78 AD AF` |

## Notifications

| Response | Data bytes | Meaning |
| --- | ---: | --- |
| `0x00` | 9 | Status snapshot. Zero-based data offset 0 is power and offset 8 is battery percentage. Offsets 1 through 7 are undocumented. |
| `0x02` | 1 | Power. `00` means off and `01` means on. |
| `0x03` | 1 | Flashing command response. `00` means disabled and `01` means enabled. |
| `0x04` | 1 | Breathing command response. `00` means disabled and `01` means enabled. |
| `0x06` | 1 | Applied brightness as an unsigned percentage. |
| `0x07` | 2 | Applied color temperature as unsigned 16-bit big-endian kelvin. |
| `0x08` | 5 | Timer state and remaining time. |
| `0x09` | 1 | Dedicated battery percentage report. |

Snapshot power uses the same `00` and `01` values as the direct power response. Both battery forms carry an unsigned integer and clients should accept values in the decimal range 0–100.

### Timer payloads

The `0x20` set payload is `01 HH MM 00`. Hours and minutes are ordinary unsigned binary values. The manufacturer controller offers `00:01` through `23:59`, and captured settings include 1 minute, 3 minutes, 2 hours, and 12 hours 12 minutes. Use `0x21` to cancel.

An active `0x08` payload is `01 01 HH MM SS`. Decode the time as `HH × 3600 + MM × 60 + SS`, using ranges of 0–23 hours and 0–59 minutes and seconds. The all-zero payload represents an inactive timer.

### Complete notification examples

| Meaning | Frame |
| --- | --- |
| Power on | `CC 02 01 01 86 BA` |
| Power off | `CC 02 01 00 81 BA` |
| Flashing enabled | `CC 03 01 01 ED BA` |
| Flashing disabled | `CC 03 01 00 EA BA` |
| Breathing enabled | `CC 04 01 01 FB BA` |
| Breathing disabled | `CC 04 01 00 FC BA` |
| Brightness 99% | `CC 06 01 63 04 BA` |
| Color temperature 4400 K | `CC 07 02 11 30 AF BA` |
| Active timer with 1 minute left | `CC 08 05 01 01 00 01 00 B1 BA` |
| Active timer with 58 seconds left | `CC 08 05 01 01 00 00 3A 02 BA` |
| Timer inactive | `CC 08 05 00 00 00 00 00 D0 BA` |
| Battery 67% | `CC 09 01 43 A3 BA` |

## Hardware-verified behavior

### Power, brightness, and color temperature

Power on and off produced the expected light output and direct responses. Hardware tests covered brightness at 10%, 50%, and 100% and color temperature at 1800 K, 4400 K, and 6500 K. Both logical power states were also observed in status snapshots.

### Lighting effects

Hardware tests confirmed direct switching between flashing and breathing. Either effect's disable command cleared the active effect, including when the other effect was running. Changing brightness during either effect preserved it. Changing color temperature returned the lantern to steady light at the selected temperature. An effect kept running across BLE disconnection. After a logical power-off and power-on using the power command, the previous active effect resumed.

### Timer

Hardware tests confirmed automatic shutoff with a 3-minute setting, cancellation of a 2-minute timer before shutoff, and a setting response for 12 hours 12 minutes. A running timer continued with the phone's Bluetooth disabled. After reconnection, a status query returned the lantern's remaining time.

Set the timer after confirming the lantern is on. The setting failed in the observed powered-off state.

### Battery

A battery reading returned during a status check matched the manufacturer controller in a follow-up comparison. A separate `0x09` report was also observed, and its percentage matched the controller reading at the time.

## Open questions

- Whether the tested lantern accepts a `23:59` setting and completes that countdown. This upper bound comes from the manufacturer controller's selectable range.
- Exact trigger and timing of the standalone `0x09` report relative to `0xC0`, behavior at low charge thresholds, and error notifications.
- Meanings of snapshot bytes 1 through 7, the command constant, and other timer flag values.
- Read behavior of the notification characteristic, the full advertisement signature, firmware variations, behavior after removing power from the lantern, and simultaneous connections to multiple lanterns.

## Minimal Python example

This uses only the standard library and emits a status query for use with a BLE client.

```python
def crc8(data: bytes) -> int:
    accumulator = 0
    for byte in data:
        accumulator ^= byte
        for _ in range(8):
            polynomial = 0x07 if accumulator & 0x80 else 0
            accumulator = ((accumulator << 1) ^ polynomial) & 0xFF
    return accumulator


command = bytes.fromhex("A0 C0 00 12 34 56 78")
query = command + bytes((crc8(command), 0xAF))

assert crc8(b"123456789") == 0xF4
assert query == bytes.fromhex("A0 C0 00 12 34 56 78 AD AF")
print(query.hex(" ").upper())
```
