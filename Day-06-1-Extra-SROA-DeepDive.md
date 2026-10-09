# Day 6 Extra: Deep Dive into SROA.cpp

Companion to [Day-06-0-SROA.md](Day-06-0-SROA.md). The file is [llvm/lib/Transforms/Scalar/SROA.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SROA.cpp). It is several thousand lines. Read it in the order below, not from line 1.

---

## Diagram

```text
   alloca { i32, i32 }          SROA slices
   +--------+--------+          +--------+  +--------+
   | field0 | field1 |   --->   | alloca |  | alloca |
   | 4 bytes| 4 bytes|          | i32    |  | i32    |
   +--------+--------+          +--------+  +--------+
        ^        ^                   |           |
   gep 0,0   gep 0,1            mem2reg      mem2reg
                                     |           |
                                     v           v
                                   SSA %a0     SSA %a1
```

SROA does not promote a struct in one step. It partitions the alloca by the offsets that loads, stores, and GEPs actually touch, rewrites those uses onto the new slices, then calls the same promoter as mem2reg. A slice whose address escapes stays memory.

## 1. Pass entry

`SROAPass::run(Function &, FunctionAnalysisManager &)`:

- Bails out if the function has the `optnone` attribute (the pass manager also skips `optnone` functions for most pipelines, but the pass checks too).
- Collects entry-block allocas. Non-entry allocas are not candidates, same rule as mem2reg, with the same reason: an alloca outside the entry may execute more than once.
- For each candidate, calls the per-alloca driver.
- Tracks whether anything changed, for `PreservedAnalyses`.

Options on the pass (see the header) include whether CFG preservation is required. When CFG preservation is set, rewrites that would need to split blocks or speculate across new control flow are refused. The default pipeline allows the usual speculation of pointer phis.

## 2. `AllocaSlices` and `Slice`

Search for `struct Slice` and `class AllocaSlices`.

A slice stores:

- begin and end offsets (64-bit, because allocas can theoretically be large; the pass also has a size cutoff and will refuse enormous allocas),
- a pointer to the using instruction,
- a flag marking whether this slice is a **splittable** tail of a wider access.

`AllocaSlices` owns the vector of slices, sorted. It knows whether the alloca escaped during the build (`isEscaped()`).

The partition iterator walks the sorted slices and yields groups. Reading `partitions()` is more useful than reading the sort comparator. The comparator's job is only "fixed, non-overlapping ranges fall out".

## 3. `SliceBuilder`

This is an inst-visitor that records slices. It is the predicate "can SROA see through this use?"

Typical visits:

- **Load / store.** The pointer operand must be the alloca or a constant-offset GEP of it. The size is the store size from the data layout (`DL.getTypeStoreSize`). Record `[offset, offset+size)`. Volatile: the builder usually marks the alloca unsplittable or escaped, depending on the path. Look at the `isVolatile` branch rather than assuming.
- **GEP.** If all indices are constant, compute the byte offset (`DL.getIndexedOffsetInType` or the GEP helper the file uses) and continue into the users of the GEP, carrying the offset. If an index is not constant, escape.
- **Lifetime intrinsics.** Ignored for partitioning, or recorded as covering the whole object. They must not block promotion.
- **`memcpy` / `memmove` / `memset`.** Recorded as a range of the length operand when the length is constant. Variable-length memcpy escapes.
- **Call.** Unless it is one of the recognized intrinsics, the pointer escaping into the call marks the alloca escaped.
- **Phi / select of the pointer.** Not an immediate escape. The builder tries to follow all incoming values. If every incoming pointer is a constant-offset address inside this same alloca, SROA can speculate the phi. If one incoming value is "some other pointer", escape.

`insertUse` is the function that either appends a slice or sets the escaped flag. When you are debugging "why didn't SROA fire?", this is the function that decided.

## 4. Splitting

Before rewriting, overlapping slices get pre-split. Search for `presplit` or `splitAlloca`.

The idea: a store of `i64` at offset 0 and a load of `i32` at offset 4 overlap badly if the alloca is two `i32`s. The wide store is rewritten into two `i32` stores (or a store plus a shift sequence) so that each partition has clean edges.

Integer splitting uses shifts and truncs. Endianness comes from `DataLayout::isLittleEndian()`. Getting this wrong is a miscompile, which is why SROA tests exist for both endiannesses and why you should not reimplement the splitter as an exercise against production bitcode without those tests.

Accesses that cannot be split (volatile, a too-wide atomic, a call) cause the overlapping partition to stay whole or cause the alloca to be abandoned. The pass prefers "give up" over "tear a volatile store".

## 5. `AllocaSliceRewriter`

An `InstVisitor` constructed for one partition. It receives the new alloca (or the SSA value) that represents that partition and rewrites every slice in the partition.

`visitLoadInst`:

- If the load is exactly the partition type, replace uses with a load of the new alloca, or with the current SSA value if the partition was already promoted in the rewriter's internal state.
- If the load is a slice of a wider integer partition, emit a shift and a trunc.

`visitStoreInst` is the mirror image: a store of a narrow value into a wide integer becomes a masked insert (load, and, or, store) or, once the value is in SSA, an `and`/`or` on values. After promotion those memory operations should themselves be gone.

`visitMemTransferInst` rewrites a memcpy that covers this partition into a load/store pair of the partition type, or into a memcpy of the new smaller alloca if the other side is still memory.

GEPs that only existed to compute the address are left with no uses and deleted afterwards. The rewriter does not always delete them itself; a dead-code sweep at the end of the alloca does.

## 6. Promotion

Search for `PromoteMemToReg` in `SROA.cpp`. After rewriting, SROA has a set of new allocas that satisfy `isAllocaPromotable`. It calls the Day 5 utility on that set with the dominator tree it already requested.

This is why the output of `opt -passes=sroa` looks like mem2reg output. The slicing is internal. If promotion fails, you will see the new, smaller allocas in the output, which is a useful debugging state: slicing worked, promotion did not.

## 7. Pointer-phi speculation

Search for `isSafeSelectToSpeculate` and the phi equivalent. The pattern:

```llvm
%p = phi ptr [ %px, %left ], [ %py, %right ]
%v = load i32, ptr %p
```

where `%px` and `%py` are both GEPs into the same alloca. SROA rewrites this into a load of `x` and a load of `y` (which then become SSA values) and a `phi` or `select` of those values. The load is no longer a load.

This is legal when the load is not volatile and when speculating both sides cannot introduce a side effect. It is one of the few places SROA "moves" computation. It still does not change the CFG.

## 8. Limits you will find as `cl::opt`s

Near the top of the file there are command-line options: a maximum alloca size, a switch for forcing SSA form, options related to splitting. Read the `cl::opt` declarations. They document the intended limits better than a blog post, and they show you how to override them while debugging.

Do not memorize the default numbers. They move. Do know that "SROA refused a 100 KB local array" is the size cap working as designed. Promoting that array into SSA would explode compile time and register pressure.

## 9. What SROA does not use

- No alias analysis. Two different allocas never alias, by IR rules, so AA is unnecessary for the accesses SROA cares about. The question is only whether *this* alloca's address escapes.
- No ScalarEvolution. Variable indices are a hard no, not a "maybe if the trip count is 4".
- No cost model beyond size thresholds. Unlike the vectorizer, SROA does not ask TTI whether a split is profitable on this CPU. Getting values into registers is treated as always profitable under the size cap.

## 10. Debugging recipe

Asserts build:

```bash
opt -passes=sroa -debug-only=sroa -S in.ll
```

Look for the alloca pointer and then either "rewriting" or a reason the builder aborted.

No asserts build:

```bash
opt -passes=sroa -print-changed=quiet -S in.ll
```

If nothing changed, the alloca escaped or was over the size limit or was not in the entry. Print the function and classify each use of the alloca by hand against section 3.

## 11. Tests

`llvm/test/Transforms/SROA/` is large and excellent. Pick:

- a test whose name mentions `struct` or `split-aggregate`,
- a test that expects SROA to fail (`CHECK: alloca` remains),
- a `phi` speculation test.

Reading three tests is more useful than reading another thousand lines of the rewriter.
