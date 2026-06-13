# Question: Why must the alignment of a composite type match the strictest (largest) alignment of its members?

# Answer

The alignment of a composite type must match the strictest (largest) alignment of its members for a straightforward reason: **the struct must be safely accessible wherever it is placed in memory.**

Consider a struct with a `double` (8-byte alignment) and a `char` (1-byte alignment). If the struct itself only had 1-byte alignment, it could be placed at an address like `0x1001` — which is not 8-byte aligned. That means the `double` member would also sit at a misaligned address, causing:

- **Performance penalties** on hardware that handles misaligned access slowly (e.g., x86 performs the access but with overhead).
- **Hardware faults** on architectures that outright prohibit misaligned access (e.g., ARM, SPARC).

By giving the struct the alignment of its most restrictive member (8 bytes in this case), the language guarantees that every member is also properly aligned, no matter where the struct is placed. This is also why **padding** is inserted between and after members — it ensures each member lands at an address satisfying its own alignment, while the struct's overall size is a multiple of its alignment so that arrays of the struct remain correctly aligned.

In short: the composite's alignment is the **least common multiple** of its members' alignments because that is the *minimum* alignment that satisfies *every* member's requirement simultaneously.
