# My notes on the stack programs

These are my explanations of the six programs in this folder. For each one,
I followed the stack from `_start` until the program exited and recorded
the register values and stack changes from GDB.

| Program | My explanation | Mode | Most stack space used below the starting pointer |
| --- | --- | --- | --- |
| [01.asm](01.asm) | [01_README.md](01_README.md) | 64-bit | 16 bytes |
| [02.asm](02.asm) | [02_README.md](02_README.md) | 64-bit | 32 bytes |
| [03.asm](03.asm) | [03_README.md](03_README.md) | 64-bit | 8 bytes |
| [04.asm](04.asm) | [04_README.md](04_README.md) | 32-bit | 16 bytes |
| [05.asm](05.asm) | [05_README.md](05_README.md) | 64-bit | 16 bytes |
| [06.asm](06.asm) | [06_README.md](06_README.md) | 32-bit | 8 bytes |

## What I noticed across the programs

I used S to stand for the stack pointer at the first instruction of `_start`.
The actual address can change between runs, so S makes it easier to follow
how far RSP or ESP moves. Linux has already put startup information on the
stack before the program begins.

The stack grows towards lower addresses. A push moves the pointer down and
stores a value. A pop reads the top value and moves the pointer back up.
These examples use 8-byte entries in 64-bit mode and 4-byte entries in
32-bit mode. The last value pushed is the first one popped.

`call` also uses the stack by saving a return address, and `ret` removes
that address to continue in the caller. In `02.asm` and `04.asm`, the stack
also holds a saved frame pointer and two local variables.
The space totals above include all of those entries, but not the startup
information already on the stack.

## How I ran the files

From the repository root, I first opened the stack folder:

~~~sh
cd 07_stack
~~~

I assembled each `.asm` file with NASM, linked its `.o` file with `ld`,
and ran the executable with `./01`, `./02`, and so on. Each program's
README shows the exact commands I used to assemble, link, run, and
open it in GDB.

All six programs printed nothing, produced no error output, and exited
with status 0. In each GDB trace, the stack pointer was back at S before
the program exited. The individual READMEs contain the values I checked.

For `04.asm` and `06.asm`, I used fresh ELF32 builds because the existing
executables were ELF64. Writing `BITS 32` in the source does not by itself
make the executable run in 32-bit mode. I used `nasm -f elf32` and
`ld -m elf_i386` for those two files.
