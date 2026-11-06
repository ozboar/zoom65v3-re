# Screen Module Knowledge Base

Everything known about the stock LCD module on the Zoom65 v3: its hardware, ROM bootloader, firmware and UART protocol.

| Page | Contents |
|---|---|
| [hardware.md](./hardware.md) | MCU, pin map, SPI flash, ribbon wiring |
| [bootloader.md](./bootloader.md) | Entering the ROM bootloader, protocol, protection state, dumping |
| [firmware.md](./firmware.md) | Image layout, boot flow, peripherals, flash map, function index |
| [protocol.md](./protocol.md) | UART command table and media upload framing |
| [lcd.md](./lcd.md) | Panel init sequence and driver details |

All firmware addresses refer to the dump with this SHA-256:

```
d9d74c10c5bb65c10428b86c9ada3d390c9365be0ccb801debc23262ebf00cd1
```

## Evidence labels

Claims are tagged with where they come from:

- **measured**: observed on one keyboard through the probe firmware
- **firmware**: read from the disassembly, not exercised on hardware
- **board**: from board design documents, not checked on hardware
- **manual**: from the [AT32F403A/407 reference manual](https://arterychip.com/download/RM/RM_AT32F403A_407_EN_V2.06.pdf), which assumes that family is the right one
- **inferred**: a conclusion that fits the evidence but has not been proven

## Open questions

- Exact MCU part number. The chip marking has not been read.
- Whether sLib is enabled. The bootloader refuses to read `SLIB_STS0` and `SLIB_STS1`. The firmware never references the sLib unlock key or status registers, and all of main flash reads back.
- Why the first bootloader entry out of stock firmware fails. See [bootloader.md](./bootloader.md#entry-with-boot0).
- Whether `A5 00 FE` reaches the bootloader in practice. See [bootloader.md](./bootloader.md#entry-from-stock-firmware).
- W25Q128 contents. It has not been read, and the bootloader's access to the `0x08400000` window is untested.
- `EXT_FLASH_KEY` option bytes at `0x1FFFF820`, which decide whether external flash is encrypted. Not read yet.
- Panel resolution and exact controller part.
- How the blue and pink themes differ. See [protocol.md](./protocol.md#theme).
