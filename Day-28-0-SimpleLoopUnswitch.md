# Day 28: Simple Loop Unswitch (`simple-loop-unswitch`)

Loop unswitch pulls a loop-invariant branch (or switch) out of a loop by duplicating the loop for each successor. The “simple” pass replaces the older `LoopUnswitch` and is MemorySSA-aware. Deep dive: [Day-28-1-Extra-SimpleLoopUnswitch-DeepDive.md](Day-28-1-Extra-SimpleLoopUnswitch-DeepDive.md). Examples: [Day-28-2-Extra-SimpleLoopUnswitch-Examples.md](Day-28-2-Extra-SimpleLoopUnswitch-Examples.md).

## Why this exists

Invariant conditions inside loops waste work every iteration:

```c
for (int i = 0; i < n; ++i) {
  if (mode)  /* mode invariant */
    a[i] += 1;
  else
    a[i] -= 1;
}
```

Each iteration re-branches on `mode`. Unswitching creates two loops specialized to `mode==true` and `mode==false`, with a single branch outside. That removes the inner branch, enables further LICM/vectorize/idiom passes on simpler bodies, and can improve prediction.

The cost is **code size**: the loop body is cloned. Non-trivial unswitch is therefore gated by a cost model (`unswitch-threshold`).

## Diagram

```text
   for (...) {                         if (cond) {
     if (cond)          unswitch         for (...) { A }
       A;              -------->       } else {
     else                               for (...) { B }
       B;                              }
   }

   `cond` does not change in the loop,
   so the branch is invariant
```

SimpleLoopUnswitch hoists an invariant branch out of the loop and duplicates the loop into the two arms. Each copy then has straight-line control where the branch used to be, so LICM and the vectorizer see a simpler body. The pass will not unswitch a branch that is trivial to fold another way, and it refuses when duplicating the body would blow the size threshold. Non-trivial unswitching freezes a possibly poison condition rather than branching on poison.

## The idea

### Non-trivial unswitch

For an invariant `br i1 %cond` inside the loop where both successors stay in (or meaningfully relate to) the loop:

1. Clone the entire loop.
2. In the original, force the branch one way; in the clone, the other way (or rewrite CFG so each copy is straight-line w.r.t. that condition).
3. Insert a branch on `%cond` in the preheader selecting which loop copy to enter.

### Trivial unswitch

When one successor of an invariant branch **exits** the loop and the other continues, you may not need a full clone: rewrite control so the exit edge is hoisted. This is cheaper and often done first.

### Freeze and poison

If `%cond` can be poison, two cloned loops that each branch on the “same” SSA value could theoretically observe different poison-related UB behaviors. With `freeze-loop-unswitch-cond` enabled (default true in modern trees), unswitch inserts `freeze` on the condition before branching to the copies so both sides see a fixed arbitrary value of the same bits.

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| Pass name | `simple-loop-unswitch` |
| File | [llvm/lib/Transforms/Scalar/SimpleLoopUnswitch.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SimpleLoopUnswitch.cpp) |
| Debug | `-debug-only=simple-loop-unswitch` (DEBUG_TYPE; older docs sometimes say `loop-unswitch`) |
| Cost | `-unswitch-threshold` (default commonly 50) |
| Form | Non-trivial unswitch expects **LoopSimplify + LCSSA** |
| Analysis | Updates **MemorySSA** |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes u.c -o u.ll
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -S u.ll
```

## Core concepts

### Loop-invariant condition

The branch condition must be invariant w.r.t. the loop (SCEV/`isLoopInvariant` / no def inside the loop). Partial invariance injection is an advanced optional feature (`simple-loop-unswitch-inject-invariant-conditions`)—skip on first read.

### Cost model

Non-trivial unswitch estimates the size of duplicated code. If cost exceeds `unswitch-threshold`, skip. Nested unswitches multiply size (`2^k` copies risk)—the pass accounts for exponential blow-up.

### PGO / profile

Branch weights and profile estimates (`simple-loop-unswitch-estimate-profile`, PGO metadata) can bias which conditions are worth unswitching (hot loop + biased invariant branch is more valuable).

### MSSA threshold

`simple-loop-unswitch-memoryssa-threshold` limits how hard the pass works with MemorySSA on large regions—another compile-time guard.

## The algorithm (non-trivial sketch)

1. Ensure simplified loop + LCSSA.
2. Find a conditional branch/switch whose condition is loop-invariant.
3. Classify trivial vs non-trivial.
4. For non-trivial: compute duplication cost; compare to threshold.
5. Insert `freeze` on the condition when required.
6. Clone loop CFG, fix phis/LCSSA, redirect preheader through the invariant branch.
7. Update MemorySSA via updater APIs; preserve DT/LI invariants.

## Where to read the code

1. [SimpleLoopUnswitch.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SimpleLoopUnswitch.cpp) — top cl::opt block, then trivial unswitch helpers, then non-trivial clone path, then freeze insertion comments near `freeze-loop-unswitch-cond`.
2. Pass registry entry `simple-loop-unswitch` in [PassRegistry.def](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Passes/PassRegistry.def).
3. Tests under `llvm/test/Transforms/SimpleLoopUnswitch/`.

## Worked before / after

Before (shape): one loop, inner `br i1 %mode`.

After: 

```llvm
  %mode.fr = freeze i1 %mode
  br i1 %mode.fr, label %loop.a.preheader, label %loop.b.preheader
```

Two specialized loops follow without the inner mode branch.

## Reading order

1. Trivial vs non-trivial distinction in this note.
2. Freeze section in the .cpp file.
3. Cost threshold logic.
4. DeepDive + Examples.

## Hands-on

```bash
cat > unswitch.c <<'EOF'
void f(int *a, int n, int mode) {
  for (int i = 0; i < n; ++i) {
    if (mode)
      a[i] += 1;
    else
      a[i] -= 1;
  }
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes unswitch.c -o unswitch.ll
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -S unswitch.ll -o unswitch.out.ll
```

Search for a second loop header / frozen condition.

## Pitfalls

- **Skipping LoopSimplify/LCSSA** breaks non-trivial unswitch assumptions.
- **Huge bodies** silently do nothing when over threshold—lower `-unswitch-threshold` only for experiments.
- **Poison**: without freeze, duplicated branches on poison are a correctness footgun; do not “optimize away” freezes casually in tests.
- **Debug flag name**: use `simple-loop-unswitch`, not the legacy `loop-unswitch` string, unless your build still defines that.

## Neighbors

- LoopRotate (Day 26) — CFG prep.
- LICM — often after unswitch cleans the body.
- Loop vectorize — simpler bodies vectorize better.
- Day 37 MemorySSA — what the updater maintains.

## Check yourself

1. **Q:** Trivial vs non-trivial?  
   **A:** Trivial hoists an exit-edge style invariant branch without full duplication; non-trivial clones the loop for each side.

2. **Q:** Why `freeze`?  
   **A:** So both copies observe one concrete condition value if the original could be poison.

3. **Q:** Pass and threshold flag?  
   **A:** `simple-loop-unswitch`, `-unswitch-threshold`.

4. **Q:** Required form for non-trivial?  
   **A:** LoopSimplify + LCSSA.

5. **Q:** What analysis does it update for memory?  
   **A:** MemorySSA.

## Code-size blow-up intuition

Unswitching a nest of three independent invariant branches can clone the loop up to eight ways. The cost model multiplies estimated body size by the branching factor and compares to `unswitch-threshold`. Nested loops make this worse: unswitching an outer invariant condition duplicates the entire inner nest. When teaching, nest two small invariant ifs and watch either dual unswitch or a threshold refusal as you lower `-unswitch-threshold`.

## Partial invariance (advanced)

`simple-loop-unswitch-inject-invariant-conditions` tries to rewrite conditions so more branches become invariant. This can create new unswitch opportunities but also risks churn. Skip on a first read; revisit after you can manually spot trivial vs non-trivial cases in IR.

## Guards and implicit null checks

Options around `simple-loop-unswitch-guards` and dropping non-trivial implicit null checks interact with LLVM’s guard/`make.implicit` machinery. They matter for languages that lower checks specially. For C study notes, know the flags exist so debug strings do not surprise you.

## MSSA verification recipe

```bash
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -verify-memoryssa -S in.ll
```

If verification fails after unswitch, the bug is almost always in MemoryAccess cloning or MemoryPhi edge updates—excellent MSSA practice (Day 37).

## Freeze and select

Not only `br i1 %cond` but also branches fed by `select` or frozen selects appear. Comments near freeze insertion discuss selecting a defined condition before duplicating control flow that would otherwise branch on poison. Compare IR with `-freeze-loop-unswitch-cond=true/false` on an i1 argument condition.

### Additional study points (1)

Unswitching a nest of three independent invariant branches can clone the loop up to eight ways. The cost model multiplies estimated body size by the branching factor and compares to `unswitch-threshold`. Nested loops make this worse: unswitching an outer invariant condition duplicates the entire inner nest. When teaching, nest two small invariant ifs and watch either dual unswitch or a threshold refusal as you lower `-unswitch-threshold`.
`simple-loop-unswitch-inject-invariant-conditions` tries to rewrite conditions so more branches become invariant. This can create new unswitch opportunities but also risks churn. Skip on a first read; revisit after you can manually spot trivial vs non-trivial cases in IR.
Options around `simple-loop-unswitch-guards` and dropping non-trivial implicit null checks interact with LLVM’s guard/`make.implicit` machinery. They matter for languages that lower checks specially. For C study notes, know the flags exist so debug strings do not surprise you.
```bash
- opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -verify-memoryssa -S in.ll
```
If verification fails after unswitch, the bug is almost always in MemoryAccess cloning or MemoryPhi edge updates—excellent MSSA practice (Day 37).
Not only `br i1 %cond` but also branches fed by `select` or frozen selects appear. Comments near freeze insertion discuss selecting a defined condition before duplicating control flow that would otherwise branch on poison. Compare IR with `-freeze-loop-unswitch-cond=true/false` on an i1 argument condition.

```bash
# revisit with debug (session 1)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

### Additional study points (2)

Unswitching a nest of three independent invariant branches can clone the loop up to eight ways. The cost model multiplies estimated body size by the branching factor and compares to `unswitch-threshold`. Nested loops make this worse: unswitching an outer invariant condition duplicates the entire inner nest. When teaching, nest two small invariant ifs and watch either dual unswitch or a threshold refusal as you lower `-unswitch-threshold`.
`simple-loop-unswitch-inject-invariant-conditions` tries to rewrite conditions so more branches become invariant. This can create new unswitch opportunities but also risks churn. Skip on a first read; revisit after you can manually spot trivial vs non-trivial cases in IR.
Options around `simple-loop-unswitch-guards` and dropping non-trivial implicit null checks interact with LLVM’s guard/`make.implicit` machinery. They matter for languages that lower checks specially. For C study notes, know the flags exist so debug strings do not surprise you.
```bash
- opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -verify-memoryssa -S in.ll
```
If verification fails after unswitch, the bug is almost always in MemoryAccess cloning or MemoryPhi edge updates—excellent MSSA practice (Day 37).
Not only `br i1 %cond` but also branches fed by `select` or frozen selects appear. Comments near freeze insertion discuss selecting a defined condition before duplicating control flow that would otherwise branch on poison. Compare IR with `-freeze-loop-unswitch-cond=true/false` on an i1 argument condition.

```bash
# revisit with debug (session 2)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

## More on trivial vs non-trivial decision points

Trivial unswitch is attempted aggressively because it is cheap. Non-trivial unswitch asks cost first. A branch that looks “almost trivial” (one edge exits a child loop but not the parent) can fall into non-trivial cloning of a subset of the nest. When IR surprises you, map each edge as “exits L?” for the loop L being processed—SimpleLoopUnswitch is loop-scoped, not function-scoped.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
