# Wireless Module Protocol

How the stock keyboard firmware talks to the wireless module, traced from `firmware/ZOOM65V3_ANSI_v1.27.bin` with Ghidra. Addresses are in that image. The ISO image was not analyzed.

The stock firmware is built on ChibiOS, like QMK. The module protocol matches the vendor's open source driver (`keyboards/westberry/wireless` on the vendor QMK fork) for everything stock uses, but stock uses only part of that driver's command set.

## Pairing the 2.4 GHz receiver

1. Plug the receiver in.
2. Hold the 2.4 GHz key (Fn+R) for 3 seconds. The key blinks fast while the module is pairing.
3. Unplug the receiver and plug it back in while the key is still blinking.

Observed on hardware:

- Pairing completed once the receiver was replugged during step 2, on both stock and the open source QMK port, which sends the same commands.
- The receiver enumerates as `0c45:fefe` "2.4G Dongle" both before and after pairing, so its USB identity says nothing about whether it is paired.
- Pairing the receiver with `51` alone kept the Bluetooth pairings.
- Holding the key and replugging a receiver that is already paired may only reconnect it, so a test against an already paired receiver doesn't show whether a real pairing works.

### Commands from the vendor driver

These were tried on the open source port, which otherwise sends what stock sends.

| Sent before `51` | Result |
|---|---|
| `52` | Did not pair, even with the replug. The receiver no longer reconnected afterwards, and the next plain `51` pairing worked. Bluetooth had to be paired again after each `52` |
| `30`, `AB`, `AC`, `AD`, `52` | Same as `52` alone |
| `AB`, `AC`, ASCII or UTF-16 | Reconnected. The receiver still showed as `0c45:fefe` "2.4G Dongle" |
| `AB`, `AC` in UTF-16, `AD` | Same as above |

| `52`, wait 2 s, `51` | Paired, but did not connect until 2.4 GHz was selected again |
| `52`, wait 2 s, `30`, `51` | Paired and connected |
| `52`, wait 2 s, `30`, `AB`, `AC`, `AD`, `51` | Paired. Still `0c45:fefe` "2.4G Dongle" |
| `30`, `AB`, `AC`, `AD`, `52`, wait 2 s, `30`, `51` (vendor order) | Same as above |

So `52` clears the receiver pairing, the Bluetooth pairings and the selected channel, which is why a `51` straight after it fails. Sending `30` again before `51` fixes it. The receiver kept its generic identity through real pairings in both orders, so it does not take names or IDs from this module, which fits stock never sending them. In these two tests the names were ASCII or UTF-16 depending on the module's version reply, which was never confirmed to arrive. A reconnect test with forced UTF-16 names didn't change the identity either.

Each Bluetooth re-pair also used a different Bluetooth address, so the host collects an old entry for each one.

## Physical link

| Item | Value |
|---|---|
| Pins | `A9` TX, `A10` RX |
| Peripheral | UART1 at `0x40003800`, ChibiOS serial driver SD1 |
| Settings | 115200 baud, 8N1 |

## Frames to the module

Every frame is a type byte, its payload, then a checksum: the 8-bit sum of all bytes before it. Built by `0x0800a918` from a queue filled by `0x0800a7b8`.

| Type | Payload | Use |
|---|---|---|
| `A1` | 8 bytes | Keyboard report |
| `A2` | 14 bytes | NKRO report |
| `A3` | 2 bytes | Consumer usage, little endian |
| `A4` | 1 byte | System usage |
| `A5` | 1 byte | Named "fn" in the vendor driver, purpose unknown |
| `A6` | 1 byte | Device control, see below |
| `A8` | 5 bytes | Mouse report |
| `A9` | length, then the name | Bluetooth name. The length comes from `strlen`, so no terminator is sent |

The keyboard also acknowledges every frame from the module, except a resend request, with the 3 bytes `61 0D 0A` (`a\r\n`), sent with no checksum.

### Device control commands

Stock sends only these. Sent by `0x0800a900`.

| Command | Meaning |
|---|---|
| `11` | Use USB |
| `30` | Use 2.4 GHz |
| `31`, `32`, `33` | Use Bluetooth 1, 2 or 3 |
| `51` | Pair the current device |
| `53` | Query the battery level |
| `55` | Enable sleep after 30 minutes in Bluetooth mode, per the vendor driver |

The vendor driver defines more, and stock uses none of them. That includes the receiver's manufacturer and product strings (`AB`, `AC`), its USB IDs (`AD`), raw HID (`AF`), clearing pairings (`52`), forced pairing (`62`, `63`) and reading the module's firmware version (`70`).

## Frames from the module

Most frames are 3 bytes: type, value, and the 8-bit sum of the two. Parsed by `0x0800a918` through a 3-byte sliding window, so it resynchronizes after a corrupt byte.

| Frame | Meaning |
|---|---|
| `61 0D 0A` (`a\r\n`) | Acknowledges the frame in flight |
| `5A` value | Host LED state. Accepted only when the top 5 bits are clear |
| `5B` value | Event, see below |
| `5C` value | Battery percent |
| `5B 43 9E` | Drop the frame in flight without resending |

### Events

| Value | Meaning | Stock response |
|---|---|---|
| `21` | Battery low | Shows 10% |
| `22` | Battery about to cut out | Shows 1%, flags it critical |
| `23` | Battery back to normal | Clears the flags, sends `53` |
| `31` | Pairing started | Sets a pairing flag |
| `32` | Connected | Sets connected |
| `33` | Disconnected | Clears connected |
| `34`, `35` | Unknown, named "done" and "reconnect" in the vendor driver | Sets a flag |
| `36` | Pairing rejected | Sends `51` again |
| `42` | Resend the frame in flight | Resends. This event is not acknowledged |

## Flow control

Only one frame is in flight at a time. After about 6 ms without an acknowledgement, the frame is sent again. After 4 tries it is dropped. Timings assume the firmware's millisecond timer.

## Devices and keys

Stock numbers devices 0 USB, 1-3 Bluetooth 1-3, and 4 for 2.4 GHz. It has no Bluetooth 4 or 5.

| Keycode | Key |
|---|---|
| `7E00`-`7E02` | Bluetooth 1-3 |
| `7E03` | 2.4 GHz |
| `7E04` | USB |
| `7E0B` | Battery level, sends `53` on press |

### Selecting a device

Pressing a device key calls the device switch at `0x0800066c`, but only when that device isn't already selected. The switch:

1. Moves the USB connection over when changing between USB and wireless.
2. Saves the device, and clears the connected state.
3. Sends the device's command, `11` or `30`-`33`.

### Pairing

The key handler records when each device key goes down. A loop in the wireless task, `0x08000918`, calls `0x0800077c` once a key has been held for 3 seconds. That callback only acts if the held key's device is already the current one. It then calls the device switch in pairing mode, which sends `51` alone, with no device command before it.

So pairing is the device command when the key goes down, if the device changed, then `51` after 3 seconds.

## Background tasks

The wireless task at `0x08000918` also:

- Sends the name, then the saved device's command (or `11` for USB), then `55`, 2 seconds after boot.
- Queries the battery with `53` every 10 seconds.
- Uses a 3 minute idle timeout (`179999` ms).

## Reproducing

```sh
G=~/zoom65v3-ghidra   # Ghidra rejects project paths with a component starting with "."
mkdir -p $G/proj $G/out
nix shell nixpkgs#ghidra -c ghidra-analyzeHeadless $G/proj kb \
    -import firmware/ZOOM65V3_ANSI_v1.27.bin -overwrite \
    -loader BinaryLoader -loader-baseAddr 0x08000000 \
    -processor 'ARM:LE:32:Cortex' -cspec default \
    -scriptPath tools/ghidra \
    -preScript Setup.java -postScript Export.java $G/out
```

The pairing callback `0x0800077c` is reached only through a function pointer in a table in RAM, so Ghidra does not find it. Disassemble it with:

```sh
arm-none-eabi-objdump -D -b binary -marm -Mforce-thumb --adjust-vma=0x08000000 \
    --start-address=0x0800077c --stop-address=0x08000870 firmware/ZOOM65V3_ANSI_v1.27.bin
```
