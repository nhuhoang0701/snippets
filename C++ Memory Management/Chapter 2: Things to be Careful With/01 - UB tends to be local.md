# Question: What does it mean when one says "UB tends to be local"?

Illustration example:
```cpp
int g(int);
int f(int *p) {
    if(p != nullptr)
        return g(*p); // Ok, we know p is not null
    return *p; // oops, if p == nullptr this is UB
}
```

# Answer
This is a classic and surprising consequence of **undefined behavior (UB)** in C++. Here's what it means:

## The Core Logic

The C++ standard says that if a program encounters undefined behavior, the implementation (compiler) can do *anything* — the program has no meaningful behavior from that point onward. Compilers exploit this by **reasoning backwards**:

1. If `p` were `nullptr`, then `return *p` (line 5) would be undefined behavior.
2. The programmer surely didn't intend UB (so the compiler's reasoning goes), therefore `p` must never be `nullptr`.
3. If `p` is never `nullptr`, then the check `if(p != nullptr)` is always true and is **redundant**.
4. The compiler is therefore allowed to **optimize away the null check entirely**, rewriting the function as:

```cpp
int f(int *p) {
    return g(*p);  // null check removed!
}
```

## Why This Matters

This is deeply counterintuitive. You'd expect the null check *protects* part of the code, and only the unchecked access is the bug. But because UB gives the compiler **blank permission** to assume the impossible never happens, the UB "poisons" the entire function — even the parts that *look* safe. The compiler can reason globally: "if this function is ever called with `nullptr`, the program is already undefined, so I can assume it's never called with `nullptr` and simplify accordingly."

The key takeaway: **UB doesn't just mean "the bad line might crash."** It means the compiler is free to restructure your code in ways that can silently remove safety checks you relied on, making the bug *harder* to detect, not easier. This is why UB is so dangerous in C++ — its effects are not always local or obvious.
