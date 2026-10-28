# Chip identity and fuse interface

**MMIO base `0x3d100000`**, 32-bit accesses. The block supplies identity and
revision fields used to select PHY and board behavior. Disk clock-gate ID35
(`0x23`) gates it; this is a gate ID, not a PWRCON bit. The read routine
establishes no fuse programming sequence.

`disk mem:0x08038c58` caches a 10-byte descriptor at software RAM
`mem:0x080a3594`:

| MMIO field | Cached destination | Meaning |
|---|---|---|
| `00 bit0` | validity decision | must be set to accept the chip identity |
| `08 bits31:16` | descriptor u16+0 | must equal `0x8730`, otherwise the fallback1 value is written |
| `04 bits26:24` | descriptor u16+2 | revision field, encoding unknown |
| `04 bits31:28` | descriptor u16+4 | revision/configuration nibble, meaning unknown |
| `04 bit4` | descriptor byte+6 | boolean hardware variant |
| `08 bit0` | descriptor byte+7 | selects the disk USB PHY ref-clock |
| `08 bits12:11` | descriptor byte+8 | variant field; the bootloader provider maps 0 -> 0, 1 -> 1, 3 -> 2, 2 -> 3 |

The function enables its gate, tests the fields, disables the gate and copies
10 bytes of cached data to the caller. Later calls use the cache.

## Registers and bootloader protocol

| MMIO | Name | Use | Evidence |
|---|---|---|---|
| `+0x00` bit 0 | valid | CID_VALID | `bootloader ChipId.pe+0x27c` |
| `+0x04` | CHIPIDL | Diag splits it as VER = `& 0xff00003f` (bits 31:24 and 5:0) and CAL = the rest; bits 26:24 and 31:28 are the revision fields; bit 4 is the variant flag (set -> 3, clear -> 2, `+0x37c`) | `diag TestM2ChipID.pe+0xb44..0xb5c` |
| `+0x08` | CHIPIDH | bits 31:16 = `0x8730`; bits 12:11 map 0 -> 0, 1 -> 1, 3 -> 2, 2 -> 3; bits 10:4 are a 7-bit field; bits 3:2 map 1 -> 0, 2 or 3 -> 1, 0 -> 2; bit 1 is one flag | `bootloader ChipId.pe+0x2b8`, `+0x3bc` |
| `+0x0c` / `+0x10` | DIEID low / high | die ID, unique per chip | `bootloader ChipId.pe+0x308` |
| `+0x14` | ECID version | | `bootloader ChipId.pe+0x344` |

The provider `bootloader ChipId.pe` enables clock gate `0x23` around every access (clock protocol slot +4, id `0x23`, 1 / 0).
Interface slots (table `+0x4a4`, code `+0x240..0x3b2`): +0 valid = `+0x00` bit 0; +4 id fields = `08 >> 16`, `04[26:24]`, `04 >> 28`;
+8 variant = `04` bit 4 ? 3 : 2; +0xc the `08` fields above; +0x10 die ID = `+0x0c` (low), `+0x10` (high); +0x14 ECID version = `+0x14`;
+0x18 CHIPIDL = `+0x04`, CHIPIDH = `+0x08`.

The diag command `m2chipid` (GUID `ddee1ef2-dc49-4799-90e6-4b0c8bd36810`) prints `CID_VALID`, `CHIPIDL_VER`, `CHIPIDL_CAL`, `CHIPIDH`,
`DIEIDL`, `DIEIDH`, `ECID_VERSION` (`diag TestM2ChipID.pe+0xb10..0xb8e`, strings `+0xb94..0xc04`).

Device run: CHIPID word `08` = `0x8730000b` (device run). `0x87300005` is rejected by the WTF boot-mode check. `0x8730000b` is one observed unit, not a universal reset
value. OSOS's board word and the chip identity are separate structures.

## Unknown

Fuse array beyond the die-ID words `+0x0c/+0x10`, programming voltages, lock semantics, unused bits, the encoding of the revision fields and of the 7-bit `08` field. The
routine only reads chip fields; no write path is known.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Extracted fields and validity | `disk mem:0x08038c58` | observed (decompilation) |
| Gate and descriptor cache | same function | observed; offsets are software memory, not registers |
| PHY consumer | [usb-phy.md](usb-phy.md) | observed; only bit 0 consumer known |
| Register fields, provider slots, die ID at `+0x0c/+0x10` | `bootloader ChipId.pe+0x240..0x3b2`; `diag TestM2ChipID.pe+0xb10..0xb8e` | observed |
