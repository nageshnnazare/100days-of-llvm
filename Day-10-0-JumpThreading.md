# Day 10: Jump Threading

Jump threading looks at a branch whose condition is already determined on one incoming edge, and rewires that edge to the destination the branch would have picked. The branch stays for the other incoming edges. The effect is fewer branches on the hot path and larger straight-line regions for InstCombine, GVN, and the backend.

It is a CFG pass. It duplicates instructions when it threads through a block that has other predecessors.

---

## 1. Why this exists

```c
int f(int x) {
  int y = 0;
  if (x == 1)
    y = 1;
  if (x == 1)
    return y + 10;
  return y;
}
```

On the path where the first condition is true, `x` is 1, so the second condition is true. The program still contains both tests. Jump threading rewrites the first true-edge so it lands in the `return y + 10` block (or a copy of it), and never evaluates `x == 1` again.

This pattern shows up constantly after inlining. The caller has a test, the inlined callee repeats a related test, and the IR has two branches that are correlated. Threading undoes that.

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

## 2. The idea

While processing a block `B` with a conditional branch:

1. Ask, for each predecessor `P`: is the branch condition a known constant when control arrives from `P`?
2. If predecessor `P` always takes successor `S`, and `P` is not the only predecessor, rewrite the edge `P -> B` into `P -> S`.
3. If `B` contains instructions that `S` needs, and other predecessors still enter `B`, **duplicate** the relevant part of `B` onto the new edge. The duplicated block is `P`'s private copy. The original `B` remains for the other predecessors.

"Known constant" comes from:

- a phi in `B` whose incoming value from `P` is a constant, and the branch compares that phi,
- `LazyValueInfo`, which tracks constant ranges implied by dominating branches (Day 14),
- cheap local folding, such as a compare of two constants.

Threading through a switch works the same way: one incoming edge may imply one case.

## 3. Where it sits in the pipeline

Pass name: `jump-threading`. File: [llvm/lib/Transforms/Scalar/JumpThreading.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/JumpThreading.cpp). Class `JumpThreadingPass`.

It wants `DominatorTree`, `LazyValueInfo` (through the analysis manager; the pass computes LVI queries internally), and target info for the cost of duplicating a block. It preserves analyses only when it did not change the CFG. When it threads, it updates the dominator tree or invalidates it. Look at the pass's use of `DomTreeUpdater`.

The pipeline runs it in the simplification phase, near SimplifyCFG and CVP. They overlap but do not subsume each other:

- SimplifyCFG folds a branch that is **unconditionally** constant.
- CVP folds comparisons and switches using ranges, often without duplicating blocks.
- Jump threading handles the **partial** case: constant on one edge, not on the others, and is willing to duplicate.

```bash
opt -passes=jump-threading -S in.ll
```

Threshold: the pass will not duplicate a large block. `-jump-threading-threshold` is the instruction budget. A block full of calls will not be threaded through.

## 4. Core concepts

### Threadable conditions

The best case is a phi:

```llvm
join:
  %p = phi i1 [ true, %left ], [ %c, %right ]
  br i1 %p, label %t, label %f
```

Arriving from `%left`, `%p` is `true`, so the branch always goes to `%t`. The edge `%left -> join` can target `%t` directly. Arriving from `%right`, `%p` is `%c`, and the branch must remain.

Correlated facts that are not phis need LVI. "In `%left` we branched on `%x == 1`, and this block, dominated by that edge, branches on `%x == 1` again." LVI answers "on this edge, `%x` is the constant 1".

### Duplication

If `%join` contains more than the branch — say an `add` that both successors use — the predecessor that skips `%join` still needs the `add`. The pass clones the instructions into a new block placed on that edge. SSA values are remapped. This is `CloneBasicBlock`-style copying, not a phi rewrite alone.

Duplication is the cost. Two copies of a block mean more code, more compile time in later passes, and sometimes worse I-cache. The threshold exists for that reason. Profile data (`!prof`) can justify a larger copy on a hot edge; the pass looks at block frequencies when they are available.

### Freeze and poison

If the condition could be poison, "known true on this edge" might be a poison compare, and threading must not turn poison into a concrete branch that skips a side effect the poison would have allowed to be undefined in a different way. Modern jump threading inserts `freeze` when it needs a well-defined boolean from a value that might be poison. Search the file for `freeze`. You will see it in the output IR of aggressive tests. Do not delete those freezes in a downstream pass unless you can prove the value is not poison.

### What threading must not create

- **Irreducible control flow,** when it can avoid it. The pass has checks to avoid turning a natural loop into an irreducible loop. It sometimes still declines to thread across a loop header.
- **An infinite compile-time loop** of thread, simplify, thread. The pass has thresholds and "times threaded" limits. If you add a pass that recreates the pattern jump threading just removed, the pipeline oscillates. That is a real class of bugs; the fix is usually a cost check, not another run.
- **Broken phis.** Every rewritten edge must update phi incoming blocks. The implementation uses the standard CFG rewrite helpers. If you are reading a crash, it is almost always a phi whose predecessor list does not match the new CFG.

### Across-block patterns

The pass also threads through a chain of blocks with a single predecessor when the condition at the end is implied by the condition at the start. And it can thread a `select` that feeds a branch, by turning the select into two edges. Search for `ProcessThreadableEdges` and `TryToUnfoldSelect`.

Unfolding a select increases the number of branches temporarily so that threading can see them. Combined with SimplifyCFG, the typical result is still fewer branches than you started with.

## 5. The algorithm

`JumpThreadingPass::run` iterates blocks until no change or a budget runs out.

For each block, `ProcessBlock`:

1. If the terminator is not a branch or switch, skip.
2. `ComputeValueKnownInPredecessors` fills a list of `(predecessor, concrete destination)` pairs.
3. For a predecessor that has a concrete destination, `ThreadEdge(From, TheBlock, To)`.
4. `ThreadEdge` may clone `TheBlock`, rewire the edge, and fix phis and the dominator tree.
5. If the block became dead or folded, erase it.

`ComputeValueKnownInPredecessors` is the analysis half. `ThreadEdge` is the transform half. Read them in that order.

## 6. Where to read the code

1. [llvm/lib/Transforms/Scalar/JumpThreading.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/JumpThreading.cpp) — `ProcessBlock`, `ComputeValueKnownInPredecessors`, `ThreadEdge`.
2. [llvm/lib/Analysis/LazyValueInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyValueInfo.cpp) — what "known on this edge" means for non-phi values. Day 14 uses the same analysis from the other direction.
3. Tests in `llvm/test/Transforms/JumpThreading/`.

## 7. Worked example

```llvm
define i32 @thread(i1 %c, i32 %x) {
entry:
  br i1 %c, label %left, label %right
left:
  br label %join
right:
  br label %join
join:
  %p = phi i32 [ 1, %left ], [ %x, %right ]
  %cmp = icmp eq i32 %p, 1
  br i1 %cmp, label %is_one, label %not_one
is_one:
  ret i32 10
not_one:
  ret i32 %p
}
```

From `%left`, `%p` is 1, so `%cmp` is true, so the edge should not visit `join`'s compare.

```bash
opt -passes=jump-threading -S thread.ll
```

**Expected:** `%left` branches directly to `%is_one` (possibly through a small cloned block). `%right` still enters a block that tests `%x == 1`. There should be no path from `%left` that evaluates `icmp eq i32 1, 1` at runtime.

If the threshold were 0, the pass would refuse to clone and might still thread because `join`'s only real work is the compare, which is cheap. Look at the output rather than guessing how much was cloned.

## 8. Reading order

1. The phi example on paper: draw the edge `%left -> is_one` before you open the source.
2. `ComputeValueKnownInPredecessors`.
3. `ThreadEdge`, paying attention to the clone versus rewrite decision.
4. One test from `llvm/test/Transforms/JumpThreading/` that contains the word `threshold` or `freeze`.

## 9. Hands-on

```bash
opt -passes=jump-threading -print-changed=quiet -S thread.ll
opt -passes=jump-threading,simplifycfg,instcombine -S thread.ll
```

The second command is what the pipeline effectively does. Threading creates blocks; SimplifyCFG merges the trivial ones.

Force a refusal by stuffing the threaded block with many unrelated `add`s and setting `-jump-threading-threshold=1`. The compare should stay.

## 10. Pitfalls and invariants

- **Threading is a duplication pass.** A win on branch count can be a loss on code size. Read the threshold before you "fix" a missed thread by calling the pass twice.
- **It is not CVP.** CVP would replace a compare with `true` when the compare is true on every path. Threading is for the per-edge case. If the compare is true on every path, SimplifyCFG or CVP deletes it without a clone.
- **LVI is a lattice, and it gives up.** "Not threaded" often means LVI returned "overdefined", not that the fact is false. Day 14 shows the queries.
- **Domtree updates are mandatory.** A pass that clones a block and forgets `DomTreeUpdater` will fail `verify<domtree>` later in the pipeline.
- **`musttail` and `noduplicate` calls** cannot be cloned. The pass refuses to thread through them. Inline assembly with side effects is similarly expensive and often blocks threading.

## 11. Neighbors

| Day | Relationship |
| --- | --- |
| Day 4 | Dominator tree must be updated when edges move |
| Day 14 CVP | Same facts, different rewrite: CVP changes values, threading changes edges |
| Day 18 SimplifyCFG | Cleans up the blocks threading leaves behind |
| Day 28 Loop unswitch | The loop-aware cousin: hoist an invariant branch by duplicating the loop |
| Day 42 Inliner | Creates most of the correlated branches threading then removes |

## 12. Check yourself

**The condition is true for every predecessor. Is jump threading the pass you want?**
SimplifyCFG or CVP. Threading is for a condition that is known on a subset of the incoming edges.

**Why does threading sometimes clone a block?**
Other predecessors still need the original block. The predecessor being threaded needs the block's computations on its new private edge.

**What does LazyValueInfo provide?**
A constant or a constant range for a value at a particular context (a block or an edge), inferred from dominating branches and assumes.

**Why would the pass insert `freeze`?**
To turn a possibly-poison condition into a defined boolean before it duplicates paths based on that condition.

**A block contains forty calls. Why did the pass do nothing?**
The duplication threshold. Cloning forty calls to skip one compare is not profitable.

> [!TIP]
> `ThreadEdge` and LVI queries are in [Day-10-1-Extra-JumpThreading-DeepDive.md](Day-10-1-Extra-JumpThreading-DeepDive.md). Full IR before and after, including a case that must not thread, is in [Day-10-2-Extra-JumpThreading-Examples.md](Day-10-2-Extra-JumpThreading-Examples.md).
