# Konami 056734 "ESC" Coprocessor — Programmer's Reference

*Reconstructed from the firmware and game programs shipped in Konami GX
arcade titles (1994–1996). Nothing here comes from Konami documentation;
every statement was derived by decoding the ROM images and confirming the
behaviour by execution. Items marked* **(?)** *are consistent with all
observations but not independently proven.*

---

## 1. Overview

The 056734, called "ESC" by Konami's own library (`esc_ctrl.c`,
`esc_communication.c`, "ESC control library ver1.10 programmed by
H.Takahama"), is a security coprocessor fitted to the game cartridge of
some Konami GX titles. It is a 32-bit RISC-style core with:

- 32 general registers and 32 special registers, all 32 bits wide;
- a private word-addressed SRAM ("local memory") for code, data and stack;
- bus-master access to the 68020 host bus through a 24-bit byte-addressed
  big-endian window ("host memory");
- a mailbox register at host address `0xCC0000`, an interrupt output to
  the host (IRQ 4), and a small internal boot ROM;
- a hardware multiplier/divider and an instruction-fetch descrambler
  configured per chip.

The chip holds a per-title secret. The game ROM carries, at a fixed
address, a boot program and an encrypted command interpreter ("kernel");
the game later uploads one small encrypted program and invokes it every
frame. In the titles examined, those programs build the sprite list for
the 053246/055673 sprite hardware, copy blocks of memory, or compute a
table pointer. Without the chip the games hang at start-up.

Titles carrying ESC programs: Crazy Cross / Taisen Puzzle-dama, Taisen
Tokkae-dama, Tokimeki Memorial Taisen Puzzle-dama, Run and Gun 2 / Slam
Dunk 2 (program present but unused), Daisu-Kiss, Twin Bee Yahhoo!, Dragoon
Might, Gokujou Parodius (program present but unused), Salamander 2 and
Sexy Parodius.

---

## 2. Programming model

### 2.1 Registers

| Register | Role |
|---|---|
| `r0` | Always reads as zero. Writes are discarded. Used as a null operand and to set flags (`sub r0, rB, rC` is a compare). |
| `r1`–`r31` | General purpose. By software convention inside game programs: `r1` = parameter block pointer, `r2` = workspace pointer, `r3` = data-section pointer, `r4`–`r19` callee-saved, `r20`–`r30` scratch, `r31` = return value. |
| `s0` | Control. Bit 4 enables burst (sequential) host access; the kernel sets it around block transfers and clears it afterwards. Other bits unknown. |
| `s1` | Stack pointer. Word address into local memory; the stack grows downward; `push`/`pop`/`call`/`ret` use it. |
| `s2` | Program counter as seen by software. Reading it yields the word index of the current instruction **plus two** (a two-stage fetch pipeline). Used with `jr` for position-independent calls and jump tables. |
| `s6` | Host handshake. Bit 0 is set by firmware before it waits for the host ("ready"); bit 1 is pulsed (`ori.b s6,#1` / `#3` / `#1`) to assert IRQ 4 on the host. |
| `s7` | Host mailbox. Holds the last longword the 68020 wrote to `0xCC0000`; zero when empty. The chip clears it by itself storing to `0xCC0000`. |
| `s8` | Multiply/divide result (see `mul`, `div`). |
| `s9` | Unknown; the boot code's diagnostic dump reports it next to `s8`. |
| `s10` | Per-chip secret (the low 16 bits are used by the loader). |
| `s11` | Boot object address: bits 23:0 = `0x200100`; bits 27:24 and 31:28 are configuration nibbles that also feed the loader. |
| `s22`, `s23` | Scratch, used by the kernel's cipher routines. |
| `s30` | Unknown. Appears only in the image terminator word (§6.3). |
| `s31` | Link register: `ret s31` is the return instruction. |

Unlisted special registers (`s3`–`s5`, `s12`–`s21`, `s24`–`s29`) were
never referenced.

### 2.2 Flags

Add, subtract and compare instructions set **Z** (zero), **N** (sign bit
of the result within the operation size), **C** (carry out, or borrow for
subtraction) and **V** (signed overflow) **(?)**. Logical operations and
shifts set Z and N. Loads and stores do not affect flags.

### 2.3 Memory spaces

**Local memory** is word-addressed (32-bit words, one address per word).
Programs and the kernel address at least 8K words (`0x0000`–`0x1FFF`);
the exact size is not known. The kernel occupies words `0x0000`–`0x069B`,
its variables and tables follow, the loader allocates program blocks
above them, and the stack top is `0x2000`. Byte and halfword local
accesses touch the low part of a word (loads zero-extend; stores keep the
upper bits of the word).

**Host memory** is the 68020 address space, 24 bits, byte-addressed,
big-endian. Unaligned halfword and word accesses work. Everything the
68020 can see is reachable: BIOS ROM, game ROM (`0x200000`–), work RAM
(`0xC00000`–`0xC1FFFF`), sprite RAM (`0xD20000`–), and GX control
registers.

Addresses held in registers are 32 bits; the host bus uses bits 23:0 and
local memory uses the low word-address bits. The linker tags local
addresses with segment bits above the address (`0x10000` for data,
`0x20000` for workspace) that the hardware ignores.

### 2.4 Instruction fetch and code storage

Instruction words are fetched from local memory through a descrambler:
each chip is programmed with a permutation of the sixteen 2-bit lanes of
the 32-bit word plus a 32-bit XOR constant. Code is therefore stored in
local memory (and in ROM) in the scrambled form for that chip, and
appears in plaintext only inside the fetch unit. Data words are not
descrambled. The bit numbering used in this document is a canonical
reference order: Sexy Parodius's storage order with its constant
(`0x1886AE1D`) removed. The constants and lane maps of the other chips
are in §8.4. The order on the chip's internal bus is not observable and
does not matter to software.

---

## 3. Instruction encoding

All instructions are one 32-bit word. The word is viewed as sixteen 2-bit
lanes, lane *k* = bits 2k+1:2k. Fields are assembled from lanes in the
order shown (most significant first):

| Field | Lanes (high → low) | Width | Meaning |
|---|---|---|---|
| rA | 5, 15, 9 | 6 | first register (destination or compared source); 0–31 = `r0`–`r31`, 32–63 = `s0`–`s31` |
| rB | 2, 1, 4 | 6 | second register (base or first source) |
| off10 | 6, 14, 13, 12, 11 | 10 | signed offset, or function/third register: rC = off10 ≫ 3 |
| imm16 | 2, 1, 4, 6, 14, 13, 12, 11 | 16 | immediate (rB and off10 together) |
| class | 0 | 2 | 3 = ALU / memory, 2 = register & stack, 0 = branch / jump / load-immediate, 1 = absolute host access |
| size | 7 | 2 | 0 = word (`.w`), 1 = byte (`.b`), 2 = halfword (`.h`) |
| opcode | 10, 8, 3 | 6 | operation within the class |

Opcodes are written below as a five-digit pattern giving lanes
10, 8, 7, 3, 0 in that order; `s` stands for the size lane (0/1/2).

Two formats do not follow the table:

- **Branches** are recognised by lanes 5, 7, 9, 15, 0 = 2, 2, 2, 0, 0.
  The condition is lanes 10, 8, 3; the signed 16-bit displacement in the
  imm16 lanes is in words relative to the *next* instruction.
- **Load immediate** (`li`) is class 0 and not a branch or jump. The
  destination register is encoded in lanes 8 (bit 0), 3 and 10:
  `rD = (lane8 & 1) << 4 | lane3 << 2 | (lane10 >> 1) << 1 | (1 − (lane10 & 1))`,
  and the immediate is 24 bits: bits 23:22 = lane 7, bits 21:16 = the rA
  lanes, bits 15:0 = imm16. The same 24-bit field is an absolute host
  address in the class-1 instructions.

The all-zero word is a no-op.

**Worked examples** (canonical form):

| Word | Lanes 15…0 | Decodes as |
|---|---|---|
| `40110002` | `1000 0000 0001 0001 0000 0000 0000 0010` | `push r4` (`11002`, rA = 4) |
| `c01d0c42` | `1100 0000 0001 1101 0000 1100 0100 0010` | `ret s31` (`11012`, rA = 63) |
| `402101c3` | `0100 0000 0010 0001 0000 0001 1100 0011` | `ld.w(l) r4, [r1]` (`21033`) |
| `0012c040` | `0000 0000 0001 0010 1100 0000 0100 0000` | `li r4, #0xC00000` (lane 7 = 3 → bits 23:22) |
| `000ac440` | `0000 0000 0000 1010 1100 0100 0100 0000` | `li r5, #0xD20000` (rA lanes = 0x12) |
| `3f38bb3c` | `0011 1111 0011 1000 1011 1011 0011 1100` | `bne −3` (condition 300, displacement −4 → target = index − 3) |

---

## 4. Instruction set

Syntax: destination first. `[rB+d]` is host memory at rB + d;
`[rB+d](l)` is local memory (the mnemonic carries the `(l)`). `#n` is an
immediate. A size suffix `.b`/`.h` restricts the operation to the low
8/16 bits of the operands; for register-to-register forms the untouched
upper bits of the result come from the **first source operand** (so
`or.h rA, r0, rB` is a zero-extending halfword move), for immediate and
memory-operand forms they come from rA.

### 4.1 Arithmetic and logic (class 3)

| Pattern | Syntax | Operation | Flags |
|---|---|---|---|
| `12s03` | `add rA, rB, rC` | rA = rB + rC | Z N C V |
| `12s13` | `sub rA, rB, rC` | rA = rB − rC (`sub r0, …` = compare) | Z N C V |
| `12s23` | `and rA, rB, rC` | rA = rB & rC | Z N |
| `12s33` | `xor rA, rB, rC` | rA = rB ^ rC | Z N |
| `32s13` | `or rA, rB, rC` | rA = rB \| rC (`mov rA, rB` when rC = r0) | Z N |
| `32s33` | `nor rA, rB, rC` | rA = ~(rB \| rC) (`not rA, rC` when rB = r0) | Z N |
| `13s03` | `addi rA, #imm16` | rA = rA + imm16 | Z N C V |
| `13s13` | `subi rA, #imm16` | rA = rA − imm16 | Z N C V |
| `13s23` | `andi rA, #imm16` | rA = rA & imm16 | Z N |
| `13s33` | `xori rA, #imm16` | rA = rA ^ imm16 | Z N |
| `33s13` | `ori rA, #imm16` | rA = rA \| imm16 | Z N |

Immediates in this group are zero-extended 16-bit values (`subi.h r20,#0x8000` compares against 0x8000; `andi r20,#3` masks 32-bit r20).

### 4.2 Loads and stores (class 3)

| Pattern | Syntax | Operation |
|---|---|---|
| `20s33` | `ld rA, [rB+d]` | rA = host memory (zero-extended for `.b`/`.h`) |
| `30s33` | `st rA, [rB+d]` | host memory = rA (low 8/16/32 bits) |
| `21s33` | `ld(l) rA, [rB+d]` | rA = local word (masked to 8/16 bits for `.b`/`.h`) |
| `31s33` | `st(l) rA, [rB+d]` | local word = rA; `.b`/`.h` replace only the low part of the word |

`d` is the signed 10-bit offset (−512…+511) in bytes for host memory and
in words for local memory.

### 4.3 Memory-operand arithmetic (class 3, local memory)

| Pattern | Syntax | Operation |
|---|---|---|
| `01s03` | `add(m) rA, [rB+d]` | rA = rA + local |
| `01s13` | `cmp(m) rA, [rB+d]` | flags of rA − local (rA unchanged) |
| `01s23` | `and(m) rA, [rB+d]` | rA = rA & local |
| `01s33` | `xor(m) rA, [rB+d]` | rA = rA ^ local |
| `21s13` | `or(m) rA, [rB+d]` | rA = rA \| local |
| `11s03` | `(m)add rA, [rB+d]` | local = local + rA **(?)** |
| `11s13` | `(m)sub rA, [rB+d]` | local = local − rA; flags of local − rA (`(m)sub r0, …` tests memory) |
| `11s23` | `(m)and rA, [rB+d]` | local = local & rA **(?)** |
| `11s33` | `(m)xor rA, [rB+d]` | local = local ^ rA **(?)** |
| `00s13` | `cmp.m rA, [rB+d]` | flags of rA − host memory |
| `10s13` | `cmpm [rB+d], rA` | flags of host memory − rA (`cmpm.b [rB], r0; bmi` tests bit 7 of a byte) |

### 4.4 Register, shift and stack instructions (class 2)

| Pattern | Syntax | Operation |
|---|---|---|
| `11002` | `push rA` | s1 = s1 − 1; local[s1] = rA |
| `31002` | `pop rA` | rA = local[s1]; s1 = s1 + 1 |
| `11012` | `ret s31` | return: pop the address saved by `call` into the program counter |
| `31012` | `ret s31` (variant) | used once, at the end of the kernel's diagnostic dump routine; assumed to return **(?)** |
| `10032` | `lih rA, #imm16` | rA[31:16] = imm16, low half unchanged (`li` + `lih` builds a 32-bit constant) |
| `30012` | `swap16 rA, rB` | rA = rB rotated by 16 |
| `12s02` | `shl rA, rB` | rA = rB ≪ 1 (logical) |
| `12s12` | `asl rA, rB` | rA = rB ≪ 1 (same result as `shl`) |
| `12s22` | `rol rA, rB` | rA = rB rotated left by 1 within the size |
| `32s02` | `shr rA, rB` | rA = rB ≫ 1 (logical) |
| `32s12` | `asr rA, rB` | rA = rB ≫ 1 (arithmetic, sign preserved) |
| `32s22` | `ror rA, rB` | rA = rB rotated right by 1 **(?)** |
| `10212` | `ext.h rA, rB` | rA = rB[15:0] sign-extended to 32 bits |
| `10112` | `ext.b rA, rB` | rA = rB[7:0] sign-extended **(?)** |
| `10022` | `mul rA, rB` | s8 = rA × rB (32 × 32, low 32 bits) |
| `10222` | `mul.h rA, rB` | s8 = rA[15:0] × rB[15:0], signed 16 × 16 → 32 |
| `30222` | `div.h rA, rB` | s8 = rA[15:0] ÷ rB[15:0], unsigned; rA unchanged |
| `30002` | `neg rA, rB` **(?)** | a one-operand transform of rB into rA. Used only by the boot code's RAM test (boot words 173 and 194), as `neg r31, r31; subi r31, #1` both when filling and when verifying, so any one-to-one transform passes the test and the operation cannot be identified from it. Negation (making the pair a complement) is a guess. |

Shifts and rotates operate on the low 8/16/32 bits selected by the size
and set Z and N. All shifts are by one bit; multi-bit shifts are written
as sequences.

**Multiplier/divider timing.** `mul` and `div` start the operation and
continue; the result becomes valid in `s8` later. Firmware reads `s8`
two instructions after a `div.h` and seven to fifteen instructions after
a `mul`, padding with no-ops; the boot code waits sixteen no-ops after a
`mul`. Exact latencies are not known.

### 4.5 Control transfer (class 0)

| Pattern / condition | Syntax | Operation |
|---|---|---|
| `10200` | `jr rA[+d]` | pc = rA + d (d in words) |
| `11200` | `jsr rA` | push return address; pc = rA |
| cond `100` | `jmp target` | unconditional |
| cond `110` | `call target` | push return address; pc = target |
| cond `200` | `beq target` | branch if Z |
| cond `300` | `bne target` | branch if !Z |
| cond `001` | `bmi target` | branch if N |
| cond `101` | `bpl target` | branch if !N |
| cond `201` | `bcs target` | branch if C (borrow set after a subtract: unsigned lower) |
| cond `301` | `bcc target` | branch if !C (unsigned higher or same) |
| cond `102` | `bgt target` | branch if signed greater **(?)** (used as a count-down loop back-edge) |
| cond `302` | `bhi target` | branch if unsigned higher **(?)** (same use) |
| cond `003` | `blt target` | branch if signed less **(?)** |

Branch targets are `index + 1 + displacement`. The return address pushed
by `call`/`jsr` is the index of the following word; `ret` pops it. The
boot code uses `call 86; st.h(l) r0,[s1]; ret` to jump to address 0 —
proof that the return address lives on the `s1` stack.

Register-indirect position-independent calls are written
`mov r17, s2; jmp routine` … `jr r17` (s2 already points past the
`jmp`), and PC-relative jump tables as `add.h r20, s2, r20; jsr r20`
where the table holds offsets from the word two past the `add`.

### 4.6 Load immediate and absolute host access (classes 0 and 1)

| Form | Syntax | Operation |
|---|---|---|
| class 0, not branch/jump | `li rD, #imm24` | rD = 24-bit immediate, zero-extended |
| class 1, lane 8 bit 1 clear | `ld.abs rD, [addr24]` | rD = host word at the 24-bit address **(?, width assumed 32-bit)** |
| class 1, lane 8 bit 1 set | `st.abs rD, [addr24]` | host word at the address = rD **(?)** |

`li` covers the whole host address space in one word (`li r5, #0xD20000`);
32-bit constants use `li` followed by `lih`. The absolute forms appear in
firmware as `st.abs r0, [0xCC0000]` (clear the mailbox) and in one game
program for counters in work RAM.

---

## 5. Host interface

### 5.1 Mailbox and interrupt

The 68020 starts every transaction by writing a longword to `0xCC0000`.
The chip latches it in `s7`. Firmware waits for `s7 ≠ 0`, copies it to a
register, and clears the latch with `st.abs r0, [0xCC0000]`. A zero
write from the host is therefore harmless (the games issue them as
keep-alives).

The value is a pointer into host memory. Two kinds of target exist:

- the **initialisation word**, a longword holding `0x0108DB04`;
- a **command packet** (§5.2).

On completion of a packet the firmware writes the packet's state byte and
pulses `s6` bit 1, which asserts 68020 interrupt level 4 if the GX
control register enables it (bit 4 of the IRQ-enable byte at `0xD56000`).
The library also polls the state byte, so operation without the
interrupt is possible.

### 5.2 Command packet

Packets live in work RAM (`0xC00000`–`0xC1FFFF`) and may be unaligned.

| Offset | Size | Field |
|---|---|---|
| +0 | 4 | magic `0xFEF724FB` |
| +4 | 4 | handle (a program instance identifier chosen by the host) |
| +8 | 1 | command |
| +9 | 1 | state: 0 idle, 1 accepted/busy, 2 END, 3 ERROR |
| +0xC | 4 | argument 1 (pointer or value) |
| +0x10 | 4 | argument 2 |

| Command | Name | Argument 1 | Argument 2 | Action |
|---|---|---|---|---|
| 0 | — | | | rejected (state 3) |
| 1 | RUN | pointer to four longwords | | copy the parameters, invoke the program's RUN entry, store its result at +0xC |
| 2 | LOAD | pointer to a program image | pointer to four longwords | parse, allocate, decrypt, verify, invoke the program's LOAD entry with the four longwords |
| 3 | RESTART **(?)** | pointer to four longwords | | invoke the program's third entry, then its LOAD entry with new parameters |
| 4 | UNLOAD **(?)** | | | invoke the third entry, release the program's blocks and handle |
| 5 | RESET | pointer to a 12-byte reply buffer | | restart the kernel; it then writes `{magic, free, free}` to the buffer (free = free local words) and completes |
| 6 | QUERY **(?)** | | | store the amount of free local memory at +0xC |

Command numbers above 6 are treated as 0. **RESET must come first**: until
a RESET with a reply buffer has been processed the kernel dispatches
through a restricted table in which every other command is rejected with
ERROR. After each later command the kernel refreshes the free-memory word
at reply buffer +4 as long as the buffer still starts with the magic.

The kernel keeps eight handle records, so up to eight programs may be
resident; every game examined loads one.

### 5.3 Initialisation word and error reporting

The library's INIT routine points the mailbox at a longword holding
`0x0108DB04` and spins for a fixed time. The kernel ignores packets that
do not start with the magic, so a healthy chip leaves the word untouched.
If the boot self-tests failed, the boot code sits in an error loop that
answers the next mailbox pointer by **overwriting the word with an error
code** (bit flags describing which RAM-test pass failed); the library
treats a changed word as failure and skips the device.

### 5.4 Diagnostic packet

Whenever the chip is waiting for a mailbox write, in the boot code's error
loop as well as in the kernel's wait routine, it also recognises a packet
whose first longword is `0xC759E285`. It answers by dumping `r1`–`r30`,
the two words at the top of the stack, `s1`, `s8`, `s9` and `r31` into
the packet at +0x94…+0x120, copying 32 local words from the local address
given at +0xC to +0x10…, and setting the longword at +4 to 1. This is a
factory or development readout facility; the games never use it.

---

## 6. Boot sequence and program upload

### 6.1 Reset

At power-up the internal boot ROM transfers the **boot object** from host
memory into local memory and starts executing it at local address 0. The
boot object sits at a fixed address in every game ROM:

| Host address | Content |
|---|---|
| `0x200100` | longword `0x00000259` (601): number of longwords that follow |
| `0x200104`–`0x200A67` | 527 words of boot code (scrambled for this chip) + 74 words of text ("Warning: Sub Board is not connected with Mother Board…", 1994 Konami) |
| `0x200A68` | longword `0x00002000`: initial stack pointer |
| `0x200A6C` | the kernel image (§6.3), 6792 bytes |

`s11` holds `0x200100` (with configuration nibbles above bit 24) and is
the only absolute address the boot code relies on; everything else is
computed from it.

### 6.2 Boot code

1. Pulse bit 7 of the GX write port at `0xD56000` (0, 0x80, 0x80, 0): the
   motherboard watchdog kick. Repeated between the long self-test phases.
2. Compare the configuration nibble `s11[31:28]` against a 16-entry table
   embedded in the object (an identity check); a match aborts.
3. Clear the mailbox; load the stack pointer from `0x200A68`.
4. Local RAM test of words `0x1000`–`0x1FFF`: fill with a linear
   congruential sequence (`x = x × 0x41C64E6D + 12345`, using `mul`
   with sixteen no-ops of latency), read back and compare, four passes
   with different seeds/patterns.
5. Copy the whole object to `0x1000`, continue from the copy, and test
   words `0x0000`–`0x0FFF` the same way. A failure in either test leads
   to the error loop (§5.3) with a bit code identifying the pass.
6. Motherboard check: byte-sum host addresses `0x000000`–`0x00797B` (the
   GX BIOS) and require `0x000C27DF`; decode the object's 69-word warning
   text and compare it with the same text stored in plain form in the
   BIOS at `0x002558`. The chip refuses to start on any other board.
7. Kernel load: locate the kernel at `s11 + 4 + 4 × 601 + 4`, check its
   magic, decrypt the code section (word count in the header) to local
   `0x0000` and the data section behind it, compare the 16-bit checksum of
   the result with the header, and record the first free word after
   code + data + workspace.
8. Jump to local 0 with `r24` = first free local word and `r25` = stack
   top. The boot copy at `0x1000` is abandoned (it lies inside the region
   the loader later hands to programs and the stack).

### 6.3 Image format

Both the kernel and every game program are **images** with a 24-byte
header followed by a code section and a data section. The image is a
contiguous block in the game ROM (unaligned addresses are common); the
host passes only its address.

| Offset | Field | Notes |
|---|---|---|
| +0 | magic `0xFEF724FB` | |
| +4 | code words (A) | XOR `0x3C96` |
| +6 | data words (B) | XOR `0xB9E5` |
| +8 | workspace words (C) | XOR `0x67A5`; uninitialised local memory given to the program |
| +0xA | LOAD entry (D) | XOR `0x3616`; word offset into the code, invoked once after loading |
| +0xC | RUN entry (E) | XOR `0x7D8D`; invoked by command 1 (0 in every image) |
| +0xE | third entry (F) | XOR `0x57BB`; invoked by commands 3 and 4 (a bare `ret` in every image) |
| +0x10 | flags (G) | XOR `0x796C`; bit 12 = image is stored in plain text; bits 11:0 must equal `((A + B) ^ D ^ E ^ F) & 0xFFF`; bits 15:13 select the decryption variant |
| +0x12 | checksum | 16-bit sum of all halfwords of the stored (decrypted) code and data words |
| +0x14 | salt | 32 bits mixed into the program key |
| +0x18 | code section | A words; encrypted unless G bit 12 is set; stored scrambled for the chip's fetch unit |
| +0x18 + 4A | data section | B words; encrypted like the code section (and with the cipher restarted), but not scrambled |

The code section ends with the entry points D and F, each a bare `ret`
in the shipped images; when the code word count would otherwise be odd a
pad word `0xD7DE2E7F` (which decodes as a harmless `sub s31, s30, #0x25F`)
is appended, because the cipher works on 64-bit blocks. Every image
length is a multiple of 8 bytes.

The kernel image is decrypted by the boot code with a stream cipher;
program images are decrypted by the kernel with a block cipher keyed from
the chip secret (`s10`), the configuration nibble in `s11`, the flags word
and the salt. Both are per-title: the same program encrypted for another
chip would fail the checksum and never run. Both ciphers and every
title's constants are in §8.

### 6.4 The kernel

After the boot code jumps to 0, the kernel initialises its variables
(local `0x69C`: base pointers, stack tops, the packet pointer of the last
RESET), its eight 12-word handle records (`0x6C0`), and its block
allocator, then enters the main loop:

```
loop:  ori.b s6, #1              ; ready
       wait for s7 != 0          ; mailbox
       r4 = s7; st.abs r0,[0xCC0000]
       if [r4] != 0xFEF724FB: goto loop
       [r4+9] = 1                ; state: accepted
       cmd = [r4+8]; if cmd > 6: cmd = 0
       r24 = r4; jsr table[cmd]  ; handler returns r31 = 1 (ok) / 0 (fail)
       [r4+9] = r31 ? 2 : 3      ; END / ERROR
       call post-reset hook
       ori.b s6,#1; ori.b s6,#3; ori.b s6,#1   ; IRQ 4 pulse
       goto loop
```

A **handle record** holds: the handle id, the image address, three block
descriptors (code, data, workspace), the three entry offsets E, D, F, and
four parameter words. Blocks are allocated from the free area above the
kernel in 16-word units.

### 6.5 LOAD, step by step

Given a LOAD packet with argument 1 = image address and argument 2 =
pointer to four longwords:

1. Find a free handle record (or the record already holding this handle
   id, for a reload).
2. Parse the header into a temporary record: unmask A…G, keep the
   checksum and salt, note the body address (+0x18). Reject the image if
   the consistency field in G does not match.
3. Check that A, B and C (each rounded up to 16 words) fit in free memory.
4. Allocate the code block, the data block (if B > 0) and the workspace
   block (if C > 0).
5. Derive the program key from `s10`, the `s11` nibble, G and the salt.
6. Transfer the code section into the code block and the data section
   into the data block, decrypting 8 bytes at a time (or copying plainly
   when G bit 12 is set). Host reads are done in burst mode (`s0` bit 4).
7. Sum the halfwords of both blocks and compare with the header checksum;
   on mismatch the kernel hangs in an error loop (the game will hang
   waiting).
8. Store D, E, F and the block descriptors in the handle record.
9. Copy the four longwords at argument 2 into the record's parameter
   words and **invoke entry D** (§6.7). Its return value is ignored.
10. Return success: state END, interrupt.

### 6.6 RUN, step by step

1. Find the handle record; fail if absent.
2. Copy the four longwords at argument 1 into the record's parameter
   words (burst mode).
3. Invoke entry E.
4. Store the program's `r31` into the packet at +0xC, overwriting the
   parameter pointer. (The 68020 library exposes this as the command's
   result; the sprite programs leave the workspace address there, the
   Crazy Cross program a computed table pointer.)
5. State END, interrupt.

### 6.7 Invocation and calling convention

The kernel's invoke routine:

1. converts the three block descriptors to local addresses;
2. sets `r1` = address of the four parameter words in the handle record
   (`[r1]` = p1, `[r1+1]` = p2, …);
3. saves `r4`–`r19` and `s1` in the kernel variables area, switches `s1`
   to the program stack (kernel stack top − 0x80);
4. sets `r3` = data block, `r2` = workspace block;
5. `jsr` to code block + entry offset;
6. on return restores `r2`, `r3`, `s1`, `r4`–`r19`; `r31` is the result.

A program therefore sees:

| Register | Contents at entry |
|---|---|
| `r1` | pointer to p1..p4 (local words) |
| `r2` | workspace (C words, contents preserved between invocations, uninitialised at load) |
| `r3` | data section |
| `r31` | workspace address (the default result) |
| `s1` | program stack |

Programs must preserve `r4`–`r19` if they use them (all shipped programs
push and pop them), may use `r20`–`r30` freely, and return with
`ret s31`. Code is position-independent: branches are relative, tables
are `s2`-relative, and data/workspace are reached through `r3`/`r2`.

### 6.8 A frame in the life of a sprite program

Taking Daisu-Kiss as the example: the game builds up to 128 "object"
entries of 256 bytes at `0xC00000`, each pointing at a ROM-resident
"set" of tile records with a global position, flip, zoom, colour
overrides and a priority. Once per frame it issues RUN with
p2 = `0xD20000` or `0xD21000` (double-buffered sprite RAM) and, when the
END state arrives, points the sprite chip at that buffer. The ESC program
sorts the objects by priority, expands every set into 16-byte sprite
entries with zoom scaling and screen clipping, and writes exactly 256
entries. The 68020 never touches the sprite list itself; a board without
the chip shows no sprites and hangs at the first RUN.

Other shipped programs: Tokimeki Memorial Taisen Puzzle-dama and Dragoon
Might copy 4 KB into sprite RAM (a job the 68020 could do itself),
Tokkae-dama the same but the 68020 repeats the copy anyway, Crazy Cross
computes `p2 + ((byte[p1+2] & 7) << 3)` and the 68020 recomputes it,
Salamander 2 converts object lists with a priority histogram, and Sexy
Parodius adds an after-image effect by re-reading the previous frame's
list from the other buffer.

### 6.9 Sexy Parodius: what the host sends

Observed in MAME 0.289 with a Lua tap on writes to `0xCC0000` and on CPU
accesses to work RAM `0xC00000`–`0xC003FF` (stage 1 in play, frames
2600–2602 from power-on):

- Once, at start-up: RESET (command 5, reply buffer `0xC00034`), then
  LOAD (command 2) of the image at host `0x2FBA11`.
- Every frame, one RUN (command 1). The packet moves through work RAM
  (`0xC002D0`, `0xC002A4`, `0xC00278`, …, 0x2C bytes apart); argument 1
  is always `0xC00058`. The host library writes the four parameters
  there first (68020 PC `0x288AAC`):

  | Frame | p1 | p2 | p3 | p4 |
  |---|---|---|---|---|
  | 2600 | 2 | 0 | `0xD20000` | `0x2ED` |
  | 2601 | 2 | 0 | `0xD21000` | `0x2EE` |
  | 2602 | 2 | 0 | `0xD20000` | `0x2EF` |

  p3 alternates between the two halves of sprite RAM (the double buffer
  of §6.8, here in the third parameter rather than the second) and p4
  counts frames.
- After each RUN the host writes zero to the mailbox twice (keep-alives)
  and reads the packet's state through the library.

MAME's high-level emulation (`konamigx.cpp`, `sexyparo_esc`) turns every
RUN into `generate_sprites(0xC00604, 0xD20000, 0xFC)`: it ignores all
four parameters, always writes the first half of sprite RAM, and never
runs the uploaded program. On the PCB, Sexy Parodius shows snow (stage 1),
rain and stars drawn over the other sprites; MAME and an FPGA
reimplementation that follows it show none of them, and at those moments
none is present in work RAM's object table, in sprite RAM or in the
tilemaps (checked from a save state taken on hardware during the snow).

The decrypted program (§8, `konami_056734/prog_sexyparo.txt`) generates
them. Words 0–230 are the Daisu-Kiss sprite builder of §7 with the object
table at `0xC00604` (248 entries) and the destination taken from p3. Words
231–242 then dispatch on p1 to an overlay that fills the rest of the list,
padding with index-only entries up to its first slot:

| p1 | Words | Slots | Effect | Word 0 / colour | Tiles |
|---|---|---|---|---|---|
| 0 | 243 | — | none: pad to 256 | | |
| 1 | 249 | `0xE0`–`0xFF` | starfield | `0x8000` / `0xE6` | data 8–11 by `slot & 3` |
| 2 | 314 | `0xE0`–`0xFF` | snow | `0x8000` / `0xA6` | data 108–139, 32 frames of a tumbling flake |
| 3 | 391 | `0xE0`–`0xFF` | rain | `0xA400` / `0xC4` | `0x12B5` |
| 4 | 451 | `0xC0`–`0xFF` | particles at object positions | `0xA000` / `0x66` | data 140–147 |

Every overlay keeps its state in the previous frame's list, read from the
other half of sprite RAM (`p3 ^ 0x1000`): a slot whose previous tile is
the one the overlay would have drawn there is still alive, and its
position is carried on (for the starfield and snow, x as 16.16 with the
fraction in the unused word +0xE). A dead slot respawns when `(p4 >> k) + slot` is a multiple of 32
(k = 3 for snow, 2 for rain), so one particle starts every few frames.
Motion is skipped while p2 is non-zero (pause):

- **Starfield:** x starts at data 12–43 (`0x18`–`0x108`), y at data 76–107;
  x moves left by data 44–75 (−1/16 to −1/2 pixel per frame).
- **Snow:** spawns at y = 0, x = data 12–43; y += 1, x −= 1/2 per frame;
  the tile is `data[108 + ((y + slot) & 31)]`.
- **Rain:** spawns at y = 0, x = data 12–43; y += 8, x −= 2 + `[r1+4]`.
- **Particles:** for each object from `data[148 + (p4 & 3)]` (`0xC00E04`,
  `0xC03B04`, `0xC06E04`, `0xC06E04`) to `0xC0DE04`, a slot spawns at the
  object's position (+4 in x) when `slot × 8 ≡ p4 & 0xF8`; it then moves by
  −(p4 & 1) in y and −`[r1+4]` in x, and its tile is
  `data[140 + ((y >> 2) + slot) & 7]` (index 0 ends it).

In every case x wraps by +0x120 when it goes negative. `[r1+4]` is the
word after the four parameters in the handle record (§6.4); where its
value comes from is not known.

The Saturn port of the game (Sexy Parodius, Japan) draws the same four
overlays in SH-2 code: the VDP1 list builder at `0x0602231C` dispatches on
the same 0–4 value to routines at `0x06022594` (starfield), `0x06022758`
(snow), `0x06022978` and `0x06022B34`, and its snow spawn table at
`0x06080D54` holds the same 32 values as data 12–43.

---

## 7. Annotated program: Daisu-Kiss sprite list generator

Image at host `0x2F8AAD`: 258 code words, 8 data words, 256-word
workspace. Entries: LOAD = 256 (`ret`), RUN = 0, third = 257 (`ret`).
Data section: `0, 0x400, 0x800, 0xC00, 0, 0x100, 0x200, 0x300`.

Input object entry (256 bytes at `0xC00000 + 0x100·n`, n = 0…127):

| Offset | Field |
|---|---|
| +0 | 32-bit pointer to the set (0 = unused entry) |
| +4 | global x |
| +8 | global y |
| +0xC | flip x (non-zero = flip) |
| +0xE | flip y |
| +0x10 | colour: bit 15 → replace colour[4:0] with bits 4:0; bit 14 → rotate colour[4:0] by bits 4:0 |
| +0x12 | bit 15 → replace colour[7:5] with bits 7:5 |
| +0x14 | zoom x (0 → 0x40 = 1:1) |
| +0x16 | zoom y |
| +0x18 | bit 15 → colour[11:10] = data[bits 1:0] |
| +0x1A | bit 15 → colour[9:8] = data[4 + bits 1:0] |
| +0x1C | priority 0…255 (≥ 256 = skip) |
| +0x1E | link to the next entry of the same priority (written by this program) |

A set is a 16-bit record count followed by 10-byte records: tile, flip,
colour, y, x; a tile of `0xFFFF` continues at the address formed by the
next two halfwords.

Output entry (16 bytes at p2 + 16·i, i = 0…255): +0 flip ^ global flip |
index, +2 tile, +4 y, +6 x, +8 zoom y, +0xA zoom x, +0xC colour, +0xE
unused.

Register use: r4 table base, r5 current object, r6 list-head cursor, r7
output pointer, r9 loop counter, r10 output index, r11 = 256 limit, r12
records left in the set, r13 set record pointer, r14 = global x + 0x10,
r15 = global y, r16 global flip bits, r17/r18 flip x/y flags, r19 zoom
pair, r22 zoom, r23 = 0x800, r24/r25 x/y scale factors, r26 colour OR
value, r27 colour AND mask, r28 colour rotate.

```
   0 001d0402  push r19            ; save r4..r19 (callee-saved)
   1 00190402  push r18
   2 00150402  push r17
   3 00110402  push r16
   4 c01d0002  push r15
   5 c0190002  push r14
   6 c0150002  push r13
   7 c0110002  push r12
   8 801d0002  push r11
   9 80190002  push r10
  10 80150002  push r9
  11 80110002  push r8
  12 401d0002  push r7
  13 40190002  push r6
  14 40150002  push r5
  15 40110002  push r4
  16 403a0243  mov r6, r2          ; workspace = 256 list heads, one per priority
  17 00021080  li r9, #0x100
  18 003102c7  st.w(l) r0, [r6]    ; clear them
  19 405b0003  addi r6, #0x1
  20 80578043  subi.h r9, #0x1
  21 3f38bb3c  bne 18
  22 0012c040  li r4, #0xc00000    ; object table base
  23 0002c060  li r5, #0xc08000    ; one past the last of 128 entries
  24 40120497  and r20, r20, r0
  25 00331040  li r22, #0x100      ; priority limit
  26 40171043  subi r5, #0x100     ; --- pass 1: thread objects into priority lists, top down
  27 402005c7  ld.w r20, [r5]      ; set pointer
  28 00120057  sub r0, r20, r0
  29 01e88800  beq 37              ; unused entry
  30 472085c7  ld.h r20, [r5+0x1c] ; priority
  31 2c128057  sub.h r0, r20, r22
  32 01388840  bcc 37              ; priority >= 256: ignore
  33 441a0017  add r6, r20, r2     ; &head[priority]
  34 402506c7  ld.w(l) r21, [r6]   ; old head
  35 47b405c7  st.w r21, [r5+0x1e] ; entry.link = old head  (written into work RAM)
  36 403502c7  st.w(l) r5, [r6]    ; head = entry
  37 0a120047  sub r0, r4, r5
  38 3ce8bb7c  bcs 26              ; while entry > base
  39 406d01c3  ld.w(l) r7, [r1+0x1]; r7 = p2 = destination buffer
  40 403a0243  mov r6, r2          ; --- pass 2: walk priorities 0..255
  41 801a028b  and r10, r10, r0    ; output index = 0
  42 00221080  li r11, #0x100
  43 00021080  li r9, #0x100
  44 402502c7  ld.w(l) r5, [r6]    ; r5 = list head for this priority
  45 00120147  sub r0, r5, r0      ; (loop head for the list)
  46 2e288800  beq 231             ; empty: next priority
  47 c02401c7  ld.w r13, [r5]      ; set pointer
  48 c02081cf  ld.h r12, [r13]     ; record count
  49 c0970003  addi r13, #0x2
  50 c12881c7  ld.h r14, [r5+0x4]  ; global x
  51 c41b8003  addi.h r14, #0x10   ;   + 16 (hardware offset)
  52 c22c81c7  ld.h r15, [r5+0x8]  ; global y
  53 c01c834e  ext.h r15, r15      ;   sign-extended
  54 00128493  and.h r16, r16, r0  ; global flip bits
  55 032485c7  ld.h r17, [r5+0xc]  ; flip x flag
  56 22128043  sub.h r0, r0, r17
  57 00688800  beq 59
  58 00338447  ori.h r16, #0x1000  ;   -> bit 12
  59 03a885c7  ld.h r18, [r5+0xe]  ; flip y flag
  60 24128043  sub.h r0, r0, r18
  61 00788800  bne 63
  62 0033844b  ori.h r16, #0x2000  ;   bit 13 set when NOT flipped (inverted sense)
  63 00230240  li r23, #0x800      ; 0x40 << 5
  64 401a0697  and r22, r22, r0
  65 45a885c7  ld.h r22, [r5+0x16] ; zoom y
  66 403c8696  div.h r23, r22      ; s8 = 0x800 / zoom_y  (result read at 85)
  67 801a869b  and.h r26, r26, r0  ; colour OR = 0
  68 b43e84c3  not.h r27, r26      ; colour AND = 0xffff
  69 462085c7  ld.h r20, [r5+0x18]
  70 40138463  subi.h r20, #0x8000
  71 01888840  bmi 78              ; bit 15 clear: no override
  72 00070040  li r21, #0x10000    ; data segment tag
  73 46160517  add r21, r21, r3    ; &data[0]
  74 40d30483  andi r20, #0x3
  75 68120517  add r20, r21, r20
  76 80298457  or.h(m) r26, [r20]  ; OR |= {0,0x400,0x800,0xc00}[bits 1:0]
  77 bfdfb4bf  andi.h r27, #0xf3ff ; AND &= ~0x0c00
  78 2c128043  sub.h r0, r0, r22   ; zoom y == 0 ?
  79 00f88800  bne 83
  80 10330040  li r22, #0x40       ;   default 1:1
  81 8016859b  and.h r25, r25, r0  ;   y scale = 0 (means "no scaling")
  82 00d88800  jmp 86
  83 00000000  nop                 ;   divider latency
  84 00000000  nop
  85 8036046b  mov r25, s8         ;   y scale = 0x800 / zoom_y
  86 003c0656  swap16 r19, r22     ; r19 = zoom_y << 16
  87 40108496  mul.h r20, r20      ; (no effect on the result; r20 is overwritten below)
  88 46a085c7  ld.h r20, [r5+0x1a]
  89 40138463  subi.h r20, #0x8000
  90 01888840  bmi 97
  91 01070040  li r21, #0x10004    ; &data[4]
  92 46160517  add r21, r21, r3
  93 40d30483  andi r20, #0x3
  94 68120517  add r20, r21, r20
  95 80298457  or.h(m) r26, [r20]  ; OR |= {0,0x100,0x200,0x300}[bits 1:0]
  96 bfdf87bf  andi.h r27, #0xfcff
  97 44a085c7  ld.h r20, [r5+0x12]
  98 40138463  subi.h r20, #0x8000
  99 00c88840  bmi 103
 100 78138483  andi.h r20, #0xe0
 101 b43a8457  or.h r26, r20, r26  ; OR |= bits 7:5
 102 87dfb7bf  andi.h r27, #0xff1f
 103 401a0697  and r22, r22, r0
 104 452885c7  ld.h r22, [r5+0x14] ; zoom x
 105 403c8696  div.h r23, r22      ; s8 = 0x800 / zoom_x (read at 124)
 106 442485c7  ld.h r21, [r5+0x10]
 107 40320557  mov r20, r21
 108 40138463  subi.h r20, #0x8000
 109 00c88840  bmi 113
 110 47d38483  andi.h r20, #0x1f
 111 b43a8457  or.h r26, r20, r26  ; OR |= bits 4:0
 112 b81fb7bf  andi.h r27, #0xffe0
 113 c012849f  and.h r28, r28, r0  ; rotate = 0
 114 40320557  mov r20, r21
 115 40178493  andi.h r21, #0x4000
 116 00a88800  beq 119
 117 47d38483  andi.h r20, #0x1f
 118 c0320457  mov r28, r20        ; rotate = bits 4:0
 119 2c128043  sub.h r0, r0, r22   ; zoom x == 0 ?
 120 00f88800  bne 124
 121 503b8443  ori.h r22, #0x40
 122 8012849b  and.h r24, r24, r0  ;   x scale = 0
 123 00588800  jmp 125
 124 8032046b  mov r24, s8         ;   x scale = 0x800 / zoom_x
 125 263e0657  or r19, r22, r19    ; r19 = zoom_y << 16 | zoom_x
 126 402085cf  ld.h r20, [r13]     ; --- per record: tile
 127 7fd3b77f  subi.h r20, #0xffff
 128 01388800  bne 133
 129 c0a401cf  ld.w r13, [r13+0x2] ; 0xffff: continue at the linked set
 130 c02081cf  ld.h r12, [r13]     ;   new count
 131 c0970003  addi r13, #0x2
 132 3e58bb3c  jmp 126
 133 422085cf  ld.h r20, [r13+0x8] ; x
 134 22128043  sub.h r0, r0, r17
 135 00688800  beq 137
 136 68128443  sub.h r20, r0, r20  ; flip x: negate
 137 80108496  mul.h r24, r20      ; s8 = x * scale  (used only if scaling)
 138 30128043  sub.h r0, r0, r24
 139 01b88800  bne 146
 140 5c128417  add.h r20, r20, r14 ; unscaled: x += global x + 16
 141 10030040  li r21, #0x40
 142 68168517  add.h r21, r21, r20
 143 68179443  subi.h r21, #0x1a0
 144 14788840  bcc 226             ; clip unless -0x40 <= x < 0x160
 145 04588800  jmp 163
 146 00000000  nop                 ; multiplier latency (7 words)
 147 00000000  nop
 148 00000000  nop
 149 00000000  nop
 150 00000000  nop
 151 00000000  nop
 152 00000000  nop
 153 4032846a  asr.h r20, s8       ; x = (x * 0x800/zoom) >> 5
 154 40328456  asr.h r20, r20
 155 40328456  asr.h r20, r20
 156 40328456  asr.h r20, r20
 157 40328456  asr.h r20, r20
 158 5c128417  add.h r20, r20, r14 ; += global x + 16
 159 00031040  li r21, #0x100
 160 68168517  add.h r21, r21, r20
 161 4817b443  subi.h r21, #0x320
 162 0ff88840  bcc 226             ; clip unless -0x100 <= x < 0x220
 163 41138403  addi.h r20, #0x4    ; + 4
 164 41b087c7  st.h r20, [r7+0x6]  ; out.x
 165 41a085cf  ld.h r20, [r13+0x6] ; y
 166 24128043  sub.h r0, r0, r18
 167 00688800  beq 169
 168 68128443  sub.h r20, r0, r20  ; flip y: negate
 169 40108456  ext.h r20, r20
 170 80140496  mul r25, r20        ; s8 = y * scale
 171 32128043  sub.h r0, r0, r25
 172 01b88800  bne 179
 173 5e128417  add.h r20, r20, r15 ; unscaled: y += global y
 174 0c030040  li r21, #0x30
 175 68168517  add.h r21, r21, r20
 176 58179443  subi.h r21, #0x160
 177 0c388840  bcc 226             ; clip unless -0x30 <= y < 0x130
 178 06588800  jmp 204
 179 00000000  nop                 ; multiplier latency (15 words)
 ...           nop
 193 00000000  nop
 194 4032046a  asr r20, s8         ; y = (y * 0x800/zoom) >> 5
 195 40320456  asr r20, r20
 196 40320456  asr r20, r20
 197 40320456  asr r20, r20
 198 40320456  asr r20, r20
 199 5e120417  add r20, r20, r15   ; += global y
 200 3c030040  li r21, #0xf0
 201 68168517  add.h r21, r21, r20
 202 7817a443  subi.h r21, #0x2e0
 203 05b88840  bcc 226             ; clip unless -0xf0 <= y < 0x1f0
 204 413087c7  st.h r20, [r7+0x4]  ; out.y
 205 40a085cf  ld.h r20, [r13+0x2] ; record flip bits
 206 601284d7  xor.h r20, r20, r16 ;   ^ global flip
 207 54328457  or.h r20, r20, r10  ;   | output index
 208 403087c7  st.h r20, [r7]      ; out.word0
 209 402085cf  ld.h r20, [r13]
 210 40b087c7  st.h r20, [r7+0x2]  ; out.tile
 211 023c07c7  st.w r19, [r7+0x8]  ; out.zoom y, zoom x
 212 412085cf  ld.h r20, [r13+0x4] ; colour
 213 6812879b  and.h r20, r27, r20 ;   & mask
 214 6832865b  or.h r20, r26, r20  ;   | overrides
 215 38128043  sub.h r0, r0, r28
 216 01288800  beq 221
 217 6816841f  add.h r21, r28, r20 ;   rotate: colour[4:0] = (colour + rotate) & 0x1f
 218 47d78483  andi.h r21, #0x1f
 219 7813b7bf  andi.h r20, #0xffe0
 220 68328557  or.h r20, r21, r20
 221 433087c7  st.h r20, [r7+0xc]  ; out.colour
 222 441f0003  addi r7, #0x10      ; next output entry
 223 805b8003  addi.h r10, #0x1
 224 1612824b  sub.h r0, r10, r11
 225 03788840  bcc 239             ; 256 entries written: done
 226 c2970003  addi r13, #0xa      ; next record
 227 c0538043  subi.h r12, #0x1
 228 2678bb3c  bne 126
 229 47a401c7  ld.w r5, [r5+0x1e]  ; next object in this priority list
 230 1198bb3c  jmp 45
 231 405b0003  addi r6, #0x1       ; next priority
 232 80570043  subi r9, #0x1
 233 10b8bb3c  bne 44
 234 803883c7  st.h r10, [r7]      ; --- pad the list: word0 = index only
 235 441f0003  addi r7, #0x10
 236 805b8003  addi.h r10, #0x1
 237 1612824b  sub.h r0, r10, r11
 238 3ee8bb7c  bcs 234
 239 40310002  pop r4              ; restore r4..r19; r31 still holds the workspace
 240 40350002  pop r5              ;   address, which the kernel reports at packet+0xC
 241 40390002  pop r6
 242 403d0002  pop r7
 243 80310002  pop r8
 244 80350002  pop r9
 245 80390002  pop r10
 246 803d0002  pop r11
 247 c0310002  pop r12
 248 c0350002  pop r13
 249 c0390002  pop r14
 250 c03d0002  pop r15
 251 00310402  pop r16
 252 00350402  pop r17
 253 00390402  pop r18
 254 003d0402  pop r19
 255 c01d0c42  ret s31
 256 c01d0c42  ret s31             ; LOAD entry (D = 256): nothing to initialise
 257 c01d0c42  ret s31             ; third entry (F = 257)
```

Points worth noticing in this listing:

- Words 26–38 show the compare-and-branch idiom: `sub r0, a, b` sets the
  flags, `bcs`/`bcc` test the borrow (unsigned compare), `bmi`/`bpl`
  test the sign, `beq`/`bne` test zero.
- Words 66 and 85 and words 137/170 and 153/194 show the `div`/`mul`
  protocol: start the operation, do other work or no-ops, then read `s8`.
  The program keeps the no-op padding even on the path that skips the
  read.
- Words 72–76 and 91–95 show data-section access: a tagged constant
  added to `r3`, then a memory-operand OR.
- Words 35/229 show that the program treats the game's own object table
  as scratch: it stores link pointers into work RAM.
- The output index rather than the priority goes into word 0 (207), and
  x carries a fixed +20 (51 and 163) that the game's sprite offset
  registers absorb.

---

## 8. Encryption and per-chip constants

The boot code is only scrambled (§2.4), so it can be read once a chip's
XOR constant and lane map are known. It holds the kernel's stream cipher
and key derivation; the decrypted kernel holds the program cipher. Every
kernel and program image of the nine titles below decrypts with these
and passes its header checksum, and the Daisu-Kiss program decrypts to
the listing of §7 word for word. Listings are in
[`konami_056734/`](konami_056734/): `boot.txt`, `kernel.txt` and
`prog_<set>.txt` (canonical encoding), with `esccipher.py` (both ciphers
and the constants table) and `escdis.py` (the disassembler that produced
them).

### 8.1 Kernel key and stream cipher (boot words 335–383)

```
key = (s11[27:24] * 0x01010101) ^ 0x36AE2592 ^ (s10[15:0] * 0x00010001)

a = key ^ 0x6E8EF8FC;  b = key
for each 32-bit word w (code section, then data section; both start from key):
    plain = w ^ a
    a = a + b
    b = b ^ rot16(a)
```

The checksum of §6.3 is taken over the plain words. Because `s10` and the
nibble only enter as this XOR, 16 different (`s10`, nibble) pairs give
the same kernel key; the program key below separates them.

### 8.2 Program key (kernel words 875–925)

```
k0 = s11[27:24] << 28 | (G & 0xFFF) << 16 | s10[15:0]
k1 = k0 ^ salt
k2 = (G >> 13) << 3 | ((E ^ s10) & 3)            (8 bits)
```

### 8.3 Program cipher (kernel words 1387–1665)

A FEAL-8 variant on 64-bit blocks, each block two big-endian 32-bit words
(L, R) of the image; the data section restarts from the same key. With
`S0(a,b) = rol2((a+b) mod 256)` and `S1(a,b) = rol2((a+b+1) mod 256)`:

- **Key schedule:** FEAL's: A = k0, B = k1, D = 0; eight times
  `B' = fk(A, B ^ D)`, then D = A, A = B, B = B'; each B' gives two 16-bit
  subkeys, high half first: K0…K15.
- **Rounds:** `(k2 >> 2) & 6` rounds, or 8 when that is 0 (every shipped
  image has G bits 15:13 = 4, so 8).
- **Block:** `R ^= K1:K0; L ^= K3:K2; R ^= L`; per round
  `(L, R) = (R, L ^ f(R, K[4 + r]))`; then `L ^= R; L ^= K13:K12;
  R ^= K15:K14` (for 8 rounds); the output is (R, L).
- **Round function** `f(a, k)`, chosen by `k2 & 3`; a0…a3 are the bytes
  of a (a0 most significant), b0/b1 the high/low byte of the subkey, and
  the result is f0…f3:

  | `k2 & 3` | x | y | then |
  |---|---|---|---|
  | 0 (FEAL's f) | a0^a1^b0 | a2^a3^b1 | f1=S1(x,y) f2=S0(y,f1) f0=S0(a0,f1) f3=S1(a3,f2) |
  | 1 | a0^a2^b0 | a1^a3^b1 | f2=S1(x,y) f1=S0(y,f2) f0=S0(a0,f2) f3=S1(a3,f1) |
  | 2 | a2^a3^b0 | a0^a1^b1 | f3=S1(x,y) f0=S0(y,f3) f2=S0(a2,f3) f1=S1(a1,f0) |
  | 3 | a0^a1^b1 | a2^a3^b0 | f0=S1(x,y) f3=S0(y,f0) f1=S0(a1,f0) f2=S1(a2,f3) |

- **Subkey update:** after every block, with o1/o2 the two output words,
  two of these run in order, the first chosen by `m1 = (k2 ^ k0) & 3`, the
  second by `m2 = ((k1 & 3) | 2) ^ m1` (16-bit arithmetic):

  | m | update |
  |---|---|
  | 0 | `K[o1 & 15] ^= K[(o2 >> 16) & 15]` |
  | 1 | `K[(o1 >> 16) & 15] += 1` |
  | 2 | `K[(o2 >> 16) & 15] += K[o1 & 15]` |
  | 3 | `K[(o2 >> 16) & 15] -= 1` |

  So the same plaintext block never encrypts the same way twice in one
  image.

The routines in the kernel take the round function and the two updates
by address from its data section (data 16–23).

### 8.4 Per-chip constants

`s10` and `s11[27:24]` are the only values per title that the ciphers
need; they were found by searching the 2^16 kernel keys for the one
matching the kernel checksum, then the 16 pairs it allows against the
program checksum. The descrambler of §2.4 is given as the XOR constant
and the lane map: canonical lane k is lane `map[k]` of
`(stored word ^ XOR)`. Both come from the boot code, which is the same
program on every chip: words 122–137 and 142–157 are sixteen no-ops each,
so they hold the XOR constant itself, and the lane map follows from
comparing the 527 code words between chips.

| Set | Title | `s10[15:0]` | `s11[27:24]` | XOR | lane map (k = 0…15) |
|---|---|---|---|---|---|
| `puzldama` | Taisen Puzzle-dama / Crazy Cross | `5D32` | 3 | `91C2C6FB` | 11 8 12 9 4 2 5 6 10 3 7 13 14 0 1 15 |
| `tokkae` | Taisen Tokkae-dama | `9E8E` | B | `6BFBB5B9` | 12 2 6 9 13 0 5 10 8 11 3 4 14 15 7 1 |
| `tkmmpzdm` | Tokimeki Memorial Taisen Puzzle-dama | `4924` | C | `88600EF4` | 9 14 5 1 13 2 0 6 11 15 3 7 8 10 12 4 |
| `daiskiss` | Daisu-Kiss | `89EE` | D | `39556CC0` | 9 12 1 0 6 10 3 15 7 14 2 13 4 5 8 11 |
| `tbyahhoo` | Twin Bee Yahhoo! | `0424` | A | `DE78B8AE` | 15 4 12 2 3 13 11 1 5 8 10 0 6 14 7 9 |
| `gokuparo` | Gokujou Parodius | `8D8C` | 3 | `2C3EA1C4` | 15 11 13 14 9 3 0 7 6 1 8 5 12 10 4 2 |
| `dragoonj` | Dragoon Might | `5963` | B | `049DB8E5` | 4 7 9 3 14 8 6 13 5 0 15 1 11 12 2 10 |
| `salmndr2` | Salamander 2 | `1EC6` | A | `B3F135B3` | 7 8 2 4 10 14 15 6 9 1 0 12 3 5 13 11 |
| `sexyparo` | Sexy Parodius | `896A` | E | `1886AE1D` | 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 |
| `rungun2`, `slamdnk2` | Run and Gun 2 / Slam Dunk 2 | `1593` | 0 | `6862ABCF` | 4 9 12 3 5 15 8 0 7 14 1 10 11 6 2 13 |

Every set in MAME 0.289's `konamigx.cpp` was checked. Clones carry their
parent's chip: `crzcross` (puzldama's), `mtwinbee` (tbyahhoo's),
`sexyparoa`, `dragoona`, `salmndr2a`; their program images sit at other
addresses (`0x2FE67B`, `0x2FB883`, `0x251C78`, `0x234E50`) and decrypt with
the same key. `fantjour` and `fantjoura` carry gokuparo's kernel (key
`B821AB1D`) and no program. The other sets have no kernel at `0x200A6C`:
`le2` and its clones, `ggreats2`, `opengolf`, `opengolf2`, `racinfrc`,
`racinfrcu`, `rushhero`, `soccerss` and clones, `vsnetscr` and clones,
`winspike` and clones, `sexyparoebl`.

Run and Gun 2's program (`prog_rungun2.txt`; Slam Dunk 2's is the same at
`0x254F40`) copies blocks of bytes, the routine chosen by the low two bits
of the fourth parameter.

`s10[31:16]` and `s11[31:28]` are not used by the ciphers; `s11[31:28]` is
the boot code's identity check (§6.2). The upper half of `s10` is not known.

### 8.5 Running the chip

An instruction-level model of the chip (this document's semantics, plus
`neg` for `30002`) boots Sexy Parodius from reset to the kernel's
mailbox wait in 1,834,459 instructions, given only: local memory loaded
with the 601 boot words from `0x200104`, pc = 0, `s10` and `s11` as above
(with `s11[31:28]` = 0, which passes the identity check), and the BIOS
and game ROM on the host bus. The boot code passes its RAM tests and the
BIOS checksum, decrypts the kernel and jumps to it. Fed the packets that
MAME's 68020 writes (captured with a Lua tap), the kernel completes the
init word, RESET (911 instructions), LOAD (287,128 instructions, most of
them the decryption) and each RUN (about 6,800 instructions in the
attract demo) with state END. What the internal boot ROM does before
word 0 is not known; this setup is the simplest one that works.
