# Math review (2026-10-23)

Every result below was observed by running `bin/luna`. Source: `compiler/runtime/luna_rt_math.lu`
(functions), `luna_rt_ops.lu` (operators), `compiler/parser.lu` (unary minus). `//` is the comment
marker in Luna, so it is **not** a candidate for floor division.

Legend: OK = behaves as documented and sensibly. BUG = wrong result or crash. GAP = missing or
inconsistent behavior. Fixes are proposals; nothing has been changed yet.

## Operators

| Item | Status | Observed | Fix |
|---|---|---|---|
| `+ - *` on ints | OK | overflow is `integer overflow in '+'` | none |
| `/` on ints | OK | truncates toward zero; `INT_MIN / -1` errors; `5 / 0` errors | none |
| `%` on ints/floats | OK | sign follows the dividend (`-5 % 3` is `-2`); `% 0` errors | document it |
| `-(-3)`, `- -3`, `- -1.5` | **BUG** | clang fails: IR contains `i64 --3` | `parser.lu:343` p_unary: when the operand literal already starts with `-`, remove the `-` instead of prefixing another one. Add a test |
| `-x` for ints | OK | `-INT_MIN` errors with overflow | none |
| `3 > 2 > 1` | OK | Type Error (bool vs int) | none |
| int/float mixing | GAP | `3 == 3.0` true; `9007199254740993 == 9007199254740992.0` is also true (int converted to double) | compare exactly: convert float to int when it is whole and in range, else unequal; document |
| `2 ** 3`, `&`, `\|`, `^`, `<<`, `>>` | GAP | syntax errors | add `**` and bitwise ops, see plan |
| literals `0x1F`, `1_000`, `1e3`, `2.5e-4` | GAP | syntax errors | lexer additions |

## Basic utilities

| Function | Status | Observed | Fix |
|---|---|---|---|
| `abs(x)` | OK | `abs(INT_MIN)` is an overflow error; wrong types are a Type Error | none |
| `min(a, b)`, `max(a, b)` | GAP | exactly 2 args (`max(1,3,2)` is an Argument Error); with NaN they return NaN; `min(2, 2.0)` is `int` but `min(2.0, 2)` is `float` (result type depends on argument order) | accept 2+ args or one list; on ties return the first argument; define NaN behavior (or make NaN impossible, see below) |
| `clamp(x, lo, hi)` | GAP | `clamp(5, 10, 0)` silently returns `10` | error `clamp() lo must not be greater than hi`; keep input type |
| `sign(x)` | GAP | always returns `int`; `sign(NaN)` is `0` | fine for numbers; error on NaN once NaN is removed |

## Powers and roots

| Function | Status | Observed | Fix |
|---|---|---|---|
| `pow(a, b)` | BUG/GAP | always float: `pow(2, 62)` is a float (exact only while < 2^53, prints `4.61169e+18`); `pow(0, -1)` is `inf`; `pow(-8.0, 0.333)` is `nan`; `pow(10, 400)` is `inf` | int base and non-negative int exponent: exact int result with `integer overflow in 'pow'`; otherwise float; `0` to a negative power is a `division by zero` error; negative base with fractional exponent is a domain error; overflow to inf is an error |
| `pow(-8, 1/3)` | trap | `1/3` is the **int** `0`, so the result is `1` | not a runtime bug; document the int division trap next to `pow`/`lerp` |
| `sqrt(x)` | GAP | `sqrt(-4)` prints `-nan` | domain error: `sqrt() of a negative number` |
| `cbrt(x)` | OK | negatives work; exact cubes made exact | none |
| `exp(x)` | GAP | `exp(710)` is `inf` | overflow error |
| `ln(x)`, `log10(x)` | GAP | `ln(0)` is `-inf`; `ln(-1)` is `-nan`; `log10(-5)` is `nan` | domain error for `x <= 0` |

## Rounding

| Function | Status | Observed | Fix |
|---|---|---|---|
| `floor/ceil/round/trunc(x)` | **BUG** | `floor(pow(2, 63))` returns `-9223372036854775808` (wrong sign): the range check in `_round_to_int` uses `9.3e18`, above 2^63 = `9.223e18`, so values between them overflow the float-to-int conversion | `luna_rt_math.lu:264`: accept only `-9223372036854775808.0 <= r < 9223372036854775808.0` |
| `floor/ceil/round/trunc` of huge, NaN or inf | GAP | docs say they return an int, but `floor(pow(10,30))` returns a float `1e+30` and `floor(sqrt(-1))` returns `-nan` | raise `result does not fit in an integer` (matching `int()`, which already errors) |
| `round(x)` | OK | halves away from zero (`round(2.5)` 3, `round(-2.5)` -3) | add `round(x, digits)` returning a float |
| `fract(x)` | GAP | `fract(-5.9)` is `0.1` (x minus floor), not `-0.9`; `fract(5)` returns float | document the definition; keep it |

## Remainder and interpolation

| Function | Status | Observed | Fix |
|---|---|---|---|
| `mod(x, y)` | GAP | `mod(5, 3)` returns float `2`, but `5 % 3` is int `2`; duplicates `%` | return int for two ints; or state that `mod` is the float remainder and `%` the integer one |
| `lerp(a, b, t)` | OK | `t` outside 0..1 extrapolates (`lerp(0,100,2)` is 200); always float | document extrapolation; add `clamp01`-style helper if wanted |
| `deg_to_rad`, `rad_to_deg` | OK | fine | none |

## Trigonometry

| Function | Status | Observed | Fix |
|---|---|---|---|
| `sin cos tan atan sinh cosh tanh` | OK/GAP | correct; `sinh(1000)` is `inf` silently | overflow error for sinh/cosh |
| `asin(x)`, `acos(x)` | GAP | `asin(2)` is `nan` | domain error when `x` is outside [-1, 1] |
| `atan2(y, x)` | OK | `atan2(0, 0)` is `0` (C convention) | document |
| `tan` near pi/2 | OK | huge but finite | none |

## Random numbers

| Function | Status | Observed | Fix |
|---|---|---|---|
| `rand()` | OK | float in [0, 1) with 53 bits | none |
| `rand(max)`, `rand(min, max)` | **BUG** | `rand("a")`, `rand(null)`, `rand([1])` return `0` with no error (the code reads the payload without checking the tag) | check type with `_arg_num` style helper; ints only, or truncate floats explicitly |
| `rand(1, 2, 3)` | **BUG** | prints `Runtime Error: ...` to stderr and returns `null`; the program keeps running | use `lv_arity` like every other builtin so it is an Argument Error that exits |
| `rand(-INT_MIN.., INT_MAX)` | BUG (edge) | span is 2^64, `range == 0` is treated as "return lo", so it always returns `INT_MIN` | when the span wraps to 0, return the raw 64-bit value |
| `rand(5.9)` | GAP | silently truncated to `5` | either error on floats or document truncation |
| `rand(-5)` | GAP | swaps to `rand(-5, 0)`; fine but undocumented for the one-arg form | document |
| modulo bias | GAP | `urem(next, range)` is slightly biased for big ranges | rejection sampling (only matters for huge ranges) |
| `srand(seed)` | **BUG** | `srand("x")`, `srand([1])` are accepted; a float is cast with `f2i` on the bit pattern of the payload, not the value; `srand(1.5)` gives garbage | accept ints only (or call the same numeric helper); error otherwise |
| `trand()` | GAP | `trand(5)` ignores the extra argument | check arity 0 |
| `shuffle`, `ssort` | OK | in place, return `null`; `ssort` name is a joke algorithm (Stalin sort deletes elements) | consider renaming/documenting prominently; not math-critical |

## Conversions that feed math

| Function | Status | Observed | Fix |
|---|---|---|---|
| `int(v)` | OK | float out of range, NaN and `null` raise errors | none |
| `int("99999999999999999999")` | **BUG** | silently saturates to `9223372036854775807` | overflow error, or use the proposed `parse_int` returning `null` |
| `int("abc")`, `to_int("")`, `to_int("-")` | GAP | return `0` | `parse_int` returning `null`; keep old functions documented as lenient |
| `to_float("1.5x")`, `float("")` | GAP | `1.5`, `0` | `parse_float` returning `null` |
| `float("nan")`, `float("inf")`, `float("0x10")` | GAP | accepted (libc `strtod`) so NaN/inf can enter a program | reject these in the string parser |
| `float(x)` / float display | GAP | printing uses 6 significant digits: `123456789.0` shows `1.23457e+08`, and `1000000.0` shows `1e+06`; precision is invisible | keep `print` short, but add `fmt_float(x, digits)`; consider shortest-round-trip printing for `to_string` |
| `-0.0` | GAP | prints `-0`; `abs(-0.0)` prints `0` | normalise on print, or document |

## Vector math (dense lists)

| Function | Status | Observed | Fix |
|---|---|---|---|
| `dense_list(n, fill)` | GAP | `dense_list(3, 0)` is an error because the fill must be a float; message `expects (size: int, fill: float)` is the same for every bad input (negative size, huge size, string fill) | accept ints (convert), give a precise message per failure |
| `vec_add/sub/mul/div` | **BUG** | length mismatch silently truncates (`[1,2]+[1,2,3]` is `d[2,4]`); non-number element silently counts as 0 (`vec_add([1,"a"],[1,2])` is `d[2,2]`); `vec_div` by 0 yields `inf` | error on length mismatch; Type Error on non-numbers; `division by zero` error for `vec_div` |
| `vec_mul_inline` | GAP | wrong argument types print `Error: ...` with no line number and the program continues with `null` | use `lv_error` like the rest |
| `mat_mul` | GAP | dimension mismatch prints `Runtime Error: ...` to stderr and returns `null`, no exit | use `lv_error` with a source line |

## Priority

1. **Real bugs** (small, safe, each with a test):
   - `-(-3)` compiler bug in `p_unary`.
   - `_round_to_int` range check.
   - `rand`/`srand` type and arity checks.
   - `int("9999...")` saturation.
   - `vec_*` silent truncation and silent zero for non-numbers.
2. **Policy change** (needs your decision): turn NaN/inf into errors (`sqrt`, `ln`, `log10`, `asin`, `acos`, `pow`, `exp`, `sinh`, `cosh`), so no NaN can ever exist in a program. This also removes the NaN rows in `min`, `max`, `sign`, `floor`.
3. **New features**: `**`, bitwise ops, numeric literal forms, `min/max/sum` over lists, exact integer `pow`, `round(x, digits)`, `parse_int/parse_float`, `fmt_float`.
