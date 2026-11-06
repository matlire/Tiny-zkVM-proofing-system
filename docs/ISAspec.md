## ISA — RV32I

|Feature|Explanation|
|---|---|
| Instruction width     | 32 bits                                               |
| Register width        | 32 bits (`XLEN = 32`)                                 |
| Integer registers     | 32 registers, `x0`–`x31`                              |
| Zero register         | `x0` always reads as zero; writes to it are discarded |
| Program counter       | `pc` holds the address of the current instruction     |
| Sequential execution  | the next instruction is at `pc + 16`                   |

### Register Names and Conventional Roles

RV32I has 32 integer registers, `x0`–`x31`. ABI names are aliases for these same registers, not additional registers.

| Register | ABI Alias | Conventional Role |
|---|---|---|
| `x0`          | `zero`        | Constant zero; writes are discarded                       |
| `x1`          | `ra`          | Return address                                            |
| `x2`          | `sp`          | Stack pointer                                             |
| `x3`          | `gp`          | Global pointer                                            |
| `x4`          | `tp`          | Thread pointer                                            |
| `x5`–`x7`     | `t0`–`t2`     | Temporary registers; caller-saved                         |
| `x8`          | `s0` / `fp`   | Callee-saved register; optional frame pointer             |
| `x9`          | `s1`          | Callee-saved register                                     |
| `x10`–`x17`   | `a0`–`a7`     | Function arguments; `a0`–`a1` also carry return values    |
| `x18`–`x27`   | `s2`–`s11`    | Callee-saved registers                                    |
| `x28`–`x31`   | `t3`–`t6`     | Temporary registers; caller-saved                         |

**Caller-saved:** the caller must save a value if it needs it after a call.
**Callee-saved:** the called function must restore the register if it modifies it.
The callee must also restore `sp`; `gp` and `tp` are reserved by convention.

### Instruction Fields

| Field | Width | Meaning |
|---|---:|---|
| `opcode`  | 7 bits | Identifies the instruction class                     |
| `funct3`  | 3 bits | Selects an operation within an instruction class     |
| `funct7`  | 7 bits | Further distinguishes operations where required      |
| `rs1`     | 5 bits | First source register index                          |
| `rs2`     | 5 bits | Second source register index                         |
| `rd`      | 5 bits | Destination register index                           |
| `imm`     | Varies | Immediate value encoded in the instruction           |


### Instruction Formats

Most significant bit (`31`) -> Least significant bit (`0`)

#### Register Format — R-Type

```text
funct7 | rs2 | rs1 | funct3 | rd | opcode
   7      5     5      3      5      7     | bits
```

#### Immediate Format — I-Type

```text
imm[11:0] | rs1 | funct3 | rd | opcode
   12        5      3      5      7     | bits
```

#### Store Format — S-Type

```text
imm[11:5] | rs2 | rs1 | funct3 | imm[4:0] | opcode
    7         5     5      3         5         7     | bits
```

#### Branch Format — B-Type

```text
imm[12] | imm[10:5] | rs2 | rs1 | funct3 | imm[4:1] | imm[11] | opcode
   1          6         5     5      3         4          1         7     | bits
```

The reconstructed byte offset has an implicit zero in bit `0`.

#### Upper Immediate Format — U-Type

```text
imm[31:12] | rd | opcode
    20        5      7     | bits
```

The reconstructed immediate has twelve zero bits at the bottom.

#### Jump Format — J-Type

```text
imm[20] | imm[10:1] | imm[11] | imm[19:12] | rd | opcode
   1         10          1           8         5      7     | bits
```

The reconstructed byte offset has an implicit zero in bit `0`.

### Immediate Generation

The following expressions reconstruct 32-bit immediates from the instruction word `inst`.

```verilog
// I-type: sign-extended 12-bit immediate
imm_i = {{20{inst[31]}}, inst[31:20]};

// S-type: sign-extended 12-bit immediate
imm_s = {{20{inst[31]}}, inst[31:25], inst[11:7]};

// B-type: sign-extended 13-bit byte offset; bit 0 is zero
imm_b = {{19{inst[31]}}, inst[31], inst[7],
         inst[30:25], inst[11:8], 1'b0};

// U-type: upper 20 bits followed by 12 zero bits
imm_u = {inst[31:12], 12'b0};

// J-type: sign-extended 21-bit byte offset; bit 0 is zero
imm_j = {{11{inst[31]}}, inst[31], inst[19:12],
         inst[20], inst[30:21], 1'b0};
```

### Operation Notation

In the descriptions below:

- `rs1`, `rs2`, and `rd` denote register **values**
- `s32(x)` interprets a 32-bit value as a signed two's-complement integer
- `u32(x)` interprets a 32-bit value as an unsigned integer
- `sextN(x)` sign-extends an `N`-bit value to 32 bits
- `zextN(x)` zero-extends an `N`-bit value to 32 bits
- `LSL(x, n)` shifts left, filling with zeros
- `LSR(x, n)` shifts right, filling with zeros
- `ASR(x, n)` shifts right, replicating the sign bit
- `loadN(address)` and `storeN(address, value)` read and write `N` bits

Arithmetic results and address calculations wrap modulo `2^32`. Arithmetic overflow does not cause a trap. All source operands and `pc` refer to their values **before** the instruction executes.

### Supported Instructions

The RV32I base contains **40 instructions**

#### Instruction Encodings

All field values below are binary. `opcode = inst[6:0]`, `funct3 = inst[14:12]`, and `funct7 = inst[31:25]`.

`—` means there is no separate function selector in that position; those bits may encode an immediate or other fields. It does not mean the bits must be zero or may always be ignored when decoding.

| Instruction | Format | opcode [6:0] | funct3 [14:12] | funct7 / shift imm[11:5] [31:25] |
|---|---|---|---|---|
| `ADD`     | `R` | `0110011` | `000` | `0000000` |
| `SUB`     | `R` | `0110011` | `000` | `0100000` |
| `SLT`     | `R` | `0110011` | `010` | `0000000` |
| `SLTU`    | `R` | `0110011` | `011` | `0000000` |
| `SLL`     | `R` | `0110011` | `001` | `0000000` |
| `SRL`     | `R` | `0110011` | `101` | `0000000` |
| `SRA`     | `R` | `0110011` | `101` | `0100000` |
| `XOR`     | `R` | `0110011` | `100` | `0000000` |
| `OR`      | `R` | `0110011` | `110` | `0000000` |
| `AND`     | `R` | `0110011` | `111` | `0000000` |
| `ADDI`    | `I` | `0010011` | `000` | — |
| `SLTI`    | `I` | `0010011` | `010` | — |
| `SLTIU`   | `I` | `0010011` | `011` | — |
| `SLLI`    | `I` | `0010011` | `001` | `0000000` |
| `SRLI`    | `I` | `0010011` | `101` | `0000000` |
| `SRAI`    | `I` | `0010011` | `101` | `0100000` |
| `XORI`    | `I` | `0010011` | `100` | — |
| `ORI`     | `I` | `0010011` | `110` | — |
| `ANDI`    | `I` | `0010011` | `111` | — |
| `LUI`     | `U` | `0110111` | — | — |
| `AUIPC`   | `U` | `0010111` | — | — |
| `LB`      | `I` | `0000011` | `000` | — |
| `LBU`     | `I` | `0000011` | `100` | — |
| `LH`      | `I` | `0000011` | `001` | — |
| `LHU`     | `I` | `0000011` | `101` | — |
| `LW`      | `I` | `0000011` | `010` | — |
| `SB`      | `S` | `0100011` | `000` | — |
| `SH`      | `S` | `0100011` | `001` | — |
| `SW`      | `S` | `0100011` | `010` | — |
| `BEQ`     | `B` | `1100011` | `000` | — |
| `BNE`     | `B` | `1100011` | `001` | — |
| `BLT`     | `B` | `1100011` | `100` | — |
| `BGE`     | `B` | `1100011` | `101` | — |
| `BLTU`    | `B` | `1100011` | `110` | — |
| `BGEU`    | `B` | `1100011` | `111` | — |
| `JAL`     | `J` | `1101111` | — | — |
| `JALR`    | `I` | `1100111` | `000` | — |
| `FENCE`   | `I` | `0001111` | `000` | — |
| `ECALL`   | `I` | `1110011` | `000` | — |
| `EBREAK`  | `I` | `1110011` | `000` | — |

#### Integer Register–Register Operations — R-Type

Register-controlled shifts use only the lowest five bits of `rs2`: `shamt = rs2 & 31`.

**Arithmetic**

```text
ADD  rd, rs1, rs2 — Add:      rd = rs1 + rs2
SUB  rd, rs1, rs2 — Subtract: rd = rs1 - rs2
```

**Comparisons**

```text
SLT  rd, rs1, rs2 — Set Less Than (Signed):   rd = 1 if s32(rs1) < s32(rs2), else 0
SLTU rd, rs1, rs2 — Set Less Than (Unsigned): rd = 1 if u32(rs1) < u32(rs2), else 0
```

**Shifts**

```text
SLL  rd, rs1, rs2 — Shift Left Logical:     rd = LSL(rs1, rs2 & 31)
SRL  rd, rs1, rs2 — Shift Right Logical:    rd = LSR(rs1, rs2 & 31)
SRA  rd, rs1, rs2 — Shift Right Arithmetic: rd = ASR(rs1, rs2 & 31)
```

**Bitwise Operations**

```text
XOR  rd, rs1, rs2 — Bitwise XOR: rd = rs1 ^ rs2
OR   rd, rs1, rs2 — Bitwise OR:  rd = rs1 | rs2
AND  rd, rs1, rs2 — Bitwise AND: rd = rs1 & rs2
```

#### Integer Register–Immediate Operations — I-Type

For non-shift instructions, `imm` is a signed 12-bit immediate (`-2048` to `2047`), and `I = sext12(imm)`.

**Arithmetic**

```text
ADDI  rd, rs1, imm — Add Immediate: rd = rs1 + I
```

**Comparisons**

```text
SLTI  rd, rs1, imm — Set Less Than Immediate (Signed):   rd = 1 if s32(rs1) < s32(I), else 0
SLTIU rd, rs1, imm — Set Less Than Immediate (Unsigned): rd = 1 if u32(rs1) < u32(I), else 0
```

`SLTIU` still sign-extends the immediate before interpreting it as unsigned.

**Shifts**

For these instructions, `shamt` is an unsigned 5-bit value (`0` to `31`).

```text
SLLI rd, rs1, shamt — Shift Left Logical Immediate:     rd = LSL(rs1, shamt)
SRLI rd, rs1, shamt — Shift Right Logical Immediate:    rd = LSR(rs1, shamt)
SRAI rd, rs1, shamt — Shift Right Arithmetic Immediate: rd = ASR(rs1, shamt)
```

**Bitwise Operations**

```text
XORI rd, rs1, imm — Bitwise XOR Immediate: rd = rs1 ^ I
ORI  rd, rs1, imm — Bitwise OR Immediate:  rd = rs1 | I
ANDI rd, rs1, imm — Bitwise AND Immediate: rd = rs1 & I
```

#### Upper Immediate Operations — U-Type

Here, `imm20` denotes the encoded 20-bit field. The effective value is `U = imm20 << 12`.

```text
LUI   rd, imm20 — Load Upper Immediate:          rd = U
AUIPC rd, imm20 — Add Upper Immediate to the PC: rd = pc + U
```

`AUIPC` uses the address of the `AUIPC` instruction itself.

#### Load Instructions — I-Type

The effective address is `address = rs1 + sext12(offset)`.

```text
LB  rd, offset(rs1) — Load Byte (Signed):       rd = sext8(load8(address))
LBU rd, offset(rs1) — Load Byte (Unsigned):     rd = zext8(load8(address))

LH  rd, offset(rs1) — Load Halfword (Signed):   rd = sext16(load16(address))
LHU rd, offset(rs1) — Load Halfword (Unsigned): rd = zext16(load16(address))

LW  rd, offset(rs1) — Load Word:                rd = load32(address)
```

#### Store Instructions — S-Type

The effective address is `address = rs1 + sext12(offset)`.

```text
SB rs2, offset(rs1) — Store Byte:     store8(address, rs2[7:0])
SH rs2, offset(rs1) — Store Halfword: store16(address, rs2[15:0])
SW rs2, offset(rs1) — Store Word:     store32(address, rs2[31:0])
```

#### Conditional Branch Instructions — B-Type

A taken branch sets `pc = old_pc + offset`. Otherwise, execution continues at `old_pc + 4`.

`offset` is a signed byte displacement relative to the branch instruction, with a range of `-4096` to `4094` in steps of two bytes.

**Equality**

```text
BEQ rs1, rs2, offset — Branch if Equal:     take branch if rs1 == rs2
BNE rs1, rs2, offset — Branch if Not Equal: take branch if rs1 != rs2
```

**Signed Comparisons**

```text
BLT rs1, rs2, offset — Branch if Less Than:        take branch if s32(rs1) <  s32(rs2)
BGE rs1, rs2, offset — Branch if Greater or Equal: take branch if s32(rs1) >= s32(rs2)
```

**Unsigned Comparisons**

```text
BLTU rs1, rs2, offset — Branch if Less Than Unsigned:        take branch if u32(rs1) <  u32(rs2)
BGEU rs1, rs2, offset — Branch if Greater or Equal Unsigned: take branch if u32(rs1) >= u32(rs2)
```

#### Unconditional Jump Instructions — J-Type and I-Type

Both instructions save `old_pc + 4` in `rd`. Using `rd = x0` discards the return address.

```text
JAL  rd, offset      — Jump and Link:
                       rd = old_pc + 4
                       pc = old_pc + offset

JALR rd, offset(rs1) — Jump and Link Register:
                       target = (old_rs1 + sext12(offset)) & 0xFFFFFFFE
                       rd = old_pc + 4
                       pc = target
```

- `JAL` uses J-type encoding. Its byte offset ranges from
  `-1048576` to `1048574` in steps of two bytes.
- `JALR` uses I-type encoding. Its signed 12-bit offset is measured
  in bytes and is **not** multiplied by two.
- `JALR` clears bit `0` of the computed target.

For base RV32I, instruction targets must be four-byte aligned. A taken branch or jump to a misaligned target raises an instruction-address-misaligned exception.

#### Memory Ordering — FENCE

```text
FENCE pred, succ — Order selected predecessor accesses before selected successor accesses.
```

`pred` and `succ` select combinations of:

| Flag | Access category |
|---|---|
| `i`   | Device input  |
| `o`   | Device output |
| `r`   | Memory reads  |
| `w`   | Memory writes |

For example, `FENCE iorw, iorw` orders all four categories.

The `i` flag means device input, not instruction fetching.

#### Environment Instructions — SYSTEM

```text
ECALL  — Environment Call:  raise an environment-call exception.
EBREAK — Environment Break: raise a breakpoint exception.
```

The execution environment defines how these exceptions are handled. Neither instruction is inherently a program-exit instruction.

### Assembler Pseudoinstructions

Pseudoinstructions are assembler conveniences, not additional RV32I instructions.

| Pseudoinstruction | Expansion |
|---|---|
| `NOP`         | `ADDI x0, x0, 0`  |
| `MV rd, rs`   | `ADDI rd, rs, 0`  |
| `NOT rd, rs`  | `XORI rd, rs, -1` |
| `NEG rd, rs`  | `SUB rd, x0, rs`  |
| `J offset`    | `JAL x0, offset`  |
| `JR rs`       | `JALR x0, 0(rs)`  |
| `RET`         | `JALR x0, 0(x1)`  |

`LI rd, value` loads a 32-bit constant. TinyProof's accepted literal range and exact one- or two-instruction expansion are defined in `spec.md`, under Load-Immediate Pseudoinstruction. Source mnemonics are lowercase; uppercase names here are documentation notation.
