# ASMC Support

[![License](https://img.shields.io/github/license/Orbinuity/asmc-support)](https://orbinuity.nl/license)
[![Last Commit](https://img.shields.io/github/last-commit/Orbinuity/asmc-support)](https://github.com/Orbinuity/asmc-support/commits)
[![Version](https://img.shields.io/badge/Version-1.0.0-orange)](https://github.com/Orbinuity/asmc-support/releases/v1.0.0)
[![Made By](https://img.shields.io/badge/Made%20by-Orbinuity-teal)](https://orbinuity.nl/)
[![Read the docs](https://img.shields.io/badge/Read%20the%20docs-red)](https://help.orbinuity.nl/doc?project=asmc-support)

Support the custom ASM Craft language.

## License

Before copying any part of this project, please read the [LICENSE](https://orbinuity.nl/license) file to understand the terms and conditions.

## 1. Program Structure

An ASMC source file consists of two primary sections, optional inline comments, and code labels.

```asmc
; Lines starting with or containing a semicolon are comments
section .data
    ; Variable definitions go here

section .text
    ; Executable instructions and labels go here

```

* **Comments**: Any text following a `;` is ignored by the compiler.
* **Labels**: Defined with a trailing colon (e.g., `main:` or `loop_start:`). Labels act as jump targets or call locations.

---

## 2. Registers & Calling Conventions

ASMC provides 16 general-purpose 64-bit virtual registers.

| Register | Internal ID | Syscall Alias | Standard Usage / Function |
| --- | --- | --- | --- |
| `rax` | 0 | `ret` | Return value / Syscall number |
| `rbx` | 1 | — | General purpose |
| `rcx` | 2 | — | General purpose |
| `rdx` | 3 | `arg3` | General purpose / 3rd syscall argument |
| `rsi` | 4 | `arg2` | General purpose / 2nd syscall argument |
| `rdi` | 5 | `arg1` | General purpose / 1st syscall argument |
| `rbp` | 6 | — | Base pointer |
| `rsp` | 7 | — | Stack pointer (Initialized to `0x8000`) |
| `r8` | 8 | `arg5` | General purpose / 5th syscall argument |
| `r9` | 9 | `arg6` | General purpose / 6th syscall argument |
| `r10` | 10 | `arg4` | General purpose / 4th syscall argument |
| `r11` | 11 | — | General purpose |
| `r12` | 12 | — | General purpose |
| `r13` | 13 | — | General purpose |
| `r14` | 14 | — | General purpose |
| `r15` | 15 | — | General purpose |

---

## 3. Data Section (`section .data`)

Variables must be defined in `section .data`. Each entry allocates a fixed-size buffer in memory and populates it with string literals or raw byte arrays.

### Declaration Syntax

```asmc
<variable_name> <directive> <value>

```

### Allocation Directives

| Directive | Size Allocated | Description |
| --- | --- | --- |
| `ds` | 16 bytes | Data Small |
| `dm` | 32 bytes | Data Medium |
| `db` | 64 bytes | Data Big |

* If the defined string or numeric sequence is smaller than the allocation size, it is padded with `0x00` (null bytes).
* If it exceeds the maximum allocation size, it is truncated.
* Escape sequences (e.g., `\n`, `\t`) within string literals are interpreted.

#### Example

```asmc
section .data
    msg      dm "Hello, World!\n"    ; Allocates 32 bytes
    user_buf db 0                    ; Allocates 64 bytes

```

---

## 4. Instruction Set Architecture (ISA)

ASMC instructions support three types of operands:

1. **Registers**: e.g., `rax`, `rdi`
2. **Variables**: Refers to the memory address of a defined variable.
3. **Immediates**: Integer values or labels, e.g., `42`, `0x10`, `my_label`

---

### Data Transfer Instructions

* `mov dest, src`
Moves the value of `src` (register, variable address, or immediate) into the destination register `dest`.
* `push src`
Decrements `rsp` by 4 bytes and stores the 32-bit value of `src` on the stack.
* `pop dest`
Pops a 32-bit value off the stack into `dest` and increments `rsp` by 4 bytes.

---

### Arithmetic & Logical Instructions

All arithmetic operations write their result directly to the destination register (`dest = dest <op> src`).

* `add dest, src` — Adds `src` to `dest`.
* `sub dest, src` — Subtracts `src` from `dest`.
* `mul dest, src` — Multiplies `dest` by `src`.
* `div dest, src` — Performs integer division (`dest // src`). Halts execution on division by zero.
* `xor dest, src` — Bitwise XOR operation.

---

### Comparison & Control Flow

Branching relies on internal CPU flags set by the `cmp` instruction:

* **Zero Flag (`zf`)**: Set to `True` if `v1 == v2`.
* **Sign Flag (`sf`)**: Set to `True` if `v1 < v2`.

#### Comparison

* `cmp arg1, arg2` — Compares `arg1` and `arg2` and sets internal status flags.

#### Unconditional Jump & Subroutines

* `jmp target` — Jumps unconditionally to `target` (label or address).
* `call target` — Pushes current Instruction Pointer (IP) onto the stack and jumps to `target`.
* `ret` — Pops saved IP from stack and returns execution control.

#### Conditional Jumps

* `je target` — Jump if equal (`zf == True`)
* `jne target` — Jump if not equal (`zf == False`)
* `jg target` — Jump if greater (`zf == False` and `sf == False`)
* `jl target` — Jump if less (`sf == True` and `zf == False`)
* `jge target` — Jump if greater or equal (`sf == False`)
* `jle target` — Jump if less or equal (`sf == True` or `zf == True`)

---

### System Calls

* `syscall`
Triggers a system call based on the integer value stored in `rax`.

---

## 5. System Calls Reference (Firmware v1)

| `rax` Code | Name | Argument 1 (`rdi`) | Argument 2 (`rsi`) | Argument 3 (`rdx`) | Argument 4 (`r10`) | Argument 5 (`r8`) | Argument 6 (`r9`) | Return Value (`rax`) | Description |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **0** | `sys_exit` | Exit Code | — | — | — | — | — | — | Terminates program execution with specified code. |
| **1** | `sys_write` | Buffer / Address | — | — | — | — | — | — | Prints string or memory content located at address to stdout. |
| **2** | `sys_read` | — | — | — | — | — | — | Read String/Bytes | Reads a string from stdin and stores the output in `rax`. |

---

## 6. Code Examples

### Example 1: Hello World

```asmc
section .data
    msg dm "Hello, World!\n"

section .text
main:
    ; Set up sys_write (1)
    mov rax, 1
    mov rdi, msg
    syscall

    ; Set up sys_exit (0)
    mov rax, 0
    mov rdi, 0
    syscall

```

### Example 2: Interactive Echo & Loop

```asmc
section .data
    prompt  dm "Enter text: "
    newline dm "\n"

section .text
main:
    ; Prompt user
    mov rax, 1
    mov rdi, prompt
    syscall

    ; Read user input into rax
    mov rax, 2
    syscall

    ; Store user input in rbx and print back
    mov rbx, rax
    mov rax, 1
    mov rdi, rbx
    syscall

    ; Exit program
    mov rax, 0
    mov rdi, 0
    syscall

```