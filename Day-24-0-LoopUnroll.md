# Day 24: Loop Unroll (`loop-unroll`)

Loop unroll clones the loop body a chosen number of times, folds exit tests where trip counts allow, and optionally builds a remainder (epilogue/prologue) loop for runtime trip counts. Policy lives in [LoopUnrollPass.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopUnrollPass.cpp); the transform is `UnrollLoop` in [LoopUnroll.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopUnroll.cpp). Extra: [Day-24-1-Extra-LoopUnroll-DeepDive.md](Day-24-1-Extra-LoopUnroll-DeepDive.md). Examples: [Day-24-2-Extra-LoopUnroll-Examples.md](Day-24-2-Extra-LoopUnroll-Examples.md).

## 1. Why this exists

A tight loop pays branch overhead and hides ILP:

```c
for (int i = 0; i < 4; ++i)
  a[i] = b[i] + 1;
```

Full unroll turns this into straight-line code—no latch, often constant GEPs. Partial / runtime unroll reduces branch frequency and exposes vectorization/ILP while keeping a smaller loop. The cost is code size; TTI and thresholds decide when size wins.

## Diagram

```text
   for (i = 0; i < 4; i++)           i = 0
     a[i] = i;                       a[0] = 0
                                     a[1] = 1
   unroll count 4, no remainder:     a[2] = 2
                                     a[3] = 3

   for (i = 0; i < n; i++)           if n is not a multiple of 4:
     a[i] = i;                       a prologue or epilogue loop
                                     runs the leftover iterations,
                                     and the main loop steps by 4
```

Full unrolling copies the body and deletes the branch when the trip count is a small constant. Runtime unrolling peels off a remainder so the unrolled body can assume a multiple of the unroll factor. The pass estimates code size against a threshold; a large body with a huge trip count stays rolled.

## 2. The idea

### Full unroll

When SCEV gives a small constant trip count (or pragma forces full), clone the body `TripCount` times and remove the backedge → `LoopUnrollResult::FullyUnrolled`.

### Partial unroll

Clone `Count` times but keep a loop with a reduced trip count.

### Runtime unroll

Trip count unknown at compile time. Create a **remainder** strategy (commonly epilogue; prologue variants exist via runtime remainder helpers) so the main unrolled loop runs `floor(N/Count)*Count` iterations and a remainder loop handles `N % Count`. Implemented with `UnrollRuntimeLoopRemainder` then `UnrollLoop`.

### Peel

Peeling runs first iterations (or last) separately via [LoopPeel](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopPeel.h) / `peelLoop`—related, often decided in the same pass heuristics, but not the same as multi-copy unroll. Explicit peel and unroll-count conflict checks exist in the pass.

### Unroll-and-jam

**Different pass:** `loop-unroll-and-jam`. Outer unroll + jam inner statements together. Do not confuse with `loop-unroll`. Pragma `llvm.loop.unroll_and_jam` can suppress ordinary unroll so the jam pass can act.

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `loop-unroll` |
| Driver | [LoopUnrollPass.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopUnrollPass.cpp) |
| Transform | [LoopUnroll.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopUnroll.cpp) — `UnrollLoop` |
| API header | [UnrollLoop.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/UnrollLoop.h) |
| Debug | `-debug-only=loop-unroll` |
| Needs | LoopSimplify form; PreserveLCSSA true in normal pipelines |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa,loop-unroll' -S file.ll -o file.unroll.ll
```

Knobs (selection): `-unroll-count=N`, `-unroll-threshold=N`, `-unroll-runtime`, `-unroll-allow-partial`, plus aggressive/default threshold variants in LoopUnrollPass.cpp.

## 4. Core concepts

### TTI `getUnrollingPreferences`

`gatherUnrollingPreferences` starts from defaults then calls `TTI.getUnrollingPreferences(L, SE, UP, &ORE)`. Targets set partial/runtime/full thresholds, default runtime count (often 8), and architecture-specific biases.

### Cost model

`UnrollCostEstimator` + `computeUnrollCount` weigh rolled size vs unrolled size against `UP.Threshold` / partial / full-max counts. Pragma paths use larger budgets (`pragma-unroll-threshold`).

### Metadata / pragmas

Clang `#pragma unroll` / `clang loop unroll(...)` become loop metadata:

- `llvm.loop.unroll.full`
- `llvm.loop.unroll.enable`
- `llvm.loop.unroll.count`
- `llvm.loop.unroll.runtime.disable`
- follow-up metadata helpers in UnrollLoop.h (`LLVMLoopUnrollFollowup*`)

`UnrollPragmaInfo` parses these for priority over heuristics.

### Remainder loop

After runtime remainder setup, an epilogue (or prologue) loop handles leftover iterations. `UnrollLoop` can return the remainder via an out-parameter. Follow-up metadata can mark remainder vs unrolled bodies for later passes.

### Form requirements

`tryToUnrollLoop` remarks `NotInLoopSimplifyForm` when the loop is not simplified. `UnrollLoop` itself requires a preheader and latch; refuses address-taken headers and non-cloneable bodies (`indirectbr`).

## 5. The algorithm

Policy (`tryToUnrollLoop` / `computeUnrollCount` sketch):

1. Reject non-simplified loops; respect unroll-and-jam pragmas.
2. Gather UP from TTI + cl::opt overrides + profile.
3. Query SCEV trip count / max / multiple; build cost estimator.
4. Priority roughly: explicit `-unroll-count` / pragma count / full pragma → peel → partial → runtime (if enabled) → give up.
5. If peeling selected (`PeelCount`), call `peelLoop` and return.
6. If runtime needed, `UnrollRuntimeLoopRemainder` then `UnrollLoop`.
7. Else `UnrollLoop` with chosen `UnrollLoopOptions` (`Count`, `Force`, `Runtime`, `UnrollRemainder`, SCEV budget, …).
8. `simplifyLoopAfterUnroll` cleanup; preserve LCSSA / fix outer LCSSA after full unroll when exit PHIs existed.

Transform (`UnrollLoop`): clone blocks `Count` times, stitch latches, fold constant exit tests, handle breakout trips for partial exits, optionally completely remove the loop.

## 6. Where to read the code

1. [LoopUnrollPass.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopUnrollPass.cpp) — preferences, pragmas, `tryToUnrollLoop`.
2. [UnrollLoop.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/UnrollLoop.h) — `UnrollLoop`, `UnrollRuntimeLoopRemainder`, `LoopUnrollResult`.
3. [LoopUnroll.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopUnroll.cpp) — cloning mechanics.
4. [LoopUnrollRuntime.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopUnrollRuntime.cpp) — remainder scaffolding (sibling file).
5. Tests: `llvm/test/Transforms/LoopUnroll/`.

## 7. Worked before/after

```c
void add4(int *a, int *b) {
  for (int i = 0; i < 4; ++i)
    a[i] = b[i] + 1;
}
```

```bash
opt -passes='loop-simplify,lcssa,loop-unroll' -S add4.ll
# or force: opt -passes='loop-simplify,lcssa,loop-unroll' -unroll-count=4 ...
```

**After full unroll:** four store/load sequences (possibly further folded); no loop latch back to a 4-trip header.

Runtime example (`n` unknown) with `-unroll-runtime` / target UP.Runtime: main loop steps by `Count`, remainder loop handles the tail.

## 8. Reading order for today

1. Full vs partial vs runtime vs peel vs unroll-and-jam (five different things).
2. `gatherUnrollingPreferences` + pragma priority.
3. `UnrollLoop` preconditions (preheader, latch, clone safety).
4. Day 24 DeepDive; Day 24 Examples.
5. Day 25 deletion (sometimes removes empty leftovers).

## 9. Hands-on commands

```bash
cat > unroll.c <<'EOF'
void add4(int *restrict a, int *restrict b) {
  for (int i = 0; i < 4; ++i)
    a[i] = b[i] + 1;
}
void addn(int *restrict a, int *restrict b, int n) {
  for (int i = 0; i < n; ++i)
    a[i] = b[i] + 1;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes unroll.c -o unroll.ll
opt -passes='loop-simplify,lcssa,loop-unroll' -S unroll.ll -o unroll.out.ll
opt -passes='loop-simplify,lcssa,loop-unroll' -unroll-runtime -unroll-count=4 -S unroll.ll -o unroll.rt.ll
# opt -debug-only=loop-unroll -passes='loop-simplify,lcssa,loop-unroll' -S unroll.ll
```

## Full vs partial in IR terms

**Full:** after transform, `LoopUnrollResult::FullyUnrolled`—no backedge for that loop; body copies inlined into the linear CFG (possibly still inside an outer loop).

**Partial:** still a loop; IV steps as if `Count` iterations were fused; exit tests adjusted using `BreakoutTrip` / trip multiples from SCEV.

Reading IR: count how many times a unique call or store pattern repeats between latch branches—that is your effective unroll factor.

## Remainder loop anatomy (runtime)

Typical epilogue shape after successful runtime unroll with count 4:

1. Compute `n / 4` and `n % 4` (or equivalent IV bounds).
2. Main loop: four copies of the body; IV += 4.
3. Remainder loop: original body, runs 0–3 times.

Prologue remainder flips which fragment runs first. Either way, **two loops** after one “unroll” pass is expected.

## Cost model intuition

`UnrolledSize ≈ RolledSize * Count` (with UP tweaks for eliminated branches / constants). If that exceeds `UP.Threshold`, refuse. Pragma paths substitute a larger threshold so source hints beat timid defaults—still not infinite (`pragma-unroll-full-max-iterations`).

## 10. Pitfalls and invariants

- Expecting unroll without LoopSimplify — remark / no-op.
- Confusing `loop-unroll-and-jam` with `loop-unroll`.
- Ignoring code-size thresholds — small `-unroll-threshold` prevents partial/runtime.
- Assuming pragma always wins size checks — full pragma still has max iteration guards.
- Forgetting remainder loops when reading runtime-unrolled IR — two loops is success, not failure.
- Address-taken loop headers — `UnrollLoop` refuses; try rotate first.

## 11. Neighbors

- **LoopSimplify / LCSSA** — required form.
- **LoopRotate** — helps runtime/full patterns.
- **LICM** — often run before; cleaner body → better cost.
- **LoopPeel** utils — peel path inside the same decision framework.
- **LoopUnrollAndJam** — separate pass.

## 12. Check yourself

1. **Q:** Where is policy vs transform?  
   **A:** Policy in LoopUnrollPass.cpp; transform `UnrollLoop` in LoopUnroll.cpp.

2. **Q:** What is runtime unroll’s extra CFG?  
   **A:** A remainder (epilogue/prologue) loop for `N % Count` iterations.

3. **Q:** TTI hook name?  
   **A:** `getUnrollingPreferences`.

4. **Q:** Pragma metadata examples?  
   **A:** `llvm.loop.unroll.full`, `.enable`, `.count`, `.runtime.disable`.

5. **Q:** Is unroll-and-jam this pass?  
   **A:** No — `loop-unroll-and-jam` is separate.

6. **Q:** Debug type string?  
   **A:** `loop-unroll` (`-debug-only=loop-unroll`).
## Forced count vs heuristic

```bash
opt -passes='loop-simplify,lcssa,loop-unroll' -unroll-count=2 -S addn.ll
```

forces a factor when legal. Without it, `computeUnrollCount` may pick no unroll, partial, or runtime depending on TTI—record knobs when comparing dumps across targets.

Targets differ: the same `-unroll-count` legality still depends on clone safety and LoopSimplify form.

> Tip: Deep dive in `Day-24-1-Extra-LoopUnroll-DeepDive.md`. Examples in `Day-24-2-Extra-LoopUnroll-Examples.md`.
