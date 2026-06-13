## `has_unique_object_representations_v<T>`

This is a C++17 type trait that answers one question: **"Does every byte of T's storage matter for its value?"**

Formally, it's `true` when:
1. `T` is **trivially copyable**, *and*
2. Any two objects of type `T` with the **same value** have the **same object representation** (byte-for-byte identical).

### The key distinction: value representation vs. object representation

Every object has two "representations":

- **Object representation**: the entire `sizeof(T)` bytes of storage — the raw memory footprint.
- **Value representation**: the subset of bits that actually determine the object's value.

They differ because of **padding** and **redundant encodings**:

```cpp
struct T {
    char c;   // 1 byte
    int  i;   // 4 bytes
};            // sizeof(T) == 8 (3 padding bytes inserted for alignment)
```
Those 3 padding bytes are part of the *object representation* but not the *value representation*. Two `T` objects with `c='a'` and `i=42` might have different padding byte values but are still equal — same value, different byte pattern. That means `has_unique_object_representations_v<T>` is **false**.

### What fails and what passes

| Type | Result | Why |
|---|---|---|
| `int`, `unsigned int` | ✅ true | No padding bits (on typical implementations) |
| `struct { char c; int i; }` | ❌ false | Padding between members |
| `float` | ❌ false* | IEEE-754 has +0 and −0 (same value, different bytes) |
| `struct { int a; int b; }` | ✅ true | No padding, each byte matters |
| Types with `[[no_unique_address]]` members | ❌ false | Empty members may or may not occupy storage |

*\*Implementation-defined — an implementation not using IEEE-754 could make it true.*

### Why does this exist? **Hashing.**

The primary motivation is **generic hashing**. If you want to hash a trivially copyable object by reading its bytes:

```cpp
template<typename T>
size_t hash(const T& obj) {
    auto bytes = reinterpret_cast<const unsigned char*>(&obj);
    return hash_bytes(bytes, sizeof(T));
}
```
This only produces **correct, consistent** results if the byte representation is uniquely determined by the value. If padding bytes are involved, two equal objects could hash differently. If `float`'s ±0 issue exists, two equal values hash differently. `has_unique_object_representations` tells you when this shortcut is safe.

That's why it **subsumes `is_trivially_copyable`** as a prerequisite — if the type isn't trivially copyable, its byte representation isn't even a reliable starting point. The trait is a *refinement* of trivially copyable: trivially copyable means "bytes define a valid copy"; unique object representations means "bytes uniquely define the value."

### In one line

> `has_unique_object_representations_v<T>` is `true` when hashing `T` by simply hashing its raw bytes is guaranteed to be correct.
