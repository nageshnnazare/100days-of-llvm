# Day 4 Extra: Deep Dive into DominatorTree

Companion to [Day-04-0-Dominator-Trees.md](Day-04-0-Dominator-Trees.md).

Files, in the order you should open them:

1. [llvm/include/llvm/IR/Dominators.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Dominators.h)
2. [llvm/lib/IR/Dominators.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/Dominators.cpp)
3. [llvm/include/llvm/Support/GenericDomTree.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Support/GenericDomTree.h)
4. [llvm/include/llvm/Support/GenericDomTreeConstruction.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Support/GenericDomTreeConstruction.h)
5. [llvm/include/llvm/Analysis/DomTreeUpdater.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/DomTreeUpdater.h)

The algorithm is templated on the node type so the same code builds a dominator tree for IR basic blocks, machine basic blocks, and the Clang CFG. `DominatorTree` is `DomTreeBase<BasicBlock>` plus a few IR-specific helpers.

---

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

## 1. What `DominatorTreeAnalysis` returns

The new-PM analysis is a class with a `run(Function &F, FunctionAnalysisManager &)` method in `Dominators.cpp`. It constructs a `DominatorTree`, calls `recalculate(F)`, and returns it by value into the analysis cache.

`recalculate` is the batch build: it clears the old tree and runs Semi-NCA from `GraphTraits<Function *>`. The entry node is `F.getEntryBlock()`. Blocks not reachable from the entry do not get nodes.

`DominatorTreePrinterPass` is the pass behind `print<domtree>`. `DominatorTreeVerifierPass` (and `DT.verify()`) checks that the tree matches a freshly built one. When a test fails with "dominator tree not up to date", this is the comparison that fired.

There is a separate `PostDominatorTreeAnalysis`. It is not the same object with a flag. It builds `DomTreeBase<BasicBlock>` on the inverse graph. Look at `PostDominatorTree::recalculate` for the virtual-exit handling.

## 2. `DomTreeNodeBase`

Each reachable block has a node:

- `getBlock()` / `getIDom()`
- `getChildren()` — the blocks this one immediately dominates
- `getDFSNumIn()`, `getDFSNumOut()` — the O(1) dominance test
- `getLevel()` — depth in the tree, used by the incremental algorithm and by some heuristics

`DominatorTree::getNode(const BasicBlock *)` returns null for unreachable blocks. `getRootNode()` is the entry.

The dominance test on nodes, conceptually:

```text
bool dominates(A, B) {
  return A->In <= B->In && B->Out <= A->Out;
}
```

Sibling subtrees occupy disjoint DFS ranges. A parent's range covers its descendants. This is the same numbering trick as "is this node in that XML subtree".

## 3. Instruction-level `dominates`

`DominatorTree::dominates(const Instruction *, const Instruction *)` in `Dominators.cpp` adds the same-block case the tree cannot see, because the tree only talks about blocks.

Rules worth memorizing, because the verifier uses them:

1. A block dominates itself, but an instruction does not dominate instructions that appear **before** it in the same block.
2. Phis are ordered as a prefix. A phi does not dominate another phi in the same block in a useful way; their "uses" are at predecessors.
3. For a `Use` that is a phi input, the use position is the incoming block, at the terminator. So a value defined in the predecessor can feed the phi even though the predecessor does not dominate the phi's block.
4. An invoke's value is defined at the invoke, which is in the normal-successor sense a terminator. Uses on the unwind edge have their own dominance rules; ignore them until you are looking at landing pads.

When you see `Instruction does not dominate all uses!`, pick the use the verifier names and apply rules 1–3 on paper before you change the pass.

## 4. Semi-NCA, without the path compression

Read the comment at the top of `GenericDomTreeConstruction.h`. Then find the function that runs the batch algorithm (`Calculate` / `runSemiNCA`). The structure is:

**DFS.** `runDFS` numbers nodes, records the parent in the DFS tree, and records the predecessor via which the DFS discovered the node. Unreachable nodes are never numbered.

**Semi-dominators.** For each node `w` in reverse DFS order, look at every CFG predecessor `v` of `w`. If `v` is an ancestor of `w` in the DFS tree, it is a candidate semi-dominator. If `v` is not an ancestor, the candidate is the semi-dominator of a node that can reach `v` through already-processed paths (this is the "eval" path compression). The semi-dominator of `w` is the candidate with the smallest DFS number.

Intuition: the semi-dominator is "the highest place from which there is a jump into this node", which is almost the immediate dominator, except it can be slightly too high when that jump itself is dominated by something deeper.

**Correction.** A second walk turns semi-dominators into immediate dominators. If the semi-dominator's own semi-dominator chain says "the real idom is higher", the algorithm lifts it. This is the NCA part: the immediate dominator is a nearest common ancestor in the DFS tree of the relevant semi-dominator candidates.

**Tree numbering.** Once each node has an idom, the children lists are filled and `AssignDFSNumbers` writes in/out numbers.

You can understand mem2reg without being able to reimplement step "eval". You cannot safely *update* a dominator tree by hand without this file. Use `DomTreeUpdater`.

## 5. `DomTreeUpdater`

Passes used to call `DT.insertEdge(From, To)` and `DT.deleteEdge` directly. The updater exists because a pass often makes a batch of edits, and some of those edits are easier to describe as "these blocks were deleted" than as edge pairs.

Typical use from a transform:

```cpp
DomTreeUpdater DTU(DT, DomTreeUpdater::UpdateStrategy::Lazy);
DTU.applyUpdates({
  {DominatorTree::Delete, OldPred, Succ},
  {DominatorTree::Insert, NewPred, Succ},
});
// ... more IR edits ...
// destructor or flush() applies the batch
```

`Lazy` strategy accumulates. `Eager` applies immediately.

If you also have a post-dominator tree, the updater can track both; look at the constructor overloads. Loop passes often update `DominatorTree`, `LoopInfo`, and `MemorySSA` together. Forgetting one of them is the usual "pass works alone and fails in the pipeline" bug.

`DomTreeUpdater::deleteBB` handles the case where the block goes away entirely, not just an edge.

## 6. Iterated dominance frontier

[llvm/include/llvm/Analysis/IteratedDominanceFrontier.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/IteratedDominanceFrontier.h) exposes:

```cpp
template <class NodeTy, bool IsPostDom>
class IDFCalculator { ... void calculate(SmallVectorImpl<NodeTy *> &IDFBlocks); }
```

Mem2Reg's promoter configures it with the dominance tree and the set of defining blocks for one alloca. The result is the set of blocks that receive a phi.

"Iterated" means: a phi is itself a definition. After you place phis, those blocks join the defining set, which can require more phis. The calculator computes the fixed point. For a single alloca written in two different arms of a diamond, the fixed point is just the join. For a loop where the definition is inside a conditional, the frontier includes the loop header, because the header is where the backedge meets the incoming edge.

The implementation walks the dominator tree bottom-up and unions child frontiers, which is the Cytron DJ-graph algorithm. It is much faster than "for each block, scan all other blocks".

## 7. `LoopInfo` peeks ahead

Natural loops are defined using dominance: the header dominates the body, and a backedge is an edge whose target dominates its source. [llvm/include/llvm/Analysis/LoopInfo.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/LoopInfo.h) uses the dominator tree to find backedges during `LoopInfo::analyze`. You will use this from Day 21 onward. Today, notice only that a stale dominator tree means a stale loop nest, because `LoopInfo` is rebuilt or updated from the tree.

## 8. Machine dominators

`MachineDominatorTree` in `llvm/include/llvm/CodeGen/MachineDominators.h` is the same template on `MachineBasicBlock`. After instruction selection the CFG is a different graph (critical edges may have been split, EH pads lowered). IR dominance facts do not carry over. Day 76's MachineLICM asks the machine tree, not the IR tree.

## 9. Debug and verify

```bash
opt -passes='print<domtree>' -disable-output f.ll
opt -passes='verify<domtree>' -disable-output f.ll
```

The verify syntax follows the new PM's analysis-verifier pattern (`verify<domtree>`). If your `opt` is old and rejects it, call the printer and read the tree by hand.

In a debugger:

```text
DT.dump()
DT.verify()
```

`verify()` rebuilds a fresh tree and compares parent pointers. It is the right check after you think `applyUpdates` did the right thing.

There is no `-debug-only=domtree` that traces Semi-NCA step by step in a way that is pleasant to read. The dumps are the interface.

## 10. What a pass is expected to do

Checklist when your pass edits branches:

1. Request `DominatorTreeAnalysis` if you need queries *before* the edit.
2. Make the IR edit with the helpers in `BasicBlockUtils` when possible; several of them take a `DomTreeUpdater *`.
3. If you edit by hand, record every deleted and inserted edge.
4. On the `PreservedAnalyses` you return, either the updater has flushed and you `preserve<DominatorTreeAnalysis>()`, or you do not preserve it.

Returning `PreservedAnalyses::all()` out of habit is the bug. The analysis manager believes you.
