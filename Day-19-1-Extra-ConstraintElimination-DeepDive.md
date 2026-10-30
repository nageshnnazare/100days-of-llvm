# Day 19 Extra: Constraint Elimination — Deep Dive

Companion to [Day-19-0-ConstraintElimination.md](Day-19-0-ConstraintElimination.md). Examples: [Day-19-2-Extra-ConstraintElimination-Examples.md](Day-19-2-Extra-ConstraintElimination-Examples.md).

## Diagram

```text
   %c1 = icmp uge i32 %x, %a
   %c2 = icmp ule i32 %x, %b
   %and = and i1 %c1, %c2
   br i1 %and, label %in, label %out

   inside %in the pass records the row:
        %a  <=  %x  <=  %b

   a later  icmp ult i32 %x, %a
   contradicts the row, so it is false
```

ConstraintElimination stores a system of inequalities taken from branches that dominate the current block. A later compare is tested against that system. If the system proves it, the compare becomes a constant. The arithmetic is on the integer predicates the branch actually established, so a signed fact is not reused as an unsigned one.

## 1. Module layout

| Piece | Path |
|-------|------|
| Pass | [ConstraintElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/ConstraintElimination.cpp) |
| Pass header | [ConstraintElimination.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/ConstraintElimination.h) |
| Solver | [ConstraintSystem.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ConstraintSystem.cpp) |
| Solver API | [ConstraintSystem.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/ConstraintSystem.h) |

`DEBUG_TYPE` is `"constraint-elimination"`.

Notable knobs in the pass file (names):

- `constraint-elimination-max-rows`
- `constraint-elimination-dump-reproducers`
- Debug counter `conds-eliminated`

## 2. `ConstraintSystem` API (mental model)

Think of each row as coefficients `[c0, c1, …, ck]` over variables `x1…` plus a constant component such that the row encodes a linear inequality (exactly which normalization—`<= 0` vs `>= 0`—is defined in the class; read `addVariableRow` / related helpers).

Important operations to locate while reading:

- Adding a constraint row
- `isConditionImplied` — does the system entail this row?
- `mayHaveSolution` — is the system still satisfiable?
- `dump` — debug print using IR value names when a `Value2Index` map is provided
- Negation / strictify helpers (`negate`, `toStrictLessThan`) used when reasoning about complementary predicates

The implementation uses elimination (comments historically mention Fourier–Motzkin). Treat it as “Gaussian-like / FM elimination over rationals or integers with care,” and trust the in-tree comments over third-party summaries.

## 3. `ConstraintInfo` in the pass

ConstraintElimination does not dump raw rows without IR mapping. A wrapper (commonly named `ConstraintInfo` in the `.cpp`) owns:

- `UnsignedCS` and `SignedCS`
- `Value2Index` maps per signedness
- DataLayout reference for type sizes / GEP reasoning
- Helpers to turn `CmpInst` predicates + operands into coefficient vectors

`addFact(Pred, A, B, …)` inserts the fact into the appropriate system. Query paths build a row for a later icmp and call into `isConditionImplied`.

## 4. Control-flow fact scoping

Facts are tied to program points. A correct approach:

1. Traverse in dominance order (or DFS with explicit stacks).
2. When taking the true edge of `br i1 %c, label %T, label %F`, push constraint(s) implied by `%c` being true while processing `%T`’s dominated region.
3. Pop / replace the system when leaving that region.
4. At merges, only facts valid on **all** incoming paths remain (intersection)—often implemented by not carrying edge-local facts past the join.

Reading tip: search for where the pass records “pending” facts for edges and where it removes the last constraint when backtracking (`LLVM_DEBUG` messages about adding/removing constraints are excellent breadcrumbs).

## 5. From icmp to coefficients

High-level recipe the pass follows:

1. Canonicalize predicate where needed.
2. Decompose operands into linear forms over base SSA values (add/mul-by-const/zext/GEP pieces).
3. Fail soft if decomposition hits an unknown op—**no fact / no query**, leave the icmp alone.
4. On success, emit a row and either add as fact or query implication.

Unsigned `icmp ult %a, %b` roughly means `%a <= %b - 1` in integer math when no wrap issues intervene—encoding details are in the decomposition helpers; follow them rather than inventing coefficients from memory.

## 6. GEPs and bounds checks

Array indexing IR:

```llvm
%idx = zext i32 %i to i64
%p = getelementptr inbounds i32, ptr %base, i64 %idx
%v = load i32, ptr %p
```

often sits next to `icmp ult i32 %i, %n`. The pass may reason about indexes and lengths when GEPs and extents fit the linear subset (inbounds helps semantic arguments elsewhere; CE still needs an explicit relational fact).

Do not assume every inbounds GEP magically adds `%idx < length`—length is not in the GEP. The **icmp or assume** must provide that fact.

## 7. `llvm.assume` integration

```llvm
call void @llvm.assume(i1 %cond)
```

is treated as knowledge at that point: `%cond` holds for dominated uses. Earlier passes (and programmers via builtins) plant assumes; CE is a consumer that turns them into folded icmps downstream.

## 8. `run()` path

New PM: `ConstraintEliminationPass::run` obtains analyses (DominatorTree mandatory; other analyses appear in includes—LoopInfo, ScalarEvolution, TLI, ORE, etc. depending on revision) and calls the function-level elimination driver (search `eliminateConstraints` or similarly named static entry in the `.cpp`).

Preservation: replacing icmps with constants typically preserves CFG **shape** until a later SimplifyCFG run folds edges; CE itself may not remove blocks.

## 9. What is **not** handled (or handled weakly)

- Nonlinear relations (`%a * %b` both variable)
- Most floating-point compares
- Relational facts across memory (loads) without value forwarding
- Full alias-aware “this pointer + n is in bounds of allocation” without explicit numeric facts (AllocationSize reasoning may exist in parts—verify helpers before claiming; if unsure, treat as best-effort)
- Replacing the need for LICM/SCEV in loops with non-linear trip counts

## 10. Debugging workflow

```bash
opt -passes=constraint-elimination -debug-only=constraint-elimination -S in.ll -o out.ll 2>ce.log
```

Look for:

- Constraint rows printed with value names
- Messages about removing constraints when leaving scopes
- Which icmp was proved true/false

Dump IR reproducers if `constraint-elimination-dump-reproducers` is enabled on your build.

Bisect with the debug counter:

```bash
opt -passes=constraint-elimination -debug-counter=conds-eliminated-skip=0,conds-eliminated-count=1 -S in.ll
```

(Exact counter flag syntax follows LLVM DebugCounter conventions.)

## 11. Interaction with other passes

Recommended mental order for experiments:

```
instcombine → constraint-elimination → simplifycfg → dce
```

InstCombine normalizes icmps; CE constant-folds redundant ones; SimplifyCFG deletes dead arms; DCE cleans leftovers.

CVP may overlap on some range proofs; comparing `-passes=cvp` vs `constraint-elimination` on the same file is a good lab.

## 12. Soundness checklist when reading patches

1. Does the fact only apply where dominated?
2. Is signedness consistent?
3. Are `nuw`/`nsw`/wrap cases respected?
4. Is the expression decomposition incomplete (should refuse, not guess)?
5. Does implying false vs true use the correct complementary encoding?

## 13. Study exercises

1. By hand: from `i < n` and `j == i`, justify `j < n`—then find whether CE’s decomposition sees the equality.
2. Build `i < n` on one branch only; confirm the icmp inside the other branch does not fold.
3. Flip `ult` vs `slt` on the same bit width and observe non-transfer.
4. Add `llvm.assume` for `i < n` without a branch; confirm a later check folds.

## 14. Takeaway

Constraint Elimination is **dominance-scoped linear implication for icmp**. The cleverness is in fact scoping and expression decomposition; the solver is a reusable Analysis library class.

## 15. Row budget and compile time

`constraint-elimination-max-rows` caps how large a system may grow. When the cap hits, new facts may stop being added even though more icmps remain. Symptoms: some obvious folds in huge functions disappear while small reduced tests still work. If that happens, either raise the cap for an experiment or reduce the IR; do not assume the algorithm is unsound.

## 16. Optimization remarks / ORE

The pass includes OptimizationRemarkEmitter headers on many trees. When present, successful eliminations can emit remarks you can surface with `-pass-remarks=constraint-elimination` (exact remark group strings follow the pass’s `ORE`-emitting calls). Pair remarks with `-debug-only` when learning; prefer remarks when teaching others without drowning in constraint dumps.
