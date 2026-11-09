# 06.asm: saving and restoring 32-bit registers

[Source: 06.asm](06.asm)

This is the 32-bit counterpart of `05.asm`. It saves EAX and EBX on the stack,
then restores EBX followed by EAX. The stack operations use 4-byte entries
and update ESP.

## Execution and stack changes

Let S be `ESP` at entry to `_start`. Linux has already placed startup data
on the process stack. There is no caller return address for `_start` itself.
This program uses temporary stack memory below S.

| Instruction or stage | ESP after execution | Effect |
| --- | --- | --- |
| Enter `_start` | S | Initial process stack. |
| `mov eax, 0x11223344` | S | Load EAX; no stack access. |
| `mov ebx, 0x1234` | S | Load EBX with `0x00001234`. |
| `push eax` | S - 4 | Decrease ESP by 4 and store EAX at the new top. |
| `push ebx` | S - 8 | Decrease ESP by 4 and store EBX above the earlier entry in stack order. |
| `nop` | S - 8 | No change; both entries remain on the stack. |
| `pop ebx` | S - 4 | Read `0x00001234` from the top and increase ESP by 4. |
| `pop eax` | S | Read `0x11223344` and increase ESP by 4. |
| `mov eax, 1` | S | Replace EAX with the 32-bit Linux exit-call number. |
| `xor ebx, ebx` | S | Replace EBX with exit status 0. |
| `int 0x80` | S immediately before exit | Terminate the process. |

At the `nop`, lower addresses contain the most recently pushed value:

~~~text
Address    Contents
S - 8      0x00001234           <- ESP, saved EBX
S - 4      0x11223344              saved EAX
S          Initial process data
~~~

Maximum additional stack use is 8 bytes, compared with 16 in the 64-bit
example. Even though `0x1234` fits in 2 bytes, `push ebx` saves 4 bytes because
EBX is a 32-bit operand. In little-endian byte order, the 8 bytes from ESP
are `34 12 00 00 44 33 22 11`.

The last value pushed is the first value popped, so restoring EBX before EAX
matches the save order. Neither register is modified between the pushes and
pops; restoring them therefore leaves the same visible register values.
The changing ESP and saved memory show the actual stack work. Pops release
the slots without clearing their bytes.

The program creates no frame and calls no functions. After the pops, the exit
setup intentionally replaces both restored values. `int 0x80` enters the
kernel to exit; it does not use a user-space return address. Kernel entry
bookkeeping is separate from this program's user stack.

## Observations from running

The fresh ELF32 build printed nothing to stdout or stderr and exited with
status 0. At the `nop`, GDB showed `ESP - S = -8` and the two 4-byte entries
`0x00001234` and `0x11223344`. Immediately after the pops, ESP was S,
`EBX = 0x1234`, and `EAX = 0x11223344`. Before exit, EAX was 1 and EBX was 0.

The pre-existing `07_stack/06` executable was ELF64. Its push/pop opcode bytes
decode as 64-bit register operations in that mode, so even an apparently
successful run would not establish the 4-byte ESP behavior described here.
These observations come from the correct ELF32 build below. `BITS 32` must
be paired with the appropriate assembler output format and linker mode.

## Build and inspect

From the repository root:

~~~sh
stack_build=$(mktemp -d)
nasm -f elf32 -g -F dwarf 07_stack/06.asm -o "$stack_build/06.o"
ld -m elf_i386 "$stack_build/06.o" -o "$stack_build/06"
"$stack_build/06"
echo $?  # 0
gdb -q "$stack_build/06"
~~~

The host needs support for running 32-bit x86 executables. Use `break _start`,
`run`, `set $initial_sp = $esp`, and `si`. At the `nop`, use `x/2wx $esp`
to inspect the saved values or `x/8bx $esp` to inspect their bytes.
After both pops, stop before `mov eax, 1` and check
`info registers esp eax ebx` and
`p/d (long long)$esp - (long long)$initial_sp`.
