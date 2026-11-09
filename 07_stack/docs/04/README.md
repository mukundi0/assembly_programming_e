# 04.asm: a 32-bit function stack frame

[Source: 04.asm](../../04.asm)

This program is the 32-bit counterpart of `02.asm`. It calls `work`, saves EBP,
reserves two 4-byte local variables, and restores the stack before returning.
It uses ESP and EBP rather than RSP and RBP.

## Execution and stack changes

Let S be `ESP` on entry to `_start`. Linux has already placed startup
information on the process stack. `_start` has no caller return address.
Old EBP means the value in EBP before `work` saves it.

| Instruction or stage | ESP after execution | Effect |
| --- | --- | --- |
| Enter `_start` | S | Initial process stack. |
| `call work` | S - 4 | Push the address of `mov eax, 1`, then enter work. |
| `push ebp` | S - 8 | Save old EBP in a 4-byte entry. |
| `mov ebp, esp` | S - 8 | Establish EBP = S - 8 as a fixed frame reference. |
| `sub esp, 8` | S - 16 | Reserve two 4-byte local-variable slots. |
| Store and load the locals; `nop` | S - 16 | Access local data without changing ESP. |
| `mov esp, ebp` | S - 8 | Release all 8 bytes of local storage. |
| `pop ebp` | S - 4 | Restore old EBP; the return address becomes the top. |
| `ret` | S | Pop the 4-byte return address into EIP and resume the caller. |
| `mov eax, 1`; `xor ebx, ebx` | S | Prepare the 32-bit Linux exit call with status 0. |
| `int 0x80` | S immediately before exit | Terminate the process. |

The allocation does not initialize the locals. The stores put `100` into
`[ebp - 4]` and `200` into `[ebp - 8]`. The subsequent loads set `EAX = 100`
and `EBX = 200`. These instructions access existing slots; they do not push
the values or change ESP.

At the `nop`, the layout is:

~~~text
Address    Contents
S - 16     200                  <- ESP, [EBP - 8]
S - 12     100                     [EBP - 4]
S - 8      Saved old EBP        <- EBP
S - 4      Return address          [EBP + 4]
S          Initial process data
~~~

Maximum additional stack use is 16 bytes: 4 for the return address, 4 for
saved EBP, and 8 for locals. The 64-bit version uses 32 bytes for the same
roles. Resetting ESP releases the locals without clearing memory; popping
EBP and executing `ret` then restore the caller's state in the correct order.

After returning, the exit setup overwrites EAX with 1 and EBX with 0. The
loaded local values therefore are not the final exit-call arguments.
`int 0x80` transfers control to the kernel; it does not consume a user-space
return address as `ret` does. Any kernel entry bookkeeping is separate from
the user-space frame described here.

## Observations from running

The fresh ELF32 build produced no stdout or stderr and exited with status 0.
At `work`'s `nop`, GDB showed `EAX = 100`, `EBX = 200`, and `ESP - S = -16`.
Four 4-byte entries from ESP were `200`, `100`, saved EBP (0 in this run),
and return address `0x08049005`. After `ret`, ESP was S and EBP was restored.

The existing executable `07_stack/04` was ELF64. Verification used the fresh
ELF32 build below: `BITS 32` alone does not determine the executable's runtime
mode. Using an ELF64 build can mix 64-bit stack operations with instructions
that truncate stack addresses through ESP/EBP, so it does not test this frame
correctly.

## Build and inspect

From the repository root, use both the 32-bit assembler format and linker mode:

~~~sh
stack_build=$(mktemp -d)
nasm -f elf32 -g -F dwarf 07_stack/04.asm -o "$stack_build/04.o"
ld -m elf_i386 "$stack_build/04.o" -o "$stack_build/04"
"$stack_build/04"
echo $?  # 0
gdb -q "$stack_build/04"
~~~

The Linux host needs support for running 32-bit x86 executables. In GDB, use
`break _start`, `run`, `set $initial_sp = $esp`, and `si`. At the `nop`,
inspect `info registers esp ebp eax ebx` and `x/4wx $esp`. After `ret`,
`p/d (long long)$esp - (long long)$initial_sp` should be 0.
