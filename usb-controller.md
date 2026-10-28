# USB OTG controller

The nano 5G uses a DesignWare-style controller at **MMIO `0x38400000`**,
VIC interrupt **19**. The separate PHY is documented in [usb-phy.md](usb-phy.md).
Only device-mode paths were recovered; no host stack.

## Connections and transaction lifecycle

The PHY supplies session and line-state inputs. The core owns the endpoint
FIFOs and DMA; RAM buffers must stay valid until the endpoint completes or is
cancelled. Firmware unmasks endpoint and global interrupts, receives an 8-byte
SETUP packet, picks data direction and length, completes any data packets, then
completes the opposite-direction zero-length status packet. On an IN token the
controller reads the DMA buffer itself. OUT DMA writes need cache invalidation
before the CPU reads them. FIFO depths are in words, endpoint sizes in bytes.

Evidence: `disk mem:0x08048ba4`. The observations below come
from different firmware stages and are alternatives, not one startup sequence.

## Register reference

OTG offsets are relative to **0x38400000**, PHY offsets to **0x3c400000**.
All accesses are 32-bit MMIO. W1C means write-one-to-clear (not read/modify/write).
Layouts come from ROM type recovery (reproduced below).

From ROM type recovery (offsets in the comments are hexadecimal; `u32` is 32-bit):

```c
typedef struct otg_in_ep {             /* 0x900 + n*0x20 */
    u32 DIEPCTL;  u32 rsv04;  u32 DIEPINT;  u32 rsv0c;  u32 DIEPTSIZ;  u32 DIEPDMA;  u32 DTXFSTS;  u32 rsv1c;
} otg_in_ep;
typedef struct otg_out_ep {            /* 0xb00 + n*0x20 */
    u32 DOEPCTL;  u32 rsv04;  u32 DOEPINT;  u32 rsv0c;  u32 DOEPTSIZ;  u32 DOEPDMA;  u32 rsv18;  u32 rsv1c;
} otg_out_ep;
typedef struct otg_host_ch {           /* 0x500 + n*0x20 */
    u32 HCCHAR;  u32 HCSPLT;  u32 HCINT;  u32 HCINTMSK;  u32 HCTSIZ;  u32 HCDMA;  u32 rsv18;  u32 rsv1c;
} otg_host_ch;
typedef struct otg_regs {
    u32 GOTGCTL;      /* 000 */
    u32 GOTGINT;      /* 004 */
    u32 GAHBCFG;      /* 008 ROM: 0x2b (DMA, INCR8, global irq) */
    u32 GUSBCFG;      /* 00c ROM: 0x1708 */
    u32 GRSTCTL;      /* 010 core reset */
    u32 GINTSTS;      /* 014 */
    u32 GINTMSK;      /* 018 */
    u32 GRXSTSR;      /* 01c */
    u32 GRXSTSP;      /* 020 */
    u32 GRXFSIZ;      /* 024 ROM: 0x218 */
    u32 GNPTXFSIZ;    /* 028 ROM: 0x01000218 */
    u32 GNPTXSTS;     /* 02c */
    u32 GI2CCTL;      /* 030 */
    u32 GPVNDCTL;     /* 034 */
    u32 GGPIO;        /* 038 */
    u32 GUID;         /* 03c */
    u32 GSNPSID;      /* 040 */
    u32 GHWCFG1;      /* 044 (not read by the ROM) */
    u32 GHWCFG2;      /* 048 */
    u32 GHWCFG3;      /* 04c */
    u32 GHWCFG4;      /* 050 */
    u8  pad054[0x100 - 0x54];
    u32 HPTXFSIZ;     /* 100 */
    u32 DIEPTXF[15];  /* 104 DIEPTXF1 (ROM: 0x01000318) .. DIEPTXF15 */
    u8  pad140[0x400 - 0x140];
    u32 HCFG;         /* 400 */
    u32 HFIR;         /* 404 */
    u32 HFNUM;        /* 408 */
    u32 rsv40c;
    u32 HPTXSTS;      /* 410 */
    u32 HAINT;        /* 414 */
    u32 HAINTMSK;     /* 418 */
    u8  pad41c[0x440 - 0x41c];
    u32 HPRT;         /* 440 */
    u8  pad444[0x500 - 0x444];
    otg_host_ch HC[16]; /* 500 */
    u8  pad700[0x800 - 0x700];
    u32 DCFG;         /* 800 ROM: 4 then 0 */
    u32 DCTL;         /* 804 */
    u32 DSTS;         /* 808 */
    u32 rsv80c;
    u32 DIEPMSK;      /* 810 ROM: 9 */
    u32 DOEPMSK;      /* 814 ROM: 9 */
    u32 DAINT;        /* 818 */
    u32 DAINTMSK;     /* 81c ROM: 0x10001 */
    u32 DTKNQR1;      /* 820 */
    u32 DTKNQR2;      /* 824 */
    u32 DVBUSDIS;     /* 828 */
    u32 DVBUSPULSE;   /* 82c */
    u32 DTHRCTL;      /* 830 (never written by Apple code) */
    u32 DIEPEMPMSK;   /* 834 */
    u8  pad838[0x900 - 0x838];
    otg_in_ep  DIEP[16];  /* 900 */
    otg_out_ep DOEP[16];  /* b00 */
    u8  padd00[0xe00 - 0xd00];
    u32 PCGCCTL;      /* e00 */
} otg_regs;


typedef struct usbphy_regs {
    u32 OPHYPWR;      /* 00 0 = power up, 0xff/0x1f = power down */
    u32 OPHYCLK;      /* 04 */
    u32 ORSTCON;      /* 08 reset: 1 -> 0 pulse */
    u8  pad0c[0x1c - 0x0c];
    u32 UNKCON;       /* 1c 6 around the core reset, 4 power-down */
    u8  pad20[0x28 - 0x20];
    u32 STATUS;       /* 28 b0 VBUS/session valid, b5 PHY clock running */
```

### Global controller registers

| Offset | Register | Fields/use established |
| --- | --- | --- |
| `000` | GOTGCTL | B-session valid bit 19; session/role state inputs |
| `004` | GOTGINT | OTG event latch/ack; not cleared by acknowledging the summary status |
| `008` | GAHBCFG | bit 0 global IRQ output enable, bits 4:1 burst, bit 5 DMA enable; Apple `0x2b` |
| `00c` | GUSBCFG | bit 3 PHY width; bits 13:10 turnaround; bits 8/9 SRP/HNP; bits 29/30 force host/device |
| `010` | GRSTCTL | bit 0 core reset; bit 4 RX flush; bit 5 TX flush; bits 10:6 TX FIFO selection; bit 31 AHB idle |
| `014` | GINTSTS | current mode plus pending event/summary bits |
| `018` | GINTMSK | event mask; same event bit positions as GINTSTS |
| `024` | GRXFSIZ | RX FIFO depth in words |
| `028` | GNPTXFSIZ / DIEPTXF0 | EP0/shared nonperiodic TX FIFO start and depth |
| `02c` | GNPTXSTS | free FIFO/request queue space |
| `038` | GGPIO / integration-specific GCCFG name | electrical meaning unknown |
| `03c` | GUID | Apple disk core init writes zero |
| `040` | GSNPSID | core identification |
| `044` | GHWCFG1 | endpoint directions: 0 bidirectional, 1 IN, 2 OUT, 3 reserved/unavailable |
| `048` | GHWCFG2 | architecture, PHY, endpoint/channel counts, dynamic FIFO, queue capabilities |
| `04c` | GHWCFG3 | total FIFO words in bits 31:16; packet/transfer counter widths |
| `050` | GHWCFG4 | dedicated FIFO bit 25; IN/control/periodic endpoint and filter capabilities |
| `104 + 4*(n-1)` | DIEPTXFn, n>=1 | TX FIFO start/depth; choose according to endpoint capability/FIFO mode |
| `440` | HPRT | line-state bits 11:10 used by charger/reset checks |
| `e00` | PCGCCTL | clock/power control; Apple sequences bit 0 (meaning unknown) |

Selected GINTSTS/GINTMSK bits: mode mismatch 1, OTG 2, global IN NAK 6,
global OUT NAK 7, suspend 11, reset 12, enumeration done 13, IN summary 18,
OUT summary 19, incomplete isochronous IN 20, disconnect 29, wakeup 31.
Bit 0 is current mode, not an interrupt. Endpoint summaries are recomputed
from pending endpoint events and masks, so not every GINTSTS bit is a clearable latch.

### Device and endpoint banks

| Offset | Register | Meaning |
| --- | --- | --- |
| `800` | DCFG | speed bits 1:0; NZ status-OUT handshake bit 2; address bits 10:4 |
| `804` | DCTL | soft disconnect bit 1; global IN/OUT NAK status bits 2/3; set/clear IN NAK bits 7/8 and OUT NAK bits 9/10; power-on-program-done bit 11 |
| `808` | DSTS | enumerated speed bits 2:1; frame number begins bit 8 |
| `810` | DIEPMSK | common IN interrupt mask |
| `814` | DOEPMSK | common OUT interrupt mask |
| `818` | DAINT | endpoint status summary: IN n at bit n; OUT n at bit 16+n |
| `81c` | DAINTMSK | endpoint summary masks in the same positions |
| `830` | DTHRCTL | threshold configuration |
| `834` | DIEPEMPMSK | per-IN-endpoint FIFO-empty mask |

IN bank n = `0x900 + 0x20*n`; OUT bank n = `0xb00 + 0x20*n`.
Within a bank: `+0x00` control, `+0x08` interrupt status/ack, `+0x10`
transfer size, `+0x14` DMA address. IN has TX FIFO status at `+0x18`.

DEPCTL: endpoint MPS bits 10:0 (nonzero endpoints); active 15; NAK status 17;
type 19:18 (0 control, 1 isochronous, 2 bulk, 3 interrupt); stall 21; IN TX FIFO
25:22; clear NAK 26; set NAK 27; data0/even-frame 28; data1/odd-frame 29;
disable 30; enable 31. Some fields differ by direction and type. EP0 MPS
encodings 0/1/2/3 mean 64/32/16/8 bytes.

Endpoint interrupt bits: transfer complete 0, disabled 1, AHB error 2, SETUP 3
on OUT / timeout 3 on IN, IN NAK effective 6. Acknowledge is W1C: read the
event once, save needed data, then acknowledge the sampled events.

Transfer size: bits 18:0, packet count 28:19 (nonzero endpoints). EP0 transfer
size is bits 6:0, its IN packet-count field is limited, and OUT0 has a special
packet/setup-count format with SUPCNT in bits 30:29. EP0 is armed one packet at a time.

Original SETUP arming literals differ:

| Image/path | DOEPTSIZ0 | DOEPCTL0 command |
| --- | --- | --- |
| ROM | `0x20080040` | `EPENA | CNAK` |
| WTF HAL | `0x20080006` | `EPENA | SNAK` |
| Disk/OSOS | `0x60080040` | `EPENA | SNAK` |

WTF's size-field literal is 6 bytes although a SETUP packet is 8 bytes.
Disk/OSOS records SETUP bytes as `64 - remaining` and rearms the SETUP buffer in the ISR.

### DMA and control-transfer ownership

Apple often clears bit 31 of a submitted DMA pointer (an alias, not a general
virtual-to-physical translation). The manual driver requires identity-mapped DMA
RAM. Masking CPU IRQs does not stop DMA already started.

A control request is SETUP, optional DATA, opposite-direction STATUS, separate
from class command/reply state. On a new SETUP cancel stale EP0 work; on bus
reset clear address, configuration and pending protocol state. For OUT, received
length is the programmed length minus the controller's residual count. Round OUT
DMA storage up to whole packets and own whole cache lines so invalidation does
not discard unrelated dirty data. D-cache maintenance for DMA is separate from
I-cache and branch-predictor sync after writing code.


### Core configuration (disk `usb_otg_core_init` 0x08048ba4, WTF `0x15e0`)

| Register | Apple 5G | Decode |
|---|---|---|
| wait | GINTSTS.CurMod == 0 (device), 25 ms timeout | |
| GRSTCTL | wait HPRT[11:10]==0 (1 ms), AHBIdle, CSftRst, wait clear + AHBIdle | `0x08048cd0` |
| GUID +0x3c | 0 | |
| GAHBCFG | **0x2b** | GlblIntrMsk, HBstLen 5 (INCR8), DMAEn |
| GUSBCFG | **0x1408** | PHYIf=1 (16-bit UTMI+), ULPI/FS off, USBTrdTim=**5** |
| DCFG | **0x00040804** | HS, NZStsOUTHShk, PerFrInt=1, EPMisCnt=1 |
| DTHRCTL | no write found in the compared initialization paths | inherited/reset state not established |
| DIEPMSK / DOEPMSK | disk 9 / 9; WTF mask helper 0x0d / 0x49 | XferCompl+TimeOUT / XferCompl+SetUp |
| DAINTMSK | 0x10001 | EP0 IN/OUT |
| GINTMSK | 0xa01c1806 (b1,2,11,12,18,19,20,29,31) | ModeMis, OTGInt, USBSusp, USBRst, IEP, OEP, incomplISOIN, Disconn, WkUp |
| DMA / slave | DMA (DOEPDMA0 = buf & 0x7fffffff, `0x080432ac`) | |
| PHY width / type | 16-bit UTMI+ | |
| FIFO mode | dynamic, from GHWCFG2 b19; dedicated TX FIFOs if GHWCFG4 b25 | `0x0804daa8` |

FIFO sizes are **computed** from GHWCFG1..4 (`disk mem:0x08048de8`, `wtf UsbHAL.pe:0xb00`, identical):
`GRXFSIZ = 4*nCtlEps(HWCFG4[19:16]) + 2*nOutCapableEps + 0x110` (capped at the reset GRXFSIZ);
`GNPTXFSIZ(EP0) = DedFifo ? 0x20 : 0x100` (capped); `DIEPTXF1 = min(rest, 0x200, saved low-halfword)`; the remaining
words are split evenly over IN EPs 2..n (or periodic EPs in shared mode). Total = GHWCFG3[31:16]. The GHWCFG values are read from hardware at run time; device values are below.

Soft disconnect: DCTL = 2 inside the PHY reset (`0x08039c62`); connect = **DCTL = 0xd00** (CGNPInNak, CGOUTNak,
PWROnPrgDone, SftDiscon=0) in `usb_otg_start` 0x08048adc after `usb_otg_irq_enable` (GINTMSK/GAHBCFG); `usb_otg_start` then calls `usb_phy_set_run_mode(1)` and enables VIC IRQ 19 as its last step; stop writes DCTL = 2
first (`0x08048b5c`). Bus reset/enum handling runs in the USB task, the ISR `0x08038568` only acks and defers.

### Disk endpoints and IDs (disk mode)

- Device descriptor template `disk mem:0x0807f3f7`: USB 2.0, EP0 MPS 64, VID **0x05ac**, PID set by board id
  (`0x08039fb4`, base literal 0x1265): board 0x17 -> 0x1263 (0x1264 for one sub-variant),
  board 0x1a / 0x1c -> **0x1265** (= iPod nano 5G in usb.ids; inf: the N33 board id is one of these).
- Config `0x0807f3e4`: bmAttributes 0xc0 (self powered), bMaxPower 0xfa = 500 mA.
- Interface `0x0807f440`: MSC 8/6/0x50, **EP 0x83 bulk IN, EP 0x02 bulk OUT**, MPS filled at runtime.
- SET_CONFIGURATION resets data toggles on EP1..6 (`0x080484f4`).


### Diagnostics DFU descriptors and device layer

`diag DiagDFU.pe+0x40c`: device bcdUSB `0x0200`, class 0/0/0, EP0 MPS `0x40`,
**VID `0x05ac` PID `0x1246`** (same as the WTF stage), bcdDevice 1, strings 1/2/0
("Apple Inc." `+0x9aa`, "iPod Recovery" `+0x9b5`; language id `+0x9a8`), 1 configuration.
Qualifier (length `0x0a`, type 6) MPS `0x40`. Configuration length `0x1b`, 1 interface,
attributes `0x80`, bMaxPower `0xfa` (500 mA). Interface class `0xfe`, subclass 1,
protocol 2 (DFU mode), no endpoints. DFU functional descriptor: length 9, type `0x21`,
bmAttributes = argument `| 8`, wDetachTimeOut 10, wTransferSize = argument, bcdDFUVersion `0x0100`.

`diag UsbDeviceController.pe` uses only UsbHAL vtable slots
(+0x04, +0x08, +0x0c, +0x10, +0x14, +0x18, +0x1c, +0x20, +0x24, +0x28, +0x2c, +0x30,
+0x34, +0x40, +0x54, +0x5c, +0x64, +0x70..+0x7c, +0x80..+0x90) and bus-reset / suspend
states; it has no MMIO. It differs from the WTF copy by relocation shifts (0xc) and one
extra 8-byte thunk in the WTF build. `diag UsbHAL.pe` equals `wtf UsbHAL.pe` apart from
70 header and build-path bytes ([USB PHY](usb-phy.md)).

## Device-observed configuration and suspend

Device run (one unit): GHWCFG1..4 = `00000264 228f60d0 082000e8 1bf08030`.
DSTS bit 0 is the suspend-state
input; SOF is GINTSTS bit 3. A stale suspend interrupt with active
DSTS must not leave the PMU input off. Resume shows as a new SOF as well as wakeup.
USB input budget transitions are in [pmu.md](pmu.md).

## Unknown

`GGPIO` integration bits, PCGCCTL behavior, FIFO reset-value caps, and the
stage-specific setup-size literals (kept as observed). Host, isochronous, OTG
and error-recovery modes are not decoded, nor are reserved words.

## Diagnostics image

The diagnostics DFU descriptors are VID/PID `05ac:1246`, DFU class `fe/01/02`,
bMaxPower `0xfa` (`diag DiagDFU.pe+0x40c`). The diagnostics USB HAL (`diag UsbHAL.pe`)
is the same code as the WTF copy (only the PE header and build path differ), so the
register facts above apply to it unchanged.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Core/endpoint layout and W1C events | Original USB sources, core, disk capture | observed |
| Disk initialization constants | `disk mem:0x08048ba4` | observed (decompiled) |
| Configuration words | Device run | device (one unit) |
| Diag `UsbHAL.pe` code equals `wtf UsbHAL.pe` | 70-byte diff, header and build path only | observed |
| Diag DFU descriptors 05ac:1246, DFU class fe/01/02, bMaxPower 0xfa | `diag DiagDFU.pe+0x40c` | observed |
| Diag device-controller layer: vtable slots only, no MMIO | `diag UsbDeviceController.pe` | observed |
| Physical USB and charger behavior | device tests | device |
