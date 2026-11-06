# UART Protocol

All of this comes from `uart_command_handler` at `0x08031BFC` unless noted. Host software wraps these frames in a HID report as `[88, length, frame...]`, and the keyboard forwards the frame over UART3. The root README and `zoom-sync` cover the HID side.

## Link

- USART1 at 115200 baud, set up with no parity (firmware).
- Frames are at most 32 bytes. DMA fills the buffer and the idle line interrupt marks the end of a frame.
- Each frame gets a 1 byte reply after the handler returns, chosen from two SRAM bytes (`0x200000B0` for success, `0x200307B1` for failure).

## Commands

Commands start with `A5`, then a group byte, then a command byte.

### Group 00: control

| Frame | Action |
|---|---|
| `A5 00 10` | Toggle the backlight (PA3) |
| `A5 00 20` | Next page |
| `A5 00 21` | Move down |
| `A5 00 22` | Move up |
| `A5 00 FE` | [Reboot into the ROM bootloader](./bootloader.md#entry-from-stock-firmware), no reply |
| `A5 00 FF` | System reset, no reply |

### Group 01: data

| Frame | Action | Validation |
|---|---|---|
| `A5 01 10 YY MM DD hh mm ss` | Set time, year is 2000 + `YY` | `YY` 23-99, `MM` 1-12, `DD` 1-31, `hh` up to 24, `mm` and `ss` up to 60 |
| `A5 01 20 icon cur low high` | Set weather | None |
| `A5 01 40 cpu gpu dl_hi dl_lo` | Set system info | `cpu` and `gpu` under 121 |
| `A5 01 70 n` | Select one of 6 entries from a table at `0x08032B4C`, likely a page or screen. Undocumented elsewhere | `n` up to 5 |
| `A5 01 FF t` | Set the theme, see below | |

### Group 02: media

| Frame | Action |
|---|---|
| `A5 02 E0` | Delete the image, revert to the default |
| `A5 02 E1` | Delete the GIF, revert to the default |
| `A5 02 F0 ch` | Start an upload, `ch` 1 for image, 2 for GIF |
| `A5 02 D0 len` | Set the length, big endian u32 |
| `A5 02 F1 01` | End the upload |

### Theme

`A5 01 FF t` only checks whether `t` is zero:

- `t` = 0 erases the theme flag and loads theme A.
- Any other value writes `0x12121212` to the theme flag and loads theme B.

`zoom-sync` sends blue as 1 and pink as 2, and both take the same branch here. The difference, if any, has not been found.

`A5 01 FF` without an argument reads byte 3 of the receive buffer anyway. The handler does not clear that buffer between frames, so the result may depend on the previous frame (inferred).

## Upload

1. `A5 02 F0 ch`: shows the progress screen and starts the TIM6 timeout.
2. `A5 02 D0 len`: for an image, erases 4 KB sectors from `0x08610000`. For a GIF, writes `len` to `0x08603000` and erases the GIF flag.
3. Data packets, in order.
4. `A5 02 F1 01`: writes the present flag and, for a GIF, the length.

Data packet:

```
[index (u16, big endian)] [up to 24 bytes of data] [CRC (u32, big endian)]
```

- `index` must match the expected packet number, starting at 0.
- The CRC is checked over everything before it. The algorithm is in `zoom-sync` (`boards/zoom65v3/src/checksum.rs`) and the root README. It is CRC-32/MPEG-2 style, polynomial `0x04C11DB7`, initial value `0xFFFFFFFF`.
- Only the last packet of a GIF can be short. `zoom-sync` pads it so the CRC stays 32-bit aligned.
- A wrong index or CRC cancels the upload.
- For GIFs, `0x080068E4` also runs every 512 packets. It has not been decoded, but is likely an erase.

## Differences from the root README

| Topic | Root README | Firmware |
|---|---|---|
| Upload channels | Image 0, GIF 1 | Image 1, GIF 2. Other values are rejected. `zoom-sync` already uses 1 and 2 |
| Data packets | Data and CRC | A 2 byte index comes first. `zoom-sync` already sends it |
| Default GIF size | "111x111 ?" | Built-in default is 118x118. This does not establish a required size |
| Themes | Blue 1, pink 2 | Only zero or non-zero matters |

Also missing from the root README: `A5 00 10`, `A5 00 FE`, `A5 00 FF` and `A5 01 70`. The keyboard firmware already sends `A5 00 10` for its screen toggle key.
