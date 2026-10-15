# Day 7: Dead Code Elimination (DCE and ADCE)

Once values are in SSA, a lot of instructions exist only because some earlier form needed them. Dead code elimination deletes instructions that cannot affect the observable behavior of the program. LLVM has two passes because "dead" has two useful meanings, and they are not the same cost.

---

## 1. Why this exists

After InstCombine folds `x - x` into `0`, the subtract's operands may themselves be unused. After a branch is folded to unconditional, an entire arm may be unreachable. Something has to take out the trash, or every later analysis pays for instructions that do not matter.

Observability in LLVM is not "printed to stdout" only. A dead instruction is one whose removal cannot change:

- the values that reach `ret`, stores, volatile operations, and calls that may have side effects,
- the control that decides whether those operations run,
- infinite loops, when the function is not allowed to be assumed to terminate.

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

## 2. The idea

**DCE** (`dce`) is local and cheap. An instruction is *trivially dead* when it has no uses and no side effects. Delete it. Deleting it may make its operands trivially dead. Repeat. DCE does not understand control flow. A branch is not trivially dead, even if both targets do the same thing.

**ADCE** (`adce`, aggressive DCE) walks backward from operations that are observably live. Everything not reached by that walk is deleted, including branches that only controlled dead code. ADCE can delete a diamond of pure arithmetic. It can also rewrite a conditional branch into an unconditional branch when one side is dead.

```llvm
define i32 @f(i32 %x) {
  %dead = mul i32 %x, 500
  %live = add i32 %x, 1
  ret i32 %live
}
```

Both passes delete `%dead`. Only ADCE helps here:

```llvm
define i32 @g(i1 %c, i32 %x) {
entry:
  br i1 %c, label %t, label %f
t:
  %a = mul i32 %x, 3
  br label %j
f:
  br label %j
j:
  ret i32 %x
}
```

`%a` has no uses. DCE deletes `%a` and leaves the branch. ADCE sees that the branch does not control any live operation and redirects `entry` straight at the return, after which SimplifyCFG can delete the empty blocks. (ADCE itself rewrites terminators; empty-block cleanup is still SimplifyCFG's specialty.)

## 3. Where it sits in the pipeline

| Pass | Name | File |
| --- | --- | --- |
| DCE | `dce` | [llvm/lib/Transforms/Scalar/DCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DCE.cpp) |
| ADCE | `adce` | [llvm/lib/Transforms/Scalar/ADCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/ADCE.cpp) |

`isInstructionTriviallyDead` lives in [llvm/lib/Transforms/Utils/Local.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/Local.cpp) and is used by DCE, InstCombine, and many other passes. It is the one-line version of "can I delete this?".

The `-O2` pipeline uses both ideas in more than one place: InstCombine deletes trivially dead instructions as it goes, a DCE run sweeps leftovers, and ADCE runs where control flow has been simplified enough for a backward liveness walk to pay off. BDCE (Day 13) is a third variant that deletes instructions whose *bits* are all dead even when the value has uses.

```bash
opt -passes=dce -S in.ll
opt -passes=adce -S in.ll
```

ADCE requests the post-dominator tree (`PostDominatorTreeAnalysis`). DCE needs no analysis.

## 4. Core concepts

### Trivially dead

`isInstructionTriviallyDead(I)` is true when:

- `I` has no uses, or its only uses are things that will also disappear (debug uses are handled carefully so dropping debug info is allowed and dropping a real use is not), and
- `I` does not write memory, does not throw in a way the caller must observe, is not volatile, is not a terminator, and is not a call that may have side effects.

`mayHaveSideEffects()` on `Instruction` is the coarse test. A `call` to a function marked `readnone` and `willreturn` can be trivially dead. A `call` to an unknown function cannot, even if the return value is unused, because the callee might store to a global.

Debug intrinsics (`dbg.value`) do not count as uses for this purpose. Deleting an instruction may require deleting or undef-ing the debug uses. The helper does that bookkeeping. Day 98 is about what you lose.

### Live sets in ADCE

ADCE assumes an instruction is dead unless proved live.

Always live:

- `ret`
- stores that may be observable, including volatile loads and stores
- calls that are not both side-effect-free and will-return
- instructions that may unwind if that unwind is observable

Then the worklist grows:

- If an instruction is live, its operands are live.
- If a live instruction is control-dependent on a branch, that branch is live, and therefore the branch's condition is live.

Control dependence comes from the **post-dominator frontier** (Day 4). A block that does not post-dominate its predecessor's other path is controlled by that predecessor's branch. Marking the branch live marks the instructions that compute the condition.

When the fixed point is done, unmarked instructions are deleted. Unmarked branches are replaced with unconditional branches to the unique live successor (or the pass deletes the dead region). The implementation in `ADCE.cpp` uses a `DenseSet` of live instructions and a worklist.

### `willreturn` and infinite loops

Deleting a loop that has no side effects is not always legal. In C, a loop with no side effect may be assumed to terminate (`-ffinite-loops` / the IR `mustprogress` attribute on functions). In other languages it may be a legal infinite loop, and deleting it would let the code after the loop run when it should spin.

ADCE respects `mayHaveSideEffects` and the `willreturn` attribute. LoopDeletion (Day 25) is the pass that actually removes loop structure; ADCE removes dead instructions and dead control. If you see an empty infinite loop survive, check for `mustprogress` on the function before you call ADCE broken.

### Unreachable code

A block with no predecessors is not the same as a dead branch. `removeUnreachableBlocks` in `Local.cpp` deletes blocks not reachable from the entry. ADCE and SimplifyCFG both call utilities in that family. DCE does not: an unreachable block's instructions may still "have uses" inside the block, and DCE does not reason about reachability.

## 5. The algorithms side by side

**DCE**

1. Put every instruction in a worklist (or scan and push the dead ones).
2. Pop `I`. If `isInstructionTriviallyDead(I)`, remember its operands, erase `I`, push the operands.
3. Repeat until empty.

No CFG. No post-dom. Linear in the number of deleted instructions plus the uses you touch.

**ADCE**

1. Build (or request) the post-dominator tree.
2. Mark root-live instructions.
3. Worklist: mark operands and controlling branches.
4. Delete unmarked non-terminator instructions.
5. Rewrite unmarked terminators so the CFG no longer enters dead regions.
6. Drop blocks that became unreachable.

ADCE is still cheap relative to GVN, but it is not the thing you want to run between every InstCombine fold. The pipeline places it at a few fixed points.

## 6. Where to read the code

1. `isInstructionTriviallyDead` and `RecursivelyDeleteTriviallyDeadInstructions` in [llvm/lib/Transforms/Utils/Local.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/Local.cpp). The second function *is* DCE on one instruction's fan-in. InstCombine calls it constantly.
2. [llvm/lib/Transforms/Scalar/DCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DCE.cpp) — the pass, short.
3. [llvm/lib/Transforms/Scalar/ADCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/ADCE.cpp) — `AggressiveDeadCodeElimination::performDeadCodeElimination` and `markLive`.
4. Post-dominators from Day 4, used here as control dependence.

## 7. Worked example

```c
int live_only(int x, int c) {
  int dead = x * 500;
  int y = x + 1;
  if (c) {
    int also_dead = y * 3;
    (void)also_dead;
  }
  return y;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes live.c -o live.ll
opt -passes=mem2reg,dce -S live.ll -o live.dce.ll
opt -passes=mem2reg,adce,simplifycfg -S live.ll -o live.adce.ll
```

After mem2reg+DCE, the multiply by 500 is gone, and the multiply by 3 is gone, but the `if (c)` may still be there as an empty diamond. After ADCE+SimplifyCFG, the branch should be gone and the function is `add` then `ret`.

If the empty diamond survives DCE, that is the pass working as specified, not a bug.

## 8. Reading order

1. `isInstructionTriviallyDead` in `Local.cpp`.
2. `DCE.cpp` end to end. It is small.
3. `markLive` in `ADCE.cpp`.
4. The worked example's three IR files.

## 9. Hands-on

```bash
opt -passes=mem2reg,instcombine,dce -S live.ll
opt -passes=mem2reg,instcombine,adce,simplifycfg -S live.ll
```

Try a call:

```c
int ext(int);
int keep(int x) {
  int d = ext(x);
  return x;
}
```

`d` is unused, but `ext` may have side effects. Neither DCE nor ADCE may delete the call. Mark it `__attribute__((const))` (Clang emits `readnone`, with the usual caveats about how Clang spells memory effects today) and rerun. A `readnone willreturn` call whose value is unused is dead.

## 10. Pitfalls and invariants

- **DCE will not delete a store.** A store has side effects even when you cannot see a later load in this function. The store might be to a global, or to a pointer the caller observes. Dead *store* elimination is a different pass (`dse`, Day 37's neighbor), and it needs alias information.
- **ADCE will not delete a volatile load.** Volatile is a side effect by definition.
- **Deleting a phi input requires deleting or rewriting the phi.** The utilities do this. If you hand-delete instructions, you will break phis.
- **`unreachable` is not dead code in the DCE sense.** It is a terminator that marks undefined behavior if executed. ADCE's treatment of blocks that unconditionally lead to `unreachable` is subtle: those blocks are not "live" because of a return, and folding them is often SimplifyCFG's `speculativelyExecute` / unreachable folding, not DCE's.
- **Debug uses must be stripped before the "has no uses" test succeeds.** If you write your own deleter and forget `SalvageDebugInfo` or the equivalent, you either crash the verifier or you keep dead code alive via `dbg.value`.

## 11. Neighbors

| Day | Relationship |
| --- | --- |
| Day 2 InstCombine | Calls `RecursivelyDeleteTriviallyDeadInstructions` after each fold |
| Day 4 | Post-dominance frontier supplies ADCE's control dependence |
| Day 13 BDCE | Dead *bits*, not dead values |
| Day 18 SimplifyCFG | Deletes the empty blocks ADCE leaves behind |
| Day 25 LoopDeletion | Deletes side-effect-free loops as loops, which is stronger than "this instruction is dead" |
| Day 46 GlobalDCE | Dead *functions and globals*, whole-module mark-and-sweep |

## 12. Check yourself

**An unused `add` is removed by which pass?**
DCE, or InstCombine's trivial-dead cleanup. ADCE would also remove it. You do not need ADCE for this case.

**An unused `add` inside a branch, with the branch otherwise empty, needs which pass to delete the branch?**
ADCE (then SimplifyCFG to drop empty blocks).

**Why is a call to an unknown function live even when its return value is unused?**
The callee may store, throw, or not return. Unused return value is not unused side effect.

**What analysis does DCE require?**
None.

**What does ADCE use the post-dominator tree for?**
To find which branches control live instructions (control dependence), so those branches stay live.

> [!TIP]
> The live-marking walk is in [Day-07-1-Extra-DCE-DeepDive.md](Day-07-1-Extra-DCE-DeepDive.md). Cases that look dead and are not are in [Day-07-2-Extra-DCE-Examples.md](Day-07-2-Extra-DCE-Examples.md).
