# Division and EFLAGS

I looked at `div1` and `div2`. Both use unsigned division. Check the
registers after `DIV` in GDB. The arithmetic flags after `DIV` are
undefined, so displayed values do not describe the answer.

## div1: 8-bit divisor

```text
100 / 7 = 14 remainder 2
```

`DIV BL` puts the quotient in `AL` (14) and remainder in `AH` (2).

| Flag | Status | Why |
| --- | --- | --- |
| CF, PF, AF, ZF, SF, OF | Undefined | `DIV` does not set these flags to describe its result. |
| IF | Unchanged | `DIV` does not change IF. |

If the divisor is zero or the quotient does not fit, the CPU raises a
divide-error exception (`#DE`). It does not report division overflow with OF.

## div2: 16-bit divisor

```text
50000 / 300 = 166 remainder 200
```

`DIV BX` divides `DX:AX` by `BX`. The quotient is `AX = 166` and the
remainder is `DX = 200`. The quotient fits in AX, so no `#DE` occurs.

The flags are the same as in div1: CF, PF, AF, ZF, SF, and OF are
undefined; IF is unchanged. Any values GDB displays for undefined flags
are not meaningful results of the division.
