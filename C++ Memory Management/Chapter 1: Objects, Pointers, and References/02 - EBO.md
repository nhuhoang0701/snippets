Let me work through each case systematically. I'll assume a typical 64-bit platform where `int` is 4 bytes and alignment/padding follows the standard rules.

---

## Case 1: Simple empty base (EBO applies)

```cpp
struct Empty {};
struct D : Empty { int x; };
```

**With EBO** (what compilers actually do):

| Offset | Size | Content |
|--------|------|---------|
| 0 | 4 | `int x` |
| 4 | — | *end* (sizeof(D) == 4) |

The `Empty` base subobject occupies **0 bytes**. It shares `D`'s address (offset 0) — the base "overlaps" with the start of `D`, but since it has no data, nothing is actually there.

**Without EBO** (the wasteful default the rules would produce):

| Offset | Size | Content |
|--------|------|---------|
| 0 | 1 | `Empty` base subobject |
| 1 | 3 | *padding* (alignment of `int`) |
| 4 | 4 | `int x` |
| 8 | — | *end* (sizeof(D) == 8) |

The `Empty` base forces 1 byte, then 3 bytes of padding to align `x` to a 4-byte boundary. **Double the size.**

---

## Case 2: Two different empty bases (EBO applies to both)

```cpp
struct Empty1 {};
struct Empty2 {};
struct D : Empty1, Empty2 { int x; };
```

**With EBO:**

| Offset | Size | Content |
|--------|------|---------|
| 0 | 4 | `int x` |
| 4 | — | *end* (sizeof(D) == 4) |

Both `Empty1` and `Empty2` have zero size. They are **different types**, so the standard does *not* require them to have distinct addresses — it's perfectly fine for both to share `D`'s address at offset 0.

**Without EBO:**

| Offset | Size | Content |
|--------|------|---------|
| 0 | 1 | `Empty1` base |
| 1 | 1 | `Empty2` base |
| 2 | 2 | *padding* (alignment of `int`) |
| 4 | 4 | `int x` |
| 8 | — | *end* (sizeof(D) == 8) |

---

## Case 3: Non-empty base (no EBO needed)

```cpp
struct Base { int a; };
struct D : Base { int b; };
```

| Offset | Size | Content |
|--------|------|---------|
| 0 | 4 | `Base::int a` |
| 4 | 4 | `D::int b` |
| 8 | — | *end* (sizeof(D) == 8) |

No magic here — the base has real data, so it naturally occupies its bytes. No overhead, no padding, no EBO needed.

---

## Case 4: Two indirect empty bases of the same type

```cpp
struct Empty {};
struct D1 : Empty {};
struct D2 : Empty {};
struct D : D1, D2 {};
```

This is the interesting one. `D1` and `D2` are **different types**, so their subobjects can share an address. But each contains an `Empty` subobject, and those two `Empty` subobjects are the **same type** — the standard requires them to have **distinct addresses**.

| Offset | Size | Content |
|--------|------|---------|
| 0 | 1 | `D1` subobject (contains `Empty` at offset 0) |
| 1 | 1 | `D2` subobject (contains `Empty` at offset 0 within D2) |
| 2 | — | *end* (sizeof(D) == 2) |

What's happening:
- `D1` and `D2` can each be zero-sized *as base subobjects of D*, but their contained `Empty` subobjects **can't share the same address** in `D`'s memory.
- The compiler must ensure `D1::Empty` and `D2::Empty` live at different offsets. So even with EBO, it's forced to allocate at least 1 byte for one (or both) of the intermediate bases to separate the `Empty` subobjects.
- Both `D1` and `D2` are themselves empty as complete objects (sizeof == 1), but as base subobjects of `D`, the same-type `Empty` constraint forces real storage.

This is the key nuance: **EBO can eliminate the size of an empty base, but it cannot violate the distinct-address rule for same-type subobjects.** That rule is absolute, and it's what prevents full optimization in this diamond-like case.
