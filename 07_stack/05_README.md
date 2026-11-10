# 05.asm - Saving and restoring 64-bit values

[Source code](05.asm)

In this program, I looked at how RAX and RBX can be saved on the stack and
then restored. The program runs in 64-bit mode, so each saved register
takes 8 bytes.

## How the program runs

I use S for RSP at the beginning of `_start`. Linux has already put startup
information on the stack. This program adds its own temporary values below S.

First, `mov rax, 0x1122334455667788` loads RAX.
`mov rbx, 0x1234` loads RBX with `0x0000000000001234`.
Neither instruction changes RSP.

`push rax` subtracts 8 from RSP and saves the full RAX value.
`push rbx` subtracts another 8 and saves RBX at the new top.
RSP is now S - 16. The `nop` does not change anything.

~~~text
Address    Value
S - 16     0x0000000000001234   <- RSP, saved RBX
S - 8      0x1122334455667788      saved RAX
S          Startup information
~~~

Next, `pop rbx` reads the top entry into RBX and adds 8 to RSP.
`pop rax` reads the next entry into RAX and adds another 8.
Both registers have their saved values, and RSP is back to S.

| Point in the program | RSP |
| --- | --- |
| At `_start` | S |
| After `push rax` | S - 8 |
| After `push rbx` | S - 16 |
| After `pop rbx` | S - 8 |
| After `pop rax` | S |

The program then runs `mov rax, 60` to select the Linux exit call and
`xor rdi, rdi` to set status 0. `syscall` ends execution.
The restored RAX value is therefore replaced before the program exits.
There are no function calls or local-variable frames in this example.

## What I observed

The program printed nothing and exited with status 0. At the `nop`, GDB
showed the top entries as `0x0000000000001234` and `0x1122334455667788`.
After both pops, RAX was `0x1122334455667788`, RBX was `0x1234`, and RSP
was back to S. The most stack space used below S was 16 bytes.

Even though `0x1234` is a small value, `push rbx` still uses 8 bytes because
RBX is a 64-bit register. The pops happen in the reverse order of the pushes,
which is how each value gets back to the right register. The registers are
not changed between saving and restoring, so their values look the same
before and after. The memory entries and RSP changes show the stack operations.
Popping releases the entries without erasing the bytes.

## How I ran the program

From the `07_stack` folder, I assembled, linked, and ran the program with:

~~~sh
nasm -f elf64 05.asm -o 05.o
ld 05.o -o 05
./05
echo $?  # 0
~~~

To inspect the stack, I opened the executable in GDB:

~~~sh
gdb -q 05
~~~

In GDB, I used `break _start`, `run`, and `si`.
At the `nop`, `x/2gx $rsp` shows the saved values.
After both pops, I checked `info registers rsp rax rbx` before executing
`mov rax, 60`, so the restored RAX value was still visible.
