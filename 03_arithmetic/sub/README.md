# Subtraction and EFLAGS

I looked at `sub1` and `sub2`. `sub1` subtracts 8-bit values in `AL`;
`sub2` subtracts 16-bit values in `AX`. Check EFLAGS in GDB right after
`SUB`, before the later `xor ebx, ebx` instruction.

## sub1: 8-bit subtraction

```text
50 - 80 = -30 = 0xE2 (8-bit result)
```

As an unsigned byte, `0xE2` is 226. As a signed byte, it is -30.

| Flag | Status | Why |
| --- | --- | --- |
| CF | Set | 50 is less than 80 unsigned, so subtraction borrows. |
| PF | Set | `0xE2` has four 1s, an even count. |
| AF | Cleared | The low nibbles subtract as `2 - 0`, with no borrow. |
| ZF | Cleared | The result is not zero. |
| SF | Set | Bit 7 of `0xE2` is 1. |
| OF | Cleared | -30 fits in the signed 8-bit range. |

## sub2: 16-bit subtraction

```text
1000 - 2000 = -1000 = 0xFC18 (16-bit result)
```

As an unsigned word, `0xFC18` is 64536. As a signed word, it is -1000.

| Flag | Status | Why |
| --- | --- | --- |
| CF | Set | 1000 is less than 2000 unsigned, so subtraction borrows. |
| PF | Set | The low byte `0x18` has two 1s, an even count. |
| AF | Cleared | The low nibbles subtract as `8 - 0`, with no borrow. |
| ZF | Cleared | The result is not zero. |
| SF | Set | Bit 15 of `0xFC18` is 1. |
| OF | Cleared | -1000 fits in the signed 16-bit range. |

CF shows unsigned borrow. OF shows signed overflow, so CF can be set while
OF is clear, as it is in both programs.
