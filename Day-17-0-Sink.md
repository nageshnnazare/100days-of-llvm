# Day 17: Sink

Code sinking moves an instruction from a block into a successor where it is still dominated and cheaper to execute (or only needed on that path). The mid-end pass is `sink` in [Sink.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/Sink.cpp), class `SinkingPass`. Deep dive: [Day-17-1-Extra-Sink-DeepDive.md](Day-17-1-Extra-Sink-DeepDive.md). Examples: [Day-17-2-Extra-Sink-Examples.md](Day-17-2-Extra-Sink-Examples.md).

## 1. Why this exists

Compilers often compute values “early”: before a branch, in a loop preheader-like block, or at the join of a diamond. If every use of that value lies on only one successor path, executing the instruction on the other path wastes work and can lengthen register live ranges for results that are never needed there.

Sinking is the dual of hoisting. LICM and early CSE pull invariants *up*. Sink pushes instructions *down* toward their uses when domination and speculation rules allow it. That shortens critical-path work on cold arms and can unlock further CFG cleanup when a block becomes empty enough for SimplifyCFG to fold.

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

## 2. The idea

Given instruction `I` in block `BB` with successors, Sink asks:

1. Do all (or the decisive) uses of `I` live in a region dominated by one particular successor `Succ`?
2. Is it safe to move `I` into `Succ` (aliasing, volatility, control dependence, loop nesting)?
3. Is the move profitable (speculation cost, not lengthening a hotter path unnecessarily)?

If yes, `SinkInstruction` relocates `I` into that successor (typically near the start, preserving dominance of uses). Iteration over the function repeats until a fixed point because sinking one instruction can expose another.

Mental picture: a diamond where `%t = add` only feeds the true arm → sink `%t` into the true successor so the false arm never materializes it.

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `sink` |
| Class | `SinkingPass` |
| File | [llvm/lib/Transforms/Scalar/Sink.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/Sink.cpp) |
| Analyses | `DominatorTree`, `LoopInfo`, `AAResults` |
| Debug | `-debug-only=sink` |

In PassBuilder’s default pipelines, sinking appears among scalar cleanups after substantial CFG and memory opts have run. Exact slotting moves across LLVM versions; treat it as a late-ish function pass that can run more than once in different pipelines. Locally, always invoke it explicitly:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=sink -S file.ll -o file.sink.ll
```

It preserves CFG shape (does not split/merge blocks itself) and returns `PreservedAnalyses` that keep CFG analyses when it changes instructions.

## 4. Core concepts

### Domination of uses

An instruction may sink into successor `S` only if every use is dominated by `S` (or can be rewritten consistently with that move). PHI uses count via their incoming edges: a value used as a PHI incoming from predecessor `P` is “used” in a context tied to `P`, which matters when choosing the sink target.

### Acceptable target (`IsAcceptableTarget`)

Helper `IsAcceptableTarget` checks whether sinking into a candidate successor is legal: the successor must dominate the uses, and sinking must not enter a *deeper* loop than the instruction currently lives in (LoopInfo). Moving an invariant into a nested loop would increase dynamic executions—anti-LICM.

### Safety (`isSafeToMove`)

`isSafeToMove` consults AliasAnalysis and tracks stores seen while walking the block bottom-up. Loads cannot sink past stores that may alias. Volatile memory ops, atomics, and instructions that use token types are rejected. Terminators and PHIs never sink.

### Speculation / path cost

If sinking requires executing `I` on a path where it was previously not executed, the pass only allows cheap/speculation-safe ops. The dual case—uses only on one arm—means sinking *removes* execution from the other arm, which is the common win.

### Opposite of LICM

LICM moves loop-invariant instructions to the preheader (fewer executions inside the loop). Sink refuses to move into deeper loops and prefers successors that still dominate uses. Together they push values toward the “right” nesting depth.

## 5. The algorithm

High-level flow in Sink.cpp:

1. `SinkingPass::run` fetches DT, LI, and AA, then calls `iterativelySinkInstructions`.
2. Each iteration walks every basic block via `ProcessBlock`.
3. `ProcessBlock` walks the block **bottom-up**. Walking upward matters: sinking a later instruction first can free earlier ones, and the iterator must not be invalidated incorrectly when an instruction moves out.
4. For each non-debug instruction, call `SinkInstruction`.
5. `SinkInstruction` finds a successor that dominates all uses, checks `IsAcceptableTarget` and `isSafeToMove`, then splices the instruction into the successor.
6. Repeat whole-function iterations while any sink succeeds (`NumSinkIter` / `NumSunk` statistics).

Rough decision inside `SinkInstruction`:

```
for each candidate successor S of Inst.parent:
  if not IsAcceptableTarget(Inst, S, DT, LI): continue
  if not safe w.r.t. AA / stores: continue
  move Inst into S; return true
return false
```

## 6. Where to read the code

1. [Sink.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/Sink.cpp) — entire pass is one focused file.
2. `SinkingPass::run` — analysis acquisition and preserved-set.
3. `iterativelySinkInstructions` — fixed-point loop.
4. `ProcessBlock` — bottom-up walk and store set.
5. `SinkInstruction` — core move decision.
6. `IsAcceptableTarget`, `isSafeToMove` — legality helpers.
7. Lit tests under `llvm/test/Transforms/Sink/` in-tree for expected patterns.

Header declaration lives with other scalar passes (new PM registration); the behavior is entirely in the `.cpp`.

## 7. Worked before/after

```c
int only_true(int c, int x, int y) {
  int t = x + y;   /* computed before the branch */
  if (c)
    return t;
  return 0;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes only_true.c -o only_true.ll
opt -passes=sink -S only_true.ll -o only_true.sink.ll
```

**Before (shape):**

```llvm
entry:
  %t = add i32 %x, %y
  %tobool = icmp ne i32 %c, 0
  br i1 %tobool, label %if.then, label %if.end
if.then:
  br label %return
if.end:
  br label %return
return:
  %retval = phi i32 [ %t, %if.then ], [ 0, %if.end ]
  ret i32 %retval
```

**After (shape):** `%t = add` moves into `if.then` (or the block that dominates the PHI incoming that carries `%t`), so the false path no longer computes the add. Exact block labels depend on Clang’s CFG; the invariant is: the add’s uses are dominated by the sink target, and the cold/false path does not execute it.

## 8. Reading order for today

1. This note’s domination and loop-nesting rules.
2. `ProcessBlock` + `SinkInstruction` in Sink.cpp (short file—read end to end).
3. One lit test that sinks a non-memory op across a branch; one that refuses a load past a store.
4. [Day-17-1-Extra-Sink-DeepDive.md](Day-17-1-Extra-Sink-DeepDive.md) for AA/store-set details.
5. [Day-17-2-Extra-Sink-Examples.md](Day-17-2-Extra-Sink-Examples.md) for hands-on IR.
6. Contrast with LICM (Day 22-ish in many curricula) and SimplifyCFG’s own common-code sink (Day 18).

## 9. Hands-on commands

```bash
# Produce IR without mid-end opts
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sink_demo.c -o sink_demo.ll

# Run sinking
opt -passes=sink -S sink_demo.ll -o sink_demo.out.ll

# Debug (asserts build)
opt -passes=sink -debug-only=sink -S sink_demo.ll -o /dev/null 2>sink.log

# Compare
diff -u sink_demo.ll sink_demo.out.ll
```

Useful experiments:

- Pure arithmetic used on one arm → expect sink.
- Load before a may-alias store → expect no sink past the store.
- Instruction used on both arms → stay put.
- Candidate sink into a nested loop body → refuse (LoopInfo depth).

## 10. Pitfalls and invariants

- **Volatile / atomic:** never sink; ordering and observability must stay.
- **Token-using instructions** (e.g. some callbr / exception-related patterns): rejected.
- **PHIs and terminators:** fixed to their blocks; Sink does not touch them as move candidates.
- **Loop depth:** do not sink into a deeper loop than the current block’s loop.
- **AA conservatism:** MayAlias blocks sinking a load past a store; missing TBAA can freeze an otherwise legal sink.
- **Register pressure:** sinking a multi-input op toward a use can lengthen operand live ranges while shortening the result’s—Sink does not fully model pressure (known limitation discussed in upstream issues).
- **`optnone`:** strip it; use `-O1 -Xclang -disable-llvm-passes`.
- **CFG unchanged:** if you expected block merges, that is SimplifyCFG, not Sink.

## 11. Neighbors

- **Dominators (Day 4)** — legality of moving defs relative to uses.
- **LICM** — hoist out of loops; Sink’s loop checks are the dual.
- **SimplifyCFG (Day 18)** — can sink *identical* instructions from multiple preds (`SinkCommonInsts`); different pass, overlapping vocabulary.
- **MachineSink** (`MachineSink.cpp`) — late CodeGen analogue; similar idea, MIR.
- **ADCE / BDCE** — delete dead code after sinking exposes unused paths.
- **InstCombine** — may create sinkable patterns or fold after a move.

### Lab: compare Sink vs SimplifyCFG common-sink

Build a diamond where both arms compute the **same** `add` with the same operands, and another where only one arm uses a unique `add`:

```bash
opt -passes=sink -S unique_use.ll -o unique_use.sink.ll
opt -passes='simplifycfg<sink-common-insts>' -S same_add_both_arms.ll -o both.scfg.ll
```

(Exact SimplifyCFG pass-parameter spelling varies by LLVM version—use `opt --help-hidden` / pipeline parsing docs if the angle-bracket form fails; setting `SinkCommonInsts` via a late default `-O2` pipeline is an alternative.) Expect Day-17 Sink to move the unique-use add, and SimplifyCFG’s common-sink path to merge the identical pair. Reading both diffs back-to-back cements the vocabulary split.

### Static SSA argument (why no new PHIs)

If every use of `%v` is dominated by successor `S`, then after moving the defining instruction into `S`, those uses still see a dominating definition. If some use were in a sibling of `S`, either that use is unreachable after the move (illegal) or `S` did not dominate it (rejected by `IsAcceptableTarget`). Hence Sink never needs to insert PHIs to repair SSA for a successful move.

## 12. Check yourself

**Q1.** What is the `opt` pass name and the new-PM class?  
**A1.** `sink` / `SinkingPass`.

**Q2.** Why walk the block bottom-up in `ProcessBlock`?  
**A2.** So sinking a later instruction does not invalidate the walk incorrectly, and dependent earlier instructions can sink in later iterations once uses have moved.

**Q3.** Name three analyses Sink uses.  
**A3.** DominatorTree, LoopInfo, AliasAnalysis (`AAResults`).

**Q4.** Why refuse sinking into a deeper loop?  
**A4.** That would increase dynamic execution count—anti-LICM behavior.

**Q5.** What debug flag prints Sink’s decisions?  
**A5.** `-debug-only=sink` (`DEBUG_TYPE "sink"`).

**Q6.** Does Sink merge basic blocks?  
**A6.** No. It only moves instructions between existing blocks; CFG cleanup is a neighbor’s job.

Tip: implementation walk in [Day-17-1-Extra-Sink-DeepDive.md](Day-17-1-Extra-Sink-DeepDive.md); four worked IR cases in [Day-17-2-Extra-Sink-Examples.md](Day-17-2-Extra-Sink-Examples.md).

### Statistics

On an asserts build, `-stats` reports `NumSunk` and `NumSinkIter`. A high iteration count with few sunk instructions usually means each iteration unlocked one dependent move—expected for long single-use chains.
