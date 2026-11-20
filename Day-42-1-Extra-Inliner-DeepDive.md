# Day 42 Extra: Inliner Deep Dive

Companion to [Day-42-0-Inliner.md](Day-42-0-Inliner.md). Examples: [Day-42-2-Extra-Inliner-Examples.md](Day-42-2-Extra-Inliner-Examples.md).

## 1. Pass shape: CGSCC, not module-blind

[`Inliner.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/Inliner.cpp) implements a CGSCC pass. The `run` method receives:

- The current `LazyCallGraph::SCC`
- Module / function analysis managers
- `LazyCallGraph &`
- `CGSCCUpdateResult &`

That signature is the contract from Day 41: every successful inline that removes or changes a call edge must be reflected so the manager can split/merge SCCs.

Historically LLVM also had module-level and legacy call-graph SCC inliners. When reading old docs, prefer the NPM `InlinerPass` / `ModuleInlinerWrapperPass` wiring in PassBuilder as the source of truth for *when* inlining runs in `default<O#>` pipelines.

## Diagram

```
call site
   │
   ▼
InlineAdvisor::getAdvice / shouldInline
   │
   ▼
getInlineCost(Call, Threshold, ...)
   │
   ├─ InlineCost::getNever()  → skip
   ├─ cost > threshold        → skip (remark missed)
   └─ cost ≤ threshold        → InlineFunction(...)
                                    │
                                    ▼
                              update LazyCallGraph
                              CGSCCUpdateResult fields
```

### `DefaultInlineAdvisor`

Lives in [`InlineAdvisor.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineAdvisor.cpp). It applies attribute overrides first (`alwaysinline`, `noinline`), then delegates to the classical cost model.

### ML / replay

Optional advisors exist in the same analysis library area: they replace or wrap the decision function while still eventually calling `InlineFunction` for accepted sites. Useful for research and for bisecting “did the heuristic change?” by replaying a decision log. You do not need them for day-to-day compiler work, but know they plug in *above* cost, not inside `InlineFunction`.

## 3. Inside `getInlineCost`

[`InlineCost.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineCost.cpp) walks the callee as if parameters were substituted.

Important ideas:

1. **Budget / threshold** — starts from `-inline-threshold` and adjusts for hotness, `inlinehint`, size opts (`Os`/`Oz` pipelines use lower thresholds via PassBuilder).
2. **Per-instruction cost** — ordinary ALU is cheap; some ops query TTI.
3. **Simplification during analysis** — if arg is `i32 0`, a callee branch `icmp eq %arg, 0` is treated as foldable → cost rebate / dead-side not charged fully.
4. **Call penalty** — a callee that itself calls others is less attractive (unless those are cheap/known).
5. **Alloca** — static allocas may be OK; **dynamic allocas** are heavily penalized or forbidden depending on settings.
6. **Recursive** — calling a function already on the inline stack → never (unless alwaysinline forces careful handling).
7. **`InlineCost::getInvalid`** — illegal to inline (not merely unprofitable).

Reading tip: search for threshold cl::opt definitions and for attribute early-exits at the top of the cost computation.

## 4. Inside `InlineFunction`

[`InlineFunction.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/InlineFunction.cpp) is mechanical but subtle:

1. Clone callee basic blocks into the caller (ValueMapper / VMap).
2. Remap arguments to the call operands.
3. Turn `ret` into stores to an alloca (historical) or direct SSA reconnect into a continuation block—modern path produces a merge block with PHIs for returned values.
4. Split the call site’s basic block: pre-call code, inlined region, post-call continuation.
5. Propagate parameter attributes, UB-implying attributes, and metadata carefully.
6. Handle `byval`, `inalloca`, `preallocated`, `sret`, `musttail`, eh personality / `invoke` vs `call`.

After splicing, mem2reg/instcombine (later in the pipeline) clean up allocas introduced for returns or aggregates.

### `musttail` and `noduplicate`

- **`musttail`**: the inlined region must preserve the guarantee that the call was in tail position with matching signatures/attributes. Illegal inlines are rejected before mutation.
- **`noduplicate`**: must not create another copy of a call marked noduplicate; inlining a caller that contained such a call can be blocked, or inlining into multiple sites can be blocked—see checks inside cost/legality.

## 5. Attributes in practice

```llvm
define i32 @h(i32 %x) alwaysinline { ... }
define i32 @n(i32 %x) noinline { ... }
define i32 @o(i32 %x) optnone noinline { ... } ; optnone ⇒ don’t optimize/inline
define i32 @i(i32 %x) inlinehint { ... }       ; softer preference
```

`cold` on a function or call site biases against growing cold code. Profile metadata `!prof` on calls interacts with hot-callee thresholds.

## 6. Remarks and debug

```bash
# LLVM remarks through opt
opt -passes='cgscc(inline)' \
  -pass-remarks=inline -pass-remarks-missed=inline \
  -pass-remarks-analysis=inline \
  in.ll -S -o out.ll 2>remarks.txt

# Clang front-end facing
clang -O2 -Rpass=inline -Rpass-missed=inline -Rpass-analysis=inline file.c

# Verbose decision trace (needs +asserts LLVM build)
opt -passes='cgscc(inline)' -debug-only=inline in.ll -S

# Cost printer when registered in your version
opt -passes='cgscc(inline)' -print-inline-cost in.ll -S
```

Remarks answer “why missed?”: cost too high, noinline, recursive, incompatible personality, etc.

## 7. Interaction with SCC updates (worked micro-scenario)

Before:

```
SCC{leaf} , SCC{mid} , SCC{root}
root → mid → leaf
```

Inline `leaf` into `mid`: edge `mid→leaf` removed. If `leaf` had only that caller and becomes dead, later GlobalDCE removes it—but the inliner itself focuses on call sites + CG edges.

Inline `mid` into `root`: edge `root→mid` removed. SCCs stay singletons; update result may still mark analyses invalidated for `root`.

Recursive case:

```
SCC{f,g} with f→g→f
```

Inlining `g` into `f` may leave `f→f` or remove the cycle. `CGSCCUpdateResult` must present the new SCC set; continuing to iterate with a freed SCC pointer would crash. This is why the inliner is the textbook client of Day 41’s update protocol.

## 8. What inlining unlocks downstream

After a constant-arg inline:

- InstCombine folds icmps.
- SCCP/IPSCCP may already have done similar interprocedurally—but inlining exposes CFG of the callee to *intraprocedural* passes immediately.
- SROA/mem2reg see aggregates passed by value more clearly.
- Vectorizers see larger loop nests if a loop helper was inlined.

That is why PassBuilder runs inlining interleaved with function simplification, not once at the end.

## 9. Threshold experiments worth doing

| Knob | Expectation |
| --- | --- |
| `-inline-threshold=0` | Only alwaysinline / tiny sites |
| `-inline-threshold=1000` | Aggressive growth |
| `Oz` pipeline | Fewer inlines |
| Add `inlinehint` | Borderline sites flip to yes |
| Add profile hot count | Same |

Always record both IR size (`llvm-size` / line count of `.ll`) and remarks.

## 10. Source reading checklist

1. Attribute short-circuits in cost/advisor.
2. Constant propagation bonuses while scanning callee.
3. `InlineFunction` handling of return blocks.
4. Calls into LazyCallGraph update helpers after a successful inline.
5. PassBuilder: where `ModuleInlinerWrapperPass` / `cgscc(inline)` sits relative to `function-attrs` and IPSCCP.

## 11. Common misconceptions

- **“Inliner = InlineFunction.”** No—without cost/advisor you’d inline everything legal and explode.
- **“Cost is exact cycle count.”** It is a heuristic correlated with size/latency, not a simulator.
- **“After inline the callee is deleted.”** Only if linkage allows and no remaining uses (refs, other calls, `llvm.used`).
- **“O0 IR is fine for experiments.”** `optnone` blocks you; strip it or emit with `-O1 -Xclang -disable-llvm-passes`.
