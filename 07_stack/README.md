# Stack programs

Each source file has its own README covering execution from Linux process entry
to process exit, with stack layouts, pointer changes, build commands, and
observations from a fresh run and instruction-by-instruction GDB inspection.

| Source | Program README | Mode | Maximum additional stack use |
| --- | --- | --- | --- |
| [01.asm](01.asm) | [Push and pop into different registers](docs/01/README.md) | 64-bit | 16 bytes |
| [02.asm](02.asm) | [A function with two local variables](docs/02/README.md) | 64-bit | 32 bytes |
| [03.asm](03.asm) | [The implicit stack operations of call and ret](docs/03/README.md) | 64-bit | 8 bytes |
| [04.asm](04.asm) | [A function with two local variables](docs/04/README.md) | 32-bit | 16 bytes |
| [05.asm](05.asm) | [Saving and restoring 64-bit registers](docs/05/README.md) | 64-bit | 16 bytes |
| [06.asm](06.asm) | [Saving and restoring 32-bit registers](docs/06/README.md) | 32-bit | 8 bytes |

## Reading the traces

In every README, S means the stack pointer at the first instruction of
`_start`. Linux has already placed the argument count, argument pointers,
environment pointers, and auxiliary information on the process stack. It is not
empty. `_start` is the executable entry point, rather than a function entered
through `call`, so it has no caller return address to pop.

These examples use a downward-growing stack. A register `push` first decreases
the pointer and then stores a value; `pop` reads the current top and then
increases the pointer. These programs use 8-byte stack entries in 64-bit mode
and 4-byte entries in 32-bit mode. `call` also pushes a return address, and
`ret` consumes it. Moving the pointer back does not erase the old memory.

Maximum stack use in the table is measured below S and includes return
addresses, saved frame pointers, and allocated local variables where present.
It excludes the initial process data and any separate kernel stack activity.

## Verification

All six source files were freshly assembled with NASM, linked with GNU ld,
executed, and single-stepped with GDB on Linux. Every program produced empty
stdout and stderr, exited with status 0, and restored `RSP` or `ESP` to S before
its exit instruction. GDB confirmed maximum stack depths of 16, 32, 8, 16, 16,
and 8 bytes respectively. The individual READMEs record the relevant register
and memory observations. Absolute stack addresses vary between runs; offsets
from S describe the same behavior.

The pre-existing executables named `04` and `06` were ELF64 files despite their
sources containing `BITS 32`. The checks above used fresh ELF32 builds of those
two programs. `BITS 32` selects instruction encoding in NASM; it does not by
itself make the linked executable a 32-bit process. Use `nasm -f elf32` with
`ld -m elf_i386` for them. In particular, identical push/pop opcode bytes in an
ELF64 executable can operate on 64-bit registers, so a successful exit alone
does not establish that a 32-bit stack example was tested correctly.

Each README's build commands use a temporary directory. In GDB, `si` executes
one machine instruction, including entering functions. When stopped at an
instruction, that instruction has not executed yet. Inspect the stack at each
`nop`, and check restored registers before the exit setup overwrites them.
