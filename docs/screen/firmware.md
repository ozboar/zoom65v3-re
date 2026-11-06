# Stock Firmware

Addresses refer to the dump identified in the [index](./README.md). Analysis was done with Ghidra.

## Image layout

The image is loaded at `0x08000000`.

| Offset | Size | Contents |
|---|---|---|
| `0x000000` to `0x03C47F` | 241 KB | Code and data |
| `0x03C480` | 12650 bytes | Default GIF, 118x118 |
| `0x03F9BC` | 36300 bytes | Default image, 110x110 |
| `0x048888` to `0x05C787` | 80 KB | More code and data, not decoded |
| `0x05C788` to `0x0FFFFF` | 654 KB | Blank, `0xFF` |

The firmware uses about 379 KB of the 1024 KB. The vectors give an initial stack pointer of `0x20036970` and a reset handler at `0x08000224`.

The UI looks like LVGL (inferred): there are no name strings, but the code has LVGL's blending and object patterns, and the image holds its default `Option 1\nOption 2\nOption 3` widget text. Images use 3 bytes per pixel, RGB565 plus an alpha byte, which matches LVGL's true color alpha format.

## Boot flow

1. `0x08000224` reset handler enters the C runtime.
2. `0x0802EE88` main calls `0x0800191C`, which jumps to the ROM bootloader if [the backup register flag](./bootloader.md#entry-from-stock-firmware) is set.
3. Clocks, then USART1 at 115200 baud (`0x0803B064`), then SPIM (`0x080336C8`).
4. The RTC is passed a default time of 2024-06-12 18:00:00.
5. LVGL and display setup, then the theme is loaded by checking `0x08602000`.
6. The stored GIF and image are selected, falling back to the built-in defaults.
7. The main loop runs the UI task (`0x0802E1F4`) and waits 5 ms, forever.

## Interrupts

| IRQ | Handler | Source | Use |
|---|---|---|---|
| 3 | `0x08001698` | RTC | Clock tick, redraws the time |
| 13 | `0x080006E0` | DMA1 channel 3 | LCD transfer complete, raises `CS` |
| 25 | `0x080017F4` | TIM1 update | Not decoded |
| 37 | `0x080018A8` | USART1 | Idle line, runs the command handler |
| 54 | `0x08001828` | TIM6 | Upload timeout |

All other vectors point to the default handler at `0x0800023E`.

## Peripherals

| Peripheral | Base | Use |
|---|---|---|
| USART1 | `0x40013800` | Keyboard link |
| DMA1 channel 4 | `0x40020044` | USART1 TX |
| DMA1 channel 5 | `0x40020058` | USART1 RX, 32 byte buffer at `0x20000486` |
| SPI1 | `0x40013000` | LCD |
| TIM6 | `0x40001000` | Upload timeout |
| RTC | `0x40002800` | Clock |
| BPR | `0x40006C00` | Bootloader flag |
| FLASH | `0x40022000` | External flash programming |

## External flash map

The firmware selects flash type 2 (`FLASH_SELECT`) and sets `FLASH_DA` to 0. Per the manual, `FLASH_DA` is the size of the encrypted range starting at `0x08400000`, so this should mean no encrypted range. `EXT_FLASH_KEY` has not been read.

| Address | W25Q128 offset | Contents |
|---|---|---|
| `0x084F9358` to `0x0857CD28` | `0x0F9358` to `0x17CD28` | Referenced by code as constants, likely UI assets (inferred) |
| `0x08600000` | `0x200000` | Image present flag, `0x11111111` when set |
| `0x08601000` | `0x201000` | GIF present flag, `0x11111111` when set |
| `0x08602000` | `0x202000` | Theme flag, `0x12121212` or erased |
| `0x08603000` | `0x203000` | GIF length in bytes |
| `0x08610000` | `0x210000` | Uploaded image |
| `0x08620000` | `0x220000` | Uploaded GIF |

## Function index

Names are descriptive and were assigned during analysis.

| Address | Name | Notes |
|---|---|---|
| `0x08000224` | `Reset_Handler` | |
| `0x08000728` | `lcd_set_window` | Sends `2A`, `2B`, `2C` |
| `0x080007F4` | `lcd_init` | See [lcd.md](./lcd.md) |
| `0x0800191C` | `boot_check` | Jumps to the bootloader if flagged |
| `0x08001964` | `system_reset` | Writes `AIRCR` |
| `0x08005410` | `reboot_to_bootloader` | Sets the flag, then resets |
| `0x0800533C` | `bootloader_flag_take` | Reads and clears the flag |
| `0x08005E70` | `upload_crc_check` | Checks a packet's CRC |
| `0x0800659C` | `gif_source_select` | Uploaded GIF or default |
| `0x08006608` | `image_source_select` | Uploaded image or default |
| `0x0800AD38` | `extflash_erase_sector` | |
| `0x0800B4C4` | `flash_program_word` | |
| `0x08031BFC` | `uart_command_handler` | See [protocol.md](./protocol.md) |
| `0x08031758` | `theme_a_load` | Used when the theme flag is erased |
| `0x08031948` | `theme_b_load` | Used when the theme flag is set |
| `0x080336C8` | `spim_init` | |
| `0x0802EE88` | `main` | |
| `0x0803B064` | `usart1_init` | Takes the baud rate |
