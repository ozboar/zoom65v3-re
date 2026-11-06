# Hardware

## MCU

Artery AT32 in a 48-pin package with an exposed pad. The exact part number is not known.

| Property | Value | Source |
|---|---|---|
| Flash size | 1024 KB (`F_SIZE` = `0x0400`) | measured |
| Bootloader chip ID | `0x0430` | measured |
| Crystal | 8 MHz | board |
| SRAM | Stack starts about 218 KB in, so likely the 224 KB configuration | inferred |
| Family | AT32F403A/407, from flash size and SPIM pin use | inferred |

## Pin map

| Pin | Net | Use | Source |
|---|---|---|---|
| PA3 | `BLK` | Backlight on/off, also USART2 RX for the bootloader | board, firmware |
| PA5 | `CLK` | LCD SPI1 SCK | board, firmware |
| PA6 | `RS` | LCD data/command | board, firmware |
| PA7 | `SDA` | LCD SPI1 MOSI | board, firmware |
| PB0 | `RES` | LCD reset | board, firmware |
| PB2 | `CS` | LCD chip select, also BOOT1 | board, firmware |
| PA8 | `SPIM_CS` | W25Q128 chip select | board |
| PB1 | `SPIM_CLK` | W25Q128 clock | board |
| PB10, PB11 | `IO_0`, `IO_1` | W25Q128 data | board |
| PB7, PB6 | `IO_2`, `IO_3` | W25Q128 data | board |
| PA9 | `IAP_TX` | USART1 TX to the keyboard | board, firmware |
| PA10 | `IAP_RX` | USART1 RX from the keyboard | board, firmware |
| PA13, PA14 | `SWDIO`, `SWCLK` | SWD, on 4-pin header `H1` | board |
| BOOT0 | `BOOT0` | 10k pulldown, driven from the ribbon | board, measured |
| NRST | `RESET` | 10k pullup, driven from the ribbon | board, measured |

## Ribbon to keyboard MCU

Pin numbers follow the [root README](../../README.md#ribbon-cable).

| Ribbon pin | Keyboard MCU | Screen MCU | Source |
|---|---|---|---|
| 6 | PA8 | BOOT0 | measured |
| 7 | PB13 | NRST | measured |
| 8 | PC11 (UART3 RX) | PA9 (USART1 TX) | measured |
| 9 | PC10 (UART3 TX) | PA10 (USART1 RX) | measured |
| 10 | 3.3 V | VCC | measured |

Measured means the probe confirmed behavior through the line: both BOOT0 and NRST hold against the keyboard's internal pulls, holding NRST low floats screen TX, and raising BOOT0 changes the boot path.

## External flash

SPI flash on the SPIM interface, memory mapped at `0x08400000`. See the [flash map](./firmware.md#external-flash-map).

The board design lists a W25Q128 (16 MB), but this keyboard's chip behaves like a 4 MB part (measured). Contents repeat every 4 MB across the 16 MB window, at `0x08400000`, `0x08800000`, `0x08C00000` and `0x09000000`. The sector maps match, and so did 17 sampled data blocks at every repeat. The chip marking has not been read.

Only use the first 4 MB. Anything past it reads, and would write, the same physical data. A dump of the whole window contains four copies.

## LCD connector

13-pin connector `U5`: pin 3 `SDA`, 4 `CLK`, 5 `RS`, 6 `RES`, 7 `CS`, 10 VCC, 12 `BLK`, with 8, 11 and 13 grounded. Pins 1, 2 and 9 are not connected.
