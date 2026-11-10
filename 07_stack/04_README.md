# 04.asm - A 32-bit stack frame

[Source code](04.asm)

This program does the same kind of work as `02.asm`, but in 32-bit mode.
I followed ESP for the top of the stack and EBP for the local-variable
offsets. A register push or pop changes ESP by 4 bytes here.

## How the program runs

I use S for ESP when execution begins at `_start`. The stack already contains
startup information from Linux. There is no caller return address for
`_start` itself.

`call work` saves the address of the next instruction, `mov eax, 1`, on the
stack and jumps to `work`. The saved address takes 4 bytes, so ESP becomes
S - 4.

Inside `work`, `push ebp` saves the old EBP and makes ESP = S - 8.
`mov ebp, esp` sets EBP to that position. Then `sub esp, 8` moves ESP to
S - 16, reserving space for two 4-byte local variables.

The program writes 100 to `[ebp - 4]` and 200 to `[ebp - 8]`. Those stores
give the reserved slots their values. The following instructions load the
values into EAX and EBX. None of these stores or loads moves ESP.
The `nop` also leaves the stack unchanged.

At this point, the stack is:

~~~text
Address    Value
S - 16     200                 <- ESP
S - 12     100
S - 8      Saved old EBP       <- EBP
S - 4      Return address
S          Startup information
~~~

At the end of `work`, `mov esp, ebp` moves ESP back to S - 8 and releases
the space for the local variables. `pop ebp` restores the old EBP and
raises ESP to S - 4. `ret` removes the return address and continues back
in `_start`. ESP is now S again.

Finally, `mov eax, 1` selects the 32-bit Linux exit call, `xor ebx, ebx`
sets the exit status to 0, and `int 0x80` ends the program.
EAX and EBX are changed for exit after their local-variable values have
been checked. The exit call does not pop another user-space return address.

## What I observed

There was no output, and the program exited with status 0. At the `nop`,
GDB showed EAX = 100, EBX = 200, and ESP = S - 16. The four entries from
ESP were 200, 100, saved EBP (0 in this run), and return address `0x08049005`.
After `ret`, ESP was back to S and EBP was restored.

The stack used 16 bytes below S: 4 for the return address, 4 for saved EBP,
and 8 for the locals. This is half the space used by `02.asm`.
Moving ESP back releases the local slots without clearing their old bytes.

I used a fresh ELF32 build for this check because the existing executable
was ELF64. `BITS 32` alone is not enough to make the executable run in
32-bit mode, so the commands below use `elf32` and `elf_i386`.

## How I ran the program

From the `07_stack` folder, I assembled, linked, and ran the program with:

~~~sh
nasm -f elf32 04.asm -o 04.o
ld -m elf_i386 04.o -o 04
./04
echo $?  # 0
~~~

To inspect the stack, I opened the executable in GDB:

~~~sh
gdb -q 04
~~~

In GDB, I used `break _start`, `run`, and `si` to follow `work`.
At the `nop`, `x/4wx $esp` shows four 4-byte entries and
`info registers esp ebp eax ebx` shows the pointers and local values.
Stepping through `mov esp, ebp`, `pop ebp`, and `ret` shows the cleanup.
The host needs support for running 32-bit x86 programs.
