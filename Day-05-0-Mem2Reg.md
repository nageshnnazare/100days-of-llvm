# Day 5: Mem2Reg (Promote Memory to Register)

Mem2Reg is the pass that turns Clang's naive stack slots into SSA. Before it runs, a local `int` is an `alloca`, a `store`, and a `load`. After it runs, that `int` is a value, and joins in the CFG are `phi` nodes. Almost every later optimization is weak on the load/store form and strong on the SSA form. That is why `mem2reg` (or SROA, which calls the same promoter) is near the front of the pipeline.

---

## 1. Why this exists

Frontends should not have to place phis. Placing phis correctly means computing iterated dominance frontiers, and it is easy to get the incoming blocks wrong. The contract is:

- Clang emits an `alloca` in the entry block for each local, stores on assignments, and loads on uses.
- `mem2reg` rebuilds SSA for the allocas that are simple enough.
- If an alloca is not simple (its address escapes, it is volatile, it is a variable-sized stack object), it stays in memory and the backend gives it a real stack slot.

The algorithm is the classical one from Cytron, Ferrante, Rosen, Wegman, and Zadeck, "An Efficient Method of Computing Static Single Assignment Form" (1991), specialized to the case where each "variable" is one alloca.

## Diagram

```text
   entry:  %y = alloca i32
           /            \
        then            else
     store 1, %y     store 2, %y
           \            /
            join:  %t = load i32, ptr %y
                         |
                         v
   after mem2reg:

   entry
     /        \
   then       else
     \        /
      join:  %y = phi i32 [ 1, %then ], [ 2, %else ]
```

The alloca is a stack slot only because the frontend did not want to place phis. Mem2Reg puts a phi at the join, which is the iterated dominance frontier of the stores, then deletes the alloca. A load in a block dominated by exactly one store becomes that stored value and needs no phi.

## 2. The idea

For one alloca:

1. Find every store (a definition) and every load (a use).
2. If there is a single store that dominates every load, replace the loads with the stored value. No phi.
3. Otherwise place phis at the iterated dominance frontier of the store-blocks, then rename: walk the dominator tree, keep a stack of "current value", and rewrite loads to the value on top of the stack.

After rewriting, the alloca has no uses and is deleted.

```c
int f(int c, int x) {
  int y;
  if (c) y = x; else y = 0;
  return y;
}
```

Before:

```llvm
entry:
  %y = alloca i32
  br i1 %c, label %then, label %else
then:
  store i32 %x, ptr %y
  br label %join
else:
  store i32 0, ptr %y
  br label %join
join:
  %v = load i32, ptr %y
  ret i32 %v
```

After `mem2reg`:

```llvm
entry:
  br i1 %c, label %then, label %else
then:
  br label %join
else:
  br label %join
join:
  %y = phi i32 [ %x, %then ], [ 0, %else ]
  ret i32 %y
```

## 3. Where it sits in the pipeline

Pass name: `mem2reg`. The pass class is `PromotePass` in [llvm/lib/Transforms/Utils/Mem2Reg.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/Mem2Reg.cpp). The algorithm is `PromoteMemToReg` in [llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp).

In a modern `-O1`/`-O2` pipeline, **SROA** (Day 6) runs first and promotes the scalar pieces itself by calling `PromoteMemToReg`. You still see `mem2reg` as a standalone pass in tests and in unoptimized-to-SSA experiments. Clang `-O0` does not run it: that is why `-O0` IR is full of allocas, and why debug info at `-O0` can point `dbg.declare` at those allocas.

```bash
opt -passes=mem2reg -S in.ll
```

Analyses it uses: `DominatorTreeAnalysis` and `AssumptionCache`. It preserves the dominator tree when promotion does not change the CFG, which it normally does not: mem2reg inserts phis and deletes loads and stores, but it does not add or remove edges.

## 4. Core concepts

### What is promotable

`isAllocaPromotable` answers this. An alloca can be promoted when:

- It is in the entry block (so it executes exactly once; a dynamic `alloca` in a loop is a different object every iteration).
- Every use is a load or a store of the alloca's allocated type, or a lifetime intrinsic (`llvm.lifetime.start` / `end`), or a debug intrinsic.
- Loads and stores are the full size of the alloca, not a byte of a 4-byte integer.
- Nothing takes the address and stores it somewhere, passes it to a call, or uses it in a `getelementptr` that escapes. Those are **captures**. The address must not escape.
- Volatile operations block promotion. An atomic load/store of the whole object can be promoted in some cases; do not assume all atomics block it, and do not assume none do. Read `isAllocaPromotable` if you hit one.

Aggregates (structs, arrays) usually fail this test because accesses are GEPs into the alloca. That is SROA's job: break the aggregate into scalar allocas, then call the promoter.

### Two fast paths

`PromoteMemToReg` does not always build frontiers.

- **Single block.** All loads and stores are in the alloca's block. Walk the block in order and replace each load with the last store's value. This is the common case for a temporary that does not span a branch.
- **Single store.** One store dominates all loads, and there is no store that only happens on some paths in a way that leaves a load uninitialized. Replace loads with that value. If a load is not dominated by the store, the load is a use of an uninitialized variable; LLVM inserts `undef` / poison handling rather than inventing a zero. Be careful: modern LLVM prefers `poison` for "this value was never written" in many new code paths, but mem2reg's historical behavior for an uninitialized load is an `undef` of the loaded type. Check the current `PromoteMemoryToRegister.cpp` if you need the exact constant kind; the *observable* fact is that an uninitialized load becomes a constant the rest of the pipeline treats as "no bits are known".

The general path is the one to study.

### Phi placement

For each alloca, collect the blocks that contain a store. Those are the defining blocks. Compute their iterated dominance frontier (Day 4). Insert an empty phi of the alloca's type in each frontier block that is live: a block where the value may be used without being defined again first. LLVM prunes this with a liveness walk (`ComputeLiveInBlocks`) so a phi is not inserted in a region that never loads the alloca.

The new phis count as definitions too. The IDF calculator iterates, so a phi placed at a join can force another phi further down.

### Renaming

Walk the dominator tree in preorder. Keep a stack per alloca.

- On a store, push the stored value.
- On a load, replace the load with the current top of stack (the reaching definition).
- On a phi in this block, the phi *is* the reaching definition for the rest of the block; push it.
- When leaving across an edge to a successor, if the successor has a phi for this alloca, fill in the incoming value for this predecessor with the current top of stack.
- When the DFS backtracks, pop the values this block pushed.

Because the walk is the dominator tree, when you are in a block the stack top is exactly the definition that dominates the block. That is why the algorithm is correct and linear in the size of the function plus the number of phis.

### Debug info

A `dbg.declare` of the alloca is not a real use for promotion, but deleting the alloca would drop the variable. The promoter converts declares into `dbg.value` instructions at each store (and at phis). That conversion is why `-O1` debug info is harder than `-O0`. Day 98 covers the result. The code calls into `ConvertDebugDeclareToDebugValue` helpers in `llvm/lib/Transforms/Utils/Local.cpp`.

## 5. The algorithm as `PromoteMem2Reg` runs it

Class `PromoteMem2Reg` inside `PromoteMemoryToRegister.cpp`:

1. `run()` filters the alloca list.
2. For each alloca, gather `UsingBlocks` (loads) and `DefiningBlocks` (stores).
3. Try the single-store and single-block shortcuts (`rewriteSingleStoreAlloca`, a local walk).
4. Otherwise `ComputeLiveInBlocks`, then place phis with `IDFCalculator`, then `RenamePass` over the dominator tree.
5. Delete the alloca.

`PromoteMemToReg(Allocas, DT, AC)` is the function SROA and `PromotePass` both call. `AssumptionCache` lets the promoter use `llvm.assume` when reasoning about whether a bitcast or a lifetime marker matters; the core algorithm does not depend on it.

## 6. Where to read the code

1. [llvm/include/llvm/Transforms/Utils/PromoteMemToReg.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/PromoteMemToReg.h) — the entry point you would call from your own pass.
2. [llvm/lib/Transforms/Utils/Mem2Reg.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/Mem2Reg.cpp) — `PromotePass::run`, which scans the entry block for promotable allocas.
3. [llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp) — `isAllocaPromotable`, `PromoteMem2Reg::run`, `RenamePass`. This is the whole day.
4. [llvm/include/llvm/Analysis/IteratedDominanceFrontier.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/IteratedDominanceFrontier.h) — phi locations.

Search the `.cpp` for `RenamePass` and read that function with the stack picture from section 4 next to you.

## 7. Worked example

```c
int running(int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += i;
  return s;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes run.c -o run.ll
opt -passes=mem2reg -S run.ll -o run.m2r.ll
```

In `run.m2r.ll` you should find a loop header with two phis, one for `s` and one for `i`, each with an incoming value from the preheader (or entry) and an incoming value from the latch. The `alloca`s for `s` and `i` are gone. Loads and stores of them are gone.

The blocks did not change. Only the instructions inside them did. That is the signature of this pass: CFG-neutral, value-rewriting.

## 8. Reading order

1. Section 4 of this file, with a diamond drawn on paper.
2. `isAllocaPromotable` — it is the contract.
3. `RenamePass` — it is the algorithm.
4. A before/after on your own function.
5. The deep dive for the fast paths and debug info.

## 9. Hands-on

```bash
opt -passes=mem2reg -S run.ll -o run.m2r.ll
opt -passes=mem2reg -print-changed=quiet -S run.ll
```

Force a refusal:

```c
int escapes(int *out) {
  int x = 4;
  *out = x;
  return x;
}
```

That one *does* promote `x`, because `out` is a different pointer. This one does not:

```c
void take(int *p);
int no(void) {
  int x = 4;
  take(&x);
  return x;
}
```

`take(&x)` captures the address. `opt -passes=mem2reg` leaves the alloca. SROA also leaves it. The store and the load stay until the inliner can see `take` and prove it does not capture.

## 10. Pitfalls and invariants

- **Promotion does not zero uninitialized variables.** A load with no reaching store becomes `undef` (or, in newer LLVM, a poison-like placeholder — confirm in `RenamePass` if you are writing a tool that matches bit-for-bit). That is different from C's "my compiler zeroed the stack" accident.
- **Mem2Reg will not split aggregates.** `alloca {i32, i32}` accessed through GEPs is SROA's input, not mem2reg's. If your experiment "doesn't work", check whether the uses are GEPs.
- **It requires a dominator tree that matches the CFG.** Stale DT means phis in the wrong blocks and a verifier crash, or silent miscompile if verification is off.
- **Phi incoming values must be filled for every predecessor**, including predecessors that do not store. Those predecessors contribute the stack's incoming value (the definition that dominates the predecessor).
- **`optnone` skips the pass.** See [Setup-and-Toolchain.md](Setup-and-Toolchain.md).
- **Large promotable arrays are still a bad idea.** `isAllocaPromotable` may say yes for an array that is only loaded and stored as a whole, but a 1 MB alloca promoted into SSA is a disaster. SROA's size limits are the practical guard. Mem2Reg itself is eager.

## 11. Neighbors

| Day | Relationship |
| --- | --- |
| Day 4 | IDF and the rename walk's dominator DFS |
| Day 6 SROA | Splits aggregates, then calls `PromoteMemToReg` |
| Day 7 DCE | After promotion, dead stores are already gone; DCE cleans arithmetic that fed only those stores |
| Day 18 SimplifyCFG | Phis with identical inputs, or a diamond that is now a `select` |
| Day 98 | `dbg.declare` becomes `dbg.value` here |

## 12. Check yourself

**Why does the rename walk follow the dominator tree and not reverse postorder?**
The stack top must be the dominating definition. The dominator tree guarantees that when you enter a block, every dominator has already been processed and is still on the stack. RPO does not guarantee that for siblings.

**A store in `left` and a store in `right`, loads only in `join`. Where is the phi?**
In `join`, the dominance frontier of both stores.

**Why can't mem2reg promote `alloca i32` whose address is passed to `scanf`?**
The callee can store through the pointer at any time, including after the promoter would have "finished". There is no single SSA value.

**Does mem2reg change the CFG?**
No. It inserts phis and deletes loads, stores, and the alloca. Edges stay.

**SROA and mem2reg both exist. Which one does `-O2` actually rely on?**
SROA. It slices aggregates and then calls the same `PromoteMemToReg` utility. `mem2reg` remains the name of that utility's standalone pass.

> [!TIP]
> The rename stack and the promotability predicate are walked line by line in [Day-05-1-Extra-Mem2Reg-DeepDive.md](Day-05-1-Extra-Mem2Reg-DeepDive.md). Cases that should and should not promote are in [Day-05-2-Extra-Mem2Reg-Examples.md](Day-05-2-Extra-Mem2Reg-Examples.md).
