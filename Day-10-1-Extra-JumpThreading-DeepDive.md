# Day 10 Extra: Deep Dive into JumpThreading.cpp

Companion to [Day-10-0-JumpThreading.md](Day-10-0-JumpThreading.md). File: [llvm/lib/Transforms/Scalar/JumpThreading.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/JumpThreading.cpp).

---

## Diagram

```text
        entry: %c = ...
          |
          br i1 %c
         /        \
      then         else
      %c is true   %c is false
        |            |
        br i1 %c     br i1 %c
       /    \          |
     T(dead) F        F is the only
                      possible target

   after jump threading, `else` branches
   straight to F. The dead edge is gone.
```

Threading copies a destination onto a predecessor when that predecessor already knows how the branch will go. It does not merge blocks; that is SimplifyCFG. Each threaded edge has a cost, and `-jump-threading-threshold` stops the pass from cloning a large block.

## 1. Driver

`JumpThreadingPass::run` gets the function and the analysis manager. It constructs a `JumpThreading` implementation object (the pass class and the worker have been split before; search for `runImpl`) and loops over blocks.

The loop is not a single walk. Threading one edge changes predecessors of later blocks, which can create a new opportunity. The pass uses a worklist of blocks to revisit, with a cap so a pathological function cannot pin the compiler. Find the cap (`BBDupThreshold` and a separate iteration limit) near the `cl::opt`s at the top of the file.

`BBDupThreshold` is the maximum size, in LLVM's internal cost units, of a block the pass is willing to duplicate. The cost is not "number of IR lines". Calls cost more than adds. `TargetTransformInfo` may influence the cost. The command-line flag `-jump-threading-threshold` sets the cap.

## 2. `ProcessBlock`

Given block `BB`:

- Look at the terminator. `BranchInst` with a condition, or `SwitchInst`. Unconditional branches are SimplifyCFG's problem.
- Ignore blocks that are too big to duplicate when duplication would be required. The check sometimes happens later, inside `ThreadEdge`, so a cheap thread that does not duplicate still proceeds.
- Call `ComputeValueKnownInPredecessors` on the condition (or on the switch value).
- For each predecessor with a known destination, try `ThreadEdge`.

`ProcessBlock` also calls helpers that unfold selects (`TryToUnfoldSelect`) and that thread through a block whose condition is an `and`/`or` of a known fact (`ProcessImpliedCondition` or similar). These helpers exist because real IR rarely has a phi of `i1` sitting alone; it has a chain of compares. Unfolding makes the chain look like the simple case.

## 3. `ComputeValueKnownInPredecessors`

This is a recursive, memoized query. Given a value `V` and a block `BB`, it returns a list of `(predecessor, constant)` pairs describing what `V` is when `BB` is entered from that predecessor.

Cases it handles, which you should find as a switch on the instruction type:

- **Constant.** Every predecessor sees that constant. Trivial, and usually folded earlier.
- **Phi in `BB`.** The incoming value from predecessor `P` is exactly the answer for `P`. If that incoming value is itself a phi or an instruction, the function recurses, with a depth limit.
- **`ICmp`.** If both sides have known constants on a predecessor, fold the compare for that predecessor. If one side is a constant and LVI knows a range for the other side on that edge, fold when the range proves the predicate.
- **`And` / `Or` / `Xor` of `i1`.** Boolean identities: if either side of an `and` is false on an edge, the `and` is false on that edge.
- **Casts and a few arithmetic identities,** when they are invertible enough to propagate a constant. The pass is not InstCombine; it only propagates what it needs to decide a branch.

`LazyValueInfo::getPredicateOnEdge` and `getConstantOnEdge` are the LVI entry points. LVI's lattice is `undefined`, `constant`, `constantrange`, `overdefined`. Jump threading treats "constant" as actionable and "overdefined" as "do not thread". A range threads only when the *predicate* (the compare), not the value, collapses to a constant. `x` in `[0, 10)` does not tell you `x`'s value, but it does tell you `x < 10` is true.

Recursion depth is limited because a phi cycle would otherwise loop. When the limit hits, the result is "unknown", which is safe.

## 4. `ThreadEdge`

Parameters are the predecessor `Pred`, the block being threaded through `BB`, and the successor `Succ` that this predecessor should go to directly.

Steps you should be able to point at in the source:

1. **Profitability.** If `BB` has a single predecessor, no duplication is needed; just rewire. If `BB` has many, decide whether the instructions that must be cloned fit in the budget. Instructions already available in `Pred` (dominated there) do not need cloning.
2. **Clone or reuse.** `DuplicateInstructionsInSplitBetween` or a local cloner copies the instructions from `BB` that `Succ` and the phi nodes need, into a new block between `Pred` and `Succ`. SSA values defined outside `BB` are used as-is. Values defined in `BB` are remapped to the clones.
3. **Rewire the terminator of `Pred`** so the edge that went to `BB` now goes to the clone (or to `Succ`).
4. **Update phis** in `BB` (that predecessor is no longer incoming) and in `Succ` (the incoming block may now be the clone).
5. **Update the dominator tree** through `DomTreeUpdater` or `DT->applyUpdates`. Also update `LoopInfo` if the pass has it. BPI/BFI, when present, get an update so profile weights move to the new edge instead of being dropped on the floor.

If `BB` loses all predecessors, a later iteration or a call to `DeleteDeadBlock` removes it.

## 5. Unfolding selects

`TryToUnfoldSelect` turns

```llvm
%c = select i1 %cond, i32 %a, i32 %b
%cmp = icmp eq i32 %c, %a
br i1 %cmp, ...
```

into real control flow when doing so lets a predecessor be threaded. Unfolding is itself a code-size increase. The helper is conservative and looks for selects that feed a branch or a phi. It is not a general "selects are bad" pass. The backend and SimplifyCFG turn branches *into* selects when the region is small; jump threading does the opposite when correlation across blocks makes the branch form profitable. Seeing both directions in one pipeline is normal.

## 6. Freeze

Search for `IRBuilder` and `CreateFreeze` or `new FreezeInst`. The situation: LVI says "this value is 1 on this edge", but the value could be poison, and poison compared with `icmp` is also poison, not `true`. Branching on poison is undefined. If the pass duplicates a *side-effecting* region based on that branch, it must freeze the condition so both the original and the clone see a real boolean. Freezing is a register-to-register operation at codegen time (`freeze` lowers to a copy or disappears if the value is known noundef). It is cheap insurance. You will find tests in `llvm/test/Transforms/JumpThreading/` that `CHECK` for `freeze`.

## 7. Loops

Threading an edge into a loop header can rotate or destroy a natural loop. The pass consults `LoopHeaders` or `LoopInfo` and refuses some threads that would add a second entry to a loop. LoopSimplify (Day 21) wants a single header; a pass that casually creates a second entry makes every later loop pass bail out. This refusal is a frequent "why didn't it thread?" answer on loop-heavy IR. `-debug-only=jump-threading` prints the refusal reason.

## 8. Profile metadata

When the terminator has `!prof` branch weights, cloning has to split the weights. The predecessor's probability moves to the new edge, and the original block's weights are recomputed for the remaining predecessors. Getting this wrong does not miscompile; it makes block placement (Day 78) and the inliner lay out code for a fiction. Search for `updateBlockFreqAndEdgeWeight` or `BranchProbabilityInfo` usage in the file.

## 9. Preservation and verification

After a successful thread the CFG changed, so SCEV, MemorySSA, and LVI's internal cache are invalid. LVI is a query analysis that caches; the pass must clear or update that cache when it edits the CFG, otherwise the next block's query sees facts about deleted edges. This is a known sharp edge. The pass calls something like `LVI->eraseBlock` / `threadEdge` on the LVI object. When you read the file, find that call. It is as important as the IR rewrite.

Dominator tree: preserved only if updated. The new PM pass returns `PreservedAnalyses` listing `DominatorTreeAnalysis` when the updater flushed, and omits CFG-sensitive analyses it did not touch only if they truly survived. Read the return statement. Compare it with Day 0's contract.

## 10. Debug

```bash
opt -passes=jump-threading -debug-only=jump-threading -S in.ll
opt -passes=jump-threading -print-changed=quiet -S in.ll
```

The debug stream prints "threading edge" with block names. If nothing prints, either the condition was not known in any predecessor or the threshold refused the clone. Both outcomes are reported if you read the lines rather than only the final IR.
