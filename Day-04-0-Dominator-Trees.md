# Day 4: Dominator Trees

A dominator tree is the CFG compressed to one question: "which blocks *must* have run before this one?" Mem2Reg, LICM, GVN, and the verifier's "does not dominate all uses" check are all that question in disguise.

---

## 1. Why this exists

The CFG tells you which edges exist. It does not tell you, quickly, whether every path from the entry to block B passes through block A. Recomputing that by enumerating paths is exponential. The dominator tree answers it in constant time after a near-linear build.

SSA construction needs the same fact. A phi is placed where a definition stops dominating all of its uses: the dominance frontier.

## Diagram

```text
   CFG                         dominator tree

        A                         A
       / \                        |
      B   C                       +-- B
      |   |                       |   |
      D   |                       |   +-- D
       \ /                        |
        E                         +-- C
                                  |
                                  +-- E

   A dominates everyone.
   B dominates D.
   E's dominators are only A and E:
   neither B nor C is on every path to E.

   dominance frontier of D is {E}:
   D dominates a predecessor of E,
   and D does not strictly dominate E.
   Phis for a definition in D go at E.
```

A block A dominates B when every path from the entry to B passes through A. The dominator tree stores the immediate dominator of each block, and the DFS in/out numbers answer "does A dominate B?" in constant time. The dominance frontier of a definition is exactly the set of blocks that need a phi for it. That is the set mem2reg uses.

## 2. The idea

In a function with entry block E:

- A **dominates** B if every path from E to B contains A. A dominates itself.
- A **strictly dominates** B if A dominates B and A is not B.
- The **immediate dominator** of B, written `idom(B)`, is the closest strict dominator. It is unique. The entry has no immediate dominator.
- The **dominator tree** has an edge `idom(B) -> B`. It is a tree rooted at the entry.
- The **dominance frontier** of A, `DF(A)`, is the set of blocks where A's dominance stops. Formally: B is in `DF(A)` when A dominates some predecessor of B, and A does not strictly dominate B.

Picture:

```text
        entry
       /     \
     left    right
       \     /
        join
         |
        exit
```

- `entry` dominates every block.
- `left` dominates only itself. It does not dominate `join`, because the path `entry -> right -> join` avoids `left`.
- `idom(join) = entry`. `idom(left) = entry`.
- `DF(left) = {join}`. A definition in `left` that is also live through `right` needs a phi in `join`.

**Post-dominance** is dominance on the reversed CFG (edges flipped, and all `ret` blocks wired to a virtual exit). A **post-dominates** B when every path from B to a return passes through A. ADCE (Day 7) uses the post-dominance frontier to find control dependence: "which branches decide whether this side effect runs?"

## 3. Where it sits in the pipeline

`DominatorTreeAnalysis` is an analysis, not a transform. The pass manager caches it. Any CFG edit that adds or removes an edge must either update the tree or invalidate it.

Pass name for a dump:

```bash
opt -passes='print<domtree>' -disable-output file.ll
opt -passes='print<postdomtree>' -disable-output file.ll
```

You rarely schedule it by hand. `mem2reg`, `licm`, `gvn`, `simplifycfg`, and `loop-simplify` request it through `FunctionAnalysisManager::getResult<DominatorTreeAnalysis>(F)`.

Preserving it: a pass that does not touch the CFG returns `PreservedAnalyses::all()` and the tree stays. A pass that rewrites branches and does not call `DomTreeUpdater` should return `PreservedAnalyses::none()` or a set that omits `DominatorTreeAnalysis`.

## 4. Core concepts

### DFS numbers make queries O(1)

Each dominator-tree node stores an in-number and an out-number from a DFS of the tree. A dominates B if and only if

```text
in(A) <= in(B)  and  out(B) <= out(A)
```

So `DT.dominates(A, B)` does not walk the tree. Building the tree is the expensive part; querying it is a couple of integer compares. The implementation stores these on `DomTreeNodeBase`.

### Nearest common dominator

`findNearestCommonDominator(A, B)` is the deepest block that dominates both. Loop passes use it to discover which block a hoisted instruction should land in, and to check whether two values are available at the same program point.

### An instruction dominates a use

For values, dominance is defined on their positions:

- An instruction dominates a use in the same block when the definition appears earlier in the block (phis count as "earliest").
- An instruction dominates a use in another block when its block dominates the use's block.
- A phi use is considered to happen in the **predecessor** block, not in the phi's own block. That is why a phi may legally use a value defined in the predecessor even when that value's block does not dominate the phi's block.

`DT.dominates(Instruction *Def, Use &U)` implements those rules. The verifier calls the same logic.

### The frontier is where phis go

Cytron et al., "Efficiently Computing Static Single Assignment Form and the Control Dependence Graph" (1991), is the paper behind Day 5. The iterated dominance frontier of the set of blocks that define a variable is exactly the set of blocks that need a phi for that variable.

You do not compute frontiers by hand in a pass. `llvm::calculate` in [llvm/include/llvm/Analysis/IteratedDominanceFrontier.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/IteratedDominanceFrontier.h) does it. Mem2Reg calls it.

### Post-dominators and control dependence

Block Y is **control-dependent** on edge `A -> B` when B post-dominates neither A nor the other successors in the right way: intuitively, the branch at the end of A decides whether Y runs. ADCE marks a branch live when a live instruction is control-dependent on it. Otherwise a dead diamond can be deleted even if it contains arithmetic, because the arithmetic is not control-dependent on the way to a side effect.

Post-dom trees need a single exit. LLVM inserts a virtual root and attaches every block that has no successors (`ret`, `unreachable`) under it. A function with an infinite loop has blocks that do not post-dominate in the way you might expect: they never reach the virtual exit. Read `PostDominatorTree` dumps on a loop before you trust a mental picture.

### Updating, not rebuilding

Rebuilding the tree after every edge split is wasteful but correct. LLVM's `DomTreeUpdater` batches edge inserts and deletes and applies them with the incremental algorithm in `GenericDomTreeConstruction.h`. Passes that split many critical edges (GVN, LoopUnswitch) use the updater so the rest of the pipeline still sees a valid tree.

## 5. The algorithm, at the level you need

The construction lives in [llvm/include/llvm/Support/GenericDomTreeConstruction.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Support/GenericDomTreeConstruction.h). It implements **Semi-NCA** (semi-dominators + nearest common ancestor), from Georgiadis's work on linear-time dominators, with an incremental update path.

You do not need to re-derive Semi-NCA to use the tree. You do need the picture of what it produces:

1. Depth-first number the CFG from the entry.
2. Compute, for each block, a semi-dominator: an ancestor in the DFS tree that can reach the block via a back edge or cross edge, and that is "as high as possible".
3. Correct semi-dominators into immediate dominators.
4. Number the resulting tree so dominance tests are O(1).

The file's comment names the algorithm. When you read it, track three arrays indexed by DFS number: parent in the DFS tree, semi-dominator, and immediate dominator. Everything else is the bucket-and-path-compression machinery that makes step 3 fast.

Incremental updates (`insertEdge` / `deleteEdge` / `applyUpdates`) are harder than the batch build. If you are debugging a stale tree, call `DT.recalculate(F)` in a debug build and `DT.verify()` before you blame Semi-NCA.

## 6. Where to read the code

1. [llvm/include/llvm/IR/Dominators.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Dominators.h) — `DominatorTree`, which is a thin wrapper around `DomTreeBase<BasicBlock>`, plus `DominatorTreeAnalysis` and `DominatorTreePrinterPass`. This is the API passes call.
2. [llvm/include/llvm/Support/GenericDomTree.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Support/GenericDomTree.h) — `dominates`, `properlyDominates`, `findNearestCommonDominator`, DFS numbers, `DomTreeNodeBase`.
3. [llvm/include/llvm/Support/GenericDomTreeConstruction.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Support/GenericDomTreeConstruction.h) — Semi-NCA. Read the header comment and `Calculate` first; save the incremental updater for a second pass.
4. [llvm/include/llvm/Analysis/IteratedDominanceFrontier.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/IteratedDominanceFrontier.h) — the phi-placement query.
5. [llvm/lib/IR/Dominators.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/Dominators.cpp) — the analysis pass `run` method, verification, and the wrapper that knows about unreachable blocks.

`DomTreeUpdater` is in [llvm/include/llvm/Analysis/DomTreeUpdater.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/DomTreeUpdater.h).

## 7. Worked example

```llvm
define i32 @diamond(i1 %c, i32 %x) {
entry:
  br i1 %c, label %left, label %right
left:
  %a = add i32 %x, 1
  br label %join
right:
  br label %join
join:
  %p = phi i32 [ %a, %left ], [ %x, %right ]
  ret i32 %p
}
```

```bash
opt -passes='print<domtree>' -disable-output diamond.ll
```

The dump looks like this (names may be quoted):

```text
Inorder Dominator Tree:
  [1] %entry
    [2] %left
    [2] %right
    [2] %join
```

The bracketed numbers are DFS numbers of the **tree**, not execution counts. `entry` is the parent of the other three. `left` does not parent `join`.

Now the question mem2reg asks: where does a definition in `left` stop dominating? `DF(left) = {join}`. The phi is exactly there. `%a` does not dominate `join` (the right-hand path never defined it), so the use of `%a` is legal only as a phi incoming value associated with predecessor `left`. Using `%a` in `join` *after* the phi, as an ordinary instruction, would be "does not dominate all uses".

## 8. Reading order

1. Draw the diamond and mark `idom` of every block before you open the headers.
2. `Dominators.h` public methods only.
3. The DFS-number test in `GenericDomTree.h` (`dominates` on nodes).
4. The header comment of `GenericDomTreeConstruction.h`.
5. Deep dive, then examples.

## 9. Hands-on

```bash
opt -passes='print<domtree>' -disable-output diamond.ll
opt -passes='print<postdomtree>' -disable-output diamond.ll
opt -passes=mem2reg -S alloca_form.ll
```

For the post-dom dump of the diamond: `join` post-dominates `left`, `right`, and `entry`, because every path from those blocks to the return goes through `join`. `left` does not post-dominate `entry`.

Break dominance on purpose and watch the verifier:

```llvm
define i32 @bad(i1 %c, i32 %x) {
entry:
  br i1 %c, label %left, label %join
left:
  %a = add i32 %x, 1
  br label %join
join:
  ret i32 %a
}
```

```bash
opt -passes=verify -disable-output bad.ll
```

**Expected:** an error that `%a` does not dominate its use. The path through `entry -> join` reaches the `ret` without defining `%a`.

## 10. Pitfalls and invariants

- **Unreachable blocks are not in the tree in the way you expect.** A block with no path from the entry has no dominator. `DT.getNode(BB)` returns null. Always null-check. The verifier treats uses in unreachable blocks specially.
- **A block dominates itself.** If you want "strictly above", call `properlyDominates`. Using `dominates` to answer "is this in a different, dominating block?" will be true for the same block and you will hoist an instruction above its own operands.
- **Phi uses are at the predecessor.** `dominates(Def, Phi)` is the wrong question. Use the `Use` overload.
- **Loops: the header dominates the body** when the loop is in natural-loop form (Day 21). The latch does not dominate the header. A value defined in the latch and used in the header must be a phi incoming, not a naked use.
- **Post-dominance is not "comes later in the file".** Block order in the `.ll` is irrelevant.
- **Invalidating the tree and then querying the cached one** is undefined. If you call `FAM.getResult<DominatorTreeAnalysis>` you get the cached tree. If your pass then deletes a block and queries again, you must have updated it.

## 11. Neighbors

| Day | Relationship |
| --- | --- |
| Day 3 | The CFG the tree is built from |
| Day 5 Mem2Reg | Phi placement = iterated dominance frontier |
| Day 7 ADCE | Post-dominance frontier = control dependence |
| Day 8 EarlyCSE | A scoped hash table keyed by dominance depth |
| Day 17 Sink / Day 23 LICM | "Can I move this instruction and still dominate its uses?" |
| Day 22 LCSSA | Exit phis are inserted on edges the dominator tree helps locate |

## 12. Check yourself

**Does `left` dominate `join` in the diamond?**
No. There is a path from the entry to `join` that avoids `left`.

**What is the immediate dominator of `join`?**
`entry`.

**Why is the dominance frontier of `left` equal to `{join}`?**
`left` dominates a predecessor of `join` (namely `left` itself, which is a predecessor of `join`), and `left` does not strictly dominate `join`.

**A function has two `ret` blocks. What is the root of the post-dominator tree?**
A virtual exit that both returns are connected to. Neither return post-dominates the other.

**You inserted a block on an edge and then returned `PreservedAnalyses::all()`. What is wrong?**
The cached dominator tree still describes the old CFG. Later passes will query a lie. Update the tree or do not preserve `DominatorTreeAnalysis`.

> [!TIP]
> Implementation detail is in [Day-04-1-Extra-Dominators-DeepDive.md](Day-04-1-Extra-Dominators-DeepDive.md). More graphs and the phi-placement exercise are in [Day-04-2-Extra-Dominators-Examples.md](Day-04-2-Extra-Dominators-Examples.md).
