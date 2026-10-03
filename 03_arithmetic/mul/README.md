# Multiplication and EFLAGS

I looked at `mul1` and `mul2`. `MUL` is unsigned multiplication. Check the
result and flags in GDB immediately after the instruction.

## mul1: 8-bit multiplication

```text
25 * 10 = 250 = 0x00FA
```

The product is in `AX`. Its upper half, `AH`, is zero.

| Flag | Status | Why |
| --- | --- | --- |
| CF | Cleared | The upper half `AH` is zero. |
| OF | Cleared | For `MUL`, OF follows the same rule as CF. |
| PF, AF, ZF, SF | Undefined | `MUL` does not define these flags. |
| IF | Unchanged | `MUL` does not change IF. |

## mul2: 16-bit multiplication

```text
3000 * 200 = 600000 = 0x0009:0x27C0
```

The product is in `DX:AX`: `DX = 0x0009` and `AX = 0x27C0`.

| Flag | Status | Why |
| --- | --- | --- |
| CF | Set | The upper half `DX` is nonzero. |
| OF | Set | For `MUL`, OF follows the same rule as CF. |
| PF, AF, ZF, SF | Undefined | `MUL` does not define these flags. |
| IF | Unchanged | `MUL` does not change IF. |

CF and OF show whether the product fits in the lower half. The other listed
arithmetic flags have no defined result after `MUL`.
