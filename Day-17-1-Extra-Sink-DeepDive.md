# Day 17 Extra: Sink — Deep Dive

Companion to [Day-17-0-Sink.md](Day-17-0-Sink.md). Examples: [Day-17-2-Extra-Sink-Examples.md](Day-17-2-Extra-Sink-Examples.md).

Source of truth: [llvm/lib/Transforms/Scalar/Sink.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/Sink.cpp).

## Diagram

```text
        entry
        %s = add %a, %b          used only in %cold
          |                      |
          +----------+           |  Sink moves %s
                     v           v
                   hot         cold:
                   (no use)    %s = add %a, %b
                               use %s

   %a and %b must be available in %cold
   (they dominate it, or they are arguments)
```

Sink walks uses backward. If every use of an instruction is in a successor, and the operands are available there, the instruction moves to the use. That shortens its live range. It will not sink into a block where the operands would have to be recomputed, and it will not sink a load across a possible clobber.

## 1. File map

| Symbol | Role |
|--------|------|
| `DEBUG_TYPE "sink"` | `-debug-only=sink` |
| `NumSunk`, `NumSinkIter` | Statistics |
| `isSafeToMove` | Memory / side-effect safety vs AA and store set |
| `IsAcceptableTarget` | Domination + loop-nesting check for a successor |
| `SinkInstruction` | Choose target and move one instruction |
| `ProcessBlock` | Bottom-up scan of one BB |
| `iterativelySinkInstructions` | Fixed-point over the function |
| `SinkingPass::run` | New PM entry |
| `SinkingLegacyPass` | Legacy PM wrapper (still in-file on many trees) |

Read the file top-to-bottom once; it is small enough to hold in working memory.

## 2. `SinkingPass::run` path

```
SinkingPass::run(F, AM)
  DT = DominatorTreeAnalysis
  LI = LoopAnalysis
  AA = AAManager
  if (!iterativelySinkInstructions(F, DT, LI, AA))
    return PreservedAnalyses::all()
  PA.preserveSet<CFGAnalyses>()
  return PA
```

CFG structure is unchanged, so CFG-related analyses remain valid. Instruction moves invalidate many other analyses; the pass does not claim to preserve AA or LI contents beyond what the PM default invalidation does—read the return carefully when composing pipelines.

## 3. Iteration strategy

`iterativelySinkInstructions` loops:

1. For each basic block, `ProcessBlock`.
2. If any block sank something, bump `NumSinkIter` and repeat.
3. Stop when a full scan makes no change.

Why iterate? Sinking `%b` that uses `%a` into a successor may leave `%a` with uses only in that successor (or enable a new target). A single reverse walk inside one block is not enough across the whole function.

## 4. `ProcessBlock` details

Important mechanics:

- Skip unreachable blocks (`!DT.isReachableFromEntry(&BB)`).
- Start at `BB.end()` and walk toward `begin`, predecrementing the iterator **before** sinking so moving `Inst` out of the block does not invalidate the walk.
- Skip debug/pseudo instructions.
- Maintain `SmallPtrSet` of stores encountered while walking upward. Because the walk is bottom-up, “stores already seen” are stores that appear *below* the current instruction in the original block—i.e., stores that would be *passed* if the instruction moved downward in program order within the block’s successors story. The helper uses this set together with AA when deciding load motion.

When `SinkInstruction` succeeds, increment `NumSunk`.

## 5. `isSafeToMove`

Typical rejects / concerns (study the function body for the exact checklist in your revision):

- Instructions that may read or write memory: must not cross a conflicting store (AA).
- Volatile operations.
- Ordering constraints that sinking would break.
- Calls: only if known not to write memory in a way that blocks the move (AA / attributes matter).
- Anything that is not safe to speculate when the move would introduce execution on a new path.

The store set argument avoids rescanning the whole function for every candidate: the bottom-up block walk already collected stores that sit “after” the candidate in the current block.

## 6. `IsAcceptableTarget`

Given candidate successor `SuccToSinkTo`:

1. Every use of `Inst` must be dominated by `SuccToSinkTo` under the DominatorTree (PHI incoming uses interpreted via their predecessor edge).
2. LoopInfo: the loop depth of the target must not be greater than the depth of the current block. Sinking into a child loop of a sibling region is the classic failure mode to avoid.

If multiple successors could work, the implementation picks according to its local heuristic (often the unique successor that dominates all uses, or the first acceptable). Prefer reading the actual selection order rather than memorizing a blog summary.

## 7. `SinkInstruction` move

Once a target is chosen:

- Splice / move the instruction into the successor block (usually near the beginning, after PHIs, so it dominates the rest of the successor).
- SSA remains valid because the domination check guarantees all uses still see the def.
- No new PHIs are created by Sink itself; if a value were needed on multiple paths, the domination check would have failed.

## 8. What Sink does **not** handle

- Merging identical instructions from multiple predecessors (that is SimplifyCFG `SinkCommonInsts`).
- Cross-loop LICM-style hoist.
- Machine-level sinking after ISel (`MachineSink`).
- Full register-pressure modeling for multi-operand ops.
- Speculative sinking of expensive ops onto colder paths without cost checks—the pass is conservative on speculation.
- Changing calling conventions or ABI.

## 9. Interaction with AA pipelines

When experimenting:

```bash
opt -passes=sink -aa-pipeline='basic-aa' -S t.ll
opt -passes=sink -aa-pipeline='basic-aa,tbaa' -S t.ll
```

TBAA can prove NoAlias between a load and a store Sink would otherwise treat as MayAlias, unlocking a sink. Missing metadata freezes the load above the store.

## 10. Debugging tips

```bash
opt -passes=sink -debug-only=sink -S in.ll -o out.ll 2>sink.log
```

Expect messages about sinking iterations and which instructions moved. Pair with:

```bash
opt -passes='print<domtree>' -disable-output in.ll
opt -passes='print<loops>' -disable-output in.ll
```

When a sink “should” happen but does not, check in order: (1) use domination, (2) loop depth, (3) volatile/atomic, (4) AA vs stores, (5) whether Clang left `optnone`.

## 11. Relation to CodeGen MachineSink

MIR sinking has analogous helpers and also struggles with register pressure on multi-operand ops. When an IR-level sink looks profitable but post-RA code looks worse, compare both layers—fixing that IR Sink and MachineSink share the same heuristic limitations.

## 12. Study checklist

1. Trace one `add` used only in the true arm through `SinkInstruction`.
2. Trace one `load` blocked by a may-alias `store`.
3. Construct a nested loop and confirm LoopInfo blocks sinking into the inner header.
4. Confirm PHIs never appear in the candidate list.
5. Run until `NumSinkIter` > 1 on a chain of dependent arithmetic to see iteration matter.

## 13. Types and IR conventions

All notes in this series use opaque pointers (`ptr`). Sink itself is type-agnostic beyond what AA and instruction opcodes require; it does not rewrite GEPs into typed forms.

Example load/store shape Sink must reason about:

```llvm
define void @ldst(ptr %p, ptr %q, i1 %c) {
entry:
  %v = load i32, ptr %p
  store i32 0, ptr %q
  br i1 %c, label %t, label %f
t:
  call void @use(i32 %v)
  br label %x
f:
  br label %x
x:
  ret void
}
```

Whether `%v` sinks into `%t` depends on AA between `%p` and `%q`. With MayAlias, `isSafeToMove` refuses to move the load below the store’s effect.

## 14. Takeaway for reading LICM next

When you later read LICM’s “can hoist?” checks, map them mentally onto Sink’s `IsAcceptableTarget` loop-depth test and speculation rules. Same dominance vocabulary; opposite direction.
