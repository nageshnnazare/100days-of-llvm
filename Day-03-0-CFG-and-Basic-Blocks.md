# Day 3: LLVM IR, Basic Blocks, and the CFG

Every later pass is a walk over one data structure: a `Module` of `Function`s, each function a graph of `BasicBlock`s, each block a straight-line list of `Instruction`s. If that graph is hazy, InstCombine and LICM will look like a pile of special cases. Today is the map.

Upstream introduction: [LLVM Language Reference](https://llvm.org/docs/LangRef.html) and the programmer's manual section on the IR hierarchy in [llvm/docs/ProgrammersManual.rst](https://github.com/llvm/llvm-project/blob/main/llvm/docs/ProgrammersManual.rst).

---

## 1. Why this exists

A compiler cannot optimize "C". It optimizes a representation where:

- control transfers only at the end of a block,
- every value has exactly one defining instruction (SSA),
- types and side effects are explicit.

That representation is LLVM IR. The control-flow graph (CFG) is the part of it that says which block can run after which other block.

## 2. The idea

Think of a function as a directed graph.

- A **basic block** is a maximal straight-line sequence. If the first instruction runs, the rest run, until the terminator.
- An **edge** exists when a terminator can transfer to another block: a conditional branch has two successors, a switch has many, a return has zero.
- **SSA** means each instruction defines one value. When two paths would assign the same variable, the IR uses a `phi` at the join.

## Diagram

```text
        entry
        /   \
     then   else
        \   /
        join
         |
        return
```

`join` has two predecessors. A value that differs on the two paths is a phi:

```llvm
join:
  %x = phi i32 [ 1, %then ], [ 2, %else ]
```

Read that as: "if we arrived from `then`, `%x` is 1; if we arrived from `else`, `%x` is 2."

## 3. Where it sits in the pipeline

This is not a pass. It is the IR that every pass consumes. Clang's CodeGen builds it (Day 92). The verifier (`llvm/lib/IR/Verifier.cpp`) checks its invariants before and after passes. InstCombine, SimplifyCFG, and JumpThreading rewrite blocks; DominatorTree and LoopInfo are *analyses* of this graph.

There is no `opt -passes=cfg`. You print the graph:

```bash
opt -passes=dot-cfg -disable-output file.ll
# writes .dot files; render with: dot -Tpng cfg.foo.dot -o cfg.png
```

The pass is `CFGPrinter`. The textual form is already a CFG: labels are nodes, `br` / `switch` are edges.

## 4. Core concepts

### The ownership tree

```text
Module
  GlobalVariable, Function, GlobalAlias, NamedMD
    Function
      Argument, BasicBlock, AttributeList
        BasicBlock
          Instruction  (the last one is the terminator)
            Use  ->  the Values this instruction consumes
```

`Instruction` is a `User` (it uses operands) and a `Value` (other instructions use it). The use-list of `%x` is every instruction that mentions `%x`. That is how DCE and InstCombine find "everyone who depends on this" without scanning the whole function.

Header: [llvm/include/llvm/IR/Value.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Value.h), [Instruction.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Instruction.h), [BasicBlock.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/BasicBlock.h).

### Terminators

Exactly one terminator, and it is last. The common ones:

| Instruction | Successors |
| --- | --- |
| `ret` | none |
| `br i1 %c, label %t, label %f` | two |
| `br label %t` | one (unconditional) |
| `switch` | many, plus a default |
| `unreachable` | none; executing it is undefined |
| `invoke` | a normal successor and an unwind landing pad |

Less common, still real: `callbr`, `indirectbr`, `catchswitch`, `cleanupret`. You can ignore the EH family until you see `invoke` in C++ IR.

### Phi placement rules

- Phis live at the **start** of the block, before any non-phi instruction.
- A phi has one incoming value per predecessor. The predecessor list must match the CFG.
- The incoming value must dominate the end of the predecessor block (Day 4 makes "dominate" precise).

### Opaque pointers

Since LLVM 15 the pointer type is just `ptr`. The *load or store* carries the pointee type:

```llvm
%v = load i32, ptr %p, align 4
store i32 %v, ptr %p, align 4
```

`getelementptr` still computes addresses. It takes the source-element type as a separate operand, because the pointer no longer remembers it:

```llvm
%elem = getelementptr inbounds i32, ptr %base, i64 %i
```

### Instructions that are not "the function body"

- `alloca` is a stack slot, almost always in the entry block. Days 5 and 6 exist to delete them.
- `getelementptr` does not load. It only computes a pointer.
- `icmp` / `fcmp` produce `i1`.
- Calls may be `readnone`, `readonly`, `willreturn`, or have operand bundles. Those attributes are how later passes know a call is not a black box.

### SSA is not optional

There is no "assignment to `%x`" after `%x` is defined. Updating a variable in C becomes a chain of new names, or a phi at a join. Mem2Reg (Day 5) is the pass that builds those phis from naive `alloca`/`load`/`store` IR.

## 5. Walking the graph in C++

Passes almost never index instructions by number. They use the intrusive lists:

```cpp
for (BasicBlock &BB : F) {
  for (Instruction &I : BB) {
    if (auto *Br = dyn_cast<BranchInst>(&I)) {
      for (BasicBlock *Succ : successors(&BB))
        (void)Succ;
    }
  }
}
```

Successor and predecessor iterators live in [llvm/include/llvm/IR/CFG.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/CFG.h): `successors(BB)`, `predecessors(BB)`, `succ_size`, `pred_size`.

`BasicBlock` also gives you structural queries that hide the "phis first, terminator last" layout:

- `getTerminator()`
- `getFirstNonPHI()` / `getFirstNonPHIIt()`
- `getFirstInsertionPt()` — the safe place to insert a new non-phi instruction
- `splitBasicBlock(Instruction *I)` — cut the block in two; the first half ends with an unconditional branch

Graph algorithms see the CFG through `GraphTraits<BasicBlock *>` and `GraphTraits<Function *>`. DominatorTree, loop info, and reverse-postorder walks all use that, so a pass author does not write their own DFS.

Reverse postorder (RPO) is the order SCCP and many dataflow passes want: every predecessor of a block appears before the block, except for backedges. `llvm::ReversePostOrderTraversal<Function *>` in [llvm/include/llvm/ADT/PostOrderIterator.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/ADT/PostOrderIterator.h).

### Critical edges

An edge `A -> B` is **critical** when `A` has several successors and `B` has several predecessors. You cannot insert an instruction "on that edge" without affecting other paths. Passes that need a place to insert (GVN load PRE, loop passes moving code onto an exit) call `SplitCriticalEdge` in [llvm/lib/Transforms/Utils/BreakCriticalEdges.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/BreakCriticalEdges.cpp), which inserts a fresh block in the middle.

## 6. Where to read the code

Read in this order:

1. [llvm/include/llvm/IR/BasicBlock.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/BasicBlock.h) — the block as a list of instructions, `splitBasicBlock`, `getTerminator`.
2. [llvm/include/llvm/IR/CFG.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/CFG.h) — `succ_iterator`, `pred_iterator`.
3. [llvm/include/llvm/IR/InstrTypes.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/InstrTypes.h) — `UnaryInstruction`, `BinaryOperator`, `CmpInst`, `CastInst`. `Instruction.def` is the opcode list.
4. [llvm/lib/IR/BasicBlock.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/BasicBlock.cpp) — splitting, erasing, the "phis must be grouped" checks.
5. [llvm/lib/IR/Verifier.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/Verifier.cpp) — search for `visitBr`, `visitPHINode`. This is the spec, written as checks.

The assembly parser that turns `.ll` text into these objects is `llvm/lib/AsmParser/LLParser.cpp`. You do not need it today.

## 7. Worked example

```c
int abs_diff(int a, int b) {
  int d;
  if (a > b)
    d = a - b;
  else
    d = b - a;
  return d;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes abs.c -o abs.ll
```

The unoptimized IR has an `alloca` for `d`, stores on both paths, and a load at the join. After mem2reg (Day 5) the same CFG is:

```llvm
define i32 @abs_diff(i32 %a, i32 %b) {
entry:
  %cmp = icmp sgt i32 %a, %b
  br i1 %cmp, label %then, label %else

then:
  %sub = sub i32 %a, %b
  br label %join

else:
  %sub2 = sub i32 %b, %a
  br label %join

join:
  %d = phi i32 [ %sub, %then ], [ %sub2, %else ]
  ret i32 %d
}
```

Things to notice:

- `entry` has two successors. `then` and `else` each have one. `join` has two predecessors and zero successors besides the implicit return (a `ret` is a terminator with an empty successor list).
- `%d` is not "assigned twice". It is defined once, by the phi.
- The `icmp` is an ordinary instruction in `entry`. Only `br` terminates the block.

## 8. Reading order for today

1. This file, sections 4 and 5, with `BasicBlock.h` open.
2. One function in `Verifier.cpp` (`visitPHINode`) so you see which shapes are illegal.
3. The worked example, drawn on paper with arrows.
4. The deep dive, then the examples.

## 9. Hands-on

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes abs.c -o abs.ll
opt -passes=dot-cfg -disable-output abs.ll
opt -passes=mem2reg -S abs.ll -o abs.mem2reg.ll
opt -passes=instcombine,simplifycfg -S abs.mem2reg.ll -o abs.simple.ll
```

In `abs.simple.ll`, SimplifyCFG often collapses the diamond into a `select`. The CFG got smaller because the branch was the only reason the blocks existed. That is Day 18; today, just notice that passes rewrite the graph, they do not decorate it.

Print predecessors from a debugger or from `opt -passes=print<domtree>` once Day 4 is done. Today, `grep '^  br\\|^define\\|^[a-z].*:' abs.ll` is enough to see the edges.

## 10. Pitfalls and invariants

- **A block with no predecessors is unreachable**, except the entry. Passes may leave such blocks around; ADCE and SimplifyCFG remove them. Do not assume every block in the list is live.
- **Phi incoming order is not "left then right".** It is "one entry per predecessor", and the predecessor pointer must be that actual block. Editing a branch without updating phis is an IR verifier error.
- **`getelementptr` is not a load.** Deleting it does not change memory. Deleting a `load` might.
- **`unreachable` is not "empty".** If a call is `noreturn`, CodeGen may place `unreachable` after it. Anything "after" it in the source is a different block or it is dead.
- **Splitting a block duplicates the insertion point for phis.** New edges need new phi inputs. `SplitBlockAndInsertIfThen` in `llvm/lib/Transforms/Utils/BasicBlockUtils.cpp` is the helper that gets this right.
- **The entry block cannot have predecessors.** `Function::getEntryBlock()` is the only legal home for static `alloca`s.

## 11. Neighbors

| Later day | How it uses the CFG |
| --- | --- |
| Day 4 Dominators | Dominance is a fact about paths in this graph |
| Day 5 Mem2Reg | Places phis on the dominance frontier |
| Day 10 Jump threading | Rewrites edges when a condition is already known |
| Day 18 SimplifyCFG | Deletes and merges blocks |
| Day 21 LoopSimplify | Inserts preheaders and dedicated exits |
| Day 37 MemorySSA | A second SSA graph over the same blocks, for memory |

## 12. Check yourself

**What makes a sequence of instructions a basic block?**
A single entry (the label), a single exit (the terminator), and no branches in between.

**Why is the phi in the join block and not in the predecessors?**
SSA allows one definition. The join is the first point where both definitions are visible. Each predecessor *contributes* a value; it does not define the joined name.

**`store i32 1, ptr %p` then `br` — how many successors does the store have?**
None. The store is not a terminator. The branch has the successors.

**Can a phi refer to a value defined later in the same block?**
No. Incoming values come from predecessors. A phi may refer to itself only as an incoming value from a predecessor (a loop), which is a different block.

**What is a critical edge, in one sentence?**
An edge from a block with multiple successors into a block with multiple predecessors.

> [!TIP]
> The class-by-class walk is in [Day-03-1-Extra-IR-CFG-DeepDive.md](Day-03-1-Extra-IR-CFG-DeepDive.md). Worked IR you can paste into `opt` is in [Day-03-2-Extra-CFG-Examples.md](Day-03-2-Extra-CFG-Examples.md).
