# 02.asm - Using the stack for local variables

[Source code](02.asm)

This program helped me follow how a function uses the stack for a return
address, a saved register, and local variables. It runs in 64-bit mode and
uses RSP as the stack pointer and RBP as a reference for the local variables.

## How the program runs

I use S for RSP at the start of `_start`. Linux has already prepared the
startup stack. `_start` was not entered through a `call`, so it does not have
a caller's return address.

The first instruction is `call work`. It pushes the address of the next
instruction, `mov rax, 60`, onto the stack. This reduces RSP to S - 8 and
then execution moves to `work`.

Inside `work`, `push rbp` saves the old RBP and reduces RSP to S - 16.
Then `mov rbp, rsp` sets RBP to that address. `sub rsp, 16` moves RSP down
to S - 32, making room for two 8-byte local variables. This reserves space;
it does not put any values there yet.

The program stores 100 at `[rbp - 8]` and 200 at `[rbp - 16]`. It then reads
them into RAX and RBX. These stores and loads do not change RSP. RBP stays
fixed, so the same offsets can be used to find the variables.

At the `nop`, the stack looks like this:

~~~text
Address    Value
S - 32     200                 <- RSP
S - 24     100
S - 16     Saved old RBP       <- RBP
S - 8      Return address
S          Startup information
~~~

To finish `work`, `mov rsp, rbp` moves RSP back to S - 16, releasing the
space for both local variables. `pop rbp` restores the old RBP and moves
RSP to S - 8. `ret` takes the return address off the stack and continues
after the original call. RSP is now back to S.

Back in `_start`, `mov rax, 60` and `xor rdi, rdi` prepare exit with status 0.
`syscall` ends the program. The 100 in RAX is replaced by 60 for this call.

## What I observed

There was no output, and the program exited with status 0. At the `nop`,
GDB showed RAX = 100, RBX = 200, and RSP = S - 32. Reading four 8-byte
entries from RSP showed 200, 100, saved RBP, and the return address.
In this run, saved RBP was 0 and the return address was `0x401005`.
After `ret`, RSP was back to S and the old RBP had been restored.

The program used 32 bytes below S: 8 for the return address, 8 for saved
RBP, and 16 for the local variables. I can see why the locals must be
released before `pop rbp`. Otherwise, the pop would read a local variable
instead of the saved RBP. Releasing that space does not erase its contents.

## How I ran the program

From the `07_stack` folder, I assembled, linked, and ran the program with:

~~~sh
nasm -f elf64 02.asm -o 02.o
ld 02.o -o 02
./02
echo $?  # 0
~~~

To inspect the stack, I opened the executable in GDB:

~~~sh
gdb -q 02
~~~

In GDB, I used `break _start`, `run`, and `si` to enter `work`.
At the `nop`, `x/4gx $rsp` shows the whole frame and
`info registers rsp rbp rax rbx` shows the pointers and loaded values.
Stepping through the last three instructions in `work` shows the stack
being restored.
