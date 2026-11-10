# 06.asm - Saving and restoring 32-bit values

[Source code](06.asm)

This program is similar to `05.asm`, but it uses EAX, EBX, and ESP in
32-bit mode. I followed the same save-and-restore order, with each push
or pop changing ESP by 4 bytes.

## How the program runs

I use S for ESP at the start of `_start`. The initial stack already contains
startup information from Linux. The values saved by this program go below S.

The first two instructions put `0x11223344` into EAX and `0x00001234` into EBX.
They do not change ESP.

`push eax` subtracts 4 from ESP and stores EAX at the new top.
`push ebx` subtracts another 4 and stores EBX. ESP is now S - 8.
The `nop` makes no changes, so both entries can be inspected there.

~~~text
Address    Value
S - 8      0x00001234          <- ESP, saved EBX
S - 4      0x11223344             saved EAX
S          Startup information
~~~

`pop ebx` reads the most recently pushed value into EBX and adds 4 to ESP.
Then `pop eax` reads the next value into EAX and adds another 4.
ESP has returned to S.

| Point in the program | ESP |
| --- | --- |
| At `_start` | S |
| After `push eax` | S - 4 |
| After `push ebx` | S - 8 |
| After `pop ebx` | S - 4 |
| After `pop eax` | S |

Finally, `mov eax, 1` replaces EAX with the 32-bit Linux exit-call number.
`xor ebx, ebx` replaces EBX with exit status 0. `int 0x80` ends the program.
The saved values have already been restored before those exit instructions
overwrite them. There are no function calls or local-variable frames here.

## What I observed

The program printed nothing and exited with status 0. At the `nop`, GDB
showed `0x00001234` and `0x11223344` in the two stack entries.
After both pops, EBX was `0x1234`, EAX was `0x11223344`, and ESP was back to S.
Before exit, EAX was 1 and EBX was 0.

The most stack space used below S was 8 bytes, compared with 16 bytes in
`05.asm`. `push ebx` uses 4 bytes even though `0x1234` fits in fewer bytes.
The last value pushed comes out first. Neither register changes between its
push and pop, so I used the stack entries and ESP changes to follow the work.
A pop moves ESP up without clearing the old bytes in memory.

I checked a fresh ELF32 build because the existing executable was ELF64.
In 64-bit mode, the same push/pop opcode bytes can operate on 64-bit registers.
I used `nasm -f elf32` and `ld -m elf_i386` so the run would show the 4-byte
ESP changes described here.

## How I ran the program

From the `07_stack` folder, I assembled, linked, and ran the program with:

~~~sh
nasm -f elf32 06.asm -o 06.o
ld -m elf_i386 06.o -o 06
./06
echo $?  # 0
~~~

To inspect the stack, I opened the executable in GDB:

~~~sh
gdb -q 06
~~~

In GDB, I used `break _start`, `run`, and `si`.
At the `nop`, `x/2wx $esp` shows both saved values.
After the pops, I checked `info registers esp eax ebx` before
`mov eax, 1` and `xor ebx, ebx` changed the registers for exit.
The host needs support for running 32-bit x86 programs.
