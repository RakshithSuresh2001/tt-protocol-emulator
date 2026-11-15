# Programmable Protocol Emulator: Architecture and ISA (draft v0.1)

Status: draft for review. Nothing here is implemented yet.

The chip implements protocols as programs, not fixed hardware. Four small state machines (SMs) share one
instruction memory. Each SM runs a short program that drives and samples pins with cycle-exact timing, so
UART, SPI and I2C are just example programs, and new protocols can be added after fabrication.

The design is inspired by the RP2040/RP2350 PIO block. The instruction set below is our own, but the
programming model (side-set, autopush/autopull, wrap) follows the same ideas, and the write-up should credit it.

## 1. Top level

| Block | Size |
|---|---|
| State machines | 4 (SM0 to SM3), run in parallel |
| Instruction memory | 32 x 16-bit, shared by all SMs, written by the host |
| Datapath | 16-bit (X, Y, ISR, OSR, FIFO entries) |
| FIFOs | per SM: TX 4 x 16-bit (host to SM), RX 4 x 16-bit (SM to host) |
| IRQ flags | 4, shared, for SM-to-SM and SM-to-host signaling |
| Host port | SPI slave (mode 0) used by the demo board's RP2350 for configuration and data |

### Pin map (Tiny Tapeout IO)

The SMs see one pin space of 24 indices:

| Index | Tiny Tapeout pin | Direction |
|---|---|---|
| 0 to 7 | uio[0..7] | bidirectional, direction set by PINDIRS (needed for I2C open drain) |
| 8 to 11 | ui[0..3] | input only |
| 12 to 18 | uo[0..6] | output only |

Reserved for the host SPI port: ui[4] = CFG_SCK, ui[5] = CFG_CS_N, ui[6] = CFG_MOSI, uo[7] = CFG_MISO.
ui[7] is spare (planned: global run/stop).

Pin groups are given as base index plus count and wrap modulo 24. Writing to an input-only pin is ignored.
Reading an output-only pin returns the value being driven.

## 2. State machine registers

| Register | Width | Purpose |
|---|---|---|
| PC | 5 | program counter into shared instruction memory |
| X, Y | 16 | scratch registers and loop counters |
| OSR | 16 | output shift register, filled from the TX FIFO by PULL |
| ISR | 16 | input shift register, emptied into the RX FIFO by PUSH |
| OSR count, ISR count | 5 each | bits shifted out of OSR / into ISR since last refill/empty |
| Delay counter | 5 | remaining delay cycles of the current instruction |

## 3. Configuration (per SM, written by the host)

| Field | Bits | Meaning |
|---|---|---|
| CLKDIV | 16 | SM advances one step every CLKDIV system clocks (1 = full speed) |
| WRAP_BOTTOM, WRAP_TOP | 5 + 5 | when PC passes WRAP_TOP it goes to WRAP_BOTTOM (free loops) |
| OUT_BASE, OUT_COUNT | 5 + 5 | pins written by OUT PINS / OUT PINDIRS / MOV PINS |
| SET_BASE, SET_COUNT | 5 + 3 | pins written by SET PINS / SET PINDIRS (count 0 to 5) |
| IN_BASE | 5 | first pin read by IN PINS and WAIT PIN |
| SIDE_BASE, SIDE_COUNT | 5 + 2 | side-set pins (count 0 to 2) |
| SIDE_OPT | 1 | side-set is optional: the top DLY/SIDE bit enables it per instruction |
| SIDE_PINDIRS | 1 | side-set drives pin directions instead of values (open drain) |
| JMP_PIN | 5 | pin tested by JMP PIN |
| AUTOPULL, PULL_THRESH | 1 + 5 | refill OSR automatically after PULL_THRESH bits shifted out |
| AUTOPUSH, PUSH_THRESH | 1 + 5 | push ISR automatically after PUSH_THRESH bits shifted in |
| OUT_SHIFT_RIGHT, IN_SHIFT_RIGHT | 1 + 1 | 0 = MSB first, 1 = LSB first |

Thresholds of 0 mean 16. A stalled OUT (OSR empty with autopull) or IN (RX FIFO full with autopush) waits
without advancing, and its side-set still takes effect, so idle states are defined by the program.

## 4. Instruction format

Every instruction is 16 bits:

```
 15  13 12        8 7               0
[ OP  ][ DLY/SIDE ][    operands     ]
```

- OP (3 bits): one of 8 instructions.
- DLY/SIDE (5 bits): the top SIDE_COUNT bits are the side-set value applied to the side-set pins when the
  instruction starts; the remaining bits are a delay in SM cycles after the instruction completes.
  With SIDE_COUNT = 1 the delay range is 0 to 15; with 0 it is 0 to 31. With SIDE_OPT = 1 one more bit
  is used as a per-instruction enable, so SIDE_COUNT = 1 leaves a delay range of 0 to 7.

## 5. Instructions

| OP | Mnemonic | Operands (bits 7:0) | Action |
|---|---|---|---|
| 000 | JMP cond, addr | cond[7:5], addr[4:0] | jump if cond: ALWAYS, !X (X==0), X-- (X!=0, then X=X-1), !Y, Y--, X!=Y, PIN (JMP_PIN high), !OSRE (OSR not empty) |
| 001 | WAIT pol, src, idx | pol[7], src[6:5], idx[4:0] | stall until source == pol. src: PIN (IN_BASE+idx), GPIO (absolute idx), IRQ (flag idx[1:0]; clears it when waiting for 1) |
| 010 | IN src, n | src[7:5], n[4:0] | shift n bits from src into ISR. src: PINS, X, Y, NULL (zeros), ISR, OSR |
| 011 | OUT dst, n | dst[7:5], n[4:0] | shift n bits out of OSR into dst. dst: PINS, X, Y, NULL, PINDIRS, PC |
| 100 | PUSH / PULL | dir[7], ifcond[6], block[5] | PUSH: ISR to RX FIFO, clear ISR. PULL: TX FIFO to OSR. ifcond: only if threshold reached; block: stall if FIFO full/empty |
| 101 | MOV dst, op src | dst[7:5], op[4:3], src[2:0] | dst: PINS, X, Y, PINDIRS, PC, ISR, OSR. op: none, invert, bit-reverse. src: PINS, X, Y, NULL, STATUS, ISR, OSR |
| 110 | IRQ set/clear/wait, idx | clr[6], wait[5], idx[1:0] | set or clear a shared flag; with wait, stall until another SM clears it |
| 111 | SET dst, data | dst[7:5], data[4:0] | dst: PINS, X, Y, PINDIRS. Writes the 5-bit immediate |

NOP is assembled as `MOV Y, Y`.

## 6. Host commands (SPI port)

Each transaction: CS low, one command byte, payload bytes, CS high. Multi-byte values are MSB first.

| Cmd | Payload | Action |
|---|---|---|
| 0x10 | addr, hi, lo | write instruction memory word |
| 0x20 | sm, reg, hi, lo | write a configuration register |
| 0x30 | mask | enable SMs (bit per SM); disabling holds an SM in place |
| 0x40 | sm, addr | force an SM's PC to addr and clear its delay (used to start, restart, and enter START/STOP routines) |
| 0x50 | sm, hi, lo | push a word into the SM's TX FIFO |
| 0x60 | sm | pop a word from the SM's RX FIFO; returns hi, lo, then a status byte |
| 0x70 | none | return FIFO levels and IRQ flags for all SMs |
| 0x80 | mask, value, dir | host pin override for simple GPIO such as chip selects |

## 7. Example programs

All examples assume the default 50 MHz clock. The assembler fills in encodings and the host library writes
the configuration shown.

### 7.1 UART TX, 8N1 (4 instructions)

Config: SIDE_COUNT = 1, SIDE_OPT = 1 (only the first two instructions drive the side-set pin, so it never
fights OUT), OUT_BASE = SIDE_BASE = TX pin, OUT_COUNT = 1, OUT_SHIFT_RIGHT = 1 (LSB first),
CLKDIV = clk / (8 x baud).

```
.side_set 1 opt
    pull  block      side 1 [7]   ; idle/stop bit high, wait for a byte
    set   x, 7       side 0 [7]   ; start bit, 8 cycles
bit:
    out   pins, 1          [6]    ; one data bit
    jmp   x-- bit                 ; 8 cycles per bit in total
```

### 7.2 UART RX, 8N1 (4 instructions)

Config: IN_BASE = RX pin, IN_SHIFT_RIGHT = 1, AUTOPUSH = 1, PUSH_THRESH = 8, CLKDIV = clk / (8 x baud).

```
start:
    wait  0 pin 0                 ; falling edge of start bit
    set   x, 7         [10]       ; skip to the middle of data bit 0 (1.5 bit times)
bit:
    in    pins, 1                 ; sample
    jmp   x-- bit      [6]        ; 8 cycles per bit; autopush delivers the byte
```

Framing-error detection (check the stop bit with JMP PIN, flag with IRQ) is a planned extension.

### 7.3 SPI master, modes 0 and 2 (2 instructions)

Config: SIDE_COUNT = 1, SIDE_BASE = SCK, OUT_BASE = MOSI, IN_BASE = MISO, AUTOPULL and AUTOPUSH with
thresholds 8 (or 16), CLKDIV sets SCK = clk / (4 x CLKDIV). Chip select through host pin override (0x80).
With MSB-first shifting the host left-aligns each byte in the 16-bit TX word; received bytes arrive in the
low bits of the RX word.

```
; mode 0 (CPOL 0, CPHA 0): data changes while SCK low, sampled on rising edge
.side_set 1
    out   pins, 1     side 0 [1]   ; SCK low; stalls here when empty, so SCK idles low
    in    pins, 1     side 1 [1]   ; SCK high, sample MISO
```

Mode 2 (CPOL 1, CPHA 0) is the same program with the side-set values swapped (side 1, then side 0).

### 7.4 SPI master, modes 1 and 3 (3 instructions)

```
; mode 1 (CPOL 0, CPHA 1): data changes on rising edge, sampled on falling edge
.side_set 1
    out   x, 1        side 0       ; stalls here when empty, SCK idles low
    mov   pins, x     side 1 [1]   ; SCK high, drive MOSI
    in    pins, 1     side 0       ; SCK low, sample MISO
```

Mode 3 swaps the side-set values.

### 7.5 I2C master with clock stretching (13 instructions)

I2C lines are open drain: a pin is pulled low by making it an output that drives 0, and released by making it
an input (the board pull-up raises it). So the program drives pin directions, not values.

Config: SDA on uio pin n, SCL on uio pin n+1. OUT_BASE = SET_BASE = IN_BASE = SDA, OUT_COUNT = SET_COUNT = 1,
SIDE_BASE = SCL, SIDE_COUNT = 1, SIDE_PINDIRS = 1, output values of SDA and SCL preset to 0,
AUTOPULL and AUTOPUSH with thresholds 9, MSB first. WRAP_BOTTOM = bit, WRAP_TOP = the IN instruction.

Each 9-bit frame is 8 data bits plus the ACK bit, left-aligned in the 16-bit TX word (bits 15 to 7). In the TX word, 1 means "pull SDA low" (the host library
inverts data bits). For a write, the host sends the byte with ACK bit 0 (released) and reads the slave's ACK
from the RX word. For a read, the host sends all data bits 0 (released) and sets the ACK bit as needed.

```
.side_set 1 pindirs              ; side 1 = SCL pulled low, side 0 = SCL released
bit:
    out   pindirs, 1  side 1 [3]  ; SCL low, set SDA for this bit
    nop               side 0 [3]  ; release SCL
    wait  1 pin 1     side 0      ; clock stretching: wait for SCL (IN_BASE + 1) to really be high
    in    pins, 1     side 0 [3]  ; sample SDA while SCL is high; wraps to bit
start:                            ; host jumps here (0x40) for START or repeated START
    set   pindirs, 0  side 1 [7]  ; release SDA while SCL is low
    nop               side 0 [7]  ; release SCL
    set   pindirs, 1  side 0 [7]  ; SDA low while SCL high = START
    nop               side 1 [7]  ; SCL low
    jmp   bit         side 1      ; ready for the first frame
stop:                             ; host jumps here for STOP
    set   pindirs, 1  side 1 [7]  ; SDA low, SCL low
    nop               side 0 [7]  ; release SCL
    set   pindirs, 0  side 0 [7]  ; release SDA while SCL high = STOP
idle:
    jmp   idle        side 0      ; park with both lines released
```

A bit takes about 13 SM cycles plus any clock stretching, so 100 kHz needs CLKDIV of about 38 at 50 MHz and
400 kHz about 10.

### 7.6 Memory budget

UART TX (4) + UART RX (4) + SPI mode 0 (2) + I2C (13) = 23 of 32 words, so all three protocols can be loaded
and run at the same time on three SMs, with one SM and 9 words left for a fourth protocol.

## 8. Verification plan

- Unit tests per instruction (cocotb), including stalls, delays, side-set and wrap.
- Protocol tests: each example program against a reference model (UART decoder, SPI slave model, I2C slave
  model with clock stretching), at several CLKDIV values.
- Formal (SymbiYosys): one SM against an ISA reference model for every instruction; FIFO never loses or
  duplicates data; host port command decoding.
- Gate-level simulation of the hardened design (Tiny Tapeout gl_test) for the protocol tests.
- Reset: every control flop has a reset (lesson from Argo's gate-level X-pessimism).

## 9. Open questions for review

1. 4 SMs and 32 words, or 2 SMs and 64 words? (Area vs. number of simultaneous protocols.)
2. Is a 16-bit datapath enough? It saves area; 32-bit would simplify some protocols.
3. Add a CRC/bit-stuffing helper so USB and CAN become practical stretch goals?
4. Fractional clock divider for exact UART baud rates, or integer only?
5. Host port: SPI (simple, matches the RP2350) or a parallel bus on the uio pins (faster, but uses the pins)?
