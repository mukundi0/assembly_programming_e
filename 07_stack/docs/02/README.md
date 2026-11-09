# 02.asm: a 64-bit function stack frame

[Source: 02.asm](../../02.asm)

This program calls `work`, creates a stack frame, stores two 8-byte local
variables, loads them into registers, and removes the frame before returning.

## Execution and stack changes

Let S be `RSP` at `_start`. Linux has already prepared the initial process
stack; `_start` itself has no caller return address. Let old RBP mean the value
in RBP before `work` saves it. Its initial value is not needed by the program.

| Instruction or stage | RSP after execution | Effect |
| --- | --- | --- |
| Enter `_start` | S | Initial process stack. |
| `call work` | S - 8 | Push the address of `mov rax, 60` and jump to work. |
| `push rbp` | S - 16 | Save old RBP below the return address. |
| `mov rbp, rsp` | S - 16 | Set RBP to S - 16 as the frame's fixed reference. |
| `sub rsp, 16` | S - 32 | Reserve 16 bytes for two local variables. |
| Store and load the locals; `nop` | S - 32 | Access the reserved memory without moving RSP. |
| `mov rsp, rbp` | S - 16 | Release both local-variable slots at once. |
| `pop rbp` | S - 8 | Restore old RBP and leave the return address on top. |
| `ret` | S | Consume the return address and resume `_start`. |
| `mov rax, 60`; `xor rdi, rdi` | S | Prepare exit with status 0. |
| `syscall` | S immediately before exit | Terminate the process. |

`sub rsp, 16` only reserves space; it does not initialize it. The two stores
then write `100` to `[rbp - 8]` and `200` to `[rbp - 16]`. The following loads
set `RAX = 100` and `RBX = 200`. None of these memory `mov` instructions pushes
or pops anything.

At the `nop`, the complete frame is:

~~~text
Address    Contents
S - 32     200                  <- RSP, [RBP - 16]
S - 24     100                     [RBP - 8]
S - 16     Saved old RBP        <- RBP
S - 8      Return address          [RBP + 8]
S          Initial process data
~~~

Maximum additional stack use is 32 bytes: 8 for the return address, 8 for
saved RBP, and 16 for locals. The epilogue must discard the locals before
popping RBP, or it would read local data as the saved frame pointer.
`mov rsp, rbp` releases the local storage without erasing its contents.
`ret` then removes the address pushed by `call`. Back in `_start`, RAX is
overwritten with 60 for exit; 100 is a temporary calculation value, not the
exit status. The exit system call does not pop another return address.

## Observations from running

The fresh ELF64 build produced no stdout or stderr and exited with status 0.
At `work`'s `nop`, GDB showed `RAX = 100`, `RBX = 200`, and `RSP - S = -32`.
Reading four 8-byte entries from RSP showed `200`, `100`, the saved RBP
(0 in this run), and return address `0x401005`. After `ret`, RSP was S and RBP
was restored. The return address identified the instruction immediately after
`call work`.

## Build and inspect

From the repository root:

~~~sh
stack_build=$(mktemp -d)
nasm -f elf64 -g -F dwarf 07_stack/02.asm -o "$stack_build/02.o"
ld -m elf_x86_64 "$stack_build/02.o" -o "$stack_build/02"
"$stack_build/02"
echo $?  # 0
gdb -q "$stack_build/02"
~~~

Use `break _start`, `run`, and `set $initial_sp = $rsp`, then `si` through
the call and frame setup. At the `nop`, inspect `info registers rsp rbp rax rbx`
and `x/4gx $rsp`. Step through the epilogue and `ret`, then check
`p/d (long)$rsp - (long)$initial_sp` before the exit setup; it should be 0.
