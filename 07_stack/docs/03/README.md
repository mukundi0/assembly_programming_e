# 03.asm: the stack behind call and ret

[Source: 03.asm](../../03.asm)

This 64-bit program passes 10 to `calculate` in RAX. The function adds 5 and
returns 15 in the same register. Although there are no explicit `push` or
`pop` instructions, `call` and `ret` still use the stack.

## Execution and stack changes

Let S be `RSP` at entry to `_start`. Linux has already prepared the process
stack with arguments and other startup information. There is no caller return
address for `_start` itself.

| Instruction or stage | RSP after execution | Effect |
| --- | --- | --- |
| Enter `_start` | S | Initial process stack. |
| `mov rax, 10` | S | Put the input in RAX; no stack change. |
| `call calculate` | S - 8 | Push the address of `after_call` and enter calculate. |
| `add rax, 5` | S - 8 | Change RAX from 10 to 15; the return address stays on top. |
| `ret` | S | Read the saved address into RIP and increase RSP by 8. |
| `after_call: nop` | S | Execution resumes here with RAX = 15. |
| `mov rax, 60` | S | Replace the result with the Linux exit system-call number. |
| `xor rdi, rdi` | S | Set exit status to 0. |
| `syscall` | S immediately before exit | End the process. |

While inside `calculate`, the stack contains:

~~~text
Address    Contents
S - 8      Address of after_call  <- RSP
S          Initial process data
~~~

`call` saves the address immediately after itself, rather than the address of
`calculate`. This gives `ret` the location where the caller should continue.
The return address occupies 8 bytes, so maximum additional stack use is
8 bytes.

The value 10 is not pushed as an argument, and the result 15 is not popped
from the stack. Both travel in RAX. `calculate` creates no RBP frame and
reserves no local variables because this small function only needs a register.
`add` changes RAX and arithmetic flags without moving RSP.

After `ret`, execution resumes at `after_call` rather than falling through
into the function again. The caller exits through a system call. It does not
use a second `ret` at `_start`, where no caller return address exists.
The result must be inspected at `after_call`, before `mov rax, 60` replaces it.

## Observations from running

The fresh ELF64 build printed nothing to stdout or stderr and exited with
status 0. In GDB, RSP decreased by 8 upon entering `calculate`, stayed there
during `add`, and returned to S after `ret`. Execution resumed at
`after_call`, address `0x40100a` in this build, with `RAX = 15` (`0xf`).
This confirms that `call`/`ret` manage control flow on the stack while the
calculation's data stays in a register. The exit status was 0, not 15.

## Build and inspect

From the repository root:

~~~sh
stack_build=$(mktemp -d)
nasm -f elf64 -g -F dwarf 07_stack/03.asm -o "$stack_build/03.o"
ld -m elf_x86_64 "$stack_build/03.o" -o "$stack_build/03"
"$stack_build/03"
echo $?  # 0
gdb -q "$stack_build/03"
~~~

Use `break _start`, `run`, and `set $initial_sp = $rsp`, then step with `si`.
Inside `calculate`, use `x/gx $rsp` and `p/x &after_call` to compare the saved
address with the return target. After `ret`, use `info registers rip rsp rax`
and `p/d (long)$rsp - (long)$initial_sp`; the offset should be 0.
