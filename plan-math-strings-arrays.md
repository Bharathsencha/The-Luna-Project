# Plan: stabilise math and strings, add proper arrays and basic types

Written 2026-10-23 after probing `bin/luna` directly. Findings are observed behavior, not guesses.

## 1. Where things stand (probed)

### Math
| Finding | Detail |
|---|---|
| `pow` on ints returns a float | `pow(2, 62)` prints `4.61169e+18`; precision lost above 2^53 |
| NaN / inf leak out silently | `sqrt(-1)` prints `-nan`, `ln(0)` prints `-inf`, yet `1.0 / 0.0` is an error. Inconsistent with "errors, not silent wrongness" |
| No numeric literal forms | `0x1F`, `1_000`, `1e3`, `1.5e-7` are all syntax errors |
| No integer operators beyond `+ - * / %` | no `//`, `**`, `& \| ^ << >> ~` |
| `round(x, digits)` unsupported; `min`/`max` take exactly 2 args | no `sum`, no `min(list)` |
| Float printing is `%g` with 6 digits | `123456789.0` prints `1.23457e+08`; `to_string` loses data, no way to ask for precision |
| `to_int("abc")`, `int("12abc")` return 0 | silent failure, cannot tell "0" from "garbage" |
| `type()` calls 64-bit ints outside the 32-bit range `"long"` | docs and README say ints are 64-bit; two names for one type |
| Mixed int/float equality | `3 == 3.0` is `true`, while `'a' == "a"` is an error; rule is undocumented |

### Strings
| Finding | Detail |
|---|---|
| Byte strings with no UTF-8 awareness | `len("héllo")` is 6, `s[1]` is half a character, `reverse` produces invalid bytes |
| `substring`/`slice` clamp negatives differently | `substring("abc", -1, 2)` is `""`; `slice` clamps; `s[-1]` counts from the end; three behaviors |
| No slice syntax | `s[1:3]` is a syntax error |
| `repeat("ab", 2.5)` / `"ab" * 2` | error on float, error on operator; no `*` for strings |
| `contains`/`index_of` accept char or string but not code | fine, but undocumented edge cases (`index_of("abc", "")`) |
| `split` drops empty pieces | `split("a,,b", ",")` gives `[a, b]`, and CSV-style parsing is impossible |
| Missing basics | `lines`, `char_code`/`from_code`, `count`, `title`/`capitalize`, `trim` with chars, `replace` once, `str_cmp`, `format` specs (width, precision) |
| `format` with too few args | leaves `{}` in the output silently |

### Lists, "arrays" and basic types
| Finding | Detail |
|---|---|
| Lists are heterogeneous and growable | `[1, "x", [2]]` is fine; there is no fixed-size typed array |
| `dense_list` is the only typed array | float-only, created with `dense_list(n, fill)`, reports `type` as `"list"`, printed as `d[...]`, and stores strings with an error only at write |
| Very few list functions | `append`, `remove`, `find`, `slice`, `sort`, `shuffle`, `ssort`, `map/filter/reduce`. Missing: `insert`, `pop`, `extend`, `sum`, `min/max` of list, `any/all`, `index` variants, `unique`, `reverse` in place, `sort` with comparator, `copy`, `zip`, `enumerate` |
| `list + list` is element-wise add (numbers only) | `[1,2] + [3,4]` is `d[4, 6]`, not concatenation, and the result type changes silently to dense |
| `sort` rejects mixed number/string | ok, but no key/comparator form |
| Out-of-range write is an error | good, matches the language's stance |
| Basic types: int, float, bool, char, string, null | no byte type, no fixed-width ints, no tuple, no set; `char` vs 1-char string split is a recurring friction point |

## 2. Guiding decisions (please confirm or change)

1. **Keep the language's strictness**: no implicit conversions, errors over silent wrong results. Where
   Luna returns `0`, `-1`, `""` or NaN on bad input today, the stable version either errors or returns
   `null`, and a `try`-free `parse_int`/`parse_float` family returns `null` for bad text.
2. **"Arrays" = a new fixed-size typed array**, separate from the growable heterogeneous `list`.
   - Literal / constructors: `array(int, 5)`, `array(float, 5, 0.0)`, `array(int, [1,2,3])`.
   - Element types: `int`, `float`, `bool`, `byte`, `char`. Elements are unboxed (8 bytes, or 1 for
     `byte`/`bool`), contiguous, bounds-checked, element type enforced on every write.
   - Fixed length; `len`, `a[i]`, `a[i] = v`, `for-in`, `slice` (copy), `copy`, `fill`, `to_list`,
     `sum/min/max`, equality, printing as `array<int>[1, 2, 3]`, `type()` gives `"array"`.
   - **`dense_list` becomes `array(float, n)`**; keep `dense_list` as a deprecated alias for one release,
     and keep `vec_*`/`mat_mul`/operator math working on `array<float>`.
   - Multi-dimensional: `matrix(rows, cols)` over `array<float>` is a later phase, not v1.
3. **One integer type, 64-bit**: remove the `"long"` type name and document `int` as 64-bit. Add
   `byte` as a basic type only inside arrays and via `byte(v)` conversion (0..255, else error).
4. **Strings stay immutable byte strings, but gain a clearly separate UTF-8 layer**: `len`/indexing
   remain bytes (fast, predictable), and new `chars(s)`, `char_count(s)`, `utf8_valid(s)` and
   `to_upper`/`reverse` made UTF-8 safe, so nothing produces invalid bytes.
5. **Every behavior change gets a golden test**, and every doc edit lands with the code.

## 3. Phases

Each phase ends with `make`, `make seed` if compiler or runtime changed, `make test` green, docs updated.
Work in the order below; phases 1 and 2 are independent and can be split between sessions.

### Phase 0: baseline and safety net (small)
- Run `make && make test`; record the pass count in `discussion.md`.
- Add `test/test_math_edges.lu`, `test/test_strings_edges.lu`, `test/test_lists_edges.lu` capturing
  **current** behavior of everything in section 1 (via `# CASE` sections), so later changes show up as
  deliberate diffs.
- Add a micro-benchmark script in `bench/` (list loop, string building, vec_mul) so changes can be
  checked for speed. Also gives the README's speed claim a number.

### Phase 1: math stabilisation
1. **Lexer**: hex (`0x`), binary (`0b`), underscore separators, float exponents (`1e3`, `2.5e-4`).
   Files: `compiler/lexer.lu`, `parser.lu`; tests for each form and for overflow at the literal.
2. **Operators** (`//` is the comment marker, so no floor-division operator): `**` (power), bitwise `& | ^ ~ << >>` on ints with the same
   no-implicit-conversion rule. Files: `lexer.lu`, `token.lu`, `parser.lu` (precedence table),
   `emit_llvm.lu`, `luna_rt_ops.lu` (`lvp_*` helpers; use `addof`/`mulof`-style checks, error on shift count
   out of 0..63).
3. **`pow`/`**` semantics**: int ** non-negative int stays int with an overflow error; negative exponent
   or any float operand gives float. `0 ** -1` is an error.
4. **NaN/inf policy**: `sqrt(neg)`, `ln(<=0)`, `log10(<=0)`, `asin/acos` outside [-1,1] raise a
   `Runtime Error` (domain error). Float division by zero already errors. Provide `is_nan(x)`,
   `is_inf(x)` for code that needs them, and the literals `NAN`/`INF` are **not** added.
5. **Rounding**: `round(x, digits)`, `floor/ceil/trunc` keep returning int; add `round_to` is not needed.
   Define and document half-away-from-zero (matches today's `round(-2.5)` = -3).
6. **Aggregates**: `min`/`max` accept 1 list/array or 2+ numbers; add `sum`, `product`, `mean`
   (float), `divmod(a, b)` returning `[q, r]`, `gcd`, `lcm`, `isqrt`.
7. **Number formatting**: `to_string(float)` stays short (`%g`, 6 digits) for printing, but add
   `fmt_float(x, digits)` and `fmt_int(n, base)` (and `bin/hex` helpers); `format("{:.2}", x)` specs in phase 2.
8. **Parsing**: `parse_int(s)`/`parse_float(s)` return `null` on bad text (strict: whole string,
   optional sign, optional surrounding whitespace). Old `int("..")`, `to_int`, `to_float` keep their
   behavior but are documented as lenient; consider warning in docs only.
9. **Docs/types**: drop `"long"`, document equality rules for int vs float, document the exact
   overflow and domain errors in `docs/builtins.md#math`.

### Phase 2: string stabilisation
1. **Index semantics**: one rule for all of `s[i]`, `char_at`, `substring`, `slice`: negative counts
   from the end, out-of-range read is an `Index Error` for `s[i]`, `""` for the lenient `char_at`/`substring`
   (documented). Fix `substring("abc", -1, 2)` to follow the rule.
2. **Slice syntax**: `s[a:b]`, `s[a:]`, `s[:b]` for strings and lists, same clamping as `slice()`.
3. **`split` modes**: keep today's default (drop empties) under `split(s, d)`, add
   `split_keep(s, d)` (keeps empties) and `lines(s)`; `split(s, "")` stays "characters".
4. **New functions**: `char_code(c)`, `from_code(n)`, `count(s, sub)`, `capitalize`, `title`, `trim_chars(s, set)`,
   `replace_first`, `compare(a, b)` (-1/0/1), `is_upper/is_lower`, `center`, `zfill`.
5. **`format`**: strict argument count (error on too few/many), plus specs `{:5}`, `{:<5}`, `{:>5}`,
   `{:.2}`, `{:x}`; interpolation `"{x:.2}"` reuses them.
6. **`repeat`/`*`**: `"ab" * 3` and `3 * "ab"`; `repeat` stays; floats still an error.
7. **UTF-8 layer**: `chars(s)` (list of UTF-8 characters as strings), `char_count(s)`, `utf8_valid(s)`;
   make `reverse`, `to_upper/to_lower` (Latin-1 range at least) and `pad_*` UTF-8 safe. `len` and `[]` stay byte-based
   and documented as such.
8. **Perf check**: string building in a loop (`s += x`) is O(n^2); either add a `StringBuilder`-style
   `builder()` with `build_add`/`build_str`, or an internal rope/capacity optimisation. Decide after
   the Phase 0 benchmark.

### Phase 3: lists polish (prerequisite for arrays)
1. Add `insert(l, i, v)`, `pop(l)` / `pop(l, i)`, `extend(l, other)`, `copy(l)`, `clear(l)`, `reverse_in_place`
   (or make `reverse` in-place for lists with a documented name, e.g. `reversed` for copy), `unique`,
   `any/all`, `count`, `zip`, `enumerate`, `sort(l, cmp)` and `sort_by(l, key)`.
2. **Operator clarity**: `list + list` becomes **concatenation**; element-wise math moves to
   `vec_add` and to arrays (`array<float> + array<float>`). This is a breaking change: grep tests/docs
   for `[..] + [..]` and `vec_mul` operator uses; record deliberately.
3. List equality and ordering documented (`==` deep, `<` not allowed).

### Phase 4: the `array` type
1. **Runtime representation** in `compiler/runtime/luna_rt_core.lu`: new tag 17 `array`; object =
   header {length, elem_kind, data ptr}, data allocated *atomic* in the GC (not scanned). Reuse the dense
   list allocation path in `luna_rt_math.lu` as the starting point, generalised by element kind.
2. **Constructors and builtins** (`luna_rt_data.lu` or a new `luna_rt_array.lu`): `array`, `byte`,
   `fill`, `to_list`, `copy`, `sum/min/max` extensions; add names to `compiler/shared.lu` BUILTINS and
   `docs/builtins.md`.
3. **Operators and indexing** in `luna_rt_ops.lu`: get/set with bounds check and element-type check,
   `for-in`, `==`, `len`, printing and `to_string`, and element-wise `+ - * /` for numeric arrays of
   equal length (length mismatch is an error instead of "shorter wins").
4. **Compiler**: `array(int, 5)` needs the type name as an argument: accept the bare identifiers
   `int float bool byte char` in that position (parser special-case or pass strings first, e.g.
   `array("int", 5)`, and add sugar later). Update `emit_llvm.lu` only if sugar is chosen.
5. **GC**: arrays of non-pointer elements are atomic, so no change to marking; add them to
   `test_gc_safety`/stress runs.
6. **Migration**: `dense_list` -> alias for `array(float, n, fill)`; keep `d[...]` printing readable by
   switching to `array<float>[...]` only after the tests are re-recorded.
7. **Tests**: `test/test_arrays.lu` (construct, index, out-of-range, wrong element type, negative index,
   length mismatch on math, GC stress, nested arrays in a list, passing to closures).

### Phase 5: basic types and cleanup
- `byte` conversion and `type(v)` names finalised: `int float bool char string null list map array function ...`.
- Unify `char` vs 1-char string friction: allow `==` between a char and a 1-char string, or keep the
  error and document it (decision needed; default: keep the error).
- Consider a `set` (string/int keys, on top of the map implementation) and tuple-style
  multiple return only after the above is stable.
- Update `README.md`, `docs/data-types.md`, `docs/builtins.md`, `docs/architecture.md` (tag table),
  `CLAUDE.md`, and `discussion.md`. Run `make seed` once at the end of each phase.

## 4. Risks
- **Seeds**: every compiler/runtime edit needs `make` then `make seed`; a bad seed breaks bootstrap. Commit
  per sub-step so any seed can be rolled back with git.
- **Self-hosting subset**: the compiler source must not use the new operators until a stage that already
  supports them is seeded (use them in compiler code only in a *later* commit).
- **Breaking changes**: `list + list`, `pow` int result, NaN to error, `type()` `"long"`. All need
  recorded-test updates; review each `git diff test/expected` by hand.
- **Error-message contract**: new errors should copy the style of existing ones (class, message, hint).
- **Performance**: bounds/type checks per write; benchmark against Phase 0.

## 5. Suggested order of work
Phase 0 -> Phase 1 (items 1-4 first) -> Phase 2 (items 1-3, 6) -> Phase 3 -> Phase 4 -> remaining items.
The first useful milestone is "Phase 0 + Phase 1.1-1.4": numeric literals, the missing operators and a
sane NaN/inf policy.

## 6. Open questions for the owner
1. Is "arrays" a fixed-size typed `array` as above, or do you mean making `list` itself the array?
2. Is changing `list + list` to concatenation acceptable (breaking)?
3. Should domain errors (`sqrt(-1)`) stop the program, or return NaN as most languages do?
4. Literal sugar `int[5]` / `[1, 2, 3] as int` versus plain function calls `array(int, 5)`?
5. UTF-8: keep byte semantics as the default, or make `len`/indexing character-based (slower, O(n))?
