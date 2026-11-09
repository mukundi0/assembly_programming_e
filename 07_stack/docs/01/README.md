# 01.asm: pushing and popping into different registers

[Source: 01.asm](../../01.asm)

This 64-bit program pushes 100 and 200, then pops them into different
registers. It demonstrates last in, first out: the last value pushed is the
first value popped.

## Execution and stack changes

Let S be `RSP` on entry to `_start`. Linux has already prepared the process
stack, including the argument count at its top. There is no caller return
address at `_start`. The program's temporary values are stored below S.

| Instruction or stage | RSP after execution | Effect |
| --- | --- | --- |
| Enter `_start` | S | Start above the program's temporary stack storage. |
| `mov rax, 100`; `mov rbx, 200` | S | Load registers; no stack access. |
| `push rax` | S - 8 | Decrease RSP by 8 and store 100 at `[rsp]`. |
| `push rbx` | S - 16 | Decrease RSP by 8 and store 200 at the new top. |
| First `nop` | S - 16 | No change; both values remain on the stack. |
| `pop rcx` | S - 8 | Read 200 from the top into RCX, then increase RSP by 8. |
| `pop rdx` | S | Read 100 into RDX, then increase RSP by 8. |
| Second `nop` | S | No change; both temporary entries have been removed. |
| `mov rax, 60`; `xor rdi, rdi` | S | Select Linux exit with status 0. |
| `syscall` | S immediately before exit | Terminate the process. |

At the first `nop`, the layout from lower to higher addresses is:

~~~text
Address    Contents
S - 16     200 (0x00000000000000c8)  <- RSP, top
S - 8      100 (0x0000000000000064)
S          Initial process data
~~~

The two pushes use 16 bytes in total. Although 100 and 200 are small numbers,
each push uses 8 bytes because its operand is a 64-bit register. The values
are copied: pushing RAX or RBX does not remove their contents.

Popping into RCX and RDX transfers the saved values in reverse order. The
program does not swap RAX and RBX; those registers still hold 100 and 200
until the exit setup changes RAX. A pop advances RSP without clearing the
memory that held the value. There are no function calls or local stack frames.
The exit system call does not consume a user-space return address.

## Observations from running

The fresh ELF64 build printed nothing to stdout or stderr and exited with
status 0. GDB showed `RSP - S = -16` at the first `nop` and the two stack values
`0xc8` followed by `0x64`. At the second `nop`, it showed `RCX = 200`,
`RDX = 100`, and `RSP = S`. This confirms the last-in, first-out order and that
the stack is balanced before exit.

## Build and inspect

Run these commands from the repository root:

~~~sh
stack_build=$(mktemp -d)
nasm -f elf64 -g -F dwarf 07_stack/01.asm -o "$stack_build/01.o"
ld -m elf_x86_64 "$stack_build/01.o" -o "$stack_build/01"
"$stack_build/01"
echo $?  # 0
gdb -q "$stack_build/01"
~~~

In GDB, use `break _start`, `run`, and `set $initial_sp = $rsp`, then step
with `si`. At the first `nop`, use `x/2gx $rsp` to view both entries.
At the second `nop`, use `info registers rsp rcx rdx` and
`p/d (long)$rsp - (long)$initial_sp`. Check RCX before `syscall`, which can
clobber RCX as part of the system-call mechanism.
