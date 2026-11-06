# LCD Panel

## Controller

The `FE EF` unlock pair and the register layout point to the GalaxyCore GC9xxx family, and the sequence resembles GC9107 setups (inferred, from memory).

## Display area

- Stock firmware registers a 128x128 display with LVGL at `0x0802A068`, with no window offset in its flush path (firmware).
- The panel is mounted rotated 90 degrees from the controller: with the stock `36 C0` setting, controller rows run left to right on screen (measured).
- Stock firmware handles this in software: its display setup appears to set LVGL's `sw_rotate` and a 90 degree `rotated` value (inferred, from the LVGL struct layout as remembered).
- The visible area is 118x118: controller columns 10-127 and rows 10-127 (measured, a border drawn on these edges displays correctly).
- Draw the rest of the 128x128 area black, so nothing shows around the visible area.
- Screen position `(sx, sy)`, from the top left, is controller column `10 + sy`, row `127 - sx`.

So 128x128 is the addressable area, not the visible one. Stock art matches this: the background at `0x08566B00` is 118x118.

## Interface (firmware)

- 4-wire SPI on SPI1, write only: `CLK` PA5, `SDA` PA7, `RS` PA6, `CS` PB2 and `RES` PB0.
- `RS` is low for commands and high for data.
- The code polls SPI busy between bytes and raises `CS` after each byte.
- Pixel format is RGB565 (`3A 05`).

## Backlight (firmware)

`BLK` on PA3 is a plain GPIO output. It stays off during init and turns on after the reset pulse. `A5 00 10` toggles it. No brightness PWM was found on this pin.

## Reset (firmware)

`RES` high for 10 ms, low for 10 ms, then high and wait 120 ms.

## Init sequence (firmware)

From `lcd_init` at `0x080007F4`, with the first byte of each line sent as a command and the rest as data. This sequence has been run on the panel and works (measured).

```
FE
EF
B0 C0
B1 80
B2 27
B3 13
B6 19
B7 05
AC C8
AB 0F
3A 05
B4 04
A8 08
B8 08
EA 02
E8 2A
E9 47
E7 5F
C6 21
C7 15
F0 1D 38 09 4D 92 2F 35 52 1E 0C 04 12 14 1F
F1 16 40 1C 54 A9 2D 2E 56 10 0D 0C 1A 14 1E
F4 00 00 FF
BA FF FF
36 C0
11
(wait 120 ms)
29
```

`F0` and `F1` look like gamma tables (inferred). `36 C0` sets the memory access order, `11` exits sleep and `29` turns the display on.

## Drawing (firmware)

`lcd_set_window` at `0x08000720` sends `2A` and `2B` with their coordinates as 16-bit values through `0x08001628`, then `2C` to start pixel data. The flush callback at `0x08006708` then sends the pixels by DMA, through two 8 KB LVGL draw buffers.
