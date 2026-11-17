# Video playback in the iPod OS

The iPod OS plays MP4/MOV/M4V files with an H.264 track (hardware engine at `0x38f00000`), an
MPEG-4 Part 2 track or an H.263 track (a second hardware engine at `0x39600000`, shared with
JPEG decoding). It has no software video decoder. Audio is decoded on the CPU. Decoded pictures go
to the display compositor's video layer without being copied. Register semantics of the H.264 engine
are on [video-decoder.md](video-decoder.md), and those of the compositor on
[display-compositor.md](display-compositor.md).

Evidence markers: **observed** (read in the decompilation or disassembly of the osos image),
**inferred**.

## Flow

| Stage | Routine | What it does |
|---|---|---|
| UI | `osos mem:0x080dd364`, `0x08243aa0` | Videos menu events; `EnterVideoNowPlayingLcd` looks up the item and starts the player (observed) |
| Start | `osos mem:0x08268b98` | registers performance requester 8 and requests its level 2 (performance level section), then starts the player (observed) |
| Reader | `osos mem:0x081660cc` | MP4 reader open: one buffered file handle with a 64 KiB chunk, atom scan, track typing (observed) |
| Factory | `osos mem:0x082a3838` | picks the decoder from the track's codec byte and creates the picture pool (observed) |
| Decode task | `osos mem:0x082b479c` | `MeCCAVideoDecode` task, stack `0x4000`: read sample, decode into a pool picture, push to the A/V queue (observed) |
| A/V queue | `osos mem:0x082bb710`, `0x082c52cc`, `0x082c5408` | reorder by PTS, drop late pictures, present (observed) |
| Display | `osos mem:0x082918cc`, `0x08291960` | the picture's planes become the compositor video layer; the shown picture is released when the next completes (observed) |

## MP4 reader

- One video track is taken (`hdlr` `vide`); audio (`soun`), `sbtl`, `clcp` (`tx3g`) and `text`
  tracks are typed by `osos mem:0x08173454` (observed). There is no `ftyp` brand check, so MOV and
  3GP files go through the same parser; fragmented MP4 (`moof`) is not read (observed).
- Sample tables are read in windows: `stsz`, `stts` and `ctts` 0x4000 entries, `stco`/`co64` and
  `stsc` 0x2000 entries; `stss` and `elst` are loaded whole (`osos mem:0x081f0172`, `0x081effca`,
  `0x081734a2`, `0x081f022e`, `0x081f00a8`, `0x08173616`, `0x081efcfc`; observed). The `ctts` atom
  type is formed as `'stts' - 0x10` in code, so the string does not occur in the image (observed).
- Video sample reads (`osos mem:0x0816f114`) merge adjacent samples into one seek and read
  (`0x0817504c`); timestamps are converted to milliseconds with the `mdhd` time scale (observed).
- Seek (`osos mem:0x081dc1b0`) walks the tables to the last sync sample at or before the target.
  Streams with 0x5000 or more sync samples, or without `stss`, get a decimated sync index of three
  0x14000-byte arrays (`osos mem:0x0816e170`; observed).
- Sample-entry limits (`osos mem:0x081edc78`; observed): `avc1` needs `avcC`, profile 66, 77 or
  100 and `level_idc` below 41; `mp4v` needs `esds` with object type `0x20`, stream type 4 and an
  average bit rate of at most 5,000,000; `s263` is accepted. A byte at `osos mem:0x085508ff`
  disables both checks; its writer was not found.
- No other video container is parsed (bounded search: RIFF/AVI, Ogg, Matroska and FLV fourccs and
  extensions over the whole image; observed).

## Decoder selection

`osos mem:0x082a3838` switches on the track codec byte written by `osos mem:0x081653e4`:

| Codec byte | Sample entry | Driver | Engine |
|---|---|---|---|
| `0x0b` | `mp4v` | singleton `osos mem:0x082bc044`, vtable `0x08503b28` | `0x39600000`, IRQ 45, clock ID `0x40` |
| `0x0d` | `s263` | same driver, H.263 mode | same |
| `0x0c` | `avc1` | singleton `osos mem:0x082bc480`, vtable `0x085059e8` | `0x38f00000`, IRQ 35, clock ID `0x41` |
| other | | none: the track is not opened | |

Both drivers use the slots open `+0x10`, close `+0x14`, decode `+0x40` and get-info `+0x44`
(observed). A failed open makes the stream set-up `osos mem:0x081dc888` return an error; no other
video decoder exists (bounded: the factory is the only caller of both singletons; observed).

The H.264 open (`osos mem:0x082d9a0c`) rejects an `avcC` with version other than 1, profile other
than 66, 77 or 100, `level_idc` above 30, a NAL length size above 4 or more than two SPS (observed).
These are stricter than the parser limits on [video-decoder.md](video-decoder.md) and match the
capability list (`osos mem:0x0800784c`: H.264 maximum resolution 414,720 pixels = 720x576,
10,368,000 pixels per second = 720x480 at 30 Hz; MPEG-4 640x480 pixels within 720x576; observed).

## Picture pool

`osos mem:0x082a3838` (observed):

- A named heap `MeCCA_VideoBufferMgr` of 14 x `0x97000` bytes, or 20 x `0x97000` when the heap
  block count (`osos mem:0x0827814c`) is 20 or more, or `0x1c3800` bytes when a configuration byte
  (`+0x111` of object `0xa080`) is set. A `0x97000`-byte picture holds 640x480 (planes below:
  `0x8c000`) but not 720x480 (`0x9d800`; computed).
- A `0x40000`-byte compressed-sample buffer from the same heap.
- Pictures: DPB size + 4, or + 10 on the 20-picture heap, at most 26; the open fails for a DPB
  size of 0 or above 16 (`osos mem:0x08261a64`).
- Each H.264 picture has three separately allocated, 1 KiB-aligned planes: `w*align64(h)`,
  `w*align64(h/2)`, `w*align64(h/4)` (`osos mem:0x08300fcc`). Plane addresses sit at picture
  `+0x1c`, `+0x28`, `+0x38`.
- Pictures are reference counted (vtable `+0x1c` add, `+0x20` release). Owners seen: the decoder
  DPB, the decode task, the A/V queue, the compositor.

## H.264 driver

**Open** (`osos mem:0x082d9a0c`, observed): clock `0x41` on; `0x3e000004 |= 1`; IRQ object for
source 35 with handler `osos mem:0x082df3c8`; two scratch areas of `0x10420` and `0x20420` bytes,
1 KiB aligned; full reset `1004 |= 0x7ff` (`0x082e544c`); scratch descriptors; `10c4 = 1`,
`1808 = 0`, `10d0 = 0xff`; `osos mem:0x082e5368(1)` pulses `10d8` (3, then 1) and enables the same
pulse after every slice; clock `0x41` off.

**Decode** (`osos mem:0x082e22b4`, one call per access unit; observed):

1. Clock `0x41` on; second-output mode cleared (`osos mem:0x0822199c`).
2. For each NAL (length-prefixed): input set-up and header parsing `osos mem:0x081f0a24`
   (per-slice parser reset `1004 = (old & ~0xf) + 0xc` polled to bit 0 clear, NAL window, header).
3. First slice of the picture only: the output picture's three descriptors are written once
   (`osos mem:0x082ee360`). Descriptors of reference pictures are not rewritten.
4. Slice start `osos mem:0x082eddf0`, then a wait of up to `0x84` ticks on the semaphore
   `0x086b105c` signalled by the IRQ handler (`10c0 = 0`). The wait result is not checked.
5. Completion `osos mem:0x082f812c`: read `1074`, write it back, `10c0 = 0`, `1508 = 0`, `10d8`
   pulse.
6. Status 0 or 1: next NAL. Status bits 1..6 (VLC, macroblock type, sub-macroblock type, pattern,
   "idf", delta QP, messages at `osos mem:0x0845be68..0x0845c014`): core reset
   `osos mem:0x082ee0f8` (`1004 = 0x3e0`, poll until the register reads 0, at most 10,000 polls) and
   **continue with the next slice of the same picture**; if the reset fails, end the picture and
   skip until the next IDR. Status bit 7 (time-out): core reset and return -4, after which the
   decode task stops the stream.
7. End of picture: DPB store and output (`osos mem:0x082ee42c`, `0x08314df0`, smallest picture order
   count first), release of the output picture's driver reference, clock `0x41` off.

Error skipping: a nonsensical NAL length or a header parse failure sets a skip-until-IDR flag; the
first picture of a stream must be an IDR or an I slice (observed). A failed picture is still passed
on, tagged `0x10`; the A/V queue then skips only the per-frame listener call. There is no
concealment copy (observed).

The decoder clock is gated off between pictures. No CPU, bus or arbitration change is made by the
driver apart from `0x3e000004` bit 0 (bounded: every Thumb code site that forms `0x3e000000`, three
sites; observed).

## MPEG-4 Part 2 / H.263 engine

Register map, open sequence and picture sequence of the second engine (`0x39600000`, IRQ 45, clock ID `0x40`) are on [video-decoder.md](video-decoder.md#mpeg-4-part-2-h263-and-jpeg-decoder-0x39600000). In the player flow it takes the place of the H.264 driver for codec bytes `0xb` (`mp4v`) and `0xd` (`s263`).

## A/V queue, pacing and performance level

- `AVSManager_WorkerThread` (`osos mem:0x082bb814`, priority `0x10`, stack `0x1000`) pops pictures
  and presents them (observed).
- Push (`osos mem:0x082bb710`): a picture that is neither a sync sample nor flagged and whose PTS
  is below the reference at queue `+0x34` is dropped, and performance requester 8 is raised.
- Reorder (`osos mem:0x082c52cc`): smallest PTS first; holds back while the gap exceeds
  `deltaDts + 8` and the depth is at most 2; logs a warning above depth 3. More than three
  consecutive outputs lower requester 8 by one step.
- Pop (`osos mem:0x082c53b0`): queue depth below 4 raises requester 8.
- Present (`osos mem:0x082c5408`): a picture more than 33 ms behind the reference is dropped
  unless it is a sync sample or flagged. Which clock writes the reference was not found.
- Performance governor (`osos mem:0x22005bec`, `0x22005c9c`, `0x220061a4`, `0x22006278`): 23
  requesters, applied level = maximum over active requesters. Floor table `osos mem:0x22007090`
  (floor, request 1, request 2): video requester 8 is `2, 2, 3`. Clock-level table
  `osos mem:0x220093b8`: `01111150 01111110 22222141 22222121 62222111`. Bits 7:4 are the
  `fclk_cpu` divider (5, 1, 4, 2, 1), bits 23:12 the three CLKCON1 dividers (2 in levels 2..4), bit
  29 selects the PLL path, bit 30 (level 4 only) raises a supply value before switching (`0x4b0`,
  otherwise `0x41a`; observed; reading them as millivolts is inferred). Video therefore runs at level
  3, CPU at half the PLL rate, and reaches level 4 only through other requesters or system states
  (inferred).

## Display hand-off

The display unit gives the decoded picture itself to the compositor: `osos mem:0x082918cc` writes
luma (`pic+0x1c`) and chroma (`pic+0x28`) addresses and `pitch << 17` to the video layer at once
when the compositor is running, and keeps a pending and a shown picture; `osos mem:0x08291960`
promotes pending to shown after frame completion and releases the old one (observed). Rotation of
the video layer supports quarter turns only (codes 1 and 3 in `+0x118` bits 1:0; observed). The
LCD transfer is started per submitted frame (inferred from the IRQ14 behaviour on
[display-compositor.md](display-compositor.md)). TV out uses the same picture objects on the TV
video input at `0x39100000` (`osos mem:0x082afd4c`; observed).

## Audio during video

Audio tracks are set up by `osos mem:0x08141fe4` (codec bytes 0, 1, 3, 5, 7 accepted, five codec
descriptor tables, a `0x101c0`-byte decoder instance; observed). Output is the PCM task
`MeCCAOutputTask` (`osos mem:0x082b1296`): two `0x1000`-byte I2S DMA buffers refilled `0x400`
frames at a time (observed). No audio decoding hardware or DSP program was found (bounded: codec
strings and the hardware block list); AAC decoding on the CPU is inferred.

## Not established

The writer of the A/V reference time; the class implementing the display sink; what the second
output (`10e0`/`10e4`, H.264 dual output, only set by an in-band Baseline SPS) is used for; register
semantics of the MPEG-4 engine beyond the values written; whether the silicon decodes the MPEG-4
features the driver does not use.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Factory selects H.264 or MPEG-4/H.263 hardware driver; no software video decoder | `osos mem:0x082a3838`, `0x082bc044`, `0x082bc480` | observed; bounded |
| MPEG-4 Part 2 / H.263 hardware engine at `0x39600000`, IRQ 45, clock `0x40`, shared with JPEG | `osos mem:0x082d954c`, `0x082df430`, `0x082dfc1c`, `0x080e44b0`, `0x08551d80` | observed |
| MPEG-4 VOL features rejected; B-VOP without backward reference | `osos mem:0x082eae9c`, `0x082dfcd4` | observed; B-VOP result inferred |
| H.264 open: profile 66/77/100, level <= 3.0 | `osos mem:0x082d9a0c` (`0x082d9b0a`) | observed |
| Reader: windows, one handle, ms timestamps, `ctts` supported | `osos mem:0x0816db58`, `0x0816f114`, `0x0817504c`, `0x081734a2` | observed |
| Pool: 14/20 x `0x97000`, DPB + 4/+10, cap 26 | `osos mem:0x082a3838`, `0x08261a64` | observed |
| Output descriptors written once per picture | `osos mem:0x082ee360` called on the first slice only | observed |
| Slice error bits 1..6 recovered by core reset, picture continues | `osos mem:0x082e22b4` (`0x082e2514`), `0x082ee0f8` | observed |
| Clock `0x41` gated per access unit | `osos mem:0x082e22de`, `0x082e2648` | observed |
| Decoded pictures shown without a copy | `osos mem:0x082918cc`, `0x08291960` | observed |
| Video performance requester 8 at level 3 | `osos mem:0x08268b98`, `0x22007090`, `0x220093b8` | observed; clock reading inferred |
| No audio decode hardware | strings, hardware block list | bounded negative |
| Correction: an MPEG-4 Part 2 decode path exists (earlier text said none was found) | `osos mem:0x082d96c8` | observed |
