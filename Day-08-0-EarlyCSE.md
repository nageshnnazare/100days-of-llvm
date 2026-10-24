# Day 8: Early CSE

Common subexpression elimination removes a computation that has already been done. EarlyCSE is the cheap version that runs near the front of the pipeline, before GVN, using a scoped hash table rather than a full value-numbering analysis. It exists because many duplicates are obvious once SROA has turned memory into values, and waiting for GVN would leave them in the IR through several other passes.

---

## 1. Why this exists

```llvm
%a = add i32 %x, %y
%b = add i32 %x, %y
%c = mul i32 %a, %b
```

`%b` is the same value as `%a`. Nothing about the CFG makes this hard. A hash of `(opcode, %x, %y)` finds it. EarlyCSE does that, and it also forwards loads through stores to the same address when it can prove nothing else wrote in between.

GVN (Day 11) catches the cases EarlyCSE misses: loads separated by complex control flow, partial redundancy, and memory dependencies that need MemorySSA. EarlyCSE catches the easy cases quickly and repeatedly. The pipeline runs it several times.

## Diagram

```text
   entry
     %x = add %a, %b          hash(%a,%b,add) -> %x
     |
     +-- then --+
     |  %y = add %a, %b       same hash -> reuse %x
     |                        delete %y
     +-- else --+
        %z = add %a, %b       NOT the same scope if this
                              block is not dominated by
                              the first add... it is, via
                              entry, so %z folds too

   a store to %p followed by a load of %p, with no
   intervening write, forwards the stored value
```

EarlyCSE keeps a scoped hash table that pushes a scope when it enters a block and pops when it leaves. A value computed in a block is available in blocks it dominates. The MemorySSA mode can forward a load across a call that does not alias; the plain mode is more conservative.

## 2. The idea

Walk each block once, in dominator-tree order, keeping a hash table of available values. The table is **scoped**: when the walk enters a child block it pushes a scope, and when it returns it pops the scope. A value computed in a block is available in blocks it dominates, and not elsewhere. That is exactly the region where it is legal to reuse the value.

For memory, the simple implementation keeps a generation counter. A store to an unknown address bumps the generation, which invalidates available loads. A store to a known address can still forward to a later load of that address in the same generation.

There is a second mode, `early-cse-memssa`, that uses MemorySSA instead of the generation counter. The default pipeline uses the MemorySSA version once MemorySSA is being built anyway, because it forwards loads across stores that alias analysis can prove are independent. The hash-table CSE of pure arithmetic is the same in both modes.

## 3. Where it sits in the pipeline

| `opt` name | Class | Memory |
| --- | --- | --- |
| `early-cse` | `EarlyCSEPass` | generation counter |
| `early-cse-memssa` | `EarlyCSEPass` with MemorySSA | MemorySSA |

File: [llvm/lib/Transforms/Scalar/EarlyCSE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/EarlyCSE.cpp).

It needs `DominatorTreeAnalysis`, `TargetLibraryInfo`, and `AssumptionCache`. The MemorySSA mode also needs `MemorySSAAnalysis` and `AAManager`.

It runs after the first SROA and InstCombine, and again later. It does not rewrite the CFG.

```bash
opt -passes=early-cse -S in.ll
opt -passes=early-cse-memssa -S in.ll
```

## 4. Core concepts

### `SimpleValue` hashing

Two instructions get the same hash when they are the same operation on the same operands, with compatible flags. `add %x, %y` matches `add %y, %x` because `add` is commutative; the hasher sorts operands. `sub` does not match with swapped operands.

The hash includes the opcode, the operand pointers, the comparison predicate for `icmp`, and the type. It ignores instruction names. Names are not semantics.

`EarlyCSE` can also CSE loads of the same address when the memory generation matches, and it can CSE `readnone` calls.

### Scoped hash table

`ScopedHashTable` in `llvm/include/llvm/ADT/ScopedHashTable.h` is the data structure. `StackNode` in `EarlyCSE.cpp` pairs a scope with the block being processed and the current memory generation.

Because the walk is a DFS of the dominator tree:

- When we are in block B, the stack contains exactly the blocks that dominate B.
- An expression computed in a dominator is still in the hash table.
- An expression computed in an uncle block was pushed in a scope that has already been popped.

That is the dominance condition for CSE, implemented by stack discipline instead of by querying the dominator tree on every lookup. The tree decides the walk order; the stack does the availability.

### Memory generations

In the non-MemorySSA mode:

- Each store, call that may write, or other memory clobber increments a generation or records a last store.
- A load CSEs with a previous load of the same pointer only if the generation has not changed.
- A load of an address that was just stored, and whose value is known, is replaced by the stored value (store-to-load forwarding).
- A store of the same value to the same address as the previous store, with nothing in between, is a dead store and can be removed. EarlyCSE does a little DSE, not the full DeadStoreElimination pass.

This mode is deliberately coarse. A store to `%p` invalidates a load of `%q` even when `%p` and `%q` cannot alias. That is why `early-cse-memssa` exists.

### MemorySSA mode

MemorySSA (Day 37) gives each load a clobbering definition. Two loads of the same pointer that share the same MemorySSA defining access are the same value, even if a store to a *different* pointer sits between them in the block. EarlyCSE in this mode hashes the load together with its defining MemorySSA access.

The pass still uses the scoped hash table for arithmetic.

### Assumptions and guards

`llvm.assume(i1 %c)` tells later instructions that `%c` is true. EarlyCSE records assumes in the scope so that, for example, a later comparison that is implied by the assume can be folded. It also understands that `assume(x == y)` makes `x` and `y` interchangeable for hashing in that scope. This is best-effort. GVN and InstCombine do more with assumes.

### What it will not CSE

- Instructions with side effects, except the specific memory cases above.
- Values that are not available on every path to the use. There is no partial redundancy elimination here. If `%a = add` happens in the left arm only, a use in the join cannot reuse it without a phi. EarlyCSE does not insert phis. GVN's PRE pass does, for loads.
- Floating-point operations that are not fast-math-flagged, when the flags would make two operations differ (`nnan`, `reassoc`). Two `fadd` instructions with the same operands and the same flags can be CSE'd.

## 5. The algorithm

`EarlyCSE::run()`:

1. DFS the dominator tree from the entry.
2. At each block, push a `StackNode` (new hash scope, inherit memory state from the parent).
3. For each instruction, in order:
   - If it is a simple value and the hash table already has a leader, `replaceAllUsesWith` the leader and delete the instruction.
   - If it is a load and forwarding or load-CSE applies, replace it.
   - Otherwise insert it into the hash table if it is a pure value.
   - If it clobbers memory, update the generation or the MemorySSA state.
4. Pop the scope at the end of the block's dominated region.

Leaders are the first instruction that produced a given hash. Later duplicates point at the leader. The leader must dominate the duplicate, which the scoped walk guarantees.

## 6. Where to read the code

1. [llvm/lib/Transforms/Scalar/EarlyCSE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/EarlyCSE.cpp) — `SimpleValue::getHashValue` and `isEqual`, then `processNode`.
2. [llvm/include/llvm/ADT/ScopedHashTable.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/ADT/ScopedHashTable.h) — the scope mechanism, short and worth reading fully.
3. MemorySSA mode's load handling in the same `EarlyCSE.cpp`, searching for `MemorySSA`.

## 7. Worked example

```c
int twice(int x, int y) {
  int a = x + y;
  int b = y + x;
  return a * b;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes twice.c -o twice.ll
opt -passes=mem2reg,early-cse -S twice.ll
```

**Expected:** one `add`, and the multiply uses it twice (`mul i32 %a, %a`). Commutativity is why `y + x` matches `x + y`.

Store forwarding:

```llvm
define i32 @fwd(ptr %p, i32 %v) {
  store i32 %v, ptr %p
  %a = load i32, ptr %p
  ret i32 %a
}
```

```bash
opt -passes=early-cse -S fwd.ll
```

**Expected:** `ret i32 %v`. The load is gone.

## 8. Reading order

1. The scoped-hash picture in section 4.
2. `SimpleValue` equality in `EarlyCSE.cpp`.
3. `processNode` for the load cases.
4. Examples, including a case GVN gets and EarlyCSE does not.

## 9. Hands-on

```bash
opt -passes=early-cse -S twice.ll
opt -passes=early-cse-memssa -S alias.ll   # example 4 in the examples file
opt -passes=gvn -S alias.ll
```

Diff the last two. The difference is the MemorySSA-aware (or GVN) handling of a store that cannot alias.

## 10. Pitfalls and invariants

- **EarlyCSE does not insert phis.** If your duplicate is on another path, this pass will not create a join. That is a job for GVN PRE or for you writing the phi yourself.
- **The generation-counter mode is pessimistic about aliasing.** Do not conclude "LLVM cannot forward this load" until you have tried `early-cse-memssa` and `gvn`.
- **Flags matter.** `add nsw` and `add` without `nsw` are not the same value. CSE must not replace one with the other, because `nsw` changes poison. The hasher includes the flags that affect poison and rounding.
- **It is not a code-size pass.** CSE almost always shrinks code. The profitability question is boring here, unlike inlining or unrolling.
- **Leaders must stay.** If a later pass deletes the leader and leaves the duplicates already rewritten, that is fine because the duplicates now *are* the leader (uses were moved). If you delete an instruction that is still in the hash table mid-walk, you have a bug. The pass deletes only the duplicate, not the leader.

## 11. Neighbors

| Day | Relationship |
| --- | --- |
| Day 2 InstCombine | Folds identities; EarlyCSE folds duplicates. They alternate in the pipeline |
| Day 6 SROA | Creates the scalar duplicates EarlyCSE then sees |
| Day 11 GVN | Slower, stronger, includes load PRE |
| Day 37 MemorySSA | The precise memory state for `early-cse-memssa` |
| Day 76 MachineCSE | The same idea after instruction selection |

## 12. Check yourself

**Why is the hash table scoped by the dominator tree?**
A value may be reused only where its definition dominates the use. The scope stack mirrors the tree walk, so availability matches dominance.

**Does EarlyCSE remove a load that is redundant only on one predecessor of a phi?**
No. That is partial redundancy. EarlyCSE does not place phis or split edges.

**Why might `early-cse` miss a load CSE that `early-cse-memssa` finds?**
A store to a pointer that cannot alias sits between the loads. The generation counter treats every store as a clobber. MemorySSA does not.

**Are `add %x, %y` and `add %y, %x` the same to EarlyCSE?**
Yes. `add` is commutative and the hasher sorts operands.

**A `call` marked `readnone willreturn` with the same arguments appears twice. Can EarlyCSE keep one?**
Yes, if nothing about the attributes differs. If the call is not `readnone`, no.

> [!TIP]
> Hashing, scopes, and the memory generation counter are in [Day-08-1-Extra-EarlyCSE-DeepDive.md](Day-08-1-Extra-EarlyCSE-DeepDive.md). Side-by-side cases, including one that needs GVN, are in [Day-08-2-Extra-EarlyCSE-Examples.md](Day-08-2-Extra-EarlyCSE-Examples.md).
