# Question: Why must the alignment of a composite type match the strictest (largest) alignment of its members?

# Answer

The alignment of a composite type must match the strictest (largest) alignment of its members for a straightforward reason: **the struct must be safely accessible wherever it is placed in memory.**

Consider a struct with a `double` (8-byte alignment) and a `char` (1-byte alignment). If the struct itself only had 1-byte alignment, it could be placed at an address like `0x1001` — which is not 8-byte aligned. That means the `double` member would also sit at a misaligned address, causing:

- **Performance penalties** on hardware that handles misaligned access slowly (e.g., x86 performs the access but with overhead).
- **Hardware faults** on architectures that outright prohibit misaligned access (e.g., ARM, SPARC).

By giving the struct the alignment of its most restrictive member (8 bytes in this case), the language guarantees that every member is also properly aligned, no matter where the struct is placed. This is also why **padding** is inserted between and after members — it ensures each member lands at an address satisfying its own alignment, while the struct's overall size is a multiple of its alignment so that arrays of the struct remain correctly aligned.

In short: the composite's alignment is the **least common multiple** of its members' alignments because that is the *minimum* alignment that satisfies *every* member's requirement simultaneously.

## Proof: Composite Alignment Must Equal the Strictest Member Alignment

### Definitions

Let a composite type $S$ have $n$ members $m_1, m_2, \dots, m_n$ with alignments $a_1, a_2, \dots, a_n$ respectively. Without loss of generality, let $a_1 \leq a_2 \leq \cdots \leq a_n$ (so $a_n$ is the strictest/maximum alignment).

Let $A$ denote the alignment of $S$ itself.

**Alignment rule (fundamental):** A type with alignment $a$ may only be placed at addresses that are multiples of $a$.

---

### Part 1: $A \geq a_n$ (the composite alignment must be *at least* the maximum)

**By contradiction.** Suppose $A < a_n$.

Then $S$ could be placed at address $p = A$ (which is a valid address for $S$, since $A \mid A$).

Member $m_n$ resides at offset $o_n$ from $p$, so its address is:
$$p_n = p + o_n = A + o_n$$

The C++ memory model requires that $a_n \mid p_n$, i.e.:
$$a_n \mid (A + o_n)$$

However, this must hold for **every** valid placement of $S$. Consider two placements: address $p_1 = A$ and address $p_2 = 2A$. Then:

$$a_n \mid (A + o_n) \quad \text{and} \quad a_n \mid (2A + o_n)$$

Subtracting:
$$a_n \mid A$$

But we assumed $A < a_n$ and $A \geq 1$, so $a_n \nmid A$. **Contradiction.** $\blacksquare$

---

### Part 2: $A \leq a_n$ (the composite alignment need not *exceed* the maximum)

We show that $A = a_n$ suffices by constructing a valid layout.

**Construction:** If $A = a_n$, we can place $S$ at any address $p$ where $a_n \mid p$. For every member $m_i$:

- The compiler inserts padding such that $a_i \mid o_i$ (each member's offset is a multiple of its own alignment).
- The member's address is $p_i = p + o_i$.

Since $a_n \mid p$ and $a_i \mid o_i$ and $a_i \mid a_n$ (because $a_i \leq a_n$ and alignments are powers of 2):

$$a_n \mid p \implies a_i \mid p \quad (\text{since } a_i \mid a_n)$$

Therefore:
$$a_i \mid (p + o_i)$$

So every member is correctly aligned. The total size of $S$ is padded to a multiple of $a_n$, ensuring arrays of $S$ also respect alignment. $\blacksquare$

---

### Conclusion

From Part 1 and Part 2:

$$\boxed{A = a_n = \max(a_1, a_2, \dots, a_n)}$$

The composite type's alignment equals the maximum of its members' alignments. A smaller alignment would violate member alignment guarantees; a larger one is unnecessary and wasteful.

> **Key insight:** This proof hinges on two facts — (1) alignments in C++ are powers of two, so smaller alignments always divide larger ones, and (2) the composite must be validly placed at *any* aligned address, which forces its alignment to divide every member's address for all valid base addresses.
