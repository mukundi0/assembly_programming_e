# 03.asm - How call and ret use the stack

[Source code](03.asm)

This example shows that a program can use the stack even when there is no
`push` or `pop` written in the source. Here, the stack is used by `call`
and `ret` to remember where execution should continue.

## How the program runs

I use S for RSP when `_start` begins. Linux has already prepared the startup
stack. Since this is a 64-bit program, a return address takes 8 bytes.

First, `mov rax, 10` puts 10 into RAX. RSP does not change.
Then `call calculate` subtracts 8 from RSP and saves the address of
`after_call` at the new top. Execution jumps to `calculate` with RSP = S - 8.

Inside `calculate`, `add rax, 5` changes RAX from 10 to 15. It does not
change RSP or the saved return address. There are no local variables or
saved frame pointer in this function.

~~~text
Address    Value while calculate is running
S - 8      Address of after_call  <- RSP
S          Startup information
~~~

Next, `ret` reads the saved address into RIP, which holds the next instruction
address, and adds 8 to RSP. Execution continues at `after_call` with RSP back
at S and RAX still holding 15. The `nop` there changes nothing.

The next instruction, `mov rax, 60`, replaces 15 with the Linux exit system-call
number. `xor rdi, rdi` sets the exit status to 0, and `syscall` ends the program.
`_start` does not use `ret` to exit because it has no caller return address.

| Point in the program | RSP |
| --- | --- |
| Before `call calculate` | S |
| Inside `calculate` | S - 8 |
| After `ret` | S |
| Before exit | S |

## What I observed

The program printed nothing and exited with status 0. In GDB, RSP went down
by 8 after the call and came back up by 8 after `ret`. At `after_call`,
RAX was 15 (`0xf`). The return target in this build was `0x40100a`.

The most stack space used below S was 8 bytes, all for the return address.
The input 10 and result 15 stayed in RAX; they were not pushed or popped.
I checked the result before `mov rax, 60` because that instruction overwrites
it. The calculation result is 15, but the program's exit status is 0.

## How I ran the program

From the `07_stack` folder, I assembled, linked, and ran the program with:

~~~sh
nasm -f elf64 03.asm -o 03.o
ld 03.o -o 03
./03
echo $?  # 0
~~~

To inspect the stack, I opened the executable in GDB:

~~~sh
gdb -q 03
~~~

In GDB, I used `break _start`, `run`, and `si` to enter `calculate`.
`x/gx $rsp` shows the saved return address, and `p/x &after_call` shows
where it should return. At `after_call`, `info registers rsp rax` shows
the restored stack pointer and the result.
