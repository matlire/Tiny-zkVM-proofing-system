## Project Specification — TinyProof RV32

TinyProof is a deterministic virtual machine using a subset of standard RV32I instruction encodings.

Instruction formats and operations are described in [ISAspec.md](ISAspec.md).

Error categories and failure precedence are described in [Espec.md](Espec.md).

This document defines the supported instruction profile, execution environment, executable format, and assembler contract.

### Project Profile

| Feature | Specification |
|---|---|
| Profile               | TinyProof RV32                        |
| Executable version    | `1`                                   |
| Instruction encoding  | Standard RISC-V, 32-bit instructions  |
| Register width        | 32 bits                               |
| Integer registers     | 32 registers, `x0`–`x31`              |
| Guest address width   | 32 bits                               |
| Addressing            | Byte-addressed                        |
| Byte order            | Little-endian                         |
| Instruction alignment | 4 bytes                               |
| RAM size              | 65,536 bytes                          |
| Maximum text size     | 61,440 bytes                          |
| Execution             | Single-threaded and deterministic     |

### Value Notation

Registers contain 32-bit patterns. Individual instructions determine whether these patterns are interpreted as signed or unsigned values.

```text
u32(v)    = v modulo 2^32

s32(v)    = v,        if v < 2^31
            v - 2^32 otherwise

sext(v,n) = v,       if v < 2^(n-1)
            v - 2^n otherwise
```

`sext(v,n)` takes an unsigned n-bit pattern and returns a mathematical signed integer.


Arithmetic results and effective addresses wrap modulo `2^32`. Arithmetic overflow and underflow do not cause traps.


All source registers and the current `pc` are read from the state before the instruction executes, including when source and destination registers overlap.

### Machine State

| State | Type / Meaning |
|---|---|
| `x[32]`           | 32 unsigned 32-bit register patterns                  |
| `pc`              | Unsigned 32-bit guest byte address                    |
| `ram[65536]`      | Byte-addressed writable memory                        |
| `steps`           | Unsigned 64-bit completed-transition count            |
| `input[]`         | Immutable sequence of unsigned 32-bit values          |
| `input_position`  | Index of the next unread input value                  |
| `output[]`        | Sequence of unsigned 32-bit values                    |
| `status`          | `Running`, `Halted(exit_code)`, or `Failed(error)`    |

Validated text bytes and execution policy remain immutable during execution.

`x0` always reads as zero. Writes to it discard the result without suppressing memory checks, control flow, environment effects, or traps.

### Initial State

| Component | Initial Value |
|---|---|
| `pc`                  | Executable header's `entry`                           |
| `x2` / `sp`           | `0x00020000`                                          |
| All other registers   | `0`                                                   |
| RAM                   | Zero-filled, then initialized from the data payload   |
| `steps`               | `0`                                                   |
| `input_position`      | `0`                                                   |
| `output`              | Empty                                                 |
| `status`              | `Running`                                             |

The data payload is copied to RAM beginning at `0x00010000`.

Register initialization and the initial stack pointer are TinyProof environment rules, not general RISC-V reset requirements.

### Memory Map

All address intervals are half-open: the end address is excluded.

| Region | Address Range | Permissions |
|---|---|---|
| Text              | `[0x00001000, 0x00001000 + text_size)`    | Read, execute |
| RAM               | `[0x00010000, 0x00020000)`                | Read, write |
| Other addresses   | Everything outside the mapped regions     | None |

Unused text capacity is unmapped. The executable header is not mapped.

Guest addresses are integers checked by the VM, never host pointers. The VM does not allocate the entire 4 GiB guest address space.

#### Word Accesses

`lw` and `sw` access four consecutive bytes.

```text
address = u32(old_rs1 + signed_immediate)

load32(address) = memory[address]
                | memory[address + 1] << 8
                | memory[address + 2] << 16
                | memory[address + 3] << 24
```

For every load or store:

1. Calculate the effective address with 32-bit wrapping.
2. Require `address % 4 == 0`.
3. Check permissions for the complete four-byte access.
4. Perform the operation only if every check succeeds.

Bounds checks use wider arithmetic; `address + 4` must not wrap in the checker.

A word load may read text or RAM. A word store may write only RAM. Stores to text fail even if they would leave the bytes unchanged.

Stores are atomic: no partial write occurs on failure.

#### Stack

The stack grows downward by software convention.

The initial `sp` points one byte beyond RAM. Software must allocate stack space before storing through it.

```asm
addi sp, sp, -16
sw   a0, 0(sp)
```

There are no automatic push/pop operations, heap allocator, or stack/data collision checks. An in-range store that overwrites other data is a program bug, not a VM trap.

### Instruction Execution

Unless an instruction changes control flow, successful execution sets:

```text
pc = u32(old_pc + 16)
```

For each transition:

1. Return immediately if the machine is already terminal.
2. Check the remaining instruction budget.
3. Validate fetch alignment and the complete instruction's executable mapping.
4. Fetch and decode the instruction.
5. Read source values, calculate effects, and validate the operation.
6. Commit all effects atomically.
7. Increment `steps` by one.

A failed transition changes only status and error metadata. It does not modify registers, `pc`, RAM, input position, output, or `steps`.

Earlier completed transitions are not rolled back.

#### Control Transfers

Branch and `jal` offsets are byte displacements relative to the instruction's own address.

```text
Taken branch:
    target = u32(old_pc + offset)

jal:
    target = u32(old_pc + offset)
    link   = u32(old_pc + 4)

jalr:
    target = u32(old_rs1 + signed_immediate) & 0xFFFFFFFE
    link   = u32(old_pc + 4)
```

For a taken branch or jump, require `target % 4 == 0` before committing any effect. A failed jump does not write its link register.

An untaken branch does not validate its unused target.

Target mapping is checked on the next instruction fetch, not during the control-transfer instruction. An aligned jump outside text therefore completes before the destination fetch fails, unless the step limit takes precedence.

`jalr` clears bit `0` before checking alignment. Its immediate is in bytes and is not multiplied by two.

### Calling Convention

Ordinary function calls and environment calls use separate conventions. These rules describe software interfaces.

#### Ordinary Function Calls

This initial convention covers 32-bit integer and pointer arguments and a single 32-bit return value.

| Purpose | Register / Location |
|---|---|
| Arguments 1–8             | `a0`–`a7`, in order               |
| Arguments 9 onward        | Consecutive four-byte stack slots |
| Return value              | `a0`                              |
| Return address            | `ra` / `x1`                       |
| Stack pointer             | `sp` / `x2`                       |
| Caller-saved registers    | `ra`, `a0`–`a7`, `t0`–`t6`        |
| Callee-saved registers    | `s0`–`s11`, `sp`                  |
| Reserved registers        | `gp`, `tp`; leave unchanged       |

Call with `jal ra, target` or `jalr ra, 0(rs)`. Return with `ret`, which expands to `jalr x0, 0(ra)`.

The caller saves any caller-saved values needed after the call.

The callee restores any modified callee-saved registers before returning. 

A function making another call must preserve its own return address if needed. 

The stack grows downward and remains 16-byte aligned.

The caller allocates stack arguments; argument 9 is at `0(sp)` on callee entry, argument 10 at `4(sp)`, and so on. The caller reclaims this storage.

The VM does not enforce register preservation or 16-byte stack alignment. Violations are software bugs; actual memory accesses still undergo VM checks.

The executable entry point has no caller: terminate through the exit service.

#### Environment-Call Registers

`ecall` uses the following register convention:

| Purpose | Register |
|---|---|
| Service number before the call                | `t0` / `x5`               |
| Arguments 1–8, when defined by the service    | `a0`–`a7` / `x10`–`x17`   |
| Scalar result, when defined by the service    | `t0` / `x5`               |

TinyProof's general argument slots through `a7` are a local ABI rule; each service explicitly defines which slots it uses.

Unlike ordinary function calls, environment calls do not implicitly read additional arguments from the stack.

`ecall` does not modify `ra` or allocate a stack frame. Successful returning calls continue at the next instruction.

#### Supported Environment Calls

| `t0` Before Call | Service | Arguments | Successful Effect |
|---|---|---|---|
| `0` | Exit        | `a0` = exit code      | Set `Halted(a0)` |
| `1` | Read  u32   | None                  | Write next input value to `t0`; inc `input_position` |
| `2` | Write u32   | `a0` = output value   | Append `a0` to output |

Read overwrites `t0`, including the service number.

Exit and write preserve all registers, including `t0`.

Read preserves every register except `t0`.

Unused argument registers are ignored and need not contain zero. `a7` is an argument slot, not the service selector.

Reload the service number before each call.

```asm
# Read one value and write it back.
addi t0, zero, 1
ecall                   ; t0 = input value

mv   a0, t0
addi t0, zero, 2
ecall                   ; output.append(a0)

# Exit successfully.
addi a0, zero, 0
addi t0, zero, 0
ecall
```

Every successful call, including exit, advances `pc` by four and increments `steps` by one.

Read fails with `INPUT_EXHAUSTED` if no input remains.

Write fails with `OUTPUT_LIMIT` if output capacity is exhausted.

Other service numbers fail with `UNKNOWN_ECALL`.

A failed call commits no effects: registers, memory, input position, output, `pc`, and `steps` remain unchanged.

| Terminal Result | Meaning |
|---|---|
| `Halted(0)`       | Application success                       |
| `Halted(nonzero)` | Normal halt reporting application failure |
| `Failed(error)`   | VM execution failure                      |

Unused input is permitted. Falling off text is a fetch fault, not an exit.

### Execution Policy

Execution limits are supplied by the host and cannot be changed by the guest.

| Parameter | Accepted Range | Default |
|---|---|---|
| `max_steps`         | Integer in `[1, 10000000]`  | `1000000`              |
| `max_output_values` | Integer in `[0, 65536]`     | `4096`                 |
| Input length        | At most `65536` values      | Supplied by the caller |
| Each input value    | Integer in `[0, 2^32 - 1]`  | —                      |

`steps` counts completed TinyProof transitions, not a RISC-V hardware counter.

The step limit is checked before fetching the next instruction. A successful exit on the final permitted transition is accepted.

### Executable Format — `.tp32`

The text payload contains real RISC-V instruction bytes. The surrounding container is a TinyProof format, not ELF.

The header is exactly 24 bytes. All integer fields are little-endian.

| File Offset | Size | Field | Requirement |
|---|---:|---|---|
| `0`               | 4 bytes | `magic`                     | ASCII `TP32`: `54 50 33 32` |
| `4`               | 2 bytes | `version`                   | `1` |
| `6`               | 2 bytes | `flags`                     | `0` |
| `8`               | 4 bytes | `text_size`                 | Positive multiple of 4, at most `61440` |
| `12`              | 4 bytes | `data_size`                 | At most `65536` |
| `16`              | 4 bytes | `entry`                     | Four-byte-aligned address within text |
| `20`              | 4 bytes | `reserved`                  | `0` |
| `24`              | `text_size` bytes     | Text payload  | Mapped at `0x00001000` |
| `24 + text_size`  | `data_size` bytes     | Data payload  | Copied to `0x00010000` |

```text
file_size = 24 + text_size + data_size
```

Trailing bytes are rejected.

A raw-binary loading mode is not required.

#### Validation

Before execution, validate the complete file in the order specified in `Espec.md`. Scan every text word, including unreachable instructions.

Validation establishes container correctness and supported instruction encodings. It does not prove termination or memory safety.

Do not reject an otherwise supported branch/jump encoding because its target would be unmapped or misaligned at runtime.

Data bytes are never decoded as instructions.

After file validation, validate the execution policy and input.

### Assembler

Proposed command:

```sh
tinyasm program.s -o program.tp32
```

The assembler produces one complete executable. No linker is required.

#### Source Syntax

| Feature | Rule |
|---|---|
| Encoding                              | Valid UTF-8                                       |
| Syntax tokens and identifiers         | ASCII                                             |
| Comments                              | `;` through end of line; may contain valid UTF-8  |
| Line endings                          | LF or CRLF                                        |
| Token separation                      | Spaces and tabs                                   |
| Operand separation                    | Commas                                            |
| Instructions, directives, registers   | Lowercase and case-sensitive                      |
| Labels                                | Case-sensitive; unique across sections            |

#### Labels and Symbol Resolution

Label names are case-sensitive and match:

```text
[A-Za-z_][A-Za-z0-9_]*
```

A label definition consists of a name followed by `:`. Spaces or tabs may appear before the colon.

A line may contain zero or more label definitions followed by at most one instruction, pseudoinstruction, or directive. Label-only lines are allowed.

```asm
loop:
    addi t0, t0, -1
    bne  t0, zero, loop

first: second: addi a0, zero, 1
```

Each label binds to the current address in the section active at the point of definition. Labels emit no bytes.

- Text addresses start at `0x00001000`
- Data addresses start at `0x00010000`
- All labels share one namespace across sections
- Multiple different labels may identify the same address
- Defining the same name twice reports `DUPLICATE_LABEL`, even if both definitions would identify the same address
- Forward references are allowed
- Undefined references report `UNDEFINED_LABEL`
- Instruction and register names are not reserved as label names; their meaning is determined by syntactic position
- Numeric local labels, label arithmetic, relocation expressions, and external symbols are unsupported

Labels preceding a section-switch directive bind in the old section.

Labels preceding `.balign 4` bind before its padding is emitted.

A branch, `jal`, or `j` target operand may be either an integer byte displacement or a label. A label operand is resolved as:

```text
offset = label_address - instruction_address
```

A branch/jump label must belong to text and identify the beginning of an emitted machine instruction. Otherwise, report `BAD_LABEL_TARGET`. A label at the end of text does not identify an instruction.

Numeric branch and `jal` displacements must fit the corresponding signed range and be divisible by two. Odd displacements report `MISALIGNED_OFFSET`. Divisibility by four and destination mapping are runtime checks, not assembler checks for numeric displacements.

`.entry` requires one label operand. An undefined entry label reports `UNDEFINED_LABEL`; a defined label that does not identify a text instruction reports `BAD_ENTRY_DIRECTIVE`. Repeating `.entry` also reports `BAD_ENTRY_DIRECTIVE`.

Label operands are supported only by branches, `jal`, `j`, and `.entry`. Other operands, including `li`, `.word`, and `jalr` offsets, do not accept labels.

#### Register Names

Accept `x0` through `x31` in canonical decimal spelling and the aliases below.

| Registers | Aliases in Matching Order |
|---|---|
| `x0`–`x4`     | `zero`, `ra`, `sp`, `gp`, `tp` |
| `x5`–`x7`     | `t0`, `t1`, `t2` |
| `x8`–`x9`     | `s0` / `fp`, `s1` |
| `x10`–`x17`   | `a0`, `a1`, `a2`, `a3`, `a4`, `a5`, `a6`, `a7` |
| `x18`–`x27`   | `s2`, `s3`, `s4`, `s5`, `s6`, `s7`, `s8`, `s9`, `s10`, `s11` |
| `x28`–`x31`   | `t3`, `t4`, `t5`, `t6` |


#### Integer Literals

Accept an optional `-`, followed by:

| Form | Example |
|---|---|
| Decimal digits                | `42`, `-17` |
| `0x` and hexadecimal digits   | `0x1234`, `-0x10` |
| `0b` and binary digits        | `0b1010`, `-0b10` |

Hexadecimal digits may use either case.

#### Sections and Directives

Assembly begins in `.text`.

Section switches may occur repeatedly. Each section has its own location counter, and later content appends to that section. The executable stores all text before all data.

| Directive | Meaning |
|---|---|
| `.text`                       | Switch to text                                        |
| `.data`                       | Switch to initialized RAM data                        |
| `.entry label`                | Select the entry instruction; optional, at most once  |
| `.word integer[, integer...]` | Append little-endian 32-bit values to data            |
| `.zero count`                 | Append `count` zero bytes to data                     |
| `.balign 4`                   | Append 0–3 zero bytes to align data to four bytes     |

Rules:

- Instructions and pseudoinstructions are allowed only in text
- `.word`, `.zero`, and `.balign` are allowed only in data
- `.word` requires at least one integer, each in `[-2^31, 2^32 - 1]`
- `.word` stores each value modulo `2^32`
- `.word` requires the current data address to be divisible by four
- `.zero` accepts a count in `[0, 65536]` and may leave data unaligned
- `.balign` supports only the operand `4`
- `.entry` emits no bytes and does not switch sections
- Without `.entry`, execution begins at `0x00001000`
- Empty text is invalid

#### Load-Immediate Pseudoinstruction

Syntax:

```asm
li rd, integer
```

`li` accepts one integer literal in `[-2^31, 2^32 - 1]`. Values outside this range report `IMMEDIATE_OUT_OF_RANGE`. Labels and expressions are unsupported.

Let:

```text
v = u32(integer)
s = s32(v)
```

Expansion is deterministic.

If `-2048 <= s <= 2047`, emit exactly one instruction:

```asm
addi rd, x0, s
```

Otherwise calculate, using mathematical integers:

```text
lo = sext(v & 0xFFF, 12)
hi = ((s - lo) / 4096) modulo 2^20
```

The division is exact. Emit exactly two instructions:

```asm
lui  rd, hi
addi rd, rd, lo
```

Here `hi` is the unsigned 20-bit field encoded by `lui`. The second instruction is emitted even when `lo == 0`.

Equivalent signed and unsigned spellings of the same 32-bit value produce identical encodings. For example, `li a0, -1` and `li a0, 0xFFFFFFFF` both expand to `addi a0, x0, -1`.

Examples:

| Source | Expansion |
|---|---|
| `li a0, 2047`         | `addi a0, x0, 2047`                               |
| `li a0, 2048`         | `lui a0, 1` followed by `addi a0, a0, -2048`      |
| `li a0, 4096`         | `lui a0, 1` followed by `addi a0, a0, 0`          |
| `li a0, 0x7FFFFFFF`   | `lui a0, 0x80000` followed by `addi a0, a0, -1`   |
| `li a0, 0x80000000`   | `lui a0, 0x80000` followed by `addi a0, a0, 0`    |

`rd = x0` is permitted and uses the same expansion rules; writes are discarded normally. No temporary register is used.

Each emitted instruction is a separate VM transition and counts toward `max_steps`. A two-instruction expansion is not atomic as a whole. A label attached to `li` identifies its first emitted instruction.
