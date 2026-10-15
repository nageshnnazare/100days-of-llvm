# Day 7 Extra: Deep Dive into DCE and ADCE

Companion to [Day-07-0-DCE-and-ADCE.md](Day-07-0-DCE-and-ADCE.md).

---

## Diagram

```text
   DCE (no CFG needed)              ADCE (needs the CFG)

   %a = add ...   <--- no uses       entry
   delete %a                          |
                                      br
                                     / \
                                 live   dead block
                                 ret    (no path to a
                                         live instruction)
                                         |
                                         delete the block
                                         once its control
                                         dependence is dead
```

DCE deletes an instruction whose result nobody reads and that has no side effect. It will not delete a store, and it will not delete a branch. ADCE starts from instructions that are obviously live (stores, returns, calls) and marks everything they are control-dependent on. Code that cannot reach a live instruction goes away.

## 1. `isInstructionTriviallyDead`

File: [llvm/lib/Transforms/Utils/Local.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/Local.cpp).

The function is the shared definition so that InstCombine, DCE, and SimplifyCFG do not drift apart. Conceptually:

```text
if instruction has non-debug uses: return false
if instruction is a terminator: return false
if instruction may have side effects: return false
return true
```

"Side effects" is the subtle part. `Instruction::mayHaveSideEffects` returns true for volatile memory, for stores, for calls that are not proven side-effect free, and for a few others (`va_arg`, fences, catchpads). A load from an alloca is not a side effect. A load from an unknown pointer is not a side effect either, *unless* it is volatile: a regular load may be deleted if its value is unused, because a load without a use cannot affect later observable behavior unless it could trap in a way the language cares about. LLVM's model allows deleting non-volatile loads that have no uses. It does not allow deleting them if they might have been the only thing keeping a page mapped in a language that treats loads as observable. C and C++ do not treat ordinary loads as observable. So unused loads are dead. This surprises people who think "loads might segfault". In LLVM IR, a load of a poison pointer is already undefined behavior if executed; deleting a load that nobody uses is still done, and a load that *is* used cannot be deleted by DCE anyway.

Calls: `wouldInstructionBeTriviallyDead` consults attributes. `readnone` plus `willreturn` plus no operand bundles that have side effects means the call may be deleted when unused. `readonly` is not enough to *delete* the call if the function might not return (it could loop). `willreturn` is the termination promise. You will see both attributes in real IR; both matter.

`RecursivelyDeleteTriviallyDeadInstructions(I)` pushes `I` and then the newly-dead operands. It is a depth-first DCE from a single root. When InstCombine replaces an instruction, it calls this on the old instruction rather than running the whole-function DCE pass.

## 2. `DCE.cpp`

[llvm/lib/Transforms/Scalar/DCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DCE.cpp) is a worklist over the function:

- Seed the worklist with instructions that are already trivially dead.
- When one is erased, push its operands.

It also removes dead phi cycles in some versions: a cycle of phis that only feed each other and never feed a live instruction. A plain "no uses" test is not enough for a cycle, because each phi uses the other. Look for a function named like `DCEInstruction` and a second pass over dead phi loops. If the version you are reading handles phi cycles, that is the one interesting piece of `DCE.cpp`. Everything else is the worklist.

The pass preserves the CFG and the dominator tree. It only deletes instructions. Analyses that depend on instruction identity (MemorySSA, SCEV) are invalidated when memory operations or SCEV-relevant instructions disappear. Returning `PreservedAnalyses::none()` is safe. Preserving `DominatorTreeAnalysis` is also fine if the CFG did not change. Look at what the pass actually returns; do not assume.

## 3. `ADCE.cpp`

[llvm/lib/Transforms/Scalar/ADCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/ADCE.cpp).

Class `AggressiveDeadCodeElimination`.

**`performDeadCodeElimination`** is the driver.

**`markLive(Instruction *)`** inserts into the live set and worklist.

**Seed liveness.** Iterate every instruction. If it is always live (`isAlwaysLive`, which wraps side effects, returns, and calls the pass is not allowed to delete), mark it.

**Worklist.** Pop a live instruction.

- For each operand, mark the defining instruction live.
- Determine the control dependence of this instruction's block: which branches decide whether the block executes? The implementation uses the post-dominator tree. For each block, the post-dominance frontier tells you the blocks that have an edge into the region this block post-dominates. Those blocks' terminators are marked live.

The code builds, up front, a map from each block to the set of blocks that are control-dependent on it, or it walks the frontier on demand. Either way the information source is `PostDominatorTree`.

**Delete.** Any instruction not in the live set and not a terminator is erased. Terminators that are not live get replaced. A conditional branch whose condition was unmarked becomes a branch to the successor that post-dominates, or the dead successor is dropped. Read `updateDeadRegions` (or whatever the current name is; search for `BranchInst` in the file) before you describe this casually. The exact rewrite has changed as bugfixes landed around infinite loops.

**`mustprogress` / loops.** Search the file for `mayContainIrreducibleControl` and `mustprogress`. An infinite loop with no side effect is not "trivially dead" when removing it would let later code run. ADCE's job is not LoopDeletion, but it must not destroy the loop in a way that changes termination. The tests in `llvm/test/Transforms/ADCE/` include loops for this reason. Read one of them.

## 4. Control dependence, computed once

For a CFG edge `A -> B`, block `Y` is control-dependent on `A` when `B` is a post-dominator of `Y` (or `Y` is `B`) and `A` is not a post-dominator of `Y`. Intuitively the decision at `A` is the last decision that determines whether `Y` runs.

ADCE does not need the full academic definition in your head if you can answer: "if I delete this branch, could a live instruction stop running, or start running when it should not?" If yes, the branch is live.

Example:

```text
entry
  | \
  |  dead_arm
  | /
 live_ret
```

`live_ret` post-dominates `entry` if both arms reach it and it has the return. The branch does not control whether the return executes, only whether `dead_arm` executes. `dead_arm` contains no live instruction. The branch is not live. ADCE rewrites it.

If `dead_arm` contains a `store` to a global, the store is live, `dead_arm` is control-dependent on the branch, and the branch stays.

## 5. Interaction with `unreachable`

A block ending in `unreachable` is not a return. Instructions that only feed the `unreachable` may be dead, but the `unreachable` marker means "control must not arrive here". ADCE's deletion of a path that leads to `unreachable` is shared territory with SimplifyCFG. If your experiment does not match your prediction, run the passes separately:

```bash
opt -passes=adce -S in.ll -o one.ll
opt -passes=simplifycfg -S one.ll -o two.ll
```

Attribute the diff to the pass that produced it.

## 6. Analyses and preservation

ADCE requires `PostDominatorTreeAnalysis`. Changing terminators invalidates the post-dom tree, the dominator tree, and loop info. The pass should not preserve them unless it updates them. In practice ADCE returns a preserved set that does **not** include the CFG analyses when it changed branches. Check the `PreservedAnalyses` return at the bottom of `run`. This is the Day 0 contract applied to a real pass.

## 7. Debug

```bash
opt -passes=adce -print-changed=quiet -S in.ll
opt -passes=dce -print-changed=quiet -S in.ll
```

`-debug-only=adce` exists in asserts builds and prints which instructions were marked live. That dump is the fastest way to see "the call kept the whole function alive".

## 8. What not to copy into your own pass

Do not reimplement liveness. If you have just made one instruction dead, call `RecursivelyDeleteTriviallyDeadInstructions`. If you have rewritten the CFG and want a sweep, schedule `adce` after your pass in the pipeline rather than calling it on the side. The pass manager will invalidate analyses correctly only if `adce` runs as a pass.
