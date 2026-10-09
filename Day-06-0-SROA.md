# Day 6: SROA (Scalar Replacement of Aggregates)

Mem2Reg only promotes an alloca when every access is a full-size load or store of the whole object. Real C and C++ do not look like that. A `struct`, a small array, a `std::pair`, or a local that the compiler split into pieces is an alloca plus a pile of `getelementptr` offsets. SROA rewrites that memory into scalar SSA values.

The name is historical. The pass also promotes scalars that mem2reg would have handled, and it rewrites "integer bags of bits" that are not really aggregates. In the `-O2` pipeline it is the pass that deletes the allocas you care about.

---

## 1. Why this exists

Consider:

```c
typedef struct { int x; int y; } Point;
int manhattan(Point p) { return p.x + p.y; }
```

Even after inlining, a `Point` parameter passed by value, or a local `Point`, starts life as memory. Loads of `p.x` and `p.y` cannot be constant-folded, GVN'd, or put in registers while they are still loads. Alias analysis can *sometimes* prove the loads are independent. SROA does better: it removes the memory.

SROA is also how Clang's "everything is an alloca at -O0" IR becomes the SSA you see at `-O1`, for objects mem2reg refuses.

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

## 2. The idea

Look at one alloca. Every load, store, and `memcpy` touches a byte range `[begin, end)`.

1. Collect those ranges. Sort them. Split them into **partitions** that do not overlap in a conflicting way.
2. If a partition is a whole, well-typed slice (the `x` field, the `y` field), give that slice its own alloca, or better, rewrite the loads and stores of that slice into SSA values directly.
3. If an access spans two partitions (a `memcpy` of the whole struct, or a store of an `i64` over two `i32`s), split the access.
4. Hand any leftover scalar allocas to `PromoteMemToReg` (Day 5).

For `Point`, the two `i32` fields become two independent values. The alloca disappears.

```llvm
; before
%p = alloca { i32, i32 }, align 4
%px = getelementptr inbounds { i32, i32 }, ptr %p, i32 0, i32 0
store i32 1, ptr %px
%py = getelementptr inbounds { i32, i32 }, ptr %p, i32 0, i32 1
store i32 2, ptr %py
%lx = load i32, ptr %px
%ly = load i32, ptr %py
%s = add i32 %lx, %ly

; after sroa
%s = add i32 1, 2
```

InstCombine then folds that add. SROA's own job ended when the memory disappeared.

## 3. Where it sits in the pipeline

Pass name: `sroa`. Implementation: [llvm/lib/Transforms/Scalar/SROA.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SROA.cpp). Class `SROAPass`.

It runs early, more than once. A typical `-O2` function pipeline runs SROA, then EarlyCSE and InstCombine, then later another SROA after inlining has exposed more allocas. Inlining is the big producer of promotable memory: a by-value struct parameter is an alloca in the callee, and after inlining that alloca is in the caller, where SROA can see every access.

```bash
opt -passes=sroa -S in.ll
```

It uses `DominatorTree` (for the final promotion) and `AssumptionCache`. It can preserve the CFG. It does not preserve MemorySSA or anything that tracks stores, because it deletes stores.

Debug build:

```bash
opt -passes=sroa -debug-only=sroa -S in.ll
```

(`-debug-only` needs an asserts build.)

## 4. Core concepts

### Slices

An access is a `Slice`: an offset range, a pointer to the instruction, and whether it is a split point. Slices are sorted by begin offset, then by end offset. The type that holds them is `AllocaSlices` in `SROA.cpp`.

The alloca size comes from the data layout (`DL.getTypeAllocSize`). Offsets come from walking `getelementptr` constant indices. A GEP with a *variable* index into an array is not a fixed slice. That usually aborts SROA for the whole alloca: if any byte is touched by an unbounded index, the partitioner cannot isolate the rest.

### Partitions

`AllocaSlices::partitions()` groups slices into non-overlapping spans. The goal is a set of ranges where:

- no access partially overlaps the range in a way the rewriter cannot split, and
- each range is as large as a real typed use wants, and as small as the conflicts force.

A store of the whole struct and a load of field 0 conflict until the store is split into field stores. Splitting is allowed for integer loads and stores (an `i64` store becomes two `i32` stores) and for `memcpy` / `memmove` of a constant length. It is not allowed for a volatile access, an atomic access that cannot be torn, or a call that receives the raw pointer.

### Escaping

If any use is not a load, store, GEP with constant indices, a lifetime intrinsic, a debug intrinsic, or a `memcpy`/`memset` the pass understands, the alloca **escapes** and SROA gives up on it. `PtrUseVisitor` (in `llvm/lib/Analysis/PtrUseVisitor.cpp`) is the general "walk all the ways this pointer is used" utility. SROA has its own visitor, `AllocaSlices::SliceBuilder`, which either records a slice or marks the alloca escaped.

Captures include: passing the pointer to an unknown call, storing the pointer to memory, returning the pointer, and `ptrtoint` in the general case.

### Integer widening and vector promotion

Not every partition stays as "the field's natural type".

- Adjacent `i8` loads that are really a flag bag may be merged into a wider integer if that makes the rewrite simpler.
- A small array of `float` with only fixed indices may become a `vector` if the target likes it (`-sroa-force-ssa` / vector thresholds; the decision uses the data layout and a size cap, not a full cost model).

Both are heuristics inside the rewriter. The important invariant is not the heuristic: after a successful run, the original alloca has no uses.

### Promotion at the end

SROA's rewriter often introduces new, simpler allocas and then calls `PromoteMemToReg` on them. So a successful SROA run is "slice, then mem2reg". When you read the IR after `-passes=sroa` you usually do not see the intermediate allocas. They were promoted before the pass returned.

### `alloca`s of scalable vectors and variable size

`alloca i32, i64 %n` is a dynamic stack allocation. Its size is not a constant. SROA cannot partition it. Scalable vector allocas (`<vscale x 4 x i32>`) are similarly special. Leave them for the backend.

## 5. The algorithm

`SROAPass::runOnAlloca` (search the file; the method name has stayed stable) for each entry-block alloca:

1. **Build slices** with `SliceBuilder`. On escape, stop.
2. **Split overlapping accesses** that are splittable, so partitions become clean. This may rewrite a wide store into several narrow stores before partitioning finishes. The implementation iterates because splitting creates new slices.
3. **Rewrite each partition** with `AllocaSliceRewriter`, an `InstVisitor`. Loads become extracts from the new value or loads of a new alloca. Stores become inserts or stores. GEPs are deleted once their users have been rewritten. `memcpy` between two SROA-able allocas becomes copies of the slices.
4. **Promote** new scalar allocas via `PromoteMemToReg`.
5. **Delete** the original alloca if it is unused.

`AllocaSliceRewriter::visitLoadInst` and `visitStoreInst` are the two methods to read after the slice builder.

Speculation: SROA sometimes rewrites `select` of pointers and `phi` of pointers when every incoming pointer is a GEP into the same alloca. That is "pointer phi speculation". It lets a phi of `&p.x` / `&p.y` become a select of the loaded values. If the phi cannot be speculated, the alloca escapes through the phi and SROA stops.

## 6. Where to read the code

1. [llvm/lib/Transforms/Scalar/SROA.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SROA.cpp) — start at `SROAPass::run`, then `SliceBuilder`, then `AllocaSliceRewriter`.
2. [llvm/include/llvm/Transforms/Scalar/SROA.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/SROA.h) — the pass wrapper and options (`PreserveCFG` is a real option: some callers need SROA to avoid splitting blocks).
3. [llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp) — the promotion SROA finishes with.
4. [llvm/lib/Analysis/PtrUseVisitor.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/PtrUseVisitor.cpp) — the escape-style pointer walk SROA's builder is patterned on.

The file is large. Do not read it top to bottom on the first day. Read `SliceBuilder::visit` (how a use becomes a slice or an escape) and one rewriter method.

## 7. Worked example

```c
typedef struct { int x; int y; } Point;

int both(int a) {
  Point p;
  p.x = a;
  p.y = a + 1;
  return p.x + p.y;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes both.c -o both.ll
opt -passes=mem2reg -S both.ll -o both.m2r.ll
opt -passes=sroa -S both.ll -o both.sroa.ll
```

`both.m2r.ll` still has the alloca. `both.sroa.ll` should be an add of `%a` and `%a + 1`, or something InstCombine-ready that contains neither `alloca` nor `getelementptr`.

Then:

```bash
opt -passes=sroa,instcombine -S both.ll
```

Often this is simply `ret i32` of `%a * 2 + 1` or `(a + (a + 1))`. That second step is not SROA. Knowing which pass did which line of the diff is the skill this week is building.

## 8. Reading order

1. The `Point` before/after, until you can draw the slices `[0,4)` and `[4,8)`.
2. `SliceBuilder` in `SROA.cpp`.
3. `visitLoadInst` in the rewriter.
4. The call to `PromoteMemToReg` near the end of rewriting an alloca.
5. Examples, including the escape counterexample.

## 9. Hands-on

```bash
opt -passes=sroa -print-changed=quiet -S both.ll
```

Try a variable index:

```c
int idx(int i) {
  int a[4] = {1, 2, 3, 4};
  return a[i];
}
```

SROA cannot promote `a` in the general case, because `i` is unbounded. You should still see the initializer stores. A later pass might turn the initializer into a memcpy from a constant, which is a different story (LoopIdiom and MemCpyOpt). The point today: a non-constant GEP index is an escape hatch out of SROA.

Try fixed indices `a[0] + a[1] + a[2] + a[3]` with no variable. SROA should scalarize all four elements.

## 10. Pitfalls and invariants

- **SROA is not a license to tear volatile or atomic memory.** Those accesses mark the alloca as something the partitioner must leave alone, or they block splitting.
- **Padding bytes** exist. A `{ i8, i32 }` has padding. Slices must account for the data layout, not the sum of the field sizes. This is why SROA takes `DataLayout` and why a target-independent "3 bytes" guess is wrong.
- **Failed SROA is silent.** The alloca stays. The way to debug it is `-debug-only=sroa` and look for "escape" or "unsplittable".
- **It runs again after inlining.** Judging SROA on un-inlined IR understates it. A getter that returns `p.x` cannot be SROA'd at the call site until the getter is inlined.
- **`memcpy` of a whole local is splittable; a call that *might* write the local is not.** Unknown calls are the usual reason a "obvious" struct stays in memory.
- **The pass can increase IR size briefly** by splitting one wide operation into several, then decrease it by promotion. Do not stop reading the diff at the first new instruction.

## 11. Neighbors

| Day | Relationship |
| --- | --- |
| Day 5 | The promoter SROA calls once slices are scalar |
| Day 8 EarlyCSE | After SROA, redundant scalar arithmetic is visible |
| Day 33 / 37 | Memory dependence and MemorySSA matter for the allocas SROA could not remove |
| Day 42 Inliner | The main source of new SROA opportunities |
| Day 47 GlobalOpt | The global-variable cousin: scalarize globals, not allocas |

## 12. Check yourself

**Why doesn't mem2reg handle `Point`?**
Accesses are GEPs, not loads of the whole alloca. `isAllocaPromotable` rejects them.

**What is a slice?**
One access's constant byte range within one alloca.

**A function calls `void dirty(Point *p)` and passes `&local`. Can SROA scalarize `local`?**
No. The pointer escapes into `dirty`, which can write any byte.

**Why does `-O2` run SROA more than once?**
Inlining between the runs exposes new allocas. One SROA before inlining cannot see into the callee.

**Does SROA change control flow?**
By default it avoids needing to. Phi and select speculation rewrites values. It does not exist to delete branches; SimplifyCFG does that after the values are scalar.

> [!TIP]
> Slice construction and the rewriter are in [Day-06-1-Extra-SROA-DeepDive.md](Day-06-1-Extra-SROA-DeepDive.md). Side-by-side IR is in [Day-06-2-Extra-SROA-Examples.md](Day-06-2-Extra-SROA-Examples.md).
