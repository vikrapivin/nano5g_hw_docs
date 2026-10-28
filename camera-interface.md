# Camera capture interface and auxiliary block

Camera pixels enter CAMIF at **`0x39900000`**; auxiliary control is at
**`0x39a00000`**. Completion is **IRQ57**. Both are separate from the video
encoder and from DMAC1 (`0x38700000`). The software uses two rotating frame
slots. Frame ownership, clocks and sensor streaming must all be active before a
valid frame arrives.

## Clocks and external signals

- GPIO `0x82` is port 16 bit 2; GPIO `0x83` is port 16 bit 3.
- Sensor on: output mode 1, drive `0x82` low and `0x83` high. Sensor off: `0x82` high, `0x83` low (`osos mem:0x081d0b40`). Reset/power-down names for the two pins are not established. The diagnostics use the same pair as selectors `0x100002` / `0x100003` (off: `0x100002` high `0xf`, `0x100003` low `0xe`; on: reversed) (`diag IspDriver.pe+0x24b8`).
- Sensor clock register `0x3c500014`: OSOS keeps the low 16 bits and replaces the high 16 bits with `((divisor&15)<<16)-0x10000`. Off ORs bit 31. The 7692 probe uses factor 1; 7739 uses factor 4. The diagnostics write the whole word: 7739 `0x0003ffff`, 7692 `0x00008000`, then Stall 100 ms (`diag IspDriver.pe+0x24b8`).
- Diagnostic sensor-on sequence (`diag IspDriver.pe+0x894`, `+0x241c`): PMU reg `0x10` bit 5 set (off: cleared) and reg `0x1c = 0x10` (on only), re-read of `0x10` and `0x1c`, Stall 100 ms (rail role inferred; PMU `0x10` bit 5 is also set by the early PMU init); ClockAndReset gates `0x47` then `0x49` (diagnostics service numbering, argument 1 on, 0 off in the same order); pad function 2 on port 1 bits 2, 3 (`0x10002/3`), port 15 bits 0..6 (`0xf0000..6`) and port 16 bits 0, 1, 5 (`0x100000/1/5`). Port 1 bits 2, 3 and port 16 bit 5 already read function 2 in the PreEfi table (`0xe0ee22ee`, `0xee2eefee`); port 15 and port 16 bits 0, 1 read `e` there.
- Clock source frequency and pin electrical limits are not measured.

OSOS clock ID `0x46` (table entry `mem:0x08551e10`) clears `0x3c500048` mask
`00020000` and `0x3c500068` mask `00008000`. ID `0x48` clears `0x3c500048`
mask `00040000`. Disable sets those masks. GPIO enable waits 100 ms; control
reset also waits 100 ms before its status poll.

## CAMIF registers

32-bit accesses; "old" means read/modify/write. The
table covers all 47 reconstructed source accesses, including unexplained
writes.

| Offset | Firmware use |
|---|---|
| `+00` | bit 0 capture enable; configuration separately ORs `10`, `80`, `40`, `20`, `02`; reset ORs `1000`, waits while `2000` set (diagnostics: fixed 100 ms stalls, no poll; see below) |
| `+10` | low 5 bits input mode: mode 2 = `1e`, mode 3 = `18`; mode 0/1 trap in OSOS (the diagnostics support them through `+14`). The diagnostics set `+00` bit 1 and `+10 = (+10 & ~1f) + mode` for modes `18` and `1e` |
| `+14` | diagnostics only: mode 0 clears `+00` bit 1 and `+14` bit 2; mode 1 clears `+00` bit 1 and sets `+14` bit 2; both then `+14 = (+14 & ~3) \| arg` and clear `+10` low 5 bits (`diag IspDriver.pe+0x6f8`) |
| `+18` | format 1/3: low 2 bits 0/1, clear `300`; format 2: low 2 bits 2, clear `30`. Diagnostics put a caller value in the field instead of clearing it: format 0 low 2 bits 0 and bits 9:8 = arg; format 1 low 2 bits 1 and bits 9:8 = arg; format 2 low 2 bits 2 and bits 5:4 = arg (the start path always passes 0: `778(0,0)` for 4:2:0, `778(1,0)` for 4:2:2); formats 1/2 return `0x80000007` when `+10` low 5 bits = `18` (`diag IspDriver.pe+0x778`, `+0x2236..0x2248`) |
| `+20` | height bits 29:16, width bits 13:0; preserve top 2 of each halfword |
| `+24` | pitch in pixels, low 14 bits (diagnostics write the full low halfword) |
| `+30`, `+34` | plane 0 / plane 1 byte addresses; OSOS strips address bit 31. Diagnostics: plane 1 = plane 0 + width*height, bit 31 stripped (`diag IspDriver.pe+0x7dc`, `+0x7f0`) |
| `+40` | bit 0 pending; bit 4 frame ready; bit 5 clears control +44 bit 0; bit 6 AND any of bits 8..10 indicates overflow |
| `+44` | start/stop handshakes, bits 0/1; timing unknown |
| `+50` | ID/self-check: the diagnostics require `0x0a` after capture start, else EFI status `0x80000003` (`diag IspDriver.pe+0x84c`) |

Auxiliary registers, offsets from `0x39a00000`:

| Offset | Firmware sequence / meaning |
|---|---|
| `+00` | on: OR 2, then OR 1, as separate writes; off: write 0; reset pulse: OR `10`, wait 2 ms, clear `10`, wait 2 ms (diagnostics: wait 100 ms each, `diag IspDriver.pe+0x5a0`). Diagnostic enable (`+0xa38`): `on ? (+00 \| 1) : 0`, Stall 1 ms |
| `+04` | enable toggles bit 0. Diagnostic enable writes `off_arg ? (+04 & ~1) : 0x98000001` (`diag IspDriver.pe+0xa38`) |
| `+08` | control routine clears bit 0 (diagnostics too); rest unknown |
| `+10` | start writes `ffffffff`; stop writes 0; per-bit meaning unknown |
| `+14` | start writes `ffffffff` before +10; per-bit meaning unknown (the diagnostic start writes `+14 = +10 = ffffffff`, `diag IspDriver.pe+0x580`) |
| `+18` | ID/self-check: the diagnostics require `0x03` after capture start, else EFI status `0x80000003` (`diag IspDriver.pe+0x84c`) |

## Configuration, streaming and completion

Configure is `osos mem:0x0810aadc`, probe is `osos mem:0x0810a940`. Configure:

1. Enable both gates.
2. Reset.
3. Copy the config.
4. Select input 2 / output 1.
5. Set dimensions and pitch.
6. Auxiliary reset.
7. Set the CAMIF +00 masks in the order listed above.
8. Install IRQ57.

Normal preparation `mem:0x0810a8de`: 640×480, pitch 640; turn on the sensor
clock; reset and initialize the selected sensor; select size 0; start the
auxiliary block; start sensor streaming. Ordered sensor writes are in
[camera-sensors.md](camera-sensors.md).

Start `mem:0x0810aa84`: clear +40 bit 0 and +44 bits 0/1, then set +00 bit 0.

ISR `osos mem:0x081281dc` (decompiled):

- Checks pending before other reads.
- Overflow increments a counter.
- Frame ready completes only when the event, next buffer and expected dequeue index are all valid. It then copies the same 32-bit timestamp to frame `+40` and `+44`, signals, programs the next buffer and toggles the active slot.
- Acknowledge is `+40 &= ~1` (not write-one-to-clear).

Stop `mem:0x0810a9bc`:

1. Clear capture enable.
2. If +44 bit 0 is clear, wait for bit 1 with 100-unit sleeps and a 3000-unit budget (OS sleep units, not milliseconds); assert on timeout.
3. Clear handshake/pending bits.
4. Dispose both buffers and zero the software indices.
5. Stop the auxiliary block, power off the sensor, gate CAMIF.

## Diagnostic driver (`diag IspDriver.pe`)

Differences from the OSOS sequence above, in addition to the register-table entries:

1. CAMIF reset: before it `+00 &= ~0xf0`, `+10 &= ~1`, clear `+40` and `+44`, install the IRQ handler (IRQ 57), then `+00 |= 0xf0`. The reset is `+00 |= 0x1000`, Stall 100 ms, clear, Stall 100 ms, with no poll of `2000` (`+0x674`, `+0x60c`, `+0x638`). Aux reset is `+00 |= 0x10`, Stall 100 ms, clear, Stall 100 ms (`+0x5a0`).
2. IRQ 57 handler: if `+44` bit 0 is set, `0x39000004 |= 1` (starts the [scaler](scaler.md)), then `+44 = 0`, `+40 = 0` (`+0x5e8`, `+0x5d8`).
3. Capture stop: `+00 &= ~1`, clear `+40`/`+44`, Stall 10 ms (`+0x820`).
4. After start: Stall 500 ms (`0x7a120` µs) before the first frame; 7692 only: dummy reads of sensor registers `82` and `38` (`+0x22b2`, `+0x1d20`).
5. Camera supply and sensor-on gates: see Clocks and external signals.

Capture modes (`+0x1c6c`, `+0x4f0`, `+0x21d4..0x22b0`, `+0x22fc`):

| Mode | Size | Mode | Size |
|---|---|---|---|
| 0 (also 8, 9, 11 and above) | 640x480 | 4 | 1280x960 |
| 1 | 800x600 | 5 | 1600x1200 |
| 2 | 1024x768 | 6 | 1920x1080 |
| 3 | 1280x720 | 7 | 2048x1536 |
| 10 | 320x240 | | |

Start accepts only w <= 640 and h <= 480 (else `0x80000002`). Format 1 (4:2:2) buffer size is w*h*2 (code 2); the other
(4:2:0) is w*h*3/2 (code 4). Two frame buffers of that size, page-aligned (`0x1000`), pre-filled with byte `0x08`;
default preview 640x480 with the chroma plane at `+0x4b000`. The [scaler](scaler.md) converts the frame for the display
(destination 640x480, pitch `0x280`).

## Buffer parameters

- Ordinary frame allocation: `width*height*2+0x800`. Frame object size `0x198`; 16 are constructed in the camera pool.
- Format 1: planar luma plus interleaved chroma. Plane 0 = `(allocation+2047)&~1023`, plane 1 = `(plane0+w*h+1023)&~1023`, pitch rounded to 8 pixels. The large allocation does not mean two bytes per visible pixel.
- Format 2: packed two-byte pixels, plane 0 aligned to 1024, no plane 1, pitch rounded to 4 pixels. Chroma byte order not determined.
- OS ownership tags 15/16 are software reference tags, not CAMIF register fields.
- Cache ownership must accompany physical DMA addresses.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| MMIO and GPIO sequences above | original VAs above | observed |
| ISR branches and acknowledge | decompilation of `osos mem:0x081281dc` | observed |
| Capture pixels / frame timing | later device reports | electrical timing and some status bits not established |
| Reset/power-down pin names, auxiliary fields | writes listed above | meaning unknown |
| Diagnostic resets, `+14` modes, `+18` argument fields, `+50` = `0a`, aux `+18` = `03`, IRQ 57 scaler kick, mode table | `diag IspDriver.pe+0x6f8`, `+0x778`, `+0x84c`, `+0x5e8`, `+0x1c6c` | observed |
| `+50` / aux `+18` constants look like ID/version registers; no other use found | `diag IspDriver.pe+0x84c` | inferred |
