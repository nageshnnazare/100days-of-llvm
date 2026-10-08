# Day 5 Extra: Deep Dive into PromoteMemoryToRegister

Companion to [Day-05-0-Mem2Reg.md](Day-05-0-Mem2Reg.md).

The file is [llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp). It is one of the better "read a real pass" files in the tree: a single algorithm, a private class, almost no target hooks.

---

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

## 1. The pass wrapper

[llvm/lib/Transforms/Utils/Mem2Reg.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/Mem2Reg.cpp) is short. `PromotePass::run`:

- Gets `DominatorTree` and `AssumptionCache` from the analysis manager.
- Walks the entry block. Only entry allocas are candidates.
- Calls `isAllocaPromotable` on each.
- Hands the vector to `PromoteMemToReg`.

If you are inserting "make this function SSA" into a custom pipeline, you do not need the pass. Call `PromoteMemToReg` yourself after you have built the allocas.

`isAllocaPromotable` is also declared in `PromoteMemToReg.h` so SROA can ask the same question of the scalar allocas it just created.

## 2. `isAllocaPromotable`

Read this function before `run()`. It walks `AI->uses()`.

Accept:

- `LoadInst` of the allocated type (the load's type equals `AI->getAllocatedType()`), non-volatile. Atomic loads are accepted only when the atomicity is compatible with a register; if you are unsure, look at the `isAtomic` check in the function rather than guessing.
- `StoreInst` whose value operand is the stored value and whose pointer operand is the alloca, same type, non-volatile.
- `llvm.lifetime.start` / `llvm.lifetime.end` on that pointer.
- Debug intrinsics (`dbg.declare`, `dbg.value`, and the debug address variants).

Reject:

- Any other opcode, including `GetElementPtrInst` and `PtrToIntInst`. A GEP means "this object has subobjects", which is SROA's problem.
- A store *of the pointer* (the alloca is the value operand, not the pointer operand). That is a capture.
- A load or store that uses the pointer through a `bitcast` or an `addrspacecast`. Opaque pointers removed pointer bitcasts; a type mismatch now shows up as a load type that differs from the allocated type, which this predicate rejects.

The predicate is local. It does not look at alias analysis. "Is the address captured?" is answered by "is there any use that is not a load, store, lifetime, or debug use?" That is conservative and fast.

## 3. `PromoteMem2Reg` fields

The class holds:

- The list of allocas.
- `DominatorTree &DT`.
- `AssumptionCache &AC`.
- Per-alloca data: the phi nodes it has placed, indexed so the rename walk can find "the phi for alloca A in block B" quickly. The implementation uses a dense map or a vector of maps; read the field declarations at the top of the class rather than assuming a layout.
- A worklist / visited set for the liveness prune.

`AllocaInfo` (search the file) records defining blocks and using blocks per alloca.

## 4. Fast path: one store

`rewriteSingleStoreAlloca`:

- There is exactly one store.
- Every load is dominated by that store. If a load is not dominated, the alloca is uninitialized on that path; the function still rewrites it, using `undef` for the uninitialized loads (search for `UndefValue` in this file to see the current choice).
- Loads are `replaceAllUsesWith`'d with the stored value, then erased.
- The store is erased.
- Debug declares are converted at the store's position.

This path is why a function that assigns a local once and then only reads it becomes straight-line SSA with no phi, even if the reads are inside branches. Dominance is the only thing required.

## 5. Fast path: one block

When every load and store of the alloca is in the same basic block as the alloca (the entry), a single forward scan is enough. There is no join inside the block. The "current value" is a local variable in the scan, not a stack of blocks. Stores update it, loads are replaced by it, and a load that appears before any store becomes `undef`.

This catches the extremely common pattern Clang emits for a temporary that is declared, assigned, and consumed inside a straight-line function body.

## 6. Liveness before phi insertion

`ComputeLiveInBlocks` answers: "in which blocks is this alloca live, meaning a load may see a value defined outside the block?"

A block is live-in for the alloca when there is a use of the alloca in the block (or in a block it can reach) that is not preceded by a store in that same block. The implementation walks predecessors from the using blocks, stopping at defining blocks. Classic dataflow, restricted to one alloca, and cheap because the CFG of one function is small.

Phis are inserted only in live-in blocks that are also in the iterated dominance frontier. A join that is on the frontier but never reaches a load does not get a phi. That keeps the IR smaller and is a visible difference versus a textbook "place phis at the whole IDF" implementation.

## 7. `RenamePass`

This is the function to read slowly.

Signature shape: it takes a basic block and the current incoming values (the vector of "what is on the stack for each alloca").

For each instruction in the block, in order:

- If it is a phi inserted for one of our allocas, record that phi as the current value of that alloca. Phis are at the top, so this happens before loads in the same block. A load in the same block as a phi sees the phi, not the incoming value from "before" the block. That is ordinary SSA.
- If it is a load of a promoted alloca, `replaceAllUsesWith` the current value and queue the load for deletion.
- If it is a store, set the current value to the stored value and queue the store for deletion.

Then for each successor:

- If the successor contains a phi for this alloca, set the incoming value for the current block to the current value. `PHINode::addIncoming` or `setIncomingValueForBlock` does this. The phi was created empty (or with a placeholder) during placement; renaming fills it.

Then recurse to the blocks this block immediately dominates, passing the current values. After the recursive calls return, the current values are discarded by virtue of going out of scope (the "pop"). Implementations usually pass the values as a vector and restore the old contents after each child, which is the explicit stack.

Why immediate dominator children, not CFG successors: CFG successors may be joins you have not finished characterizing, and they may be visited along multiple edges. The dominator children are each visited once. Successor edges are handled only to *fill phi inputs*, not to continue the walk.

## 8. Deleting instructions

The promoter queues dead loads and stores and erases them after the walk, or erases them immediately once uses are gone. Erasing a store immediately is safe because nothing uses a store's value (stores are not values you read; they are `void`). Erasing a load immediately is safe after `replaceAllUsesWith`. The alloca is erased at the end when its use list is empty.

If a use remains, the alloca was not actually promotable and you have a bug in the predicate. Debug builds assert.

Lifetime intrinsics on a promoted alloca are deleted. They only existed to bound the stack slot.

## 9. Debug conversion

Search for `dbg.declare` and `ConvertDebugDeclareToDebugValue` in this file. The idea:

- At each store of value `V`, emit `dbg.value` saying "variable Y is now V".
- At each phi, the variable's value is the phi.

This is lossy when a variable is split or when stores are later optimized away. Day 98's assignment tracking (`dbg.assign`) is the newer attempt to make this less lossy. Mem2Reg is still the moment the alloca, which debug info loved, disappears.

## 10. What this file deliberately does not do

- No alias analysis. Capture is syntactic.
- No splitting of stores into pieces. A store of `i64` to an `alloca i64` is all-or-nothing. A store of `i32` to half of it is not promotable.
- No CFG changes. It will not split critical edges. Phi placement on a critical edge is still just a phi in the successor, which is correct. Critical-edge *splitting* is for passes that need to insert a non-phi on one edge.
- No constant folding. If it inserts a phi of `(1, 1)`, a later InstCombine cleans it up. The promoter stays simple.

## 11. Debug flags

`PromoteMemoryToRegister.cpp` is light on `-debug-only` output compared with SROA. The practical tools:

```bash
opt -passes=mem2reg -print-changed=quiet -S in.ll
opt -passes='print<domtree>' -disable-output in.ll
```

If promotion did nothing, dump uses of the alloca (`opt -S` and read them, or `AI->dump()` in a debugger) and compare each use against `isAllocaPromotable`.

## 12. Tests to read

The regression tests under `llvm/test/Transforms/Mem2Reg/` are small and are the best "expected behavior" document. Open two or three `.ll` files and read the `CHECK` lines. Day 91 explains the FileCheck syntax; you can already read `CHECK-NOT: alloca` without it.
