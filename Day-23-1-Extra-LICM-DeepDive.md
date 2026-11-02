# Day 23 Extra: LICM — Deep Dive

Companion to [Day-23-0-LICM.md](Day-23-0-LICM.md). Examples: [Day-23-2-Extra-LICM-Examples.md](Day-23-2-Extra-LICM-Examples.md). Primary source: [LICM.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LICM.cpp).

## Diagram

```text
   preheader                         preheader
     |                                 %inv = add %a, %b   <-- hoisted
     v                                 |
   header                              v
     %i = phi ...                    header
     %inv = add %a, %b                 %i = phi ...
     %use = ... %inv                   %use = ... %inv
     br latch                          br latch

   the add does not depend on %i,
   and nothing in the loop writes
   the memory it would touch
```

LICM moves an instruction that is invariant and safe to the preheader, so it runs once instead of once per iteration. A load is invariant only when alias analysis says the loop does not write that memory. A store sinks to the exit only when the address is loop-invariant and no later iteration reads the old value.

## 1. File map

| Symbol / area | Role |
|---------------|------|
| `LICMPass` / `LNICMPass` | New PM loop / loop-nest entry; require MSSA |
| `LegacyLICMPass` | Legacy `licm` |
| `sinkRegion` / `hoistRegion` | Domtree walks |
| `canSinkOrHoistInst` | Per-opcode legality + memory checks |
| `isSafeToExecuteUnconditionally` | Speculation policy |
| `collectPromotionCandidates` | AST over MSSA-discovered accesses |
| `promoteLoopAccessesToScalars` | Actual scalar rewrite |
| `ICFLoopSafetyInfo` | Guaranteed-to-execute / memory-before queries |

`DEBUG_TYPE` is `"licm"`.

## 2. Why sink before hoist

The driver sinks first, then hoists, then promotes. Sinking removes instructions whose only users are outside; that can unlock hoist of their operands or shrink what promotion must consider. After promotion creates scalar PHIs, another round of sinking/hoisting may run when flags allow iteration—promotion often exposes pure arithmetic invariants.

## 3. MemorySSA clobber story for loads

For a load inside the loop, invariance is not “operands are invariant” alone. LICM asks whether any memory def in the loop **clobbers** that load’s location. MemorySSA provides the defining access / walker APIs; LICM caps expensive walks with clobber thresholds (see cl::opts near the top of LICM.cpp mentioning MemorySSA walker caps).

If a loop-carried store may alias the load, the load stays. TBAA, `noalias`, and `invariant.load` metadata tighten AA answers that MSSA consults.

## 4. `isSafeToExecuteUnconditionally`

Even an invariant load can fault. Hoisting to the preheader executes it even when the original path did not (zero-trip loop, or load under an in-loop condition).

Rough policy layers:

1. Prefer hoist when `SafetyInfo` says the instruction is guaranteed to execute whenever the loop is entered in a way that matches preheader semantics.
2. Allow speculation when the op cannot fault / has defined behavior on garbage (many integer ALU ops).
3. Refuse when a speculative fault or observable side effect would be introduced.

Rotated loops help (1): the preheader/guard already proved the body runs.

## 5. Sinking mechanics and LCSSA

`isNotUsedOrFoldableInLoop` returns true when every user is outside (PHI users attributed carefully) or the instruction can fold into an addressing mode inside the loop (`FoldableInLoop`). `sink` then clones the instruction into exit blocks as needed and rewrites users—often creating or updating LCSSA PHIs at exits (`cloneInstructionInExitBlock` handles call funclet bundles for EH).

Dedicated exits from LoopSimplify keep this from becoming a mess of shared exit blocks with mixed predecessors.

## 6. Promotion: AST still lives inside MSSA LICM

Promotion does **not** mean “old LICM used only AliasSetTracker and new uses only MemorySSA.” Current code:

1. `foreachMemoryAccess` via MSSA to find instructions.
2. Feed promotable loads/stores into an `AliasSetTracker`.
3. Keep must-alias mod sets; invalidate when non-promotable instructions alias the set.
4. Track “reads outside the set” — may restrict promotion to load-only scalarization.

Conditionally executed stores complicate AA tags: promoting inserts an **unconditional** exit store, so tags that only held on conditional paths cannot be trusted for that store (`addWithoutAATags` path).

## 7. What promotion produces

Conceptually for `*p` must-alias in the loop:

```llvm
preheader:
  %v0 = load i32, ptr %p
  br label %loop

loop:
  %v = phi i32 [ %v0, %preheader ], [ %v.next, %latch ]
  ; use %v instead of load; compute %v.next instead of store
  br i1 %c, label %loop, label %exit

exit:
  store i32 %v.lcssa, ptr %p
```

Exact SSA names depend on LCSSA and how many exits exist. Promotion refuses when exit/store insertion would be illegal (no dedicated exits, coroutine sinks, etc.—see early checks in the run function).

## 8. `llvm.invariant.start` / `invariant.end`

Helpers near the hoist path look for an `invariant.start` that:

- Dominates the load,
- Covers the byte range,
- Does not have escaping uses that would let the invariant be invalidated unnoticed.

If present, a load can be treated as invariant even when AA is otherwise conservative. Frontends and some library expansions emit these; ordinary C loops often rely on AA alone instead.

## 9. Arithmetic hoist extras

`hoistArithmetics` and related helpers reassociate GEPs, min/max patterns, and some fusions so that an invariant subexpression becomes a separate hoistable instruction. These are profitability/legality sugar on top of vanilla hoist—read them after the main sink/hoist loop makes sense.

## 10. LNICM vs LICM

`LNICMPass` runs in a loop-nest context (`LoopNest` analysis) with the same MSSA requirement. Nest mode adjusts sinking of PHI chains through outer levels (`LoopNestMode` in `isNotUsedOrFoldableInLoop`). For study, master single-loop `LICMPass` first.

## 11. Debug and remarks

`-debug-only=licm` prints hoist/sink decisions. Optimization remarks emitter threads through sink/hoist for user-visible miss reasons when enabled.

## 12. Common misconception checklist

- “LICM always hoists loads” — only with clobber-freedom + safety.
- “Promotion is the same as hoist load” — promotion also sinks stores to exits and creates a scalar cycle.
- “AliasSetTracker is dead” — still used inside candidate collection.
- “Unswitch is a LICM mode” — separate pass.
- “Bare `opt -passes=licm`” — use `loop-mssa(licm)` on new PM.

## 13. `canSinkOrHoistInst` reading guide

Open the function and skim cases: loads, stores (usually not hoisted as pure ops—promotion handles them), calls (nounwind/willreturn/memory attributes), GEPs, bitcasts, etc. MemorySSA arguments decide load legality; the boolean controlling “targeted for sink vs hoist” adjusts how users are considered.

When a call is hoisted, statistics `NumMovedCalls` increment—useful when grepping `-stats` builds.

## 14. Cordoning expensive MSSA walks

Top-of-file `cl::opt`s document caps on clobber queries and promotion effort. On huge loops LICM may give up precise clobber info for compile-time—manifesting as “missed hoist” rather than a crash. If a load “should” hoist and does not, check whether a threshold fired before blaming AA.

## 15. Stores are harder than loads

Hoisting a store to the preheader would execute it even when the loop never runs or when the store was conditional—usually wrong. Sinking stores to exits is exactly what promotion’s exit stores do, but only after proving the scalar captures all must-alias updates. Casual “hoist the store” is not a LICM feature.

## 16. Coroutine / EH footguns

Comments in the run path skip sinking stores from loops with coroutine suspends (incorrect landing). Sinking calls must thread funclet operand bundles when block colors exist. If you debug LICM on EH-heavy Windows IR, read `cloneInstructionInExitBlock` before accusing AA.

## 17. Rematerialization note

The file header mentions rematerialization of hoisted instructions under register pressure as a non-goal / separate concern. LICM may hoist aggressively; later codegen / remat passes can recompute cheap ops. Do not expect LICM itself to be the full register-pressure solution.

## 18. End-to-end study drill

1. Take `sum_scaled`; show load hoist.
2. Take `accumulate`; show promotion PHI + exit store.
3. Take conditional load; show refusal.
4. Run with `-debug-only=licm` and match one debug line to the code path (hoist vs sink).
5. Read `collectPromotionCandidates` until must-alias filtering is clear.

## 19. Related analyses checklist

| Analysis | Why LICM wants it |
|----------|-------------------|
| DominatorTree | Region walk, safety, sink placement |
| LoopInfo | Loop bounds |
| AAResults | Alias for memory |
| MemorySSA | Clobbers / promotion discovery |
| AssumptionCache | Deref / align assumptions |
| TargetLibraryInfo / TTI | Foldable addressing, costs |
| ScalarEvolution | Some arithmetic hoist / invariant helpers |
| OptimizationRemarkEmitter | User-facing miss reasons |

## 20. What LICM deliberately does not do

- Clone loops for invariant branches (unswitch).
- Change trip counts (unroll / indvars).
- Delete empty loops (loop-deletion).
- Prove general noalias between arbitrary pointers without AA metadata/attrs.

## 21. Sink vs hoist one-liner

**Hoist** moves to preheader (execute once before the loop). **Sink** moves to exits (execute after the last iteration that defined the value). Promotion is “hoist load + sink store” fused through a scalar PHI.
