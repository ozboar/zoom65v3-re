# ROM Bootloader

## Identity (measured)

Reachable over the ribbon UART, speaking the STM32 AN3155 style protocol.

| Query | Raw reply | Meaning |
|---|---|---|
| Sync `7F` | `79` | ACK |
| Get `00 FF` | `79 0B 32 00 01 02 11 21 31 44 63 73 82 92 79` | Version 3.2, command list below |
| Get Version `01 FE` | `79 32 47 03 79` | Version 3.2, option bytes `47 03` |
| Get ID `02 FD` | `79 01 04 30 79` | PID `0x0430` |

Commands offered: Get, Get Version, Get ID, Read Memory (`11`), Go (`21`), Write Memory (`31`), Extended Erase (`44`), Write Protect (`63`), Write Unprotect (`73`), Readout Protect (`82`), Readout Unprotect (`92`).

UART settings are 8E1 with automatic baud detection. 9600 through 230400 worked every time, and 460800 returned garbage once out of two tries.

USB DFU is not usable: PA11 and PA12 are not connected on the board (board design documents, not checked on hardware).

## Entry with BOOT0

1. Drive BOOT0 high.
2. Pulse NRST low.
3. Wait at least 4 seconds. 3 seconds failed every time, 4 and 5 worked every time.
4. Pulse NRST low again.
5. Wait 300 ms, then send sync.

The first reset out of stock firmware never answers, however long it waits, so the second reset is required (measured). BOOT1 is not the cause, since the bootloader does answer after the second reset.

The likely cause is PA3, which is both USART2 RX for the bootloader and the backlight line. [Artery's FAQ](https://www.arterychip.com/download/FAQ/FAQ0122_Notes_on_AT32_Bootloader_EN_V2.0.0.pdf) says activity on an unused bootloader RX pin can lock it onto the wrong interface. This is inferred, not proven.

## Entry from stock firmware

Stock firmware has a software path that does not need BOOT0 (firmware, not yet tried on hardware):

1. `A5 00 FE` writes `0x5AA5` to backup register 1 (`0x40006C04`), then requests a system reset.
2. Early in boot, `0x0800191C` checks that register. If the flag is set, it clears the flag, loads the stack pointer from `0x1FFFB000` and jumps to the address at `0x1FFFB004`, the ROM bootloader.

The screen never replies to this command, because it resets first. `uart3_command` in the QMK firmware blocks on `sdGet` waiting for a reply, so sending this through the normal HID proxy would hang the keyboard until it is power cycled. Use a path with a read timeout.

## Protection state (measured)

Read through the bootloader:

```
0x1FFFF800: A5 5A FF 00 FF 00 FF 00 FF 00 FF 00 FF 00 FF 00
```

| Item | Value | Meaning |
|---|---|---|
| FAP | `A5` / `5A` | Access protection disabled, main flash is readable |
| SSB | `FF` / `00` | Default system configuration |
| Data0, Data1 | `FF` / `00` | Unused |
| EPP0 to EPP3 | `FF` / `00` | Erased state, so erase/program protection looks unset |

The bootloader refuses reads of the flash registers at `0x40022000`, so `FLASH_USD`, `SLIB_STS0` and `SLIB_STS1` are unavailable this way.

## Hazards (manual)

- Disabling access protection erases main flash. It is currently off, so this only matters if it gets turned on.
- Erasing the option bytes without writing `0xA5` back turns access protection on.
- sLib regions cannot be read or erased without their password, and mass erase does not clear them.
- Readout Unprotect (`92`) and Readout Protect (`82`) change the protection state. The probe firmware cannot send them.

## Dumping

The dump was made by a probe firmware on the keyboard MCU, driven from a host script over raw HID. The probe can only send sync, Get, Get Version, Get ID and Read Memory, so it cannot erase or write the screen MCU.

The host reads 24 bytes per request and read the whole region twice, saving it only because both reads matched. A full 1024 KB dump took about 9 minutes at 115200 baud, or 156 reads per second.

Flashing custom firmware needs a write-capable build, which does not exist yet.
