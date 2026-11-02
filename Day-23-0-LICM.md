# Day 23: Loop Invariant Code Motion (`licm`)

LICM hoists loop-invariant instructions to the preheader, sinks instructions only used outside the loop to exits, and promotes must-alias memory into scalars (load in preheader / store at exits). Extra: [Day-23-1-Extra-LICM-DeepDive.md](Day-23-1-Extra-LICM-DeepDive.md). Examples: [Day-23-2-Extra-LICM-Examples.md](Day-23-2-Extra-LICM-Examples.md).

## 1. Why this exists

Invariant work inside a loop burns every iteration:

```c
for (int i = 0; i < n; ++i)
  s += a[i] * (*scale);  /* *scale invariant */
```

Hoisting `*scale` (when legal) turns N loads into one. Sinking a value computed in the loop but only consumed at the exit avoids carrying it on every path. Scalar promotion turns a must-alias loop-carried memory cell into a PHI + registers, which unlocks more scalar opts.

LICM is **not** control-flow unswitching: it moves instructions, not entire invariant branches (that is SimpleLoopUnswitch).

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

## 2. The idea

Three related motions:

1. **Hoist** — move an invariant instruction to the preheader when it is safe to execute there (or when the loop is known to execute / speculation rules pass).
2. **Sink** — if an instruction’s users are all outside the loop (or foldable), clone/move it to exit blocks so the loop body shrinks.
3. **Promote** — for a must-alias memory set with mod, replace in-loop loads/stores by a scalar PHI: load once before the loop (if needed), store at exits.

Modern new-PM LICM **requires MemorySSA** (`loop-mssa(licm)`). AliasSetTracker still appears inside promotion candidate gathering together with MSSA.

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `licm` (also nest flavor `lnicm`) |
| File | [llvm/lib/Transforms/Scalar/LICM.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LICM.cpp) |
| New PM | `LICMPass` — errors if MemorySSA missing |
| Debug | `-debug-only=licm` |
| Typical form | LoopSimplify + LCSSA |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa,loop-mssa(licm)' -S file.ll -o file.licm.ll
```

`loop-mssa(...)` adapts the loop pass manager to provide MemorySSA. Calling bare `licm` without MSSA in the new PM hits `reportFatalUsageError("LICM requires MemorySSA (loop-mssa)")`.

## 4. Core concepts

### Invariant instruction

Operands are invariant w.r.t. the loop (or the op is intrinsically hoistable once operands are). Loads need a stronger story: no clobbering def in the loop for that location (AA + MemorySSA defining access / clobber walk).

### Safety / speculation

`isSafeToExecuteUnconditionally` (and `ICFLoopSafetyInfo`) decide whether hoisting into the preheader is allowed when the instruction might not have executed on every entry (faulting loads, trapping ops, header-not-guaranteed). Safe cases: instruction is guaranteed to execute in the loop when the loop runs, or speculation is proven safe (e.g. certain non-trapping ops), or the preheader is only reached when the loop will run (rotated loops help).

This is why **loop-rotate** and LICM are neighbors: after rotation, body/header execution implies entry, reducing speculation.

### Sinking and LCSSA

Sinking materializes values in exit blocks and updates LCSSA-style PHIs so outside users still see one SSA value per exit. Dedicated exits (LoopSimplify) matter.

### Scalar promotion

`collectPromotionCandidates` builds an `AliasSetTracker` over potentially promotable accesses discovered via MemorySSA. Only **must-alias** sets that **mod** memory are candidates; unknown aliasing / extra writers kill the set. `promoteLoopAccessesToScalars` then rewrites loads/stores to a scalar cycle.

### `llvm.invariant.start`

LICM can treat memory as invariant for the duration started by `llvm.invariant.start` when the intrinsic dominates the load, size covers the access, and the start does not escape in ways that invalidate the claim. Helpful for frontend-proven invariants; clang does not always emit these—do not assume they exist in every C loop.

### Guarded / ICF safety

`ICFLoopSafetyInfo` tracks which instructions are guaranteed to execute on every iteration that reaches the latch / exits (irreducible-control-flow-aware safety). Hoist and promote consult it. “Guarded hoisting” in practice means: only hoist speculative ops when safety info or explicit checks say the preheader execution matches loop semantics.

## 5. The algorithm

Inside `hoistRegion` / `sinkRegion` / promotion (order in `runOnLoop`-style driver):

1. Build `ICFLoopSafetyInfo` for the loop; wrap MemorySSA in `MemorySSAUpdater`.
2. **Sink** first (post-order over the domtree region) so values only used outside leave the body; enables cleaner hoist afterward.
3. **Hoist** invariant ops to the preheader when `canSinkOrHoistInst` + `isSafeToExecuteUnconditionally` agree; optional arithmetic reassociation hoist helpers.
4. **Promote** must-alias memory sets when exits/preheader allow inserting loads/stores; may iterate when promotion exposes new invariants.
5. Preserve / verify MemorySSA when requested.

Statistics include `NumHoisted`, `NumMovedLoads`, `NumMovedCalls`, plus various arithmetic-hoist counters.

## 6. Where to read the code

1. File header comment in [LICM.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LICM.cpp) — hoist / sink / promote overview.
2. `LICMPass::run` — MemorySSA requirement.
3. `canSinkOrHoistInst`, `isSafeToExecuteUnconditionally`, `hoist`, `sink`.
4. `collectPromotionCandidates` + `promoteLoopAccessesToScalars` (AST + MSSA).
5. Invariant intrinsic helpers near `invariant.start` comments.
6. Tests: `llvm/test/Transforms/LICM/`.

## 7. Worked before/after

```c
int sum_scaled(int *a, int n, int *scale) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i] * (*scale);
  return s;
}
```

After `loop-simplify,lcssa,loop-mssa(licm)` (theme):

```llvm
preheader:
  %sc = load i32, ptr %scale
  br label %loop

loop:
  ; ... load a[i], mul by %sc (no reload of *scale) ...
```

If AA cannot prove `scale` does not alias `a[i]` stores (there are none here) or other mods, the load may stay. Restrict / noalias / TBAA can help in harder cases.

## 8. Reading order for today

1. Hoist vs sink vs promote until the three verbs are distinct.
2. Why MemorySSA is mandatory in new PM.
3. Safety / speculation vs rotated loops.
4. Day 23 DeepDive (clobber walks, AST promotion).
5. Day 23 Examples; then Day 24 unroll (consumes cleaner bodies).

## 9. Hands-on commands

```bash
cat > licm.c <<'EOF'
int sum_scaled(int *a, int n, int *scale) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i] * (*scale);
  return s;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes licm.c -o licm.ll
opt -passes='loop-simplify,lcssa,loop-mssa(licm)' -S licm.ll -o licm.out.ll
# opt -debug-only=licm -passes='loop-simplify,lcssa,loop-mssa(licm)' -S licm.ll
```

Confirm `*scale` load sits in the preheader. Then try a variant that stores through a pointer that may alias `scale` and watch the hoist disappear.

## Load invariance vs operand invariance

An `add` of two loop-invariant operands is invariant. A `load` of an invariant *address* is not automatically invariant: another iteration may store to that location. LICM therefore splits the problem:

1. Can the instruction be sunk/hoisted as a pure SSA op? (`canSinkOrHoistInst` opcode rules)
2. For memory ops, does MemorySSA + AA prove no loop-carried clobber?
3. Is executing it in the preheader safe? (`isSafeToExecuteUnconditionally`)

Failing any layer keeps the op in the loop.

## Promotion vs “just hoist the load”

Hoisting a load alone still leaves stores in the loop (or misses store-forwarding opportunities). Scalar promotion builds a **cycle**:

- Preheader: initial load (if the location is live-in)
- Header PHI: scalar carried around the latch
- Exit(s): store the final scalar

That is why promotion needs dedicated exits and careful AA for the **new** exit stores (unconditional), not only the original conditional stores’ metadata.

## Pipeline pairing with rotate

```bash
opt -passes='loop-simplify,lcssa,loop-rotate,loop-mssa(licm)' -S file.ll
```

Rotation makes the body run only when the loop is entered, so many loads that would be speculative in a while-loop header become ordinary guaranteed-to-execute ops in the rotated body—LICM’s safety info then allows more hoists.

## 10. Pitfalls and invariants

- Running `licm` without `loop-mssa` in new PM — hard error.
- Confusing LICM with unswitch — LICM does not duplicate loops for invariant branches.
- Hoisting faulting loads into a preheader when the loop may be zero-trip without a safety proof.
- Expecting promotion on may-alias sets — must-alias only.
- Forgetting LCSSA/LoopSimplify — sinking and exit stores get messy or refused.
- Using old recipes that only say `opt -licm` without MemorySSA on modern new-PM builds.

## 11. Neighbors

- **LoopSimplify / LCSSA** (Day 22) — prerequisites.
- **LoopRotate** (Day 26) — reduces speculation for hoist.
- **SimpleLoopUnswitch** (Day 28) — control-flow cousin.
- **MemorySSA / AA** — legality for loads/stores.
- **GVN / InstCombine** — clean up after motion.

## 12. Check yourself

1. **Q:** Three LICM verbs?  
   **A:** Hoist invariants to preheader; sink outside-only defs to exits; promote must-alias memory to scalars.

2. **Q:** New PM MemorySSA requirement?  
   **A:** Yes — use `loop-mssa(licm)`; `LICMPass` fatals without MSSA.

3. **Q:** Is LICM the same as unswitch?  
   **A:** No. Unswitch clones loops around invariant branches; LICM moves instructions.

4. **Q:** When is load hoist legal?  
   **A:** When the loaded location is not clobbered in the loop (AA+MemorySSA) and speculation/safety rules allow preheader execution.

5. **Q:** Debug type?  
   **A:** `-debug-only=licm`.

6. **Q:** Does promotion still use AliasSetTracker?  
   **A:** Yes—inside `collectPromotionCandidates`, fed by MSSA-discovered accesses.
## Mini case: invariant GEP + load

```c
int rowsum(int *base, int n, int stride) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += base[i * stride];
  return s;
}
```

LICM may hoist `base` / `stride` uses that are invariant, but GEPs that still depend on `%i` stay in-loop unless another pass rewrites them. Invariant *operands* are not the same as an invariant *loaded value*.

> Tip: Deep dive in `Day-23-1-Extra-LICM-DeepDive.md`. Examples in `Day-23-2-Extra-LICM-Examples.md`.
