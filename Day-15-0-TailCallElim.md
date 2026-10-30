# Day 15: Tail Call Elimination

LLVM’s scalar **tail call elimination** pass does two jobs: (1) **TRE**—turn self-recursive tail calls into loops in the entry block (with optional accumulators), and (2) **mark** eligible calls with the `tail` marker so the backend can emit sibling/tail calls. Pass name: `tailcallelim` (`TailCallElimPass`). Implementation: [TailRecursionElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/TailRecursionElimination.cpp).

Deep dive: [Day-15-1-Extra-TailCallElim-DeepDive.md](Day-15-1-Extra-TailCallElim-DeepDive.md). Examples: [Day-15-2-Extra-TailCallElim-Examples.md](Day-15-2-Extra-TailCallElim-Examples.md).

## 1. Why this exists

Naive recursion consumes stack frames. A call in **tail position**—the caller immediately returns the callee’s result—can reuse the frame if ABI and IR rules allow. Mid-end TRE rewrites obvious self-recursion into an iterative loop so even targets without perfect tail-call codegen avoid stack growth. Separately, marking `tail` on calls communicates to SelectionDAG/GlobalISel that a sibling call is legal when the backend checks the ABI.

Without this pass, recursive functional-style code stays recursive until a backend sibling-call optimization (if any), and many self-recursive helpers never become loops.

## Diagram

```text
   before                              after tailcallelim

   define i32 @fact(i32 %n)            define i32 @fact(i32 %n, i32 %acc)
     ...                                  loop:
     %r = call @fact(%n-1)                  %r = ... compute in %acc ...
     %m = mul %r, %n                        br label %loop
     ret %m                               ; the recursive call is gone
                                            ; the mul moved into the accumulator
```

A call in tail position is one whose result is returned with nothing to do after it. The recursive factorial is not in tail position until the multiply is turned into an extra argument. After that, the call is a branch back to the top, and no stack frame accumulates.

## 2. The idea

### TRE (tail recursion elimination)

If a call invokes the **same function**, sits in **tail position** (returned value forwarded, or void), and the frame has no blocking escapes/dynamic allocas, rewrite:

- Move the call into a loop structure based at entry (PHI formals for updated arguments).
- Optionally introduce an **accumulator PHI** for associative combining ops on the returned value (`add`/`mul`/bitwise, including some unary recurrence forms).

### Marking `tail`

Walk calls; if no escaping alloca-derived pointers reach the call in a bad way, and other checks pass, set `tail` on the `CallInst`. That is not the same as TRE and not the same as `musttail`.

### Distinguish neighbors

| Mechanism | Where | Meaning |
|-----------|--------|---------|
| TRE / `tail` mark | Mid-end `tailcallelim` | Loopify self-recursion; mark calls |
| Backend sibling call | Codegen | Reuse stack for eligible `tail` calls |
| `musttail` | IR verifier rule | Guaranteed tail call; strict matching |

## 3. Where it sits in the -O2/-O3 pipeline

In PassBuilder’s optimization pipeline, `TailCallElimPass` runs in the function pipeline (historically near SimplifyCFG / after some PGO-sized opts depending on revision—read `PassBuilderPipelines.cpp` for the exact neighbor list in your tree). It is a scalar function pass, not a codegen pass.

| Item | Value |
|------|--------|
| Pass name | `tailcallelim` |
| Class | `TailCallElimPass` |
| Engine class | `TailRecursionEliminator` |
| Debug | `-debug-only=tailcallelim` |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=tailcallelim -S file.ll -o file.tce.ll
```

## 4. Core concepts

### Tail position

A call whose result is returned (possibly through trivial bitcasts/PHIs as the pass understands) or a void call followed by `ret void`, with no interfering observable work after the call.

### Self-recursive TRE

`findTRECandidate` looks for a recursive call suitable for elimination; `eliminateCall` rewrites to a loop. `canTRE` returns false if any **dynamic alloca** exists (stack grows per iteration—not implemented to shrink before recurse).

### Escaping allocas / byval

`AllocaDerivedValueTracker` walks uses of allocas and byval arguments. Escapes can block `tail` marking. Byval at a call copies memory and has special TRE handling (temporary allocas / copies) when transforming recursion—byval can still **block** cases; read `markTails` / `eliminateCall` carefully.

### Accumulators

When the return value is `op(recursive_call(…), other)` for suitable associative `op`, TRE inserts an accumulator PHI initialized with the identity and folds the operation into the loop (`insertAccumulator`, `canTransformAccumulatorRecursion`).

### Attribute escape hatch

`disable-tail-calls` function attribute skips the pass work. Varargs functions skip TRE (PHI’ing args is wrong) but may still attempt marking depending on path—`eliminate` returns early after `markTails` for varargs.

## 5. The algorithm

`TailRecursionEliminator::eliminate`:

1. Bail if `disable-tail-calls`.
2. `markTails(F, …)` — set `tail` where legal.
3. If varargs → return (after marking attempt).
4. If `!canTRE(F)` (dynamic alloca) → return.
5. For each block, `processBlock` → `findTRECandidate` → `eliminateCall` to build entry loop / PHIs / accumulators.
6. `cleanupAndFinalize`.

Analyses: TTI, AA, ORE, optional BFI/PSI for remarks/coldness, DomTreeUpdater for CFG edits.

## 6. Where to read the code

- [llvm/lib/Transforms/Scalar/TailRecursionElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/TailRecursionElimination.cpp)
- [llvm/include/llvm/Transforms/Scalar/TailRecursionElimination.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/TailRecursionElimination.h)

Symbols: `TailCallElimPass::run`, `TailRecursionEliminator::eliminate`, `markTails`, `canTRE`, `findTRECandidate`, `eliminateCall`, `insertAccumulator`, `canTransformAccumulatorRecursion`, `AllocaDerivedValueTracker`.

## 7. Worked before/after

**C** (`fact_tre.c`):

```c
int fact_acc(int n, int acc) {
  if (n <= 1)
    return acc;
  return fact_acc(n - 1, acc * n);
}
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes fact_tre.c -o fact_tre.ll
opt -passes=tailcallelim -S fact_tre.ll -o fact_tre.out.ll
```

**Before:** recursive `call` in tail position updating `n` and `acc`.

**After (shape):** entry block gains a loop with PHIs for `n` and `acc`; recursive call removed; branch back to entry; no growing stack for this function’s recursion.

Accumulator-style without explicit acc:

```c
int sum(int n) {
  if (n <= 0) return 0;
  return n + sum(n - 1);
}
```

TRE may invent an accumulator for the `add` if checks pass.

## 8. Reading order for today

1. File header / `DEBUG_TYPE`.
2. `markTails` + `AllocaDerivedValueTracker`.
3. `canTRE` (dynamic alloca).
4. `findTRECandidate` + `eliminateCall`.
5. Accumulator helpers.
6. Deep dive + examples.

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes fact_tre.c -o fact_tre.ll
opt -passes=tailcallelim -S fact_tre.ll -o out.ll
opt -passes=tailcallelim -debug-only=tailcallelim -S fact_tre.ll 2>&1 | less
# Compare backend sibling calls:
clang -O2 -S fact_tre.c -o fact_tre.s
```

## 10. Pitfalls and invariants

- Dynamic allocas block TRE (`canTRE`).
- Escaping pointers from allocas block `tail` marking.
- Varargs: no TRE.
- `musttail` is a stricter IR feature—do not confuse with optional `tail`.
- Byval and readonly interactions are subtle during TRE argument updates.
- TRE changes CFG (loops); preserve DT/PDT via `DomTreeUpdater`.
- `disable-tail-calls` attribute disables the pass’s transforms.

## 11. Neighbors

- **Before:** inlining may create or destroy TRE opportunities; simplifycfg cleans returns.
- **After:** loop passes see a loop instead of recursion; codegen reads `tail`.
- **Related:** codegen tail/sibling call lowering; `musttail` for guaranteed cases (coroutines, some ABI shims).

## 12. Check yourself

**Q1.** Pass name?
**A1.** `tailcallelim`.

**Q2.** Two responsibilities?
**A2.** TRE (recursion→loop) and marking eligible calls `tail`.

**Q3.** What does `canTRE` reject?
**A3.** Functions with dynamic allocas (among the checks tied to that helper).

**Q4.** Is `tail` the same as `musttail`?
**A4.** No—`musttail` is mandatory/strict; `tail` is a hint for eligible calls.

**Q5.** Debug flag?
**A5.** `-debug-only=tailcallelim`.

**Q6.** Why accumulators?
**A6.** To TRE patterns like `return n + f(n-1)` by carrying a running associative op in a PHI.

> Tip: Implementation details are in `Day-15-1-Extra-TailCallElim-DeepDive.md`. Runnable cases are in `Day-15-2-Extra-TailCallElim-Examples.md`.

## Appendix A: What “sibling call” means in codegen

A sibling call jumps to another function while reusing the incoming stack frame layout when the callee’s stack argument area fits the caller’s. The mid-end `tail` marker is necessary but not sufficient—the backend still checks ABI, stack sizes, and callee.

## Appendix B: Stack realignment and dynamic alloca

Dynamic `alloca` changes the stack pointer in ways TRE does not currently rewind each iteration. Hence `canTRE` requires all allocas static. Realignment and special calling conventions add more backend constraints beyond this pass.

## Appendix C: Checklist before claiming TRE fired

1. Same function callee (`CallInst` to `F`).
2. Result returned (or void tail).
3. No dynamic alloca in `F`.
4. Not varargs.
5. No `disable-tail-calls`.
6. Escapes do not block the transform path you care about (`tail` mark vs TRE have related but separate checks).

If any box fails, debug with `-debug-only=tailcallelim` before blaming the optimizer.

## Appendix D: Entry-block loop shape

TRE typically introduces PHIs in the entry (or a header dominated appropriately) for each updated argument. The recursive call site becomes a branch back that feeds new PHI incomings. Returns in base cases remain `ret`. After TRE, loop passes (LICM, indvars) may further clean the function—read post-TRE IR as a loop, not as recursion.

## Appendix E: Cold callsites and marking

With profile summary + BFI, `markTails` can treat cold callsites specially for remarks / aggressiveness. Expect-only builds without entry counts may pass null BFI into `eliminate` (see `TailCallElimPass::run`). TRE still runs; marking heuristics simply have less profile input.

## Appendix F: One-line summary

`tailcallelim` = mark safe `tail` calls + rewrite self-recursive tail calls into entry loops (with accumulators when profitable).

## Appendix G: Manual IR checklist for a first TRE test

Start from a two-block function: compare → return base; else compute args → recursive call → ret. No allocas, no varargs, `fastcc` optional. Run `tailcallelim`, then `verify`. If TRE fails, add `-debug-only=tailcallelim` before complicating the IR with byval or aggregates.

## Appendix H: Debug-only reminder

```bash
opt -passes=tailcallelim -debug-only=tailcallelim -S t.ll
```
