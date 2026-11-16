# Camping BLE Protocols

Platform-independent BLE protocol notes for camping equipment. The documents give developers the GATT profiles, byte layouts, and evidence needed to build independent clients in Swift, Python, or other languages.

This unofficial project is maintained independently of the equipment manufacturers.

| Device | Document | Evidence |
| --- | --- | --- |
| Khonsu E3500 lantern | [BLE protocol](protocols/khonsu-e3500.md) | Hardware observations and validated packet examples from one lantern |
| Alpicool P15 and CarTour WD45 refrigerators | [Upstream references and verification status](research/alpicool.md) | Public research on related devices. Compatibility with these two models remains unverified. |

For the E3500, start by subscribing to notifications and sending the status query. The [packet example](protocols/khonsu-e3500.md#minimal-python-example) constructs that query without opening a Bluetooth connection.

## License

The original documentation and examples in this repository are available under the [MIT License](LICENSE). The linked upstream projects are distributed under their respective licenses.
