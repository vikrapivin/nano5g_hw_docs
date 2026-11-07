# Video encoder firmware walkthrough

The video encoder (see [video-encoder.md](video-encoder.md)) runs an embedded program that the iPod OS uploads by DMA
before recording: `0x4760` bytes of ARMv4T code for an ARM core inside the encoder. This page follows that program from the reset
vector to the end of each command: start-up, the command loop, the header and picture handlers, rate control, the
C library routines that were linked in, the data structures, and what each engine register access says about the
hardware. Register tables, host-side packing and the engine block descriptions stay in [video-encoder.md](video-encoder.md);
this page refers to them instead of repeating them.

All addresses are firmware addresses (`firmware 0xADDR`, offsets into the uploaded image, which the core executes
at address 0). Evidence tags: **static** (read from the disassembly and decompilation of the image), **dynamic** (the
image executed instruction by instruction with the engine blocks modelled, outputs decoded by an independent
H.264 parser; the models are listed in the dynamic-analysis section), **inferred**. Function and field names are descriptive
labels chosen for this document, not names found in the image (the image carries no symbols).

## Contents

1. [The image](#the-image)
2. [Start-up path](#start-up-path)
3. [The command loop](#the-command-loop)
4. [Command 1: header](#command-1-header)
5. [Command 2: picture](#command-2-picture)
6. [Commands 0, 3, 4 and 5](#commands-0-3-4-and-5)
7. [Call tree](#call-tree)
8. [Function catalogue](#function-catalogue)
9. [Data structures](#data-structures)
10. [Library inventory](#library-inventory)
11. [What the code tells us about the hardware](#what-the-code-tells-us-about-the-hardware)
12. [Dynamic analysis](#dynamic-analysis)
13. [The 1.0.1 program](#the-101-program)
14. [Open questions](#open-questions)
15. [Finding ledger](#finding-ledger)

## The image

| Range | Content |
|---|---|
| `0x0000..0x001f` | exception vector table (eight `b` instructions; `0x14` is `b .`) |
| `0x0024..0x00cb` | seven exception stubs: each loads a handler address from a RAM word and jumps to it (`0x5fc4`, `0x5fc8`, `0x5fcc`, `0x5fd0`, `0x5fd8`, `0x5fdc`, and `0x5fe0` for a seventh stub no vector reaches); literal pool `0x00d4..0x00ec` |
| `0x00cc..0x0187` | reset entry, `__main`, `__scatterload`, zero-init handler, floating-point init veneer |
| `0x0188..0x2a6f` | the encoder proper: context, headers, slice and macroblock layers, rate control, command loop (`0x1f98`), `main` (`0x2018`) |
| `0x2a70..0x4517` | C and C++ run-time library (RealView): heap, division, signals, start-up and exit, software floating point |
| `0x45c8` | entropy handler table (two `{function, adjust}` pairs) |
| `0x45d8` | six doubles: the Qstep mantissas `0.625, 0.6875, 0.8125, 0.875, 1.0, 1.125` |
| `0x4608..0x4747` | signal message table (fourteen 23-byte text records) |
| `0x474c` | scatter table, one 16-byte entry `{src 0x4760, dst 0x4760, size 0x60, handler 0x130}` |
| `0x475c` | one word, the pointer to the encoder context (stored at run time) |
| `0x4760..0x47bf` | zero-initialised library static data (`libspace`), created by the scatter table |
| `0x47c0..` | heap; the stack top is `0x5f00`; the handler pointer words `0x5fc4..0x5fe0` sit above the stack |

Built with the ARM RealView C library (observed: library message texts, scatter-load convention `__main` / `__scatterload`
/ `__rt_entry`, `__user_libspace`, semihosting calls). The image contains C++ run-time pieces (`operator new`, `operator delete`,
`terminate`, "Pure virtual fn called"); the encoder code itself uses a C++ pointer-to-member for the entropy handler
(see [Command 2](#command-2-picture)). The core executes ARM instructions only and never enables interrupts.

## Start-up path

All steps are **static** unless marked; the start-up was also executed (**dynamic**, experiment A).

1. **Reset** (`vec_reset`, `0x0000`): `b 0xcc`. `reset_entry` (`0x00cc`) calls `__main` (`0x00f0`, a `b 0xf8`), then spins. The
   code before the stack set-up uses no stack.
2. **`__scatterload`** (`0x00f8`): two literals (`0x0128`, `0x012c`, relative to `0x128`) give the scatter table bounds
   `0x474c..0x475c`. For each entry it loads `{r0 = src, r1 = dst, r2 = size, r3 = handler}`; an odd handler value means "call
   through `r7 - r3`" (a compressed-region convention that no entry uses), an even one is called directly. The only
   entry has handler `0x0130`, `__scatterload_zeroinit`, which clears `0x60` bytes at `0x4760` (16 bytes per
   `stmcs`). When the walk ends it branches to `__rt_entry` (`0x2d64`). The image has no initialised data section:
   everything mutable is zero-initialised or created at run time.
3. **`__rt_entry`** (`0x2d64`) calls, in order: hook `0x3220` (empty); `__rt_stackheap_init` (`0x301c`); hook `0x3224` (returns
   `-1,-3`, which `__rt_entry` discards by saving and restoring r0/r1); `__rt_lib_init` (`0x3118`); hook `0x3230` (calls the
   empty `0x343c`); `main` (`0x2018`); then `exit` (`0x2f98`).
   - `__rt_stackheap_init` finds `libspace` (`0x4760`, from `__user_libspace`, `0x2d58`), saves its link register and its own
     address at `libspace+0x5c/+0x58`, calls `__user_initial_stackheap` (`0x34f4`), clears sixteen words of `libspace` and
     sets the words `libspace[4] = 1`, `[6] = 0x40`, `[7] = 0x150` (a stack safety margin) and `[5] = heap start` (word indices).
   - `__user_initial_stackheap` loads the CPSR with mode bits `0xd1` (FIQ), `0xd2` (IRQ), `0xd7` (abort), `0xdb` (undefined)
     and `0xd3` (SVC) in turn (interrupts masked) and sets each mode's `sp` to `0x5f00`: there is **one stack** shared by all
     modes. It returns `r0 = 0x47c0` (heap base) and `r1 = 0x5f00`.
   - `__rt_lib_init` clears `errno` (`__rt_errno_init`), calls `_fp_init` through its veneer (`0x015c`, an empty function: there is no
     floating-point hardware to set up), `_init_alloc(0x47c0, ...)` (builds the heap descriptor at `0x47c0` and takes
     memory through `__rt_heap_extend`, which grows the heap in steps of at least `0x1000` bytes up to the stack margin), `_init_user_alloc`
     (empty), and clears `libspace+0x20..0x30` (`+0x24` is set to 1).
   - **Dynamic:** after start-up the heap runs `0x47c0..0x5298`, `sp = 0x5ee0` in the wait loop, `libspace` reads
     `[4] = 1, [5] = 0x5298, [6] = 0x40, [7] = 0x150, [8] = 0`, `[2] = 0x47c0`.
4. **`main`** (`0x2018`): writes host `a00 = 0xdeaddead` (a stage marker, the first host-visible write), host `008 = 3`, host `000 = 1`
   (a DMA start with mode 3; the length and address registers are whatever the host left; the effect is unknown), then calls
   `command_loop`. The instructions after that call (host `010 = ((bs+a8 & 0x1f) + 7) >> 3`, wait until host `03c < 8`,
   host `008 = 7`, `000 = 1`, `510 = 1`) are **never reached**: `command_loop` returns only after command 4, whose last step
   is an endless loop. They are dead code (dynamic: none of them
   executes before or after any command).
5. **`command_loop`** (`0x1f98`) first calls `ctx_create` (`0x1f78`): `operator_new(0x2ec)` (the first heap allocation, observed at
   `0x47d8`), `ctx_construct` (`0x0188`), stores the pointer at `0x475c`, and `engine_init_registers` (`0x1990`: picture engine `+00` =
   host `600 >> 1` (also kept in `ctx+0x294`), `+18` = host `120`, `+1c = 4`, `+20 = 1`, `+24` = host `124`, `+4c` = host `810`).
   `ctx_construct` allocates the 0x2a0-byte result-record array (`0x20d0`, observed at `0x4ac8`) and the 0x2c0-byte stream-slot array
   (observed `0x4d70`), clears S and P, points `ctx+0x278/+0x27c` at them, selects stream 0, and writes host
   `a00 = 0x87301018`. **That value is the ready marker the host sees while the loop waits** (dynamic, experiment A: the final value of
   `a00` after boot); `0xdeaddead` is only the earlier marker.
6. The loop then waits for commands. Nothing else runs between commands: the core is idle in `command_wait`.


## The command loop

`command_wait` (`0x1f08`) returns `(command, stream)` through two out-pointers. Its loop, per iteration:

```
stream = 0xffff; command = 0
do {
    if (host[114] != 0) { stream = (host[c0c] >> 8) & 0xff; command = host[c0c] & 0xff; }
    if (host[10c] != 0) { stream = 0; command = 4; }          // abort overrides everything
} while (!(stream <= 8 && command <= 5));                     // other values are ignored
```

- The firmware never writes host `114`; it only reads it. If the doorbell stayed set the same command would run again at the
  next call, so the hardware must clear it when the command word is read or when `50c` is written (the dynamic model clears it on
  the read of `c0c`; which of the two the silicon does is open, see [Open questions](#open-questions)). The host-side
  description of `114`/`c0c` is in [video-encoder.md](video-encoder.md).
- The abort test (`10c`) does not need the doorbell: with `10c != 0` the very next iteration yields command 4 on stream 0.
- Stream indices 9 and above, and commands 6 and above, are not accepted: the loop keeps polling with the last values
  (dynamic: stream 9 with command 1 left the core spinning in `0x1f08..0x1f77` with no output).
- Commands 0 and 3 are accepted and ignored: `command_loop` has no case for them, `50c` is not written, the loop polls again.
- `c0c` is read twice per pass (stream from bits 15:8, then command from bits 7:0), and the acceptance test is signed (`stream <= 8 && command <= 5`).

`command_loop` dispatch (`switch`):

| Command | Handler | Return to the loop? |
|---|---|---|
| 1 | `cmd1_header` (`0x1ef8`) -> `encode_headers` (`0x0bec`) | yes |
| 2 | `cmd2_frame` (`0x1ee4`) -> `encode_frame` (`0x18c4`) | yes |
| 4 | `cmd4_teardown` (`0x1e9c`), then `cmd4_halt` (`0x1e8c`) | **no**: host `510 = 1`, then `b .` at `0x1e98` |
| 5 | `cmd5_stream_update` (`0x1ebc`) | yes |
| 0, 3 | none | yes (ignored) |

The two handler stubs at `0x1ee4` and `0x1ef8` load the context pointer from `0x475c` and tail-branch to the real bodies. Decompiler listings
merge stub and body; the bodies are separate entries here (`0x18c4`, `0x0bec`).

## Command 1: header

`encode_headers(ctx, stream)` (`0x0bec`, **static**; dynamic experiments B and C):

| Step | Call / write | Effect |
|---|---|---|
| 1 | `ctx_select_stream(ctx, stream)` | `ctx+0x290 = stream`, `ctx+0x274 = slot_array + stream*0x58` |
| 2 | `stream_params_defaults` (`0x0a20`) | P and S to defaults: 352x288, Main profile (`P+0x1c = 1`), QP 26, period 0, `P+0x78 = 1`, `P+0x7c = 0x2710`, `P+0x80 = 0x32`, counters 0, S cleared |
| 3 | `load_stream_params_from_host` (`0x0924`) | width/height (`118/11c`), profile and level (`300`), entropy flag and cabac_init_idc (`310`), period (`308`), `810`, force-type (`3d4`), `80c`, `330` (ref enable, max refs), `900` (RC enables, init QP) |
| 4 | picture engine `+4c` = `P+0x28` | host `810` |
| 5 | bs `+30 = 0x32`, `+44 = 0`, `+78 = 0`, `+b4 = 0`; `ctx+0x2d8 = ctx+0x2dc = 0` | writer in raw-bit mode with emulation prevention (bit 5) enabled, counters cleared |
| 6 | `compute_geometry`, `write_sps`, `write_pps` | NAL 0x67 then NAL 0x68; see the syntax tables in [video-encoder.md](video-encoder.md#header-writer); each ends in `bs_trailing_bits` |
| 7 | host `a08 = ((sps_bits >> 3) << 16) + (pps_bits >> 3)` | the two NAL lengths in bytes |
| 8 | if `ctx+0x288 == 0`: `rate_defaults_init`; and if `P+0xc4 \| P+0xc8`: `rate_init_targets` | rate-control defaults and targets (see [Rate control](#rate-control-functions)); `rate_init_targets` receives the `P` pointer, left in `r1` by the load at `0xc8c` (the callee stores it in `ctx+4`), not the stream, while `rate_state_store` gets the stream |
| 9 | `stream_slot_store(ctx, stream)` | P saved into slot `stream` |
| 10 | if rate control on and not bypassed: `rate_state_store(ctx, stream)` | result record `stream` written |
| 11 | host `a0c = (sps_bits + pps_bits) >> 3` | header byte count |
| 12 | `bs_flush_and_wait(ctx, 1)` | 32 one-bits plus alignment filler, padding to an 8-word burst, `+30` bit 5 set, wait for `+f0 == 0`; the second argument 1 skips the `+dc == 1` wait (the end-of-frame flush passes 0 and waits for both; order of the flush in [video-encoder.md](video-encoder.md#block-0x00520000--bitstream-writer-and-entropy-coder-inferred)) |
| 13 | host `50c = 1` | command complete |

`write_sps` and `write_pps` write through `bs_put_bits` and `bs_put_ue`/`bs_put_se`. Observed outputs (dynamic) for the preset
register images the iPod OS uses (640x480, 30 fps) are in [Dynamic analysis](#dynamic-analysis): the bytes decode as a valid
Baseline level 3.0 SPS and PPS.

Properties the older notes did not record:

- `profile_idc` values other than 0, 1, 2 (host `300[1:0] = 3`) make `write_sps` **skip the 16-bit profile and constraint field entirely**
  (static: the three-way test falls through to the next write); the SPS is then malformed (dynamic experiment C, 11-byte SPS whose
  second byte is `level_idc`).
- The SPS is the same for CAVLC and CABAC; only the PPS entropy flag differs (`0x19` vs `0x1d` in the first five PPS bits).
- `P+0x20 level_idc` is host `300 >> 2` without a mask and `write_sps` sends 8 bits, so bits above 8 of `300` stay in `P`. `P+0x50` and `P+0x54` both get host `330[11:8]`; `P+0x5c` keeps its default 0. `P+0x2c slice_qp` is not loaded from the host: `write_pps` sets it to `init_qp`. `profile == 2` is tested a second time for the 7-bit High-profile block.
- `stream_params_defaults` does not clear `crop_x/crop_y` or the padded sizes; they are stale when the new size is aligned, and then `crop_flag` (cleared by the defaults, set only by `compute_geometry`) keeps them unused. `write_pps` ignores its second argument (0). `HOST+a08` uses `>> 3` per NAL, `a0c` uses `(sps + pps) >> 3`; both are exact because the trailing bits align each NAL.

## Command 2: picture

`encode_frame(ctx, stream)` (`0x18c4`, **static**):

```
ctx_select_stream(ctx, stream)
frame_setup(ctx)                       // 0x1834
stream_slot_store(ctx, stream + 4)     // pre-frame backup of P (slots 4..7)
while (!ctx->last_mb) {                // ctx+0x2cc
    slice_type_select(ctx)             // 0x1550
    slice_setup(ctx)                   // 0x352c: slice header, engine set-up
    (*ctx->mb_handler)(ctx)            // mb_loop_cavlc 0x1ca8 or mb_loop_cabac 0x1d88
    slice_close(ctx)                   // 0x1104
}
frame_end(ctx); frame_finish(ctx)      // 0x0f3c, 0x0d10
stream_slot_store(ctx, stream)
if (rate control on && !bypass) rate_state_store(ctx, stream)
```

**Frame set-up** (`frame_setup`, `0x1834`):

1. bs `+5c = 2`, then `+5c = 1` (the first write is visible only in the disassembly, the decompiler drops it; busy flag).
2. `stream_slot_load(ctx, ctx+0x290)` copies slot -> P.
3. if rate control is on and not bypassed: `rate_state_load` (restores the 21-word state, rewrites picture engine `+160..+17c`).
4. `reload_frame_params` (`0x3b88`) re-reads host `31c..348`, `3cc/3d0/3d4`, `900`, `914`, `c08`, `314`. If host `900` bit 9 is set without bit 8,
   it also sets `P+0xdc = 1` and `P+0xc8 = 1` (the engine rate flag is forced on). The `byte_thr` clamp tests the N flag of `324 - 0x780`; `vec_dump_en` is `c08 & 0x80000000` (not shifted) and `c08` is read twice. The complete list of host registers read per command is in [video-encoder.md](video-encoder.md#embedded-firmware).
5. `frame_type_select` (`0x1610`): type 2 (IDR) when `P+0xbc == 0` (first frame), when the period is 0, or when the
   frame counter `P+0xb8` equals `period + 1`; otherwise type 1 (P). Host `3d4` bit 0 overrides with the type in bits 2:1. Type 2 resets
   `P+0xb8` to 0. `ctx+0x2a0 = (type == 1)`.
6. `frame_engine_setup` (`0x4084`): clears counters, selects the handler pair from the table at `0x45c8` by `P+0x24`, derives
   `P+0x54 = min(P+0x50, frame_num - 1)` (0 on the first frame) and the reference count `P+0x5c`, programs reference-engine
   `+00/+04/+08/+0c/+14`, picture-engine `+84/+154/+14c/+148/+150/+144/+190/+30/+34/+140`, bs `+14/+58/+70`, `ref_planes_setup`, clears bs `+10` low
   bits, `bs+00 = 1`, `bs+08 = 4` (CAVLC) or `0x80` (CABAC). In bypass mode it instead copies host `91c..944`.
7. if not bypassed: the tail of `frame_setup` branches to `frame_rate_window_update` (`0x2320`).

**Slice loop.** One iteration is one slice, not one macroblock.

- `slice_type_select` (`0x1550`): slice type from the frame type, host `328` (force I) and `344` (force refresh) masks per slice, and the
  over-limit flag; stores `ctx+0x29c`, `ctx+0x2a0`, `ctx+0x2b8`.
- `slice_setup` (`0x352c`): clears slice state, `bs+00 = 4`, `write_slice_header` (`0x1340`), picture engine `+80`, for CABAC the init words
  `bs+48/+4c` and `bs+00 = 0x10`, `slice_limit_setup` (`0x12a4`, picture engine `+14`), reference engine `+0c/+04/+14`.
- The handler pair at `ctx+0x280/+0x284` is a C++ **pointer to member function** (low bit of the adjust word selects a virtual lookup,
  observed at `0x1904..0x1928`); `frame_engine_setup` loads it from the table at `0x45c8`: `{0x1ca8, 0}` (CAVLC) or `{0x1d88, 0}` (CABAC). It
  selects the **entropy coder**, not intra, inter or skip.
- `slice_close` (`0x1104`): for the last macroblock a fetch handshake (`bs+c8`), then trailing bits (CAVLC: `bs_trailing_bits`; CABAC: zero
  fill), bit totals, the over-limit flag `ctx+0x2c4`, the statistics record in table RAM, slice counter ++.

**Per-macroblock work** (`mb_loop_cavlc`, `mb_loop_cabac`):

| Step | CAVLC (`0x1ca8`) | CABAC (`0x1d88`) |
|---|---|---|
| fetch | `mb_fetch` (`bs+c8 = 1`, poll 0, `bs+10 = mb address`) | same |
| end marker | slice mode on and `rec0 == 0xffffffff`: `ctx+0x2c8 = 1`, clear `0x2000a0` when `P+0x78 == 1` | same |
| last macroblock | `P+0x34 == mb_addr + 1`: `ctx+0x2c8 = ctx+0x2cc = 1` | same |
| decode | `decode_mb_record` (record -> S), optional `vector_row_flush` when host `c08` bit 31 | same |
| syntax | `mb_layer_cavlc` (`0x3d04`) | kick `bs+04 = 0x10000`, poll `bs+20` |
| advance | `mb_advance` (`0x19d4`) | same |
| end | flush a pending skip run with `bs_put_ue` (P slices) | final kick `0x10000` with `bs+54 = 1`, then `bs+54 = 0` |

`mb_layer_cavlc` writes, in order: for a skipped macroblock only `bs+48 = cbp<<1 | 0x80` and kick 4; otherwise `bs+4c = mb_type`,
`write_mb_type` (skip run as ue in P slices, mb_type ue, `1111` for P_8x8 sub-types, reference indices), kick `0x800` (P) for the motion data the
engine supplies, `bs+0c` code mode 2 (intra) or 3 (inter) for the coded block pattern when it is not implied, `mb_qp_delta` as se, `bs+48/+4c`, kick 4.
Intra 4x4 prediction modes come from the engine on kick `0x80` (only I_NxN). The intra chroma prediction mode is written as ue after the kick.

Where the disassembly differs from the decompiler listing (disassembly wins): `mb_qp_delta` is written when `cbp != 0`, or the slice type is 0 or 2 with `mb_type != 0`, or a P macroblock has `mb_type - 6 < 0x18` (unsigned), not for every type 0; `write_mb_type` reads the type class on the inter path, overrides the type with `S.mb_type` for intra macroblocks of P pictures, maps type 3 to 4 when `ref_count == 0`, and ends with a tail branch to `bs_put_ue` after kick `0x80` (chroma mode last); after the end marker (`MBREC(0) == 0xffffffff`, read once) both macroblock loops jump to their epilogue; `mb_advance` compares `byte_thr` unsigned with `bs+44 >> 3` and `mb_addr < mb_count - 5` signed; `slice_type_select` shifts by a register, so a slice number of 32 or more tests as 0; `slice_limit_setup` in mode 1 writes only `P+0x7c = 0x7fffffff` (slice count `>= 0x1f`, signed) and never picture engine `+14`, other modes than 0, 1, 2 write nothing, and modes 0 and 2 write `0xfff` for slice count `>= 0x1f` or limit 0; `cabac_zero_words` divides (`__rt_sdiv`, `(bs+6c + 2) / 3`) before testing `n <= 0` and returns either way; the CABAC statistics word of `slice_close` is stored at `0x5ffffc + 16 * slice_count` (literal `0x1100`, one word below table RAM for slice count 0); the stats padding loop writes five words per index (`+4..+0x14`), the fifth overlapping the next record, and for index 31 reaching `0x600204`.

**Frame end** (`frame_end`, `0x0f3c`): host `918 = pic+158 & 0x3f`; for CABAC `cabac_zero_words` (`ceil((bs+6c + 2)/3)` units of `00 00 03`, added to
the bit totals as `n*0x18`); bit totals; optional minimum-size padding (zero bytes until host `3d0[30:0] >> 3` bytes, with `bs+30 &= 0x1f` so the filler is not escaped);
host `a0c = frame_bits >> 3`, `a10 = a0c - padding`; `bs_flush_and_wait(ctx, 0)`; if not bypassed: `rate_buffer_update` (rate A) and `rate_thresholds_update`;
`report_mb_counts` (host `a00`); statistics header total at `0x600000` and `stats_table_dma` when host `348` bit 31 is set; `bs+44 = bs+b4 = 0`.

**Frame finish** (`frame_finish`, `0x0d10`): bs `+5c = 0`, read-and-write-back of bs `+60`, poll host `514` bit 1 until clear, host `50c = 1`,
`P+0xb8++`, `P+0xbc++`.

Latent limit (static): the backup store uses slot `stream + 4` and the slot array has eight slots, so only streams 0..3 stay inside the array;
`command_wait` accepts streams up to 8. The slot array is `0x2c0` bytes, so `stream_slot_store(stream + 4)` writes beyond it for streams 4..8, and stream 8 also overruns the `0x2a0`-byte result-record array. The iPod OS uses streams 0 and 1 only.

### Rate control functions

Described in [video-encoder.md](video-encoder.md#rate-control); the functions are `rate_defaults_init` (`0x2580`: defaults, then `rate_engine_regs_write`, `0x24c4`),
`rate_init_targets` (`0x2104`), `frame_rate_window_update` (`0x2320`), `rate_buffer_update` (`0x2628`), `rate_thresholds_update` (`0x26f8`),
`rate_state_load` / `rate_state_store` (`0x27d4`, `0x28fc`), `qp_to_qstep` (`0x2088`).
`qp_to_qstep(qp)` computes `table[qp % 6] * 2^(qp / 6)` as a double with a software floating-point loop (`_dscalb` once per power of two, `_dfix` for
the conversion); `rate_thresholds_update` then evaluates `0x3000 / (2 * (int) Qstep)` with signed division. **For QP 0..3 `(int) Qstep` is 0 and the division raises SIGFPE**
(dynamic experiment F). The library's default signal handler prints the message through the semihosting `SWI 0x123456`. On silicon that SWI enters the exception vector
and jumps through the RAM word at `0x5fc8`, which no code in the image initialises. The shortcut `avg == 0x10` (statistic at the floor) avoids the division.
Other traps and quirks (disassembly): `rate_init_targets` faults for host `904 == 0` or `mb_count == 0`; `qp_to_qstep` with a negative remainder would index before the table (unreachable, slice QP is masked to 0..63); `rate_state_load` reads record `+0x40` twice (for `mb_stat_thr` and `mb_stat_thr2`) and never loads `+0x44`; in `rate_buffer_update` the N flag of the add (not a signed compare) clamps the level to 0 and the `p_since_i` tests are unsigned; the fourth argument of `frame_rate_window_update` is `P->frame_count` and the engine QP (`+140`) is read back only when above 0; `rate_init_targets` with `idr_period == 0` uses divisor 9 and numerator 6 times the I target. The divisions are `__rt_sdiv` / `__rt_udiv` (`/` and `%` in the sources of each routine: `h908/0x73`, `h908/h904`, `h908/(h904<<3)`, the I/P target divisions, the buffer level `/0x1d1`, `q3/mb_count`, `q2*0x24/1000000`, `0x78/k`, `stat/mb_count`, `0x3000/(2*q)`).

## Commands 0, 3, 4 and 5

- **0, 3:** ignored (see above).
- **4** (`cmd4_teardown`, `cmd4_halt`): if the context pointer is non-null, `ctx_destruct` frees the slot array and the result records, `operator_delete` frees the context. `ctx_destruct` does not clear `slot_array`, `result_records_free` frees `ctx[0]` unconditionally, and `ctx_construct` does not check its allocations.
  The pointer word at `0x475c` is not cleared. Then host `510 = 1` and `b .` at `0x1e98`. Nothing returns to the loop; the host resets the engine. Dynamic: `10c != 0` with any
  command posted reaches the same loop with no frame produced.
- **5** (`cmd5_stream_update`): `cmd5_set_bypass(ctx, stream_byte)`: `ctx+0x288 = stream_byte` and host `50c = 1`, then host `a00 = 0xbad0cafe`. The stream index
  byte is stored **as the flag**: any non-zero value sets the global bypass, 0 clears it. In bypass the header skips rate defaults and rate targets; the frame set-up copies
  host `91c..944` to picture engine `+15c..+184` (words `928` and `938` ANDed with masks built from the frame type and the engine flag) and does not touch the rate state.
  Dynamic experiment E shows the resulting writes.


## Call tree

Direct calls (`bl`) and tail branches into another function; the entropy handlers are reached through the pointer table. A function already expanded earlier is marked.

```
reset_entry  (0x00cc)
  __main  (0x00f0)
    __scatterload  (0x00f8)
      __scatterload_zeroinit  (0x0130)
      __rt_entry  (0x2d64)
        __rt_lib_init_hook0  (0x3220)
        __rt_stackheap_init  (0x301c)
          __user_libspace  (0x2d58)
          __user_initial_stackheap  (0x34f4)
        __rt_lib_init_hook1  (0x3224)
        __rt_lib_init  (0x3118)
          __rt_errno_init  (0x44a8)
            __rt_errno_addr  (0x34c8)
          __Heap_DescSize  (0x2de4)
          __fp_init_veneer  (0x015c)
          _init_alloc  (0x2ac4)
            _sys_heap_extend  (0x2f64)
              __rt_heap_extend  (0x30ac)
            __rt_raise  (0x2dac)
              __default_signal_handler  (0x2fd0)
                __default_signal_display  (0x32f0)
                  _sys_writec  (0x34b0)
              _sys_exit  (0x2f74)
              __rt_div0  (0x2f8c)
            __Heap_Initialize  (0x2dd0)
            __Heap_ProvideMemory  (0x2f0c)
              __Heap_Free  (0x2ea0)
          _init_user_alloc  (0x2a74)
        __rt_lib_init_hook2  (0x3230)
          __rt_lib_init_hook3  (0x343c)
        main  (0x2018)
          command_loop  (0x1f98)
            ctx_create  (0x1f78)
              operator_new  (0x43a4)
                malloc  (0x2b5c)
                  __Heap_Alloc  (0x2dec)
                    __Heap_Extend  (0x2a78)
              ctx_construct  (0x0188)
                result_records_alloc  (0x20d0)
                __rt_memclr  (0x2b9c)
                ctx_select_stream  (0x016c)
              engine_init_registers  (0x1990)
            command_wait  (0x1f08)
            cmd1_header  (0x1ef8)
              encode_headers  (0x0bec)
                stream_params_defaults  (0x0a20)
                load_stream_params_from_host  (0x0924)
                compute_geometry  (0x0858)
                write_sps  (0x068c)
                  bs_put_bits  (0x4374)
                  bs_put_ue  (0x3894)
                  bs_trailing_bits  (0x0528)
                write_pps  (0x05c0)
                  bs_put_se  (0x3874)
                  bs_trailing_bits  (0x0528) ...
                rate_defaults_init  (0x2580)
                  rate_engine_regs_write  (0x24c4)
                rate_init_targets  (0x2104)
                  __rt_udiv  (0x34d8)
                  __rt_sdiv  (0x2bf4)
                stream_slot_store  (0x0368)
                rate_state_store  (0x28fc)
                bs_flush_and_wait  (0x025c)
            cmd2_frame  (0x1ee4)
              encode_frame  (0x18c4)
                frame_setup  (0x1834)
                  stream_slot_load  (0x1674)
                  rate_state_load  (0x27d4)
                  reload_frame_params  (0x3b88)
                  frame_type_select  (0x1610)
                  frame_engine_setup  (0x4084)
                    ref_planes_setup  (0x3f0c)
                  frame_rate_window_update  (0x2320)
                slice_type_select  (0x1550)
                slice_setup  (0x352c)
                  write_slice_header  (0x1340)
                  slice_limit_setup  (0x12a4)
                mb_loop_cavlc  (0x1ca8)
                  mb_fetch  (0x4060)
                  decode_mb_record  (0x3688)
                  vector_row_flush  (0x1c4c)
                    vector_row_pack  (0x1b5c)
                    dma_row_chunks  (0x1a7c)
                  mb_layer_cavlc  (0x3d04)
                    write_mb_type  (0x38b0)
                  mb_advance  (0x19d4)
                mb_loop_cabac  (0x1d88)
                  decode_mb_record  (0x3688) ...
                  vector_row_flush  (0x1c4c) ...
                  mb_advance  (0x19d4) ...
                slice_close  (0x1104)
                  bs_trailing_bits  (0x0528) ...
                frame_end  (0x0f3c)
                  cabac_zero_words  (0x0e8c)
                  bs_flush_and_wait  (0x025c) ...
                  rate_buffer_update  (0x2628)
                  rate_thresholds_update  (0x26f8)
                    qp_to_qstep  (0x2088)
                      _dscalb  (0x4518)
                        __fp_trap_nan  (0x43e8)
                          __fp_exception_result  (0x44bc)
                    _dfix  (0x43f8)
                  report_mb_counts  (0x0e64)
                  stats_table_dma  (0x0d60)
                frame_finish  (0x0d10)
                rate_state_store  (0x28fc) ...
            cmd4_teardown  (0x1e9c)
              ctx_destruct  (0x0228)
                free  (0x2b78)
                result_records_free  (0x20ec)
              operator_delete  (0x4398)
            cmd4_halt  (0x1e8c)
            cmd5_stream_update  (0x1ebc)
              cmd5_set_bypass  (0x0248)
        exit  (0x2f98)
          __rt_exit  (0x2d90)
            __rt_lib_shutdown_hook  (0x323c)
            __rt_lib_shutdown  (0x3208)
              _terminate_user_alloc  (0x2a70)
```

## Function catalogue

134 entries: the 112 functions found by automatic function discovery plus 22 more that discovery folds into a neighbour or does not reach
(`handler_unused_stub`, `__scatterload_zeroinit`, `encode_headers`, `encode_frame`, `engine_init_registers`, `dma_row_chunks`, `mb_loop_cavlc`,
`frame_rate_window_update`, `rate_engine_regs_write`, `result_records_free`, `__rt_entry`, `__rt_exit`, `__Heap_Alloc`, `__Heap_Free`, `_sys_exit`, `__rt_div0`, `exit`,
`default_terminate_handler`, `__default_signal_display`, `operator_delete`, `__fp_trap_nan`, `__fp_exception_result`). Three of the discovered functions are
stubs whose bodies the decompiler merges with them (`cmd1_header`/`encode_headers`, `cmd2_frame`/`encode_frame`, `frame_setup`/`frame_rate_window_update`). Size is the distance to the next entry
(literal pools and data words included, so a function followed by a pool looks larger than its code).

| Address | Size | Name | Role | Callers |
|---|---|---|---|---|
| `0x0000` | 4 | `vec_reset` | reset vector: b reset_entry (0xcc) | hardware reset |
| `0x0004` | 4 | `vec_undefined_instruction` | exception vector: b handler_undefined_stub | hardware |
| `0x0008` | 4 | `vec_software_interrupt` | exception vector: b handler_swi_stub (the C library's semihosting SWI 0x123456 lands here on hardware) | hardware / swi |
| `0x000c` | 4 | `vec_prefetch_abort` | exception vector: b handler_prefetch_abort_stub | hardware |
| `0x0010` | 4 | `vec_data_abort` | exception vector: b handler_data_abort_stub | hardware |
| `0x0014` | 4 | `vec_reserved_spin` | reserved vector slot: b . (spins) | hardware |
| `0x0018` | 4 | `vec_irq` | exception vector: b handler_irq_stub | hardware |
| `0x001c` | 8 | `vec_fiq` | exception vector: b handler_fiq_stub | hardware |
| `0x0024` | 24 | `handler_undefined_stub` | jump through the RAM word at 0x5fc4 (no firmware code ever writes it) | `vec_undefined_instruction` |
| `0x003c` | 24 | `handler_swi_stub` | jump through the RAM word at 0x5fc8 | `vec_software_interrupt` |
| `0x0054` | 24 | `handler_prefetch_abort_stub` | jump through the RAM word at 0x5fcc | `vec_prefetch_abort` |
| `0x006c` | 24 | `handler_data_abort_stub` | jump through the RAM word at 0x5fd0 | `vec_data_abort` |
| `0x0084` | 24 | `handler_irq_stub` | jump through the RAM word at 0x5fd8 | `vec_irq` |
| `0x009c` | 24 | `handler_fiq_stub` | jump through the RAM word at 0x5fdc | `vec_fiq` |
| `0x00b4` | 24 | `handler_unused_stub` | same stub shape using the RAM word at 0x5fe0; no vector branches here | (no vector) |
| `0x00cc` | 36 | `reset_entry` | reset target: calls __main (0xf0), then spins; __main never returns | `vec_reset` |
| `0x00f0` | 8 | `__main` | RVCT __main: branch to __scatterload | `reset_entry` |
| `0x00f8` | 56 | `__scatterload` | walk the scatter table at 0x474c ({src,dst,size,handler} entries) and call each handler; then enter __rt_entry (0x2d64) | `__main` |
| `0x0130` | 44 | `__scatterload_zeroinit` | scatter handler for a zero-init region: clears r2 bytes at r1 in 16-byte steps (the only entry: 0x60 bytes at 0x4760) | __scatterload (table handler) |
| `0x015c` | 16 | `__fp_init_veneer` | veneer: pc-relative jump to _fp_init (0x43e4) | `__rt_lib_init` |
| `0x016c` | 28 | `ctx_select_stream` | ctx+0x290 = stream; ctx+0x274 = &slot_array[stream] (slot size 0x58) | `ctx_construct`, `encode_headers`, `encode_frame` |
| `0x0188` | 160 | `ctx_construct` | build the encoder context: allocate result records and the 8 stream slots, zero S and P, set pointers, write ready marker 0x87301018 to host a00 | `ctx_create` |
| `0x0228` | 32 | `ctx_destruct` | free the stream-slot array and the result-record array | `cmd4_teardown` |
| `0x0248` | 20 | `cmd5_set_bypass` | command 5: ctx+0x288 = stream byte (non-zero bypasses firmware rate control); host 50c = 1 | `cmd5_stream_update` |
| `0x025c` | 268 | `bs_flush_and_wait` | end-of-NAL flush of the bitstream engine: 32 one-bits plus alignment filler, pad to an 8-word DMA burst, set +30 bit 5, wait for +dc/+f0 | `encode_headers`, `frame_end` |
| `0x0368` | 448 | `stream_slot_store` | copy 22 P fields into slot[idx] (slot = 0x58 bytes) | `encode_headers`, `encode_frame` |
| `0x0528` | 152 | `bs_trailing_bits` | write RBSP stop bit and zero alignment based on (+44 & 7) | `write_pps`, `write_sps`, `slice_close` |
| `0x05c0` | 204 | `write_pps` | emit picture parameter set NAL 0x68 (entropy flag, ref count, pic_init_qp) and record its bit length in ctx+0x2dc | `encode_headers` |
| `0x068c` | 460 | `write_sps` | emit sequence parameter set NAL 0x67 (profile, level, refs, size, crop) and record its bit length in ctx+0x2d8 | `encode_headers` |
| `0x0858` | 204 | `compute_geometry` | derive macroblock dimensions, padded size, macroblock count and crop from P+0/+4 | `encode_headers` |
| `0x0924` | 252 | `load_stream_params_from_host` | fill P from host 118/11c/300/308/310/330/3d4/810/80c/900 | `encode_headers` |
| `0x0a20` | 460 | `stream_params_defaults` | reset P and S to defaults (352x288, Main profile, QP 26, period 0, counters 0); host values overwrite most of them via load_stream_params_from_host | `encode_headers` |
| `0x0bec` | 292 | `encode_headers` | body of command 1: select stream, defaults, host load, geometry, SPS, PPS, a08, rate init, store slot, a0c, flush, 50c = 1 | `cmd1_header` |
| `0x0d10` | 80 | `frame_finish` | clear 0x52005c, wait host 514 bit 1 clear, 50c = 1, frame counters P+0xb8/+0xbc ++ | `encode_frame` |
| `0x0d60` | 260 | `stats_table_dma` | pad the 32-entry slice statistics table in 0x600000 and DMA it to host 348[30:0] | `frame_end` |
| `0x0e64` | 40 | `report_mb_counts` | host a00 = P+0xd8 \| P+0xd4<<16 (CAVLC) else 0 | `frame_end` |
| `0x0e8c` | 176 | `cabac_zero_words` | write ceil((+6c+2)/3) units of 00 00 03 for CABAC byte stuffing; returns the count | `frame_end` |
| `0x0f3c` | 456 | `frame_end` | end of picture: last QP to 918, CABAC zero words, min-size padding, a0c/a10, rate update, a00, header-bit total to 0x600000 | `encode_frame` |
| `0x1104` | 416 | `slice_close` | close a slice: trailing bits (CAVLC) or zeros (CABAC), bit totals, over-limit flag, statistics record | `encode_frame` |
| `0x12a4` | 156 | `slice_limit_setup` | program 0x550014 (slice size limit) from P+0x78/0x80 or the host table at 0x34c | `slice_setup` |
| `0x1340` | 528 | `write_slice_header` | emit one slice NAL header (start code, NAL byte, first_mb, type, frame_num, ref/marking flags, qp delta, deblocking) | `slice_setup` |
| `0x1550` | 192 | `slice_type_select` | per slice: choose slice type (I/P) from frame type, force masks 328/344 and over-limit flag; sets ctx+0x29c/+0x2a0/+0x2b8 | `encode_frame` |
| `0x1610` | 100 | `frame_type_select` | choose frame type 2 (IDR) / 1 (P) from counters and period, honour forced type 3d4 | `frame_setup` |
| `0x1674` | 448 | `stream_slot_load` | copy 22 fields slot[idx] -> P | `frame_setup` |
| `0x1834` | 144 | `frame_setup` | per-frame set-up: 52005c = 2,1; load slot; restore rate state; reload host params; pick type; program engine; tail-branch to frame_rate_window_update | `encode_frame` |
| `0x18c4` | 204 | `encode_frame` | body of command 2: slot select, frame_setup, backup slot, slice loop (type, header, MB handler, close), frame_end, frame_finish, slot store, rate-state store | `cmd2_frame` |
| `0x1990` | 68 | `engine_init_registers` | start-up: 0x550000 <- host 600>>1, +18/+24 <- host 120/124, +1c = 4, +20 = 1, +4c <- host 810 | `ctx_create` |
| `0x19d4` | 168 | `mb_advance` | advance MB index/count/column/row, request early stop via 0x2000a0, clear skip run | `mb_loop_cavlc`, `mb_loop_cabac` |
| `0x1a7c` | 224 | `dma_row_chunks` | DMA a vector row from table RAM to host c08 address in 0x80-byte chunks (body of vector_row_flush) | `vector_row_flush` |
| `0x1b5c` | 240 | `vector_row_pack` | pack the current MB's four vector words into table RAM at 0x6002a0+16*(mb%40) | `vector_row_flush` |
| `0x1c4c` | 92 | `vector_row_flush` | pack vectors for the MB; when a row of 40 MBs (or the last MB) is complete, tail-branch to dma_row_chunks | `mb_loop_cavlc`, `mb_loop_cabac` |
| `0x1ca8` | 224 | `mb_loop_cavlc` | entropy handler (CAVLC): fetch, end-marker test, decode record, syntax via mb_layer_cavlc, advance; flush trailing skip run | encode_frame (pointer 0x45c8) |
| `0x1d88` | 260 | `mb_loop_cabac` | entropy handler (CABAC): fetch, end-marker test, decode record, kick 0x520004 = 0x10000 per MB; final kick at slice end | encode_frame (pointer 0x45c8) |
| `0x1e8c` | 16 | `cmd4_halt` | command 4 end: host 510 = 1, then spin forever (host resets the engine) | `command_loop` |
| `0x1e9c` | 32 | `cmd4_teardown` | command 4: destroy the context (ctx_destruct, operator delete) | `command_loop` |
| `0x1ebc` | 40 | `cmd5_stream_update` | command 5 handler: cmd5_set_bypass(ctx, stream byte); a00 = 0xbad0cafe | `command_loop` |
| `0x1ee4` | 20 | `cmd2_frame` | command 2 handler stub: load ctx, tail-branch to encode_frame | `command_loop` |
| `0x1ef8` | 16 | `cmd1_header` | command 1 handler stub: load ctx, tail-branch to encode_headers | `command_loop` |
| `0x1f08` | 112 | `command_wait` | poll host 114 / 10c until a command (0..5, stream 0..8) arrives; returns command and stream | `command_loop` |
| `0x1f78` | 32 | `ctx_create` | operator new(0x2ec), ctx_construct, publish to 0x475c, then engine_init_registers | `command_loop` |
| `0x1f98` | 128 | `command_loop` | create the context, then loop: command_wait; dispatch 1 header, 2 frame, 4 stop, 5 update | `main` |
| `0x2018` | 112 | `main` | C main: a00 = 0xdeaddead, DMA mode 3 + start, command_loop; tail after it is dead code | `__rt_entry` |
| `0x2088` | 72 | `qp_to_qstep` | double Qstep(qp) = table[qp%6] * 2^(qp/6) via scalbn loop | `rate_thresholds_update` |
| `0x20d0` | 28 | `result_records_alloc` | allocate the 0x2a0-byte per-stream rate-state array (8 x 0x54) into *param | `ctx_construct` |
| `0x20ec` | 24 | `result_records_free` | free(*p) and return p: counterpart of result_records_alloc (no caller in the image) | `ctx_destruct` |
| `0x2104` | 540 | `rate_init_targets` | header-time rate set-up: I/P frame targets, per-MB targets, weights, buffer size and initial level, QP margin | `encode_headers` |
| `0x2320` | 420 | `frame_rate_window_update` | per-frame rate update (tail of frame_setup): targets by frame type, clamp QP, QP window to 0x55014c, weight/bits to 0x55015c, 0x550168, 0x550178 | `frame_setup` |
| `0x24c4` | 188 | `rate_engine_regs_write` | write 0x55015c, 0x168, 0x16c, 0x170, 0x178, 0x17c, 0x180, 0x184, 0x150 from rate defaults | `rate_defaults_init` |
| `0x2580` | 168 | `rate_defaults_init` | load rate-control defaults (0x46, 0x64, 0x32, 0x90 ...) into the context, read QP limits from host 90c, then fall into rate_engine_regs_write | `encode_headers` |
| `0x2628` | 208 | `rate_buffer_update` | firmware rate control A: update virtual buffer level, shortfall and banked overshoot, write level to 0x550160 | `frame_end` |
| `0x26f8` | 220 | `rate_thresholds_update` | end-of-frame thresholds: average MB statistic, Qstep based threshold 0x3000/(2*Qstep), write 0x550168/0x55016c | `frame_end` |
| `0x27d4` | 296 | `rate_state_load` | load 21-word rate state from result record [stream] into the context, rewrite 0x550160-0x55017c | `frame_setup` |
| `0x28fc` | 372 | `rate_state_store` | store the same 21 words back to result record [stream] | `encode_headers`, `encode_frame` |
| `0x2a70` | 4 | `_terminate_user_alloc` | empty RVCT hook | `__rt_lib_shutdown` |
| `0x2a74` | 4 | `_init_user_alloc` | empty RVCT hook | `__rt_lib_init` |
| `0x2a78` | 76 | `__Heap_Extend` | ask __rt_heap_extend for more memory and insert it as a free block | `__Heap_Alloc` |
| `0x2ac4` | 152 | `_init_alloc` | initialise the heap: libspace heap descriptor at base, free list covering [base, top) | `__rt_lib_init` |
| `0x2b5c` | 28 | `malloc` | malloc(size): tail-branch to __Heap_Alloc with the libspace heap | `ctx_construct`, `result_records_alloc`, `cpp_rtglobals_get`, `cpp_rtglobals_alloc_state`, `operator_new` |
| `0x2b78` | 36 | `free` | free(ptr): tail-branch to __Heap_Free with the libspace heap | `ctx_destruct`, `result_records_free`, `operator_delete` |
| `0x2b9c` | 88 | `__rt_memclr` | zero n bytes at p in 32-byte steps; returns end pointer | `ctx_construct` |
| `0x2bf4` | 356 | `__rt_sdiv` | signed 32/32 divide, quotient r0 remainder r1; divisor 0 -> __rt_div0 | `cabac_zero_words`, `vector_row_flush`, `qp_to_qstep`, `rate_init_targets`, `rate_thresholds_update` |
| `0x2d58` | 12 | `__user_libspace` | return address of the 0x60-byte library static data (0x4760) | `_init_alloc`, `malloc`, `free`, `__rt_stackheap_init`, `__rt_heap_extend`, `__rt_lib_init`, `cpp_rtglobals_get`, `__rt_errno_addr`, `operator_new` |
| `0x2d64` | 44 | `__rt_entry` | library entry after scatter load: stack/heap init, library init, main, exit | `__scatterload` |
| `0x2d90` | 28 | `__rt_exit` | library shutdown then _sys_exit | `exit` |
| `0x2dac` | 36 | `__rt_raise` | raise(sig,type): default handler; if it returns non-zero, _sys_exit | `_init_alloc`, `__rt_div0`, `abort` |
| `0x2dd0` | 20 | `__Heap_Initialize` | heap descriptor init: [0]=0 [1]=0 [2]=self | `_init_alloc` |
| `0x2de4` | 8 | `__Heap_DescSize` | returns 0x10 | `_init_alloc`, `__rt_lib_init` |
| `0x2dec` | 180 | `__Heap_Alloc` | first-fit allocation from the free list, 8-byte granules, splits blocks, extends via __Heap_Extend | `malloc` |
| `0x2ea0` | 108 | `__Heap_Free` | insert a block into the sorted free list with coalescing | `free`, `__Heap_ProvideMemory` |
| `0x2f0c` | 88 | `__Heap_ProvideMemory` | insert a free block (address-ordered) with coalescing | `__Heap_Extend`, `_init_alloc` |
| `0x2f64` | 12 | `_sys_heap_extend` | wrapper around __rt_heap_extend | `__Heap_Extend`, `_init_alloc` |
| `0x2f70` | 4 | `terminate_veneer` | branch to terminate | (unreferenced) |
| `0x2f74` | 24 | `_sys_exit` | semihosting SYS_EXIT (swi 0x123456, r0=0x18, r1=0x20026) | `__rt_exit`, `__rt_raise`, `abort` |
| `0x2f8c` | 12 | `__rt_div0` | divide-by-zero: __rt_raise(SIGFPE=2, type 2) | `__rt_raise` |
| `0x2f98` | 24 | `exit` | exit(code): shutdown and _sys_exit | `__rt_entry` |
| `0x2fb0` | 32 | `abort` | __rt_raise(SIGABRT), library shutdown, _sys_exit | `cpp_rtglobals_get`, `terminate` |
| `0x2fd0` | 76 | `__default_signal_handler` | default action for a signal: print the message and return 1 | `__rt_raise` |
| `0x301c` | 144 | `__rt_stackheap_init` | set up libspace (0x4760): clear 16 words, stack limit 0x150, call __user_initial_stackheap, returns heap/stack | `__rt_entry` |
| `0x30ac` | 108 | `__rt_heap_extend` | grow the heap towards the stack limit, returns bytes obtained | `_sys_heap_extend` |
| `0x3118` | 240 | `__rt_lib_init` | library init: errno, _fp_init, _init_alloc(heap), _init_user_alloc, clear libspace fields +0x20..+0x30 | `__rt_entry` |
| `0x3208` | 24 | `__rt_lib_shutdown` | library shutdown: _terminate_user_alloc | `__rt_exit`, `abort` |
| `0x3220` | 4 | `__rt_lib_init_hook0` | empty hook | `__rt_entry` |
| `0x3224` | 12 | `__rt_lib_init_hook1` | returns r0=-1, r1=-3 (ignored by __rt_entry) | `__rt_entry` |
| `0x3230` | 12 | `__rt_lib_init_hook2` | calls hook3 (empty) | `__rt_entry` |
| `0x323c` | 4 | `__rt_lib_shutdown_hook` | empty hook | `__rt_exit` |
| `0x3240` | 4 | `default_terminate_handler` | default C++ terminate handler: bl abort | `pointer stored by cpp_rtglobals_get` |
| `0x3244` | 112 | `cpp_rtglobals_get` | get or create the 0x20-byte C++ runtime block at libspace+0x3c (terminate handler, exception state) | `terminate` |
| `0x32b4` | 60 | `terminate` | std::terminate: call the handler or the default, then abort | `terminate_veneer` |
| `0x32f0` | 332 | `__default_signal_display` | body of __default_signal_handler: print \\n<signal text><detail text>\\n through _sys_writec | `__default_signal_handler` |
| `0x343c` | 4 | `__rt_lib_init_hook3` | empty hook | `__rt_lib_init_hook2` |
| `0x3440` | 112 | `cpp_rtglobals_alloc_state` | malloc(0x88) for the exception state, first byte cleared | `cpp_rtglobals_get` |
| `0x34b0` | 24 | `_sys_writec` | semihosting SYS_WRITEC (swi 0x123456, r0=3): print one character | `__default_signal_display` |
| `0x34c8` | 16 | `__rt_errno_addr` | address of errno (libspace+4) | `__rt_errno_init` |
| `0x34d8` | 28 | `__rt_udiv` | unsigned 32/32 divide, quotient r0 remainder r1 | `rate_init_targets` |
| `0x34f4` | 56 | `__user_initial_stackheap` | set SP of FIQ/IRQ/abort/undef/SVC modes to 0x5f00; returns r0 = heap base 0x47c0, r1 = 0x5f00 | `__rt_stackheap_init` |
| `0x352c` | 348 | `slice_setup` | start a slice: clear slice state, write slice header, 0x550080 deblocking, CABAC init words, 0x560004/0c/14, slice_limit_setup | `encode_frame` |
| `0x3688` | 492 | `decode_mb_record` | decode the 0x200000 record into S (type, intra, class, pred modes, skip, qp delta, cbp, vectors) | `mb_loop_cavlc`, `mb_loop_cabac` |
| `0x3874` | 32 | `bs_put_se` | write se(v) through port +0c: (value<<16)\|1 | `write_pps`, `write_slice_header`, `mb_layer_cavlc` |
| `0x3894` | 28 | `bs_put_ue` | write ue(v) through port +0c: value<<16 | `write_pps`, `write_sps`, `write_slice_header`, `mb_loop_cavlc`, `write_mb_type` |
| `0x38b0` | 728 | `write_mb_type` | macroblock type, skip run, reference indices and intra chroma mode for one CAVLC macroblock; counts intra/inter in P+0xd8/0xd4 | `mb_layer_cavlc` |
| `0x3b88` | 380 | `reload_frame_params` | per frame re-read of host 31c..348, 3cc/3d0/3d4, 900, 914, c08 into P | `frame_setup` |
| `0x3d04` | 520 | `mb_layer_cavlc` | CAVLC macroblock layer: skip kick, write_mb_type, motion kick 0x800, cbp via +0c mode 2/3, qp delta, residual kick 4 | `mb_loop_cavlc` |
| `0x3f0c` | 340 | `ref_planes_setup` | program reference/reconstruction plane addresses 0x55003c..48 or the tiled variant, 0x550088, 0x55008c.. | `frame_engine_setup` |
| `0x4060` | 36 | `mb_fetch` | fetch next macroblock record: +c8 = 1 and wait until 0, then +10 = MB address | `mb_loop_cavlc`, `mb_loop_cabac` |
| `0x4084` | 752 | `frame_engine_setup` | reset per-frame counters, choose entropy handler, ref counts, program 0x520000/0x550000/0x560000 for the frame | `frame_setup` |
| `0x4374` | 36 | `bs_put_bits` | write (value, count) through the raw-bit port: wait +34, +40 = value, +3c = count, +38 = 1 | `bs_flush_and_wait`, `bs_trailing_bits`, `write_pps`, `write_sps`, `cabac_zero_words`, `frame_end`, `slice_close`, `write_slice_header`, `write_mb_type` |
| `0x4398` | 12 | `operator_delete` | free(ptr) | `cmd4_teardown` |
| `0x43a4` | 64 | `operator_new` | malloc(size) with new-handler retry loop | `ctx_create` |
| `0x43e4` | 4 | `_fp_init` | empty: no floating-point set-up needed | `__fp_init_veneer` |
| `0x43e8` | 16 | `__fp_trap_nan` | load default NaN 0x7ff80000 and go to __fp_exception_result | `_dfix`, `_dscalb` |
| `0x43f8` | 176 | `_dfix` | double -> int32 conversion (truncate, saturate, NaN -> 0) | `rate_thresholds_update` |
| `0x44a8` | 20 | `__rt_errno_init` | errno = 0 | `__rt_lib_init` |
| `0x44bc` | 92 | `__fp_exception_result` | select the default result (NaN, +-Inf, 0, +-max) from the soft FP status word 0x040200c8 | `__fp_trap_nan` |
| `0x4518` | 584 | `_dscalb` | double scalbn(x, n): add n to the exponent field, handles zero/inf/overflow | `qp_to_qstep` |

## Data structures

All structures are in the heap and are addressed through the context pointer stored at `0x475c`. Offsets are observed from the field accesses; field names are ours.

### Context (`ctx`, `0x2ec` bytes, first heap allocation, observed at `0x47d8`)

| Offset | Name | Meaning |
|---|---|---|
| `+0x00` | `result_records` | pointer to the array of 8 per-stream rate-state records, `0x54` bytes each (`0x2a0` bytes, `result_records_alloc`) |
| `+0x04` | `rate_P_ptr` | pointer to the P record the rate functions work on (set by `frame_rate_window_update`, `rate_init_targets`) |
| `+0x08` | `rate_flag` | 1 when the engine rate flag (`P+0xc8`) is set; goes to picture engine `+178` bit 7 |
| `+0x0c`, `+0x10`, `+0x14`, `+0x18` | `rc_c0c..rc_c18` | defaults `0x78`, `2`, `8`, `1` loaded by `rate_defaults_init`; `+0x0c` -> `+184`, `+0x10` -> `+178` bits 15:8, `+0x14` -> `+168` bits 31:28, `+0x18` -> `+17c` bit 16 |
| `+0x20` | `buf_level` | virtual buffer level in bits; -> picture engine `+160` |
| `+0x24` | `host910_copy` | copy of host `910` |
| `+0x28` | `buf_size` | buffer size in bits; -> `+164` |
| `+0x2c`, `+0x48` | `mb_stat_thr`, `mb_stat_thr2` | average macroblock statistic, 0x10..0x3ff (default 0x46); -> `+168` bits 25:16 and 9:0 |
| `+0x30` | `cur_weight` | weight for the current frame type (`+0x34` I = 6, `+0x38` P = 9); -> `+15c` bits 31:16 |
| `+0x34`, `+0x38` | `i_weight`, `p_weight` | 6 and 9 |
| `+0x3c`, `+0x44` | `rc_c3c`, `rc_c44` | defaults 0 and `0x90`; -> `+180` |
| `+0x40` | `cur_bits_per_mb` | per-macroblock bit target for the current type; -> `+15c` low 16 bits |
| `+0x4c`, `+0x54` | `qstep_thr_a`, `qstep_thr_b` | `0x3000 / (2 * Qstep)` clamped to 0..0x1ff (default 100 and 0x32); -> `+16c` |
| `+0x50`, `+0x58`, `+0x5c`, `+0x60`, `+0x64`, `+0x68` | `rc_c50..rc_c68` | defaults 100, 0x30, 0x30, 0x20, 0x3a, 0x5a -> `+184`, `+17c`, `+178`, `+170`, `+17c` |
| `+0x6c`, `+0x70` | `qp_max`, `qp_min` | QP window (max from host `90c[13:8]`, min floor updated per frame) |
| `+0x74` | `rc_c74` | 0; -> `+168` bit 12 |
| `+0x78` | `cur_target_bits` | frame bit target for the current type |
| `+0x7c`, `+0x80` | `i_bits_per_mb`, `p_bits_per_mb` | per-macroblock targets |
| `+0x84`, `+0x88` | `i_target`, `p_target` | per-frame targets in bits |
| `+0x8c` | `qp_floor_margin` | QP floor margin (4, or `0x18 - log` form for small frames) |
| `+0x90`, `+0x94`, `+0x98` | `shortfall`, `buf_ceiling`, `buf_floor` | rate A buffer limits |
| `+0x9c`, `+0xa0`, `+0xa4` | `banked_overshoot`, `p_since_i`, `is_p_frame` | |
| `+0xa8` | `slot_array` | pointer to 8 stream slots of `0x58` bytes (`0x2c0` bytes) |
| `+0xac` | `S` | inline per-macroblock record, `0xe0` bytes |
| `+0x18c` | `P` | inline per-stream parameter record, `0xe8` bytes |
| `+0x274` | `cur_slot` | pointer to the current slot; written by `ctx_select_stream` and never read (`ctx+0x28c` is only zeroed by `ctx_construct`) |
| `+0x278`, `+0x27c` | `S_ptr`, `P_ptr` | pointers to S and P (the code always goes through them) |
| `+0x280`, `+0x284` | `mb_handler`, `mb_handler_adj` | pointer-to-member for the entropy handler |
| `+0x288` | `bypass` | command 5 flag |
| `+0x290` | `stream` | current stream index |
| `+0x294` | `tiled_layout` | host `600 >> 1`; non-zero selects the tiled reference layout in `ref_planes_setup` |
| `+0x298` | `frame_type` | 2 IDR, 1 P, 0 I (non-IDR) |
| `+0x29c` | `slice_type` | per slice, same coding |
| `+0x2a0` | `is_p` | 1 when the slice is P; written to reference engine `+04` bit 0 |
| `+0x2a4`, `+0x2a8` | `slice_count`, `mb_in_slice` | |
| `+0x2ac`, `+0x2b0`, `+0x2b4` | `mb_addr`, `mb_col`, `mb_row` | |
| `+0x2b8` | `refresh` | P slice without reference frames forced to refresh |
| `+0x2bc` | `skip_run` | pending `mb_skip_run` |
| `+0x2c0` | `has_refs` | 1 when `P+0x58` and `P+0x5c` are non-zero |
| `+0x2c4` | `over_limit` | frame bits exceeded the limit in `P+0xa4` |
| `+0x2c8`, `+0x2cc` | `slice_end`, `last_mb` | |
| `+0x2d0`, `+0x2d4` | `esc_bits_total`, `esc_bits_slice` | escape-byte bits |
| `+0x2d8`, `+0x2dc` | `sps_bits`, `pps_bits` | bit lengths of the two header NALs |
| `+0x2e0`, `+0x2e4`, `+0x2e8` | `frame_bits`, `slice_bits`, `slice_start_bits` | |

### Per-stream parameters (`P`, `ctx+0x18c`, `0xe8` bytes)

| Offset | Name | Source / meaning |
|---|---|---|
| `+0x00`, `+0x04` | `width`, `height` | host `118`, `11c` |
| `+0x08`, `+0x0c` | `padded_width`, `padded_height` | `mb_width*16`, `mb_height*16` |
| `+0x10`, `+0x14` | `crop_x`, `crop_y` | `(padded - size) >> 1` (2-pixel units) |
| `+0x18` | `idr_period` | host `308` |
| `+0x1c`, `+0x20` | `profile`, `level_idc` | host `300[1:0]`, `300 >> 2` (no mask; `write_sps` sends 8 bits) |
| `+0x24` | `entropy_flag` | host `310` bit 0 |
| `+0x28` | `host810` | host `810` |
| `+0x2c`, `+0x30` | `slice_qp`, `init_qp` | current QP (not loaded from the host; `write_pps` sets it to `init_qp`); host `900[5:0]` |
| `+0x34`, `+0x38`, `+0x3c` | `mb_count`, `mb_width`, `mb_height` | |
| `+0x40` | `p40` | default 1; CABAC init word field |
| `+0x44` | `cabac_init_idc` | host `310[3:2]` |
| `+0x48`, `+0x4c` | `force_type_en`, `forced_type` | host `3d4` |
| `+0x50`, `+0x54`, `+0x58`, `+0x5c` | `max_refs_m1`, `num_ref_active_m1`, `ref_cfg_en`, `ref_count` | host `330` (`+0x50` and `+0x54` get the same field `[11:8]` at header time; `+0x5c` is derived per frame) |
| `+0x60` | `ref_base` | host `80c` |
| `+0x64`, `+0x6c`, `+0x70` | `deblock_idc`, `alpha_off`, `beta_off` | host `314` |
| `+0x74`, `+0x78`, `+0x7c`, `+0x80` | `slice_mode`, `limit_mode`, `byte_thr`, `slice_limit` | host `31c`, `320`, `324 - 0x780`, `324` |
| `+0x84`, `+0x88` | `force_i_mask`, `force_refresh_mask` | host `328`, `344` |
| `+0x8c` | `crop_flag` | 1 when either size is not a multiple of 16 |
| `+0x90`, `+0x94` | `stats_en`, `stats_addr` | host `348` |
| `+0x98` | `frame_bits_rate` | frame bits handed to `rate_buffer_update` |
| `+0xa0`, `+0xa4`, `+0xa8`, `+0xac` | `bit_limit_en`, `bit_limit`, `pad_en`, `pad_bits` | host `3cc`, `3d0` |
| `+0xb0` | `qpd_bits` | accumulated `bs+44` deltas from before `write_mb_type` to after the `mb_qp_delta` se: the whole macroblock header (type, references, coded block pattern, qp delta); the last `bs+44` read follows the se port write without a poll, so the final code may not yet be counted |
| `+0xb8`, `+0xbc` | `frame_num`, `frame_count` | frames since the last IDR, total frames |
| `+0xc4`, `+0xc8` | `rc_a`, `rc_engine` | host `900` bit 9, bit 8 |
| `+0xcc`, `+0xd0` | `run_val`, `run_prev` | record running value `rec4 >> 26` and its previous value |
| `+0xd4`, `+0xd8` | `inter_cnt`, `intra_cnt` | macroblock counts reported in host `a00` (`intra_cnt` low half, `inter_cnt` high half) |
| `+0xdc` | `force_engine` | engine rate flag forced on |
| `+0xe0`, `+0xe4` | `vec_dump_en`, `vec_dump_addr` | host `c08` |

### Per-macroblock record (`S`, `ctx+0xac`, `0xe0` bytes)

| Offset | Name | From |
|---|---|---|
| `+0x00` | `cbp` | `rec4[25:20]` |
| `+0x04` | `intra` | `rec0` bit 6 |
| `+0x0c` | `mb_type` | `rec0[5:0]`; rewritten by `write_mb_type` with the type actually coded |
| `+0x10` | `type_class` | `rec0[9:7]` |
| `+0x24` | `qp_delta` | `rec4[19:14]` |
| `+0x2c..+0x48` | vectors | `rec+0x20..+0x3c` per partition (x at `+0x2c/+0x34/+0x3c/+0x44`, y at `+0x30/+0x38/+0x40/+0x48`); count by type class (1, 2, 2, 4) |
| `+0x4c..+0x8b`, `+0x8c..+0xcb` | arrays | two 16-word arrays, cleared by `stream_params_defaults` and never otherwise accessed |
| `+0xcc` | `chroma_pred` | `rec0[15:14]` |
| `+0xd0` | `i16_pred` | `rec0[13:12]` |
| `+0xd4` | `skip` | `rec0` bit 10 (also set to 1 for a forced refresh) |
| `+0xdc` | `vec_row` | row counter for the vector dump |

### Stream slot (`0x58` bytes, array of 8 at `ctx+0xa8`)

Written by `stream_slot_store`, read by `stream_slot_load`: 22 words copied between P and the slot. In slot order the P fields are
`+0x1c, +0x20, +0x24, +0x44, +0x18, +0x28, +0x50, +0x60, +0x38, +0x3c, +0x34, +0x00, +0x04, +0x08, +0x0c, +0x58, +0xc4, +0xc8, +0xdc, +0x30, +0xbc, +0xb8`
(slot offsets `0x00..0x54`). Slots `stream` hold the persistent state; slots `stream + 4` hold the pre-frame backup.

### Result record (`0x54` bytes, 21 words, array at `ctx[0]`)

The rate-control state per stream (buffer level, shortfall, ceiling, floor, I/P targets, per-macroblock targets, banked overshoot, P count, `910` copy, buffer size, weights,
thresholds, QP floor and window), written by `rate_state_store` and restored by `rate_state_load` in the order given in the result-record table of
[video-encoder.md](video-encoder.md#result-record-21-words-at-ctx0--stream0x54). It is firmware state, not a report to the host.

### Library static data (`libspace`, `0x4760`, `0x60` bytes)

`+0x04` errno; `+0x08` pointer to the heap descriptor (`0x47c0`); `+0x14` current heap top (observed `0x5298`); `+0x18` and `+0x1c` stack margin fields (`0x40`, `0x150`);
`+0x20..+0x30` cleared at library init (`+0x24 = 1`); `+0x34` new-handler pointer; `+0x3c` C++ run-time block (`cpp_rtglobals_get`); `+0x58`, `+0x5c` saved by `__rt_stackheap_init`.

### Literal tables

| Address | Content |
|---|---|
| `0x45c8` | `{0x1ca8, 0}`, `{0x1d88, 0}` entropy handlers |
| `0x45d8` | six doubles, Qstep mantissas for `qp % 6 = 0..5` |
| `0x4608` | 14 records of 23 bytes: Abnormal termination; Arithmetic exception: ; Illegal instruction; Interrupt received; Illegal address; Termination request; Stack overflow; Redirect: can't open: ; Out of heap memory; User-defined signal 1; User-defined signal 2; Pure virtual fn called; C++ library exception; Out of heap |
| `0x33c4..0x3424` | texts for the floating-point exception causes (Unknown signal, Invalid Operation, Divide By Zero, Overflow, Underflow, Inexact Result) and `: Heap memory corrupted` |
| `0x474c` | scatter table |
| `0x10f4` | `0x87301018` ready marker; `0x207c` `0xbad0cafe`; `0x2084` `0xdeaddead` |
| `0x44a4` | `0x040200c8`, the soft-FP status word (rounding and trap mask) |

## Library inventory

All of this is RealView library code that was linked into the image (identified by structure and message text; compare the shape of `__rt_entry`, `__scatterload`, `__user_libspace`, `_init_alloc`, `__rt_sdiv`).

| Group | Functions (address) |
|---|---|
| start-up | `__main` `0xf0`, `__scatterload` `0xf8`, `__scatterload_zeroinit` `0x130`, `__rt_entry` `0x2d64`, `__rt_stackheap_init` `0x301c`, `__user_initial_stackheap` `0x34f4`, `__user_libspace` `0x2d58`, `__rt_lib_init` `0x3118` and its hooks `0x3220`, `0x3224`, `0x3230`, `0x343c`, `_init_user_alloc` `0x2a74`, `_fp_init` `0x43e4` (+ veneer `0x15c`), `__rt_errno_init` `0x44a8`, `__rt_errno_addr` `0x34c8` |
| exit | `exit` `0x2f98`, `__rt_exit` `0x2d90`, `__rt_lib_shutdown` `0x3208` (+ hook `0x323c`, `_terminate_user_alloc` `0x2a70`), `_sys_exit` `0x2f74`, `abort` `0x2fb0` |
| heap | `malloc` `0x2b5c` -> `__Heap_Alloc` `0x2dec`; `free` `0x2b78` -> `__Heap_Free` `0x2ea0`; `__Heap_Extend` `0x2a78`; `_init_alloc` `0x2ac4`; `__Heap_Initialize` `0x2dd0`; `__Heap_DescSize` `0x2de4`; `__Heap_ProvideMemory` `0x2f0c`; `__rt_heap_extend` `0x30ac` (+ `_sys_heap_extend` `0x2f64`); `__rt_memclr` `0x2b9c`; first-fit free list, 8-byte granules, 4-byte block header, coalescing on free |
| C++ | `operator_new` `0x43a4` (malloc, retry through the new-handler), `operator_delete` `0x4398`, `terminate` `0x32b4` (+ veneer `0x2f70`), `default_terminate_handler` `0x3240`, `cpp_rtglobals_get` `0x3244`, `cpp_rtglobals_alloc_state` `0x3440` |
| signals and diagnostics | `__rt_raise` `0x2dac`, `__default_signal_handler` `0x2fd0`, `__default_signal_display` `0x32f0`, `__rt_div0` `0x2f8c`, `_sys_writec` `0x34b0` (semihosting `SWI 0x123456`, op 3), `_sys_exit` (op 0x18) |
| integer divide | `__rt_sdiv` `0x2bf4` (quotient r0, remainder r1; zero divisor -> `__rt_div0`), `__rt_udiv` `0x34d8` |
| software floating point | `_dfix` `0x43f8` (double to int, truncating), `_dscalb` `0x4518` (`scalbn`), `__fp_trap_nan` `0x43e8`, `__fp_exception_result` `0x44bc` |

The core has no divide instruction (ARMv4T); every division in the encoder (`rate_init_targets`, `rate_thresholds_update`, `cabac_zero_words`, `vector_row_flush`'s `mb % 40`) calls the library.
Only `qp_to_qstep` uses floating point. The library message texts `Invalid Operation`, `Divide By Zero`, `Overflow`, `Underflow` and `Inexact Result` are the causes the soft-FP layer can report.


## What the code tells us about the hardware

**The embedded core.** ARMv4T (ARM state only, `bx` used, no `clz`/`blx`/halfword accesses, no CP15), no cache or MMU control, no division instruction (software `__rt_sdiv`/`__rt_udiv`), no
floating-point unit (soft FP, used only for Qstep). It executes the uploaded image in place from address 0 and has at least 24 KiB of local RAM (highest touched address `0x5fe3`).
It never enables interrupts: every wait is a polling loop on an engine register, and the host-visible handshake is the polled doorbell `114`, command word `c0c`, abort `10c`
and the completion flag `50c`. Exceptions would jump through RAM pointers (`0x5fc4..0x5fe0`) that nothing initialises.

**Everything the core does is one of five things:** (1) format H.264 syntax elements (headers, slice headers, macroblock layer) through the bitstream engine ports; (2) copy host configuration into
engine registers; (3) read per-macroblock decisions from the record engine and sequence the engine with kicks; (4) run the rate-control arithmetic that sets targets and thresholds for the engine's
own per-macroblock QP adaptation; (5) move small tables to host memory with the DMA channel. Motion estimation, intra mode decision, transform/quantisation, residual entropy coding and
reconstruction are **not** in the firmware: they happen in hardware, triggered by kicks (`bs+04`), and the firmware never sees the coefficients.

**DMA channel.** The registers `000..030` (host offsets, same window as the host drives) are used by the core with mode 5 (local RAM -> system memory, 128-byte chunks, 2 words or 1 word at the tail).
Mode 3 is written once in `main` followed by a start; mode 7 appears only in dead code. The channel completes in a polled `030 == 1`. The write order, the sites and the chunk layouts are listed in [video-encoder.md](video-encoder.md#embedded-processor-local-ram-and-dma); every program-issued copy runs inside the `bs+5c` busy window while the mode-3 start is outstanding.

**Bitstream engine facts** (derived from use, see the table below): the raw-bit port takes `(value, count)` and needs `+34` ready; `+0c` is a combined Exp-Golomb port; the writer inserts emulation-prevention
bytes (`+78` counts them) only while `+30` bit 5 is set, and the firmware clears that bit for zero padding, for `00 00 03` CABAC units and for the end-of-NAL filler; every NAL is closed by 32 one-bits (plus 8, 16 or 24 more
depending on the alignment state `+68`) and a pad to an 8-word output burst; a macroblock record advances by a `+c8` write-1-poll-0 handshake; start codes are written as the 32-bit value 1 and **clamped to one escape** by `write_slice_header`
(`if (+78 != 0) +78 = 1` right after the start code), which suggests the engine counts the zero bytes of a start code as an escape candidate even though it does not insert one.

**Every register access in the firmware.** Columns: `R`/`W` = the firmware reads/writes the register; "Used by" lists the functions with an access (static scan of the image). Meanings are the
inferred roles; unknown bit meanings are in [video-encoder.md](video-encoder.md).

### Host register file mirror (`0x500000`, host `0x3e400000`)

Offsets are the host offsets.

| Offset | Name | R/W | Used by | Role |
|---|---|---|---|---|
| `+000` | HOST_DMA_START | W | `stats_table_dma`, `dma_row_chunks`, `main` | DMA start, written 1 |
| `+008` | HOST_DMA_MODE | W | `dma_row_chunks`, `main` | DMA mode: 3 in main (before the loop, one start), 5 for every local-to-system copy (rewritten for each transfer) |
| `+00c` | HOST_DMA_WORDS | W | `stats_table_dma`, `dma_row_chunks` | DMA length in words (32 for 128-byte chunks, 2 or 1 words in the statistics copy, a tail of `(len >> 2) % 32` words for a vector row, `len` = `0x280` or `(mb % 40 + 1) * 16`) |
| `+010` | HOST_OUT_ALIGN | W | `bs_flush_and_wait`, `main` | written 0 in the end-of-NAL flush, after `bs+30 &= 0x1f` and before the polls of `03c`/`038` (output alignment); also written in the dead tail of main |
| `+014` | HOST_DMA_SYS_ADDR | W | `stats_table_dma`, `dma_row_chunks` | DMA system address (host 348[30:0] statistics table, host c08[30:0] vector rows, plus chunk offsets) |
| `+020` | HOST_DMA_LOCAL_ADDR | W | `stats_table_dma`, `dma_row_chunks` | DMA local address (0x600000 table RAM + n*0x80) |
| `+030` | HOST_DMA_DONE | R | `stats_table_dma`, `dma_row_chunks` | DMA done, polled after every chunk with an exact compare against 1 (not a bit test, fresh read, no delay); never read before a start; `008/00c/014/020` are never read |
| `+038` | HOST_OUT_DMA_STATE | R | `bs_flush_and_wait` | output DMA state, polled until 0x80 or 1 in the flush (a second read only when the first is not 0x80) |
| `+03c` | HOST_OUT_FIFO_WORDS | R | `bs_flush_and_wait`, `main` | words pending in the 8-word output burst; `03c` is read twice before the padding loop (zero test, then `8 - n`); flush pads with 32-bit words of ones until it reads 0 |
| `+10c` | HOST_ABORT_REQ | R | `command_wait` | abort request: non-zero turns any wait into command 4 |
| `+114` | HOST_DOORBELL | R | `command_wait` | doorbell: command_wait reads c0c only while it is non-zero |
| `+118` | HOST_WIDTH | R | `load_stream_params_from_host` | picture width (header) |
| `+11c` | HOST_HEIGHT | R | `load_stream_params_from_host` | picture height (header) |
| `+120` | HOST_PIC_ENGINE_18 | R | `engine_init_registers` | copied to picture engine +18 at start-up |
| `+124` | HOST_PIC_ENGINE_24 | R | `engine_init_registers` | copied to picture engine +24 at start-up |
| `+300` | HOST_PROFILE_LEVEL | R | `load_stream_params_from_host` | profile (bits 1:0) and level_idc (bits above) -> SPS |
| `+308` | HOST_IDR_PERIOD | R | `load_stream_params_from_host` | IDR period |
| `+310` | HOST_ENTROPY_CFG | R | `load_stream_params_from_host` | bit 0 entropy mode (PPS flag, handler choice); bits 3:2 cabac_init_idc |
| `+314` | HOST_DEBLOCK_CFG | R | `reload_frame_params` | deblocking: idc [1:0], beta [6:2], alpha [11:7] -> slice header and picture engine +80 |
| `+31c` | HOST_SLICE_MODE | R | `reload_frame_params` | slice mode on/off |
| `+320` | HOST_SLICE_LIMIT_MODE | R | `reload_frame_params` | slice limit mode 0/1/2 |
| `+324` | HOST_SLICE_LIMIT | R | `reload_frame_params` | slice limit (mode 0) and byte threshold (minus 0x780) |
| `+328` | HOST_FORCE_I_MASK | R | `reload_frame_params` | per-slice force-I bit mask |
| `+32c` | HOST_ENGINE_OPT | R | `frame_engine_setup` | copied to picture engine +84 |
| `+330` | HOST_REF_CFG | R | `load_stream_params_from_host`, `reload_frame_params` | bit 31 reference config enable, bits 11:8 max reference frames - 1, bits 3:0 active reference count |
| `+334` | HOST_REF_WRITE0 | R | `ref_planes_setup` | copied to picture engine +8c (when ref config enabled) |
| `+338` | HOST_REF_WRITE1 | R | `ref_planes_setup` | copied to picture engine +90 |
| `+33c` | HOST_REF_READ0 | R | `ref_planes_setup` | copied to picture engine +94 |
| `+340` | HOST_REF_READ1 | R | `ref_planes_setup` | copied to picture engine +98 |
| `+344` | HOST_FORCE_REFRESH_MASK | R | `reload_frame_params` | per-slice force-refresh bit mask |
| `+348` | HOST_STATS_ADDR | R | `reload_frame_params` | bit 31 enable, [30:0] host address of the slice statistics table |
| `+3cc` | HOST_BIT_LIMIT | R | `reload_frame_params` | bit 31 enable, [30:0] per-slice bit limit |
| `+3d0` | HOST_MIN_FRAME_SIZE | R | `reload_frame_params` | bit 31 enable, [30:0] minimum frame size in bits (zero padding) |
| `+3d4` | HOST_FORCE_TYPE | R | `load_stream_params_from_host`, `reload_frame_params` | bit 0 force picture type, bits 2:1 the type |
| `+50c` | HOST_DONE_FLAG | W | `cmd5_set_bypass`, `encode_headers`, `frame_finish` | written 1 when a command completes (header, frame, command 5) |
| `+510` | HOST_STOP_FLAG | W | `cmd4_halt`, `main` | written 1 by the command 4 stop path just before the endless loop |
| `+514` | HOST_IRQ_STATUS | R | `frame_finish` | IRQ status; frame_finish polls bit 1 until clear |
| `+600` | HOST_PICSIZE_SRC | R | `engine_init_registers` | picture size source, >>1 into picture engine +0 |
| `+800` | HOST_INPUT_PLANE0 | R | `frame_engine_setup` | input plane 0 -> picture engine +30 |
| `+804` | HOST_INPUT_PLANE1 | R | `frame_engine_setup` | input plane 1 -> picture engine +34 |
| `+80c` | HOST_REF_BASE | R | `load_stream_params_from_host` | reference base -> P+0x60 (picture engine reconstruction planes) |
| `+810` | HOST_CFG_810 | R | `load_stream_params_from_host`, `engine_init_registers` | -> P+0x28, picture engine +4c |
| `+814` | HOST_SCRATCH_ADDR | R | `frame_engine_setup` | scratch address -> picture engine +190 |
| `+900` | HOST_QP_CTRL | RW | `load_stream_params_from_host`, `reload_frame_params`, `frame_engine_setup` | QP control: [5:0] QP, bit 8 engine rate flag, bit 9 firmware rate control A; frame set-up may set bit 8 |
| `+904` | HOST_RATE_904 | R | `rate_init_targets`, `frame_engine_setup` | -> picture engine +144 and rate_init_targets divisor |
| `+908` | HOST_RATE_908 | R | `rate_init_targets`, `frame_engine_setup` | -> picture engine +148 and rate_init_targets dividend |
| `+90c` | HOST_QP_LIMITS | R | `frame_rate_window_update`, `rate_defaults_init`, `frame_engine_setup` | QP limits (max [13:8], min [5:0]) -> picture engine +14c and ctx qp window |
| `+910` | HOST_RATE_910 | R | `rate_init_targets`, `rate_defaults_init`, `frame_engine_setup` | -> picture engine +150, ctx+0x24 |
| `+914` | HOST_CFG_914 | RW | `frame_engine_setup` | -> picture engine +154; written back from +154 |
| `+918` | HOST_LAST_QP | W | `frame_end` | written with the last QP (picture engine +158 & 0x3f) |
| `+91c` | HOST_BYPASS_91C | R | `frame_engine_setup` | bypass pass-through -> picture engine +15c |
| `+920` | HOST_BYPASS_920 | R | `frame_engine_setup` | bypass pass-through -> +160 |
| `+924` | HOST_BYPASS_924 | R | `frame_engine_setup` | bypass pass-through -> +164 |
| `+928` | HOST_BYPASS_928 | R | `frame_engine_setup` | bypass pass-through (ANDed with the computed mask) -> +168 |
| `+92c` | HOST_BYPASS_92C | R | `frame_engine_setup` | bypass pass-through -> +16c |
| `+930` | HOST_BYPASS_930 | R | `frame_engine_setup` | bypass pass-through -> +170 |
| `+934` | HOST_BYPASS_934 | R | `frame_engine_setup` | bypass pass-through -> +174 |
| `+938` | HOST_BYPASS_938 | R | `frame_engine_setup` | bypass pass-through (ANDed) -> +178 |
| `+93c` | HOST_BYPASS_93C | R | `frame_engine_setup` | bypass pass-through -> +17c |
| `+940` | HOST_BYPASS_940 | R | `frame_engine_setup` | bypass pass-through -> +180 |
| `+944` | HOST_BYPASS_944 | R | `frame_engine_setup` | bypass pass-through -> +184 |
| `+a00` | HOST_RESULT_A00 | W | `ctx_construct`, `report_mb_counts`, `cmd5_stream_update`, `main` | result word: stage marker 0xdeaddead, ready marker 0x87301018, MB counts (CAVLC) per frame, 0xbad0cafe after command 5 |
| `+a08` | HOST_HEADER_LENGTHS | W | `encode_headers` | header lengths: SPS bytes << 16 \| PPS bytes |
| `+a0c` | HOST_OUT_BYTES | W | `encode_headers`, `frame_end` | output bytes of the header or frame |
| `+a10` | HOST_OUT_BYTES_NOPAD | W | `frame_end` | output bytes minus padding (frame) |
| `+c08` | HOST_VECTOR_DUMP | R | `reload_frame_params` | bit 31 enable, [30:0] host address of the vector rows |
| `+c0c` | HOST_COMMAND_WORD | R | `command_wait` | command word: stream << 8 \| command |

### Bitstream engine (`0x520000`)

Block description: [video-encoder.md](video-encoder.md#block-0x00520000--bitstream-writer-and-entropy-coder-inferred).

| Offset | Name | R/W | Used by | Role |
|---|---|---|---|---|
| `+000` | BS_MODE_CTRL | W | `write_pps`, `write_sps`, `slice_setup`, `frame_engine_setup` | writer mode: 4 raw-bit mode, 1 frame start, 2 and 0x10 (CABAC slice init) at slice set-up |
| `+004` | BS_KICK | W | `mb_loop_cabac`, `write_mb_type`, `mb_layer_cavlc` | kick: 4 residual, 0x800 motion data, 0x80 intra prediction modes, 0x10000 CABAC macroblock / slice end |
| `+008` | BS_ENTROPY_MODE | W | `frame_engine_setup` | entropy mode: 4 CAVLC, 0x80 CABAC |
| `+00c` | BS_GOLOMB_PORT | W | `bs_put_se`, `bs_put_ue`, `mb_layer_cavlc` | Exp-Golomb port: value<<16 \| (0 ue, 1 se, 2 intra cbp code, 3 inter cbp code) |
| `+010` | BS_MB_ADDR | RW | `mb_fetch`, `frame_engine_setup` | macroblock address written per macroblock; low 3 bits cleared at frame start |
| `+014` | BS_PIC_SIZE | W | `frame_engine_setup` | padded width \| padded height << 16 |
| `+020` | BS_MB_DONE | R | `mb_loop_cabac`, `mb_layer_cavlc` | macroblock done, polled |
| `+030` | BS_WRITER_CTRL | RW | `bs_flush_and_wait`, `encode_headers`, `cabac_zero_words`, `frame_end` | writer control: 0x32 at header start, &0x1f before filler, \|0x20 after the flush (bit 5 = emulation prevention) |
| `+034` | BS_PORT_READY | R | `bs_put_se`, `bs_put_ue`, `mb_layer_cavlc`, `bs_put_bits` | port ready, polled before each port write: `!= 0` for the raw-bit port (`bs_put_bits`), `== 1` for the Golomb port (`bs_put_ue`, `bs_put_se`, `mb_layer_cavlc`) |
| `+038` | BS_BIT_STROBE | W | `bs_put_bits` | raw-bit strobe (write 1) |
| `+03c` | BS_BIT_COUNT | W | `bs_put_bits` | raw-bit count |
| `+040` | BS_BIT_VALUE | W | `bs_put_bits` | raw-bit value |
| `+044` | BS_BIT_TOTAL | RW | `bs_trailing_bits`, `write_pps`, `write_sps`, `encode_headers`, `frame_end`, `slice_close`, `mb_advance`, `slice_setup`, `mb_layer_cavlc` | bit counter (total bits since last clear, excluding escape bytes); written only with 0 (header NAL starts, frame end), read elsewhere |
| `+048` | BS_MB_PARAM_A | W | `slice_setup`, `mb_layer_cavlc` | CABAC slice init word, or CAVLC macroblock parameter (skip<<7 \| cbp<<1 \| flag) |
| `+04c` | BS_MB_PARAM_B | W | `slice_setup`, `mb_layer_cavlc` | CABAC slice init word, or CAVLC macroblock type |
| `+050` | BS_KICK_DONE | R | `write_mb_type`, `mb_layer_cavlc` | motion / intra kick done |
| `+054` | BS_SLICE_END_REQ | W | `mb_loop_cabac` | slice end request (0, then 1 before the final kick) |
| `+058` | BS_MB_COUNT | W | `frame_engine_setup` | macroblock count of the picture |
| `+05c` | BS_BUSY | W | `frame_finish`, `frame_setup` | busy: 2 then 1 at frame set-up, 0 at frame end |
| `+060` | BS_STATUS_60 | RW | `frame_finish` | status; frame_finish reads it and writes the value back (write-to-clear idiom) |
| `+068` | BS_ALIGN_STATE | R | `bs_flush_and_wait`, `cabac_zero_words` | alignment state 8 / 0x10 / 0x18 (bits pending) |
| `+06c` | BS_ALIGN_BYTES | R | `cabac_zero_words` | byte count that sizes the CABAC zero-word tail |
| `+070` | BS_REF_COUNT | W | `frame_engine_setup` | active reference count |
| `+078` | BS_ESCAPE_COUNT | RW | `write_pps`, `write_sps`, `encode_headers`, `frame_end`, `slice_close`, `write_slice_header`, `slice_setup` | escape-byte (emulation prevention) count; written only with 0 (header, slice start, frame end), clamped to 1 after a slice start code |
| `+080` | BS_DUMMY_READ_80 | R | `slice_close` | dummy read in slice_close (result discarded) |
| `+0a8` | BS_STATUS_A8 | R | `main` | status, read only in the dead tail of main |
| `+0b4` | BS_CLEAR_B4 | W | `encode_headers`, `frame_end` | written 0 at header start and frame end |
| `+0c8` | BS_FETCH_HANDSHAKE | RW | `slice_close`, `mb_fetch` | macroblock fetch handshake: write 1, poll until 0 |
| `+0f0` | BS_FLUSH_DONE_B | R | `bs_flush_and_wait` | flush done B, polled until 0 |

### Picture engine (`0x550000`)

Block description: [video-encoder.md](video-encoder.md#block-0x00550000--picture--macroblock-pipeline-parameters-inferred).

| Offset | Name | R/W | Used by | Role |
|---|---|---|---|---|
| `+000` | PE_PICSIZE_HALF | W | `engine_init_registers` | host 600 >> 1 |
| `+014` | PE_SLICE_LIMIT | W | `slice_limit_setup` | slice size limit (mode 0: host 324; mode 2: table; off or slice count `>= 0x1f`: 0xfff; mode 1 never writes it) |
| `+018` | PE_HOST_120 | W | `engine_init_registers` | host 120 |
| `+01c` | PE_CONST_4 | W | `engine_init_registers` | constant 4 |
| `+020` | PE_CONST_1 | W | `engine_init_registers` | constant 1 |
| `+024` | PE_HOST_124 | W | `engine_init_registers` | host 124 |
| `+030` | PE_INPUT_PLANE0 | W | `frame_engine_setup` | input plane 0 |
| `+034` | PE_INPUT_PLANE1 | W | `frame_engine_setup` | input plane 1 |
| `+03c` | PE_REC_PLANE0 | W | `ref_planes_setup` | reconstruction plane 0 |
| `+040` | PE_REC_PLANE1 | W | `ref_planes_setup` | plane 0 + N |
| `+044` | PE_REC_PLANE2 | W | `ref_planes_setup` | plane 0 + 2N |
| `+048` | PE_REC_PLANE3 | W | `ref_planes_setup` | plane 0 + 2.5N |
| `+04c` | PE_HOST_810 | W | `encode_headers`, `engine_init_registers` | host 810 |
| `+080` | PE_DEBLOCK | W | `slice_setup` | deblocking: idc \| (alpha&0xf)<<8 \| (beta&0xf)<<3 (host `314` gives 5-bit fields; only the low 4 bits of each arrive) |
| `+084` | PE_HOST_32C | W | `frame_engine_setup` | host 32c |
| `+088` | PE_REF_CFG | W | `ref_planes_setup` | ref config: enable<<31 \| max refs<<8 \| ref count |
| `+08c` | PE_REF_RW0 | W | `ref_planes_setup` | host 334 |
| `+090` | PE_REF_RW1 | W | `ref_planes_setup` | host 338 |
| `+094` | PE_REF_RW2 | W | `ref_planes_setup` | host 33c |
| `+098` | PE_REF_RW3 | W | `ref_planes_setup` | host 340 |
| `+140` | PE_QP_START | RW | `frame_rate_window_update`, `frame_engine_setup` | host 900 (low 6 bits read back as start QP) |
| `+144` | PE_HOST_904 | W | `rate_init_targets`, `frame_engine_setup` | host 904 |
| `+148` | PE_HOST_908 | W | `rate_init_targets`, `frame_engine_setup` | host 908 |
| `+14c` | PE_QP_WINDOW | W | `frame_rate_window_update`, `frame_engine_setup` | QP window: min \| max << 8 |
| `+150` | PE_HOST_910 | W | `rate_init_targets`, `rate_engine_regs_write`, `frame_engine_setup` | host 910 |
| `+154` | PE_HOST_914 | RW | `frame_engine_setup` | host 914 (0xf when forced engine rate) |
| `+158` | PE_LAST_QP | R | `frame_end` | last QP (read, low 6 bits) |
| `+15c` | PE_BITS_PER_MB_WEIGHT | W | `frame_rate_window_update`, `rate_engine_regs_write`, `frame_engine_setup` | bits per MB \| weight << 16 |
| `+160` | PE_BUFFER_LEVEL | W | `rate_init_targets`, `rate_buffer_update`, `rate_state_load`, `frame_engine_setup` | virtual buffer level |
| `+164` | PE_BUFFER_SIZE | W | `rate_init_targets`, `rate_state_load`, `frame_engine_setup` | buffer size |
| `+168` | PE_RC_THRESH_A | W | `frame_rate_window_update`, `rate_engine_regs_write`, `rate_thresholds_update`, `rate_state_load`, `frame_engine_setup` | threshold word A: ctx+0x14<<28 \| ctx+0x2c<<16 \| ctx+0x74<<12 \| ctx+0x48 |
| `+16c` | PE_RC_THRESH_B | W | `rate_engine_regs_write`, `rate_thresholds_update`, `rate_state_load`, `frame_engine_setup` | threshold word B: ctx+0x54 \| ctx+0x4c << 16 |
| `+170` | PE_RC_PARAM_170 | W | `rate_engine_regs_write`, `rate_state_load`, `frame_engine_setup` | rate parameter (default 0x0020003a) |
| `+174` | PE_FRAME_STAT | RW | `rate_thresholds_update`, `frame_engine_setup` | frame statistic (read at frame end) |
| `+178` | PE_RC_FLAGS | W | `frame_rate_window_update`, `rate_engine_regs_write`, `frame_engine_setup` | flags: ctx+0x10<<8 \| rate flag<<7 \| ctx+0x5c |
| `+17c` | PE_RC_PARAM_17C | W | `rate_engine_regs_write`, `rate_state_load`, `frame_engine_setup` | rate parameter (default 0x00015a30) |
| `+180` | PE_RC_PARAM_180 | W | `rate_engine_regs_write`, `frame_engine_setup` | rate parameter (default 0x90) |
| `+184` | PE_RC_PARAM_184 | W | `rate_engine_regs_write`, `frame_engine_setup` | rate parameter (default 0x6478) |
| `+190` | PE_HOST_814 | W | `frame_engine_setup` | host 814 |

### Reference engine (`0x560000`)

Block description: [video-encoder.md](video-encoder.md#block-0x00560000--reference-and-prediction-set-up-inferred).

| Offset | Name | R/W | Used by | Role |
|---|---|---|---|---|
| `+000` | PR_PAD_SIZE | W | `frame_engine_setup` | padded width \| padded height << 11 |
| `+004` | PR_TYPE_PROFILE | W | `slice_setup`, `frame_engine_setup` | is_P \| profile << 4 |
| `+008` | PR_REFSET | W | `frame_engine_setup` | 0x1000000 (even frame) or 1 (odd): alternating reference set |
| `+00c` | PR_SLICE_FLAGS | RW | `slice_setup`, `frame_engine_setup` | is_P; per slice read-modify-write (low nibble kept, bit 4 = refresh) |
| `+014` | PR_CTRL | W | `frame_engine_setup` | 3 at frame start; 2 per slice from the second macroblock on in slice mode |

### Macroblock record engine (`0x200000`)

Block description: [video-encoder.md](video-encoder.md#block-0x00200000--mode-and-motion-engine-output-inferred).

| Offset | Name | R/W | Used by | Role |
|---|---|---|---|---|
| `+000` | MBREC_REC0 | R | `mb_loop_cavlc`, `mb_loop_cabac`, `decode_mb_record` | MB record word 0: type, intra, class, skip, prediction modes; 0xffffffff = end of slice |
| `+004` | MBREC_REC4 | R | `slice_close`, `mb_loop_cavlc`, `mb_loop_cabac`, `decode_mb_record` | MB record word 1: qp delta, cbp, running value |
| `+008` | MBREC_REC8 | R | `slice_close` | MB record word 2: copied into the statistics table |
| `+020` | MBREC_VEC0 | R | `decode_mb_record` | MB vector component 0 (x/y pairs per partition) |
| `+030` | MBREC_VEC4 | R | `decode_mb_record` | MB vector component 4 (x/y pairs per partition) |

### Table RAM (`0x600000`)

Block description: [video-encoder.md](video-encoder.md#table-ram-0x00600000-inferred).

| Offset | Name | R/W | Used by | Role |
|---|---|---|---|---|
| `+000` | TAB_HEADER_BITS | W | `frame_end` | header bit total (SPS + PPS bits); statistics table follows at +4 |

Accesses the scanner cannot attribute (a base register copied through a function call) are listed in the function pages above; the table is complete for the offsets named. The firmware reads bs `+80` once and discards the result; it
writes bs `+60` with the value it just read (a write-to-clear idiom or a barrier).


## Dynamic analysis

The original image was executed instruction by instruction (ARM state, SVC mode, local RAM at address 0, image loaded unchanged). Everything outside the core was modelled by simple blocks whose behaviour follows
from how the firmware uses them; the results below are therefore statements about **what the firmware does given these inputs**, not about silicon.

Models used:

- *Host register file* (`0x500000`): a plain register store, the host's values placed before a command. Reading `c0c` clears `114` (doorbell consumed). `030` reads 1 (DMA done), `038` reads `0x80`, `03c` reads the number of words pending in an 8-word burst of the output bytes written so far. Mode-5 DMA copies from table RAM into a sparse host memory.
- *Bitstream engine* (`0x520000`): raw-bit port (`+3c`, `+40`, `+38`), Exp-Golomb port `+0c` (ue, se, and the two coded-block-pattern code tables for intra and inter), bit counter `+44`, escape counter `+78` and emulation prevention gated by `+30` bit 5 (a 32-bit write of 1 is written as a start code without escaping); `+34`, `+20`, `+50` always ready; `+c8` write-1 reads back 0 and fetches the next macroblock record.
- *Macroblock record engine* (`0x200000`): a configurable sequence of records (all skip, Intra16x16 with DC prediction and no residual, P_L0_16x16 with zero motion, end-of-slice marker `0xffffffff`). Kicks produce only what is computable: kick `0x800` writes zero motion vector differences, kick 4 on a coded-block-pattern-0 Intra16x16 macroblock writes the one-bit `coeff_token` (TotalCoeff 0). Macroblocks with residual are not modelled.
- *Picture and reference engines*: register stores with the last QP read back from the written start QP and the frame statistic fixed.

The bytes the firmware produced (header and picture NALs) were decoded with an independent H.264 parser/decoder; all streams listed as valid decoded without error.

### A. Start-up to the command loop

The core reaches the first `command_wait` call after executing the scatter load, library init, `main` and context creation. State at that point: context `0x47d8`; result records `0x4ac8`; slot array `0x4d70`; S `0x4884`; P `0x4964`; heap `0x47c0..0x5298`; `sp = 0x5ee0`;
`libspace` words `[2] = 0x47c0, [4] = 1, [5] = 0x5298, [6] = 0x40, [7] = 0x150, [9] = 1`. Host writes in order: `a00 = 0xdeaddead`, `008 = 3`, `000 = 1`, `a00 = 0x87301018`; reads: `600`, `120`, `124`, `810`; picture-engine writes `+00, +18, +1c = 4, +20 = 1, +24, +4c`. `510` is not written at boot. The wait loop polls `114` and `10c` only.

### B. Command 1 with the preset register images (640x480 preset 5 and 6, 32x32 preset 11)

Preset 5 (640x480, host `300 = 0x78`, `308 = 0x1e`, `900 = 0x124`, `330 = 0x80000000`): `a08 = 0x000d0009`, `a0c = 22`:

```
00 00 00 01 67 42 00 1e 8d 68 0a 03 d9      SPS (13 bytes)
00 00 00 01 68 ce 02 9c 80                  PPS (9 bytes)
```

The parser reads SPS: `profile_idc 66`, constraint flags 0, `level_idc 30`, `sps_id 0`, `log2_max_frame_num_minus4 12`, `pic_order_cnt_type 2`, `max_num_ref_frames 1`, no gaps, 40x30 macroblocks, `frame_mbs_only 1`,
`direct_8x8_inference 1`, no cropping, no VUI. PPS: `pps_id 0`, `entropy_coding_mode 0`, `pic_init_qp_minus26 10` (QP 36), `deblocking_filter_control_present 1`, all other fields 0. Preset 6
(start QP 0) differs only in `pic_init_qp_minus26 = -26`; preset 11 (32x32, level 2.0) gives a 12-byte SPS and 8-byte PPS (`a08 = 0x000c0008`). After `a0c` bytes the model's flush writes `ff` filler (8 bytes for preset 5, 12 for preset 11).
The command ends with `50c = 1`, `a00` unchanged (`0x87301018`).

### C. Header variants (host registers varied one at a time from preset 5)

| Change | Result |
|---|---|
| `300[1:0] = 1` | `profile_idc 77` (Main), SPS same length |
| `300[1:0] = 2` | `profile_idc 100` (High), 14-byte SPS with `chroma_format_idc 1`, 8-bit depth, no bypass, no scaling matrix; decodes |
| `300[1:0] = 3` | **no profile_idc / constraint field**: 11-byte SPS `67 1e 8d ...`, decoder reads `profile_idc = 30`; malformed |
| `300 >> 2` | goes into `level_idc` unchanged (tested 30, 51) |
| `310 = 1` (CABAC) | only the PPS differs: `entropy_coding_mode 1` (first PPS payload byte `ee` instead of `ce`); `310[3:2]` does not appear in the headers |
| `330[11:8] = 2, 15` | `max_num_ref_frames 3, 16` and `num_ref_idx_l0_default_active_minus1 2, 15` (the SPS value is the register field + 1, the PPS value the field itself) |
| `900[5:0] = 0, 51, 63` | `pic_init_qp_minus26 -26, 25, 37` (no range check; 63 gives QP 63 which is outside H.264) |
| `118/11c` = 636x478 | 40x30 macroblocks, cropping right 2, bottom 1 (2-pixel units) |
| 641x481 | 41x31 macroblocks, crop right 7, bottom 7 |
| 1x1, 17x17 | 1x1 and 2x2 macroblocks, crop right 7 and bottom 7 (the crop is `(padded - size) >> 1`, 15 pixels cannot be expressed exactly) |
| 1280x720, 320x240, 4096x4096 | 80x45, 20x15, 256x256 macroblocks (no size limit check; level not adjusted) |
| `314`, `308` | no effect on SPS/PPS (deblocking appears only in slice headers) |

### D. Command 2 on a 32x32 picture (2x2 macroblocks, preset 11 registers)

Frame 0 is always type 2 (IDR). With four Intra16x16 DC macroblocks without residual:

```
00 00 00 01 65 b8 00 04 f2 72 72 72 78        (13 bytes, a00 = 4)
```

Slice header decode: `nal_unit_type 5`, `first_mb_in_slice 0`, `slice_type 2` (I), `frame_num 0`, `idr_pic_id 0`, `no_output_of_prior_pics 0`, `long_term_reference 0`, `slice_qp_delta 0`,
`disable_deblocking_filter_idc 0`, alpha/beta 0. Per macroblock the firmware wrote `mb_type` as ue (3 = I16x16, DC, no coded block pattern), `intra_chroma_pred_mode` ue(0), `mb_qp_delta` se(0); the model supplies the DC `coeff_token` bit.
An all-skip P frame is `00 00 00 01 61 e0 00 23 cb` (9 bytes: `nal_unit_type 1`, `slice_type 0` (P), `frame_num 1`, `num_ref_idx_active_override 0`, list modification 0, marking 0, then `mb_skip_run 4` as ue and the stop bit). A P frame with one P_L0_16x16
macroblock among skips gives 10 bytes (`mb_skip_run 2`, `mb_type 0`, motion vector difference `1 1`, coded block pattern code for inter pattern 0, trailing skip run 1) and `a00 = 0x10000`. Decoding the three-frame stream gives pictures of constant value 128 (DC prediction without neighbours or residual), without errors.
Other facts from the same runs: the period (`308 = 14`) gives frame types `2, 1 x 14, 2, ...` and `frame_num` `0..14, 0..`; with slice mode on (`31c = 1`, `320 = 0`, `324 = 0x4b0`) and an end marker record after two macroblocks the picture is two slices (`first_mb_in_slice` 0 and 2), each with its own start code and header; intra prediction modes
2, 1, 0, 3 on the four macroblocks decode correctly (the firmware writes the mode only through the record-driven `mb_type` formula `1 + pred + 4*chroma_cbp + 12*(luma_cbp != 0)`).

### E. Commands 0, 3, 4, 5 and the abort flag

- Commands 0 and 3 return to the wait loop with no output and `50c` unchanged.
- Command 5 with stream byte 0, 1, 2, 5 sets `ctx+0x288` to that byte, writes `a00 = 0xbad0cafe` and `50c = 1`. With the flag set, a following header and picture skip the rate-control set-up, and the picture engine receives `15c..184` straight from host `91c..944` (`168 = 0xffffefff`, `178 = 0xffffff7f` after masking in the case tried).
- Command 4 frees the context and ends at `b .` (`0x1e98`) with host `510 = 1`; the pointer word `0x475c` still holds the old context address.
- `10c = 1` with a frame command posted ends in the same loop with no bitstream output.
- Stream index 9 with command 1: the wait loop keeps polling (20 000 instructions later still in `0x1f08..0x1f77`, no output).

### F. Rate control across frames (64x64 pictures, 16 macroblocks; frame sizes fed by the model)

| Case | Result |
|---|---|
| preset 5 registers, `904 = 0x1001e` as the host writes it (bit 16 set), engine rate flag only | `rate_init_targets` divides the bit rate by `0x1001e` (65566) instead of 30: targets `98 / 36` bits per I/P frame, buffer level `194`; level constant in every frame; thresholds `168 = 0x80400040`, `16c = 0x990099` after the first frame |
| same with `904 = 0x1e` | targets `215276 / 78934` bits per I/P frame (the average frame at `2,500,000` bit/s and 30 fps is 83,333 bits), buffer size `1,604,144`, level `27598`; engine rate flag only: level never moves |
| firmware rate control A (`900 = 0x324`), I frame at 60% of target, P frames alternating 60% / 140% | after the first (undersized) I frame the level is clamped up to the floor `78934` (= P target) and the shortfall counter takes the difference (`78934`); afterwards `level = max(level + bits - target, 0)` is applied per frame and picture-engine `+160` follows it; QP window `14c = 0x3316` (min 22, max 51) from the first frame, `15c` switches from the I weight word (`0x6348e`) to the P word (`0x91345`) |
| A with P frames at 140% of target | the level climbs by about 31,600 bits per frame to the ceiling `236802` (3 x P target) at frame 5, is then held at the ceiling while the excess repays the accumulated shortfall (`51336` to `0` over frames 5..7), and after that is no longer limited |
| preset-6 image (`900 = 0x300`, QP 0) | first frame ends in the library's divide-by-zero path: semihosting output `\nArithmetic exception: Divide By Zero\n`, `SYS_EXIT` (the picture's header bytes were already produced) |
| same with start QP 4 (`900 = 0x304`) | runs: thresholds `16c = 0x1ff01ff`, QP window min 4 |

The first row is a finding about the *register image the firmware sees*: if the engine's mirror of host `904` carries bit 16 as written, the bit-rate-per-frame arithmetic is wrong by a factor of two thousand; whether the mirror masks the bit is open.

### G. Per-frame options

| Host setting | Observed |
|---|---|
| `314 = 1` | slice header `disable_deblocking_filter_idc 1`, no offsets |
| `314 = (5 << 7) \| (0x1b << 2) \| 2` | idc 2, `slice_alpha_c0_offset_div2 5`, `slice_beta_offset_div2 -5` (5-bit signed field) |
| `3d4 = 1` (type 0), `= 3` (type 1), `= 5` (type 2) on every frame | non-IDR I slice (NAL 1, `slice_type 2`, `frame_num` counting), P slice, IDR again (`frame_num 0`) |
| `3d0 = 0x80000000 \| 1600` | every frame padded with zero bytes to 200 bytes; `a0c = 200`, `a10 = 13` (IDR) or 9 (P) |
| `348` enabled, two slices | 40 two-word DMAs of the header total to the table address, then the table in four 128-byte chunks and a final word; first record: header bits `0xa0`, slice 0 macroblock count `4`, bits used `0x48` |
| `c08` enabled | one vector row per picture: skip macroblocks give `0x00008000` (skip flag), the P macroblock `01230111` = `(type & 7) << 28 \| (v20 & 0x3ff) << 16 \| v30 & 0x1ff` |
| CABAC (`310 = 1`), IDR | firmware output is the 9-byte NAL header only (the macroblock data is the engine's); engine writes `+08 = 0x80`, `+48 = 0x28`, `+4c = 0x800254`, `+00 = 0x10`, then four kicks `0x10000` (one per macroblock) and a final kick with `+54 = 1` |
| CABAC P, `310[3:2] = 2` | slice header contains `cabac_init_idc 2` (P only) |

### H. Register coverage

Across the runs above (header, I/P pictures, slice modes 0, 1 and 2, statistics and vector DMAs, CABAC, rate control A and engine-only, command 5 bypass, command 4, tiled and non-tiled reference layouts) the executed code touched every register offset in the access tables of the previous section except `bs+a8` and the table-RAM header word (the first is in dead code).


## The 1.0.1 program

The iPod OS 1.0.1 uploads a `0x4624`-byte program. Structurally it is the same program; the differences below come from matching its functions against this one (functions matched by instruction sequence and call structure, then checked by hand).

| Aspect | 1.0.1 | This program |
|---|---|---|
| Library | identical routines at +0x94 (+0x90 after `0x3d04`) | - |
| Scatter table / `libspace` / context pointer | `0x45e0` / `0x4624` / `0x45f0` | `0x474c` / `0x4760` / `0x475c` |
| Context offsets from `0x274` up | +0x1c (P pointer `ctx+0x298`, S `ctx+0x294`, bypass `+0x2a4`, stream `+0x2ac`, frame type `+0x2b4`) | `0x27c`, `0x278`, `0x288`, `0x290`, `0x298` |
| Streams accepted | 0..4, slot `stream`, **no pre-frame backup store** | 0..8, backup at `stream + 4` |
| Rate control | integer only: `rate_thresholds_update` and the buffer update are one function (`0x26a0`), the threshold is `(P+0xe4 << 8) / average statistic`; a 16-byte-entry table at `0x45f4` (three entries, index `P+0xd4`) feeds `ctx+0x58..0x64`; **no Qstep table, no floating point, no `_dfix`/`_dscalb`** | Qstep via soft floating point (divide by zero for QP 0..3) |
| Result words | `a04` = statistic / macroblock count (function `0x2840`) | `a10` = bytes without padding |
| `frame_finish` | no host `514` poll | polls bit 1 |
| Entropy handler table | `0x448c` `{0x1c9c, 0}, {0x1d7c, 0}` | `0x45c8` |
| Parameters read at frame set-up | also host `c00`, `c04` | not read |


## Open questions

- Which event clears host `114`: the read of `c0c` or the write of `50c`? The firmware never writes `114`.
- What does the engine's mirror of host `904` contain: the value as written (with `0x10000`), or the masked field? It decides whether `rate_init_targets` is sane for the preset register images.
- Do the core's mode-5 copies to system memory depend on the busy window (`bs+5c == 1`) or the programmed frame configuration, and does `030` stay 1 after a transfer until the next start? The program never reads `030` before a start.
- Does the silicon accept profile 1 / 2 (Main / High) and CABAC (`310` bit 0) set by the host? The firmware path exists and produced valid headers; whether the engine's entropy coder, transform and prediction support them is unknown (the iPod OS always uses Baseline CAVLC).
- What does each of the unmodelled engine status registers mean (`bs+20`, `+30` low bits, `+50`, `+dc`, `+f0`, `+60`) and which bits does `514` raise for each firmware write of `50c`?
- What do DMA modes 3 and 7 do, and what does the `main` write of mode 3 plus start (without a programmed length) trigger?
- What happens on silicon when the core executes the C library's semihosting `SWI` (QP 0..3 divide-by-zero): the vector jumps through the RAM word `0x5fc8`.
- Slice limit mode 0: what unit does `0x550014` count, and which condition makes the engine return the end-of-slice record?
- Where the bit-counter `+44` / escape counter `+78` pair overflows for very large pictures (the 16-bit result fields `a08` hold lengths in bytes).


## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| 134 function entries; library routines identified (RealView `__main`, `__scatterload`, `__rt_entry`, heap, `__rt_sdiv/udiv`, signals, soft FP) | `firmware 0x00f0..0x4518`, message texts `0x33c4`, `0x4608` | static |
| Start-up path: reset -> `__main` -> `__scatterload` (zero-init `0x60` bytes at `0x4760`) -> `__rt_entry` -> `main` -> `command_loop` | `firmware 0x0000`, `0x00cc`, `0x00f8`, `0x2d64`, `0x2018` | static; dynamic (A) |
| One stack of top `0x5f00` shared by all modes; heap starts `0x47c0`, reaches `0x5298` | `firmware 0x34f4`, `0x30ac` | static; dynamic (A) |
| `a00 = 0xdeaddead` is written first (in `main`), `a00 = 0x87301018` second (in `ctx_construct`) and remains the visible value while idle | `firmware 0x2024`, `0x019c` | static; dynamic (A) |
| The tail of `main` (host `010`, mode 7, `510 = 1`) is unreachable | `firmware 0x2038..0x2074`, `0x1f98`, `0x1e98` | static; dynamic (A, E) |
| Command 4 writes `510 = 1` then loops at `0x1e98`; command 5 stores the stream byte as the bypass flag | `firmware 0x1e8c`, `0x1ebc`, `0x0248` | static; dynamic (E) |
| `10c != 0` forces command 4 without the doorbell; streams >= 9 and commands >= 6 are never accepted; commands 0 and 3 are ignored | `firmware 0x1f08`, `0x1f98` | static; dynamic (E) |
| Header output matches H.264 syntax for Baseline/Main/High, CAVLC and CABAC; `a08`/`a0c` as documented | `firmware 0x068c`, `0x05c0`, `0x0bec` | dynamic (B, C) with parser |
| `300[1:0] = 3` omits profile_idc/constraint bytes from the SPS | `firmware 0x068c..0x0710` | static; dynamic (C) |
| Picture output (IDR/P, skip, I16x16, P_L0_16x16, two slices, forced types, deblocking fields) decodes without error | `firmware 0x1340`, `0x3d04`, `0x38b0`, `0x1ca8` | dynamic (D, G) |
| `a00` low half counts intra macroblocks, high half non-skipped inter macroblocks  | `firmware 0x0e64`, `0x38b0` | static; dynamic (D) |
| Frame type sequence: IDR every `period + 1` frames, `frame_num` 0..period | `firmware 0x1610`, `0x0d10` | static; dynamic (D) |
| Backup store at slot `stream + 4` and the 8-slot array limit usable streams to 0..3; result records limit to 0..7 | `firmware 0x18e4`, `0x0188`, `0x20d0` | static (arithmetic) |
| Rate control: engine-only mode keeps the buffer level constant; firmware mode A follows the level/shortfall/ceiling rules | `firmware 0x2628`, `0x2104`, `0x26f8` | dynamic (F) |
| `(int) Qstep(QP) == 0` for QP 0..3 and `0x3000 / (2 * 0)` raises the library divide-by-zero fault | `firmware 0x2088`, `0x26f8`, `0x2f8c` | static; dynamic (F) |
| `rate_init_targets` divides by host `904` as written including bit 16 | `firmware 0x2104` | static; dynamic (F) |
| The firmware never writes host `114`; the clearing event is not known | `firmware 0x1f08` (read only) | static |
| Emulation prevention is gated by bs `+30` bit 5; start codes are written as the 32-bit value 1 and `+78` is clamped to 1 after one | `firmware 0x0bec`, `0x0f3c`, `0x1340`, `0x025c` | static; dynamic |
| `frame_finish` reads bs `+60` and writes it back; `slice_close` reads bs `+80` and discards it | `firmware 0x0d1c..0x0d20`, `0x1234` | static |
| Statistics DMA sends the header total 40 times, then 512 bytes plus one word | `firmware 0x0d60..0x0e60` | static; dynamic (G) |
| Vector words `(type & 7) << 28 \| (v20 & 0x3ff) << 16 \| v30 & 0x1ff`; intra/skip `intra << 31 \| skip << 15` | `firmware 0x1b5c` | static; dynamic (G) |
| 1.0.1 program: same structure, integer rate control with a table, no floating point, backup store absent | `firmware 0x26a0`, `0x2258` (1.0.1 addresses) | static |
| Writer data ports use different ready predicates (`+34 != 0` raw bits, `== 1` Golomb); raw-bit order `+40`, `+3c`, `+38`; kick done polls `+50`/`+20`; `+00` sequence per header, frame and slice | `firmware 0x4374`, `0x3874`, `0x3894`, `0x3d04`, `0x4084`, `0x352c` | static |
| Flush order; second argument 1 skips the `+dc` wait; `+f0` always polled; filler for alignment state 8 is `0x00ffffff` | `firmware 0x25c`, `0xcfc` | static |
| Core DMA accesses: write order `020, 014, 00c, 008, 000`, `030 != 1` loop, no reads of `008/00c/014/020`, no `030` read before a start, `004/018..02c/034/040+` never touched | `firmware 0xd84..0xda8`, `0xdf4..0xe5c`, `0x1a7c`, `0x2018` | static |
| Statistics copy is 40 warm-up copies, not 80 | `firmware 0xd84..0xdb4` (loop bound `0x28`) | static |
| `P+0xb0` covers the whole macroblock header; alpha/beta repacked to 4 bits; `rate_state_load` reads record `+0x40` twice; `mb_qp_delta` condition | `firmware 0x3d04`, `0x352c`, `0x27d4` | static (disassembly over decompiler) |
| Stream index 4..8 overruns the slot array, 8 the record array; `ctx+0x274` never read; `ctx_construct` unchecked allocations | `firmware 0x16c`, `0x18e4`, `0x188`, `0x228` | static |
| Division faults: `rate_init_targets` (`904 == 0`, no macroblocks), `rate_thresholds_update` (QP 0..3) | `firmware 0x2104`, `0x26f8`, `0x2f8c` | static |
