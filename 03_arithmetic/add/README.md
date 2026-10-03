# Addition and EFLAGS

I checked the results and flags for `add1` and `add2`. `add1` adds 8-bit
values in `AL`; `add2` adds 16-bit values in `AX`.

Build a program with `nasm -f elf32 -g -F dwarf add1.asm -o add1.o` and
`ld -m elf_i386 add1.o -o add1`. Replace `add1` with `add2` for the other
program. In GDB, check EFLAGS immediately after `ADD`, before `xor ebx, ebx`.

## add1: 8-bit addition

```text
120 + 10 = 130 = 0x82 = 10000010
```

GDB showed `AL = 0x82` and flags `[ PF AF SF IF OF ]`. GDB displays AL as
`-126` when signed, but the same bits equal 130 when unsigned.

| Flag | Status | Why |
| --- | --- | --- |
| CF | Cleared | 130 fits in an unsigned byte, so no carry leaves bit 7. |
| PF | Set | The result has two 1s, an even count. |
| AF | Set | `0x8 + 0xA` carries from bit 3 to bit 4. |
| ZF | Cleared | The result is not zero. |
| SF | Set | Bit 7 is 1. |
| OF | Set | 130 is greater than the signed-byte maximum, 127. |

CF is clear because the result fits unsigned; OF is set because it does not
fit as a positive signed byte.

## add2: 16-bit addition

```text
32000 + 500 = 32500 = 0x7EF4
```

GDB showed `AX = 0x7EF4` and flags `[ IF ]`.

| Flag | Status | Why |
| --- | --- | --- |
| CF | Cleared | 32500 fits in an unsigned word. |
| PF | Cleared | The low byte `0xF4` has five 1s, an odd count. |
| AF | Cleared | The low nibbles add to `0x4` with no carry. |
| ZF | Cleared | The result is not zero. |
| SF | Cleared | Bit 15 is 0. |
| OF | Cleared | 32500 is within the signed 16-bit range. |

`ADD` changes CF, PF, AF, ZF, SF, and OF. IF is unchanged.
