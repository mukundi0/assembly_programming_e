# 01.asm - Push and pop

[Source code](01.asm)

In this program, I followed how 100 and 200 are put on the stack and then
taken off into different registers. It is a 64-bit program, so each register
push or pop changes RSP by 8 bytes.

## How the program runs

I use S to mean the value of RSP when execution begins at `_start`. Linux
has already put the program's startup information on the stack. The values
pushed by this program go below that starting address because the stack grows
towards lower addresses.

First, `mov rax, 100` and `mov rbx, 200` load the registers. These instructions
do not change RSP. Then `push rax` subtracts 8 from RSP and stores 100 at
`[rsp]`. `push rbx` subtracts another 8 and stores 200 at the new top.

At the first `nop`, RSP is S - 16. The top of the stack contains 200, and the
next 8-byte entry contains 100. `nop` does not change anything; it gives a
useful place to inspect the stack.

Next, `pop rcx` reads 200 from the top and adds 8 to RSP. Then `pop rdx` reads
100 and adds another 8. RSP is now back to S. The second `nop` also leaves
everything unchanged.

| Point in the program | RSP |
| --- | --- |
| At `_start` | S |
| After `push rax` | S - 8 |
| After `push rbx` | S - 16 |
| After `pop rcx` | S - 8 |
| After `pop rdx` | S |

Finally, `mov rax, 60` selects the Linux exit system call, and
`xor rdi, rdi` sets the exit status to 0. `syscall` ends the program.
The stack is already back to its starting position before this happens.

## What I observed

The program did not print anything and exited with status 0. In GDB, the two
stack entries at the first `nop` were `0xc8` (200) and `0x64` (100).
At the second `nop`, RCX was 200, RDX was 100, and RSP was back to S.
The most stack space used below S was 16 bytes.

This shows last in, first out: 200 was pushed last, so it came out first.
Pushing copies a value without clearing its register. Popping moves the
stack pointer back up without erasing the old bytes in memory.

## How I ran the program

From the `07_stack` folder, I assembled, linked, and ran the program with:

~~~sh
nasm -f elf64 01.asm -o 01.o
ld 01.o -o 01
./01
echo $?  # 0
~~~

To inspect the stack, I opened the executable in GDB:

~~~sh
gdb -q 01
~~~

I used `break _start` and `run` in GDB, then `si` to follow the instructions.
At the first `nop`, `x/2gx $rsp` shows the two saved values.
At the second `nop`, `info registers rsp rcx rdx` shows the results.
RCX should be checked before `syscall` because that instruction changes it.
