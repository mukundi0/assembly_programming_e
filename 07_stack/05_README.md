# 05.asm: saving and restoring 64-bit registers

[Source: 05.asm](05.asm)

This program saves RAX and RBX on the stack and restores them in reverse
order. It also shows that the stack entry size follows the register operand
size, even when the value itself is small.

## Execution and stack changes

Let S be `RSP` on entry to `_start`. Linux has already prepared the process
stack with startup data, so the stack is not empty. `_start` itself has no
caller return address. The temporary entries go below S.

| Instruction or stage | RSP after execution | Effect |
| --- | --- | --- |
| Enter `_start` | S | Initial process stack. |
| `mov rax, 0x1122334455667788` | S | Load a full 64-bit value into RAX. |
| `mov rbx, 0x1234` | S | Load RBX with `0x0000000000001234`. |
| `push rax` | S - 8 | Store all 8 bytes of RAX below the original top. |
| `push rbx` | S - 16 | Store all 8 bytes of RBX at the new top. |
| `nop` | S - 16 | Leave both saved entries available for inspection. |
| `pop rbx` | S - 8 | Restore the most recently saved value, `0x1234`. |
| `pop rax` | S | Restore `0x1122334455667788`. |
| `mov rax, 60` | S | Replace the restored RAX value with the exit-call number. |
| `xor rdi, rdi` | S | Select exit status 0. |
| `syscall` | S immediately before exit | Terminate the process. |

At the `nop`:

~~~text
Address    Contents
S - 16     0x0000000000001234    <- RSP, saved RBX
S - 8      0x1122334455667788       saved RAX
S          Initial process data
~~~

Each push first subtracts 8 from RSP and then stores its value. Each pop reads
the current top before adding 8. Thus maximum additional stack use is 16 bytes.
`push rbx` does not use only 2 bytes just because `0x1234` fits in that space;
the operand is RBX, so it saves an 8-byte value.

The machine stores these values in little-endian byte order. Viewed byte by
byte, the saved RBX starts `34 12 00 00 00 00 00 00` and the saved RAX starts
`88 77 66 55 44 33 22 11`. A normal 8-byte memory display reconstructs the
original numbers.

Restoring RBX before RAX matches the reverse of the save order. No instruction
changes either register between saving and restoring, so their visible values
happen to be the same before and after the pops. The memory entries and RSP
changes establish that the save/restore operations still occurred. Popping
does not erase those entries; it makes their storage available for reuse.
There are no calls or local stack frames, and process exit does not pop a
return address.

## Observations from running

The fresh ELF64 build printed nothing to stdout or stderr and exited with
status 0. At the `nop`, GDB showed `RSP - S = -16` and the 8-byte entries
`0x0000000000001234` and `0x1122334455667788`. Immediately after both pops,
it showed `RSP = S`, `RBX = 0x1234`, and `RAX = 0x1122334455667788`.
After the next instruction, RAX was 60, so the restored value must be
inspected before that exit setup.

## Build and inspect

From the repository root:

~~~sh
stack_build=$(mktemp -d)
nasm -f elf64 -g -F dwarf 07_stack/05.asm -o "$stack_build/05.o"
ld -m elf_x86_64 "$stack_build/05.o" -o "$stack_build/05"
"$stack_build/05"
echo $?  # 0
gdb -q "$stack_build/05"
~~~

Use `break _start`, `run`, `set $initial_sp = $rsp`, and `si`. At the `nop`,
use `x/2gx $rsp` for whole values or `x/16bx $rsp` for individual bytes.
Step past both pops, stopping before `mov rax, 60`, then inspect
`info registers rsp rax rbx` and `p/d (long)$rsp - (long)$initial_sp`.
