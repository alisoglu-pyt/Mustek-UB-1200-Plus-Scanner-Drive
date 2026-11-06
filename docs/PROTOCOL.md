# GT-6801 USB protocol (SANE `gt68xx` summary)

Sources: `sane-backends/backend/gt68xx_low.c`, `gt68xx_generic.c`,
`gt68xx_gt6801.c`, `gt68xx_devices.c` — the `mustek-gt6801` command set.

## 1. Transport layer

All commands use **control transfers**. The bulk endpoint is used only for
scanned image data.

| Operation | bmRequestType | bRequest | wValue | wIndex | Data |
|---|---|---|---|---|---|
| Memory write | `0x40` | `0x01` | `0x2009` | `addr` | 64 bytes (OUT) |
| Memory read  | `0xC0` | `0x01` | `0x200A` | `addr` | 64 bytes (IN) |
| Normal command write | `0x40` | `0x01` | `0x2010` | `0x3F40` | 16 bytes |
| Normal response read | `0xC0` | `0x01` | `0x2011` | `0x3F00` | 16 bytes |
| Small command write  | `0x40` | `0x01` | `0x2012` | `0x3F40` | 8 bytes × 8 repetitions |
| Small response read  | `0xC0` | `0x01` | `0x2013` | `0x3F00` | 8 bytes |

A normal command packet is 16 bytes: `[cmd, 0x01, ...parameters]`.
`res[0] == 0x00` means success; `res[1]` is usually the same as the command
byte (some firmware revisions omit it, which is only a warning).

## 2. Firmware upload

1. `req(50 01 80)` → if `res[0]==0x00 && res[1]==0x50`, firmware is **loaded**.
2. Otherwise, using 64-byte blocks:
   - `memory_write(addr, block)` (the last block is padded with `0x00`)
   - `memory_read(0x3F00, 64)` for verification; failure condition:
     `check[0] != 0 && check[1] != 0x40`
3. Boot: `req(69 01 C0 1C 00 ...)`.

Firmware lives in RAM and is lost when power is removed.

## 3. Command table

| Bytes | Name | Parameters / notes |
|---|---|---|
| `0x10 01` | Power status | `res[0]==0` → OK |
| `0x2E 01` | Identity | `res[2:4]` VID, `res[4:6]` PID, `res[6:10]` DID, `res[10:12]` FID |
| `0x20 01` | Setup scan | see below |
| `0x22 01` | AFE settings | `[2]=R off,[3]=R gain,[4]=G off,[5]=G gain,[6]=B off,[7]=B gain` |
| `0x76 01` | Exposure | `[2]=[6]=[10]=0x04`; 16-bit R/G/B durations in `[4:6] [8:10] [12:14]` |
| `0x12 01` + `0x24 01` | Move carriage home | models with MOTOR_HOME flag use a single `0x34 01` command |
| `0x25 01` | Lamp control | `[2]` bit0 = flatbed, bit1 = transparency (no-op on CIS) |
| `0x43 01` | Start scan | |
| `0x42 01` | Stop scan | |
| `0x35 01` | Data ready? | `res[0]==0` → ready; then read bulk IN |
| `0x83 01` | Paper feed | sheetfed models only |
| `0x69 01 C0 1C` | Firmware boot | |
| `0x50 01 80` | Firmware check | |

### Setup scan (`0x20`) packet layout

| Offset | Content |
|---|---|
| `0x00` | `0x20` |
| `0x01` | `0x01` |
| `0x02` | `abs_y0` (LE, 16-bit) — starting line |
| `0x04` | `abs_ys` — number of lines |
| `0x06` | `abs_x0` — starting pixel |
| `0x08` | `abs_xs` — number of pixels |
| `0x0A` | color mode code (gray / RGB / line mode) |
| `0x0B` | `0x60` CIS lamp mode, `0x20` CCD |
| `0x0C` | `xdpi` (LE) |
| `0x0E` | `0x12` constant |
| `0x0F` | `0x00` constant |
| `0x10` | `scan_bpl` — bytes per line (multiple of 64) |
| `0x12` | `scan_ys` — lines to scan |
| `0x14` | `motor_mode_1` (bit0 mas, bit1 mbs, bit2 mds, bit3 backtrack) |
| `0x15` | `motor_mode_2` (bit0 lamp, bit2 line mode) |
| `0x16` | `ydpi` (LE) |
| `0x18` | backtrack line count |

`scan_bpl` must not exceed 16 KB. On CIS models, `line_mode` is unavailable
because there is not enough memory.

## 4. Reading data

```
loop:
    req(35 01)          -> wait while not ready
    bulk IN endpoint    -> raw lines (BGR, 8 or 16 bit)
```

Color order on this model is **BGR**, and the default gamma is `1.5`.
