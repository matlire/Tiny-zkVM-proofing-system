## Errors Specification — TinyProof RV32

This document defines validation errors, runtime traps, assembler diagnostics, and failure precedence for TinyProof RV32.

### Failure Categories

| Category | Meaning |
|---|---|
| Executable validation error   | The `.tp32` file violates the executable contract |
| Run-request validation error  | Execution policy or input is invalid |
| Runtime trap                  | A validated program fails during execution |
| Assembler diagnostic          | Source cannot be assembled under the supported contract |
| Host failure                  | Host I/O, allocation, cancellation, or internal implementation failure |

A guest exit with a nonzero code is an application failure, not a runtime trap.

Error codes are stable symbolic identifiers.

No numeric error-code mapping is defined.

### Error-Handling

- Validation and execution stop at the first failing check
- Phase order: executable validation → run-request validation -> execution
- Validation checks follow the tables' `Order` column
- Text validation scans instructions in increasing address order
- Policy checks: `max_steps` before `max_output_values`
- Input checks: length first, then values in index order
- Assembler: validate UTF-8 first; further diagnostic order is implementation-defined
- The assembler may report multiple errors, but must not produce a successful executable if any error exists
- Host failures are reported separately from guest errors

### Executable Validation Errors

| Order | Error Code | Condition |
|---:|---|---|
| 1 | `FILE_TOO_SHORT`          | File contains fewer than 24 bytes |
| 2 | `BAD_MAGIC`               | Magic is not ASCII `TP32` |
| 3 | `UNSUPPORTED_VERSION`     | Version is not supported by the implementation |
| 4 | `NONZERO_HEADER_FIELDS`   | `flags` or `reserved` is nonzero |
| 5 | `BAD_TEXT_SIZE`           | Text size is zero, not divisible by four, or greater than `61440` |
| 6 | `BAD_DATA_SIZE`           | Data size is greater than `65536` |
| 7 | `FILE_SIZE_MISMATCH`      | Actual file length differs from `24 + text_size + data_size` |
| 8 | `BAD_ENTRY`               | Entry is not four-byte aligned or is outside text |
| 9 | `UNSUPPORTED_ENCODING`    | A text word does not exactly match a supported instruction encoding |

### Run-Request Validation Errors

Validate the run request only after the executable passes validation.

| Order | Error Code | Condition |
|---:|---|---|
| 1 | `BAD_EXECUTION_POLICY` | `max_steps` or `max_output_values` is not an integer within its permitted range |
| 2 | `BAD_INPUT`           | Input contains more than `65536` values, or any value is not an integer in `[0, 2^32 - 1]` |

| Parameter | Permitted Range | Default |
|---|---|---|
| `max_steps`           | `[1, 10000000]`   | `1000000` |
| `max_output_values`   | `[0, 65536]`      | `4096`    |


### Runtime Traps

Runtime traps are terminal and cannot be caught by guest code.

| Error Code | Trigger | Associated Context |
|---|---|---|
| `STEP_LIMIT`                      | `steps == max_steps` before the next fetch | Current `pc` |
| `INSTRUCTION_ADDRESS_MISALIGNED`  | Fetch `pc`, taken branch target, or jump target is not divisible by four | Misaligned address |
| `INSTRUCTION_ACCESS_FAULT`        | The complete four-byte fetch is not within executable text | Fetch `pc` |
| `ILLEGAL_INSTRUCTION`             | Defensive runtime decoding rejects the fetched word | Current `pc`, instruction word |
| `LOAD_ADDRESS_MISALIGNED`         | `lw` effective address is not divisible by four | Effective address |
| `LOAD_ACCESS_FAULT`               | Aligned `lw` cannot read all four bytes from text or RAM | Effective address |
| `STORE_ADDRESS_MISALIGNED`        | `sw` effective address is not divisible by four | Effective address |
| `STORE_ACCESS_FAULT`              | Aligned `sw` cannot write all four bytes to RAM | Effective address |
| `UNKNOWN_ECALL`                   | Service number in `t0` is not `0`, `1`, or `2` | Service number |
| `INPUT_EXHAUSTED`                 | Read service has no remaining input value | Service number |
| `OUTPUT_LIMIT`                    | Write service is called when output length equals `max_output_values` | Service number |

#### Runtime Errors Handling

- Already halted or failed: return without changing state or the existing result
- Check order:
  1. Step limit
  2. Fetch alignment
  3. Fetch mapping
  4. Instruction decoding
  5. Instruction-specific checks
  6. Commit effects and increment `steps`
- Loads/stores: check alignment before bounds and permissions
- `ecall`: identify the service before checking its requirements
- A failed transition changes only status and error metadata

#### Branch and Jump Errors Handling

- Check target alignment only for taken branches and jumps
- For `jalr`, clear target bit 0 before checking alignment
- A misaligned target traps on the branch/jump itself
  - Error: `INSTRUCTION_ADDRESS_MISALIGNED`
  - `pc`: address of the branch/jump
  - `address`: computed target
  - No link-register write or step increment
- Check target mapping on the next fetch, not during the transfer
- An aligned jump outside text completes; the next transition reports `STEP_LIMIT` if the budget is exhausted, otherwise `INSTRUCTION_ACCESS_FAULT`

#### Environment Calls

| Service | Arguments | Validation | Successful Effect |
|---|---|---|---|
| Exit (`t0 = 0`)  | `a0` = exit code | Every 32-bit exit code is valid | Set `Halted(a0)` |
| Read (`t0 = 1`)  | None | Input must remain | Set `t0` to the next input value; advance input position |
| Write (`t0 = 2`) | `a0` = output value | Output capacity must remain | Append `a0` to output |
| Other number | Not interpreted | Always fails | Report `UNKNOWN_ECALL` |

A failing call does not:

- Change `t0` or any other register.
- Consume input.
- Append output.
- Modify RAM.
- Advance `pc`.
- Increment `steps`.

### Assembler Diagnostics

| Error Code | Meaning |
|---|---|
| `INVALID_SOURCE_ENCODING` | Source is not valid UTF-8 |
| `SYNTAX_ERROR`            | Source does not match the supported grammar |
| `UNKNOWN_MNEMONIC`        | Instruction or pseudoinstruction name is unsupported |
| `UNKNOWN_DIRECTIVE`       | Directive name is unsupported |
| `BAD_REGISTER`            | Register token is not a supported name or alias |
| `BAD_OPERAND_COUNT`       | An instruction or directive has the wrong number of operands |
| `IMMEDIATE_OUT_OF_RANGE`  | An integer operand is outside its permitted range |
| `MISALIGNED_OFFSET`       | A numeric branch or `jal` displacement is odd |
| `DUPLICATE_LABEL`         | A label is defined more than once |
| `UNDEFINED_LABEL`         | A referenced label has no definition |
| `WRONG_SECTION`           | An instruction or directive is used in a prohibited section |
| `BAD_LABEL_TARGET`        | A resolved branch/jump label does not identify an actual text instruction |
| `DATA_ALIGNMENT`          | `.word` is emitted at a data address not divisible by four |
| `SECTION_TOO_LARGE`       | Text or data exceeds its permitted size |
| `BAD_ENTRY_DIRECTIVE`     | `.entry` is repeated or does not select a valid text instruction |

### Error Records

| Field | Include When |
|---|---|
| `phase`               | Always |
| `code`                | Always |
| `steps`               | Execution has started |
| `pc`                  | A faulting execution address is applicable |
| `instruction_word`    | An instruction word has been fetched or identified during validation |
| `address`             | A relevant effective, target, or faulting address exists |
| `service_number`      | The failure concerns an environment call; use the pre-instruction `t0` |
| `file_offset`         | Applicable to loader instruction errors |
| `guest_address`       | Applicable to loader instruction errors |
| `source_line`         | Applicable to assembler errors |
| `source_column`       | Applicable to assembler errors |

#### Error-Record

- Include applicable, known fields; omit unavailable fields
- Do not use zero as an "unavailable" sentinel
- Do not run additional checks solely to populate error fields
- Runtime `pc` and `steps` describe the state before the failed transition
- Include `instruction_word` only if fetched in this transition never reuse the previous instruction
- `STEP_LIMIT`: include `pc` and `steps`; omit `instruction_word` and `address`
- Fetch faults: include `pc`, `steps`, and `address = pc`; omit `instruction_word`
- Environment-call errors: use the pre-instruction `t0` as `service_number`
- Loader instruction errors: include the word, its file offset, and guest address
- Validation errors: omit runtime `pc` and `steps`
- Assembler locations: one-based lines and byte columns; a tab counts as one byte
