# Day 19: Constraint Elimination

Constraint Elimination proves that some `icmp`s are trivially true or false from dominating linear constraints, then folds them. Pass name: `constraint-elimination`. Implementation: [ConstraintElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/ConstraintElimination.cpp), backed by [ConstraintSystem](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/ConstraintSystem.h) / [ConstraintSystem.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ConstraintSystem.cpp). Deep dive: [Day-19-1-Extra-ConstraintElimination-DeepDive.md](Day-19-1-Extra-ConstraintElimination-DeepDive.md). Examples: [Day-19-2-Extra-ConstraintElimination-Examples.md](Day-19-2-Extra-ConstraintElimination-Examples.md).

## 1. Why this exists

Bounds checks and range tests dominate C/C++ IR: after one `icmp ult %i, %n` succeeds, a later `icmp ult %i, %n` or a tighter check is redundant. InstCombine folds algebraic identities locally; SCEV reasons about addrecs in loops. Constraint Elimination fills a gap: **dominating relational facts** as a system of linear inequalities, queried for later compares—including patterns that arise from `llvm.assume` and unsigned index checks.

Redundant branches become unconditional; redundant compares feeding selects become constants. That unlocks DCE, CFG simplify, and fewer runtime checks in sanitized or defensive code once the dominating guard exists.

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

## 2. The idea

1. Walk the function in a dominance-aware order.
2. When control flow implies a condition (true edge of a branch, assume intrinsic, etc.), add a linear constraint to a `ConstraintSystem`.
3. At a later `icmp`, encode it as a query row and ask whether the system already implies the condition—or its negation.
4. If implied true/false, replace the icmp (or branch condition) with a constant.

Unsigned predicates are especially common (`ult`, `ule`, …) because C array indexes and lengths are often treated as non-negative spaces. Signed systems are tracked separately inside the pass’s `ConstraintInfo` wrapper.

Only a **subset** of IR expressions become coefficients: adds, multiplies by constants, certain `zext`s, and some GEP-derived offsets. Everything else is opaque or ignored.

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `constraint-elimination` |
| File | [ConstraintElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/ConstraintElimination.cpp) |
| Solver | [ConstraintSystem](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ConstraintSystem.cpp) |
| Debug | `-debug-only=constraint-elimination` |
| Related | DominatorTree, often LoopInfo / SCEV includes for reasoning helpers |

Scheduled among scalar optimizations after enough simplify/instcombine have exposed clean icmps. Exact PassBuilder position shifts; invoke explicitly for study:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=constraint-elimination -S file.ll -o file.ce.ll
```

## 4. Core concepts

### Linear constraints

Relations like `A - B <= c` encoded as coefficient rows over variables mapped from SSA values (`Value2Index`). Constants sit in a dedicated column.

### `ConstraintSystem`

Holds rows; supports adding facts and asking `isConditionImplied`. Internally uses elimination techniques (Fourier–Motzkin style / Gaussian-like reduction—read `ConstraintSystem.cpp` for the current algorithm). `mayHaveSolution` detects contradictions.

### Dominating facts

A fact is only live where it dominates. Leaving a branch merge pops or stops applying facts that do not hold on all paths. The pass’s worklist / DFS-style traversal with scoped fact addition implements that (see deep dive).

### Unsigned vs signed systems

`ConstraintInfo` keeps separate unsigned and signed systems. Mixing signedness carelessly is unsound (e.g. `slt` vs `ult` on the same bit pattern). Transfer between systems happens only under documented safe conditions in the pass.

### `llvm.assume`

`call void @llvm.assume(i1 %cond)` adds `%cond` as a fact at that program point—powerful for user or earlier-pass knowledge.

### Supported expression subset

Expect first-class support for:

- `add` / `sub` of tracked values
- `mul` by a **constant**
- some `zext` (zero extension preserves unsigned inequalities carefully)
- certain GEP index contributions when they lower to linear offsets

Arbitrary `call` results, `load`s, and non-linear ops do not get rich modeling.

## 5. The algorithm

Simplified:

1. Build a traversal of blocks/instructions respecting dominance (and control-edge facts).
2. On a conditional branch, schedule adding the true/false condition as a fact for the respective successor scope.
3. On `llvm.assume`, add the assumed condition.
4. On `icmp`, build a constraint row for the predicate; query implication.
5. If implied, replace uses with `i1 true/false` (and let later passes fold branches).
6. Enforce `constraint-elimination-max-rows` (and similar caps) so pathological systems do not explode compile time.

Statistics such as `NumCondsRemoved` track successful eliminations. A debug counter (`conds-eliminated`) can skip transforms for bisection.

## 6. Where to read the code

1. [ConstraintElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/ConstraintElimination.cpp) — pass logic, `ConstraintInfo`, fact addition, icmp elimination.
2. [ConstraintSystem.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/ConstraintSystem.h) / [ConstraintSystem.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ConstraintSystem.cpp) — solver API (`add*` / `isConditionImplied` / `mayHaveSolution`).
3. Header [ConstraintElimination.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/ConstraintElimination.h) — pass decl.
4. Lit tests in `llvm/test/Transforms/ConstraintElimination/`.

Do not invent line numbers; search for `addFact`, `isConditionImplied`, and the module `DEBUG_TYPE`.

## 7. Worked before/after

```c
int idx(int *a, unsigned i, unsigned n) {
  if (i < n) {
    if (i < n)        /* redundant */
      return a[i];
  }
  return 0;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes idx.c -o idx.ll
opt -passes=constraint-elimination -S idx.ll -o idx.ce.ll
```

**Before:** two `icmp ult` on the same operands; second guards the load.

**After:** the inner compare folds to true; the inner branch becomes unconditional (or disappears after simplifycfg). The dominating `i < n` fact implies the later check.

With assumes:

```llvm
call void @llvm.assume(i1 %ok)
%check = icmp ult i32 %i, %n
```

If `%ok` encodes the same relational fact (or a stronger one the system understands), `%check` may fold.

## 8. Reading order for today

1. Encode one unsigned inequality by hand as a coefficient row (paper exercise).
2. Read `ConstraintSystem` public methods.
3. Skim ConstraintElimination’s fact scoping / icmp visitor paths.
4. Run `-debug-only=constraint-elimination` on a double bounds check.
5. [Day-19-1-Extra-ConstraintElimination-DeepDive.md](Day-19-1-Extra-ConstraintElimination-DeepDive.md).
6. [Day-19-2-Extra-ConstraintElimination-Examples.md](Day-19-2-Extra-ConstraintElimination-Examples.md).

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes ce.c -o ce.ll
opt -passes=constraint-elimination -S ce.ll -o ce.out.ll
opt -passes=constraint-elimination -debug-only=constraint-elimination -S ce.ll -o /dev/null 2>ce.log

# Optional reproducers / caps (hidden flags; confirm with --help)
opt -passes=constraint-elimination -constraint-elimination-max-rows=50 -S ce.ll
```

Try: dominating `ult`; `llvm.assume`; signed vs unsigned mismatch; non-linear `mul` of two SSA values (should not magically prove much).

## 10. Pitfalls and invariants

- **Signedness:** do not expect `slt` facts to freely rewrite `ult` queries.
- **Wrapping:** LLVM `add` is two’s complement wrap; the pass is careful about when wrap disallows a constraint. Flags like `nuw`/`nsw` matter when present.
- **Partial expression language:** if something does not fold, the reason is often “could not build a row,” not “solver failed.”
- **Dominance:** facts do not flow to blocks not dominated by the branch edge that justified them.
- **Compile-time caps:** huge functions with many compares may hit max-rows and stop adding facts.
- **Not a full Presburger solver for all IR:** keep expectations at “linear subset.”
- **Poison/UB:** proving icmp true does not invent UB; it only uses already-required control facts.

### Bounds-check idiom in IR

A common Clang pattern for `a[i]` after `i < n`:

```llvm
%cmp = icmp ult i32 %i, %n
br i1 %cmp, label %inbounds, label %fail
inbounds:
  %idx = zext i32 %i to i64
  %p = getelementptr inbounds i32, ptr %a, i64 %idx
  %v = load i32, ptr %p, align 4
  br label %join
```

A second sanitizer or explicit check inside `inbounds` that repeats `icmp ult i32 %i, %n` is CE’s favorite meal. After elimination, SimplifyCFG can delete the inner branch entirely.

### Equality facts

`icmp eq` can tighten systems when one side is a tracked variable and the other is constant or another variable already in the map. Whether a particular equality transfers into a later relational query depends on decomposition—verify with `-debug-only=constraint-elimination` rather than assuming all eq facts become substitutions automatically.

### Relationship to “range metadata”

`!range` on loads and call returns informs other analyses (CVP, InstCombine). CE’s primary input remains **relational constraints from control and assumes**. You can still benefit indirectly when InstCombine uses `!range` to canonicalize icmps into forms CE understands.

## 11. Neighbors

- **InstCombine** — local icmp canonicalization that feeds cleaner constraints.
- **CVP / SCCP** — value-range style propagation; overlapping goals, different machinery.
- **SCEV / IndVars** — loop-oriented symbolic ranges.
- **SimplifyCFG (Day 18)** — cleans branches after compares become constants.
- **AssumeSimplify / Knowledge retention** — produce `llvm.assume`s this pass consumes.
- **Sanitizer checks** — often create the redundant second checks CE removes when safe.

## 12. Check yourself

**Q1.** Pass name and debug type?  
**A1.** `constraint-elimination` / `-debug-only=constraint-elimination`.

**Q2.** What solver class holds the inequalities?  
**A2.** `ConstraintSystem` (Analysis library).

**Q3.** Why separate signed and unsigned systems?  
**A3.** Predicates mean different relations on the same bits; mixing is unsound.

**Q4.** Name two fact sources.  
**A4.** Dominating branch conditions; `llvm.assume`.

**Q5.** Give an expression CE typically cannot model well.  
**A5.** Product of two non-constant SSA values; opaque calls/loads.

**Q6.** What happens when a later icmp is implied true?  
**A6.** It becomes constant true (and the branch can fold later).

Tip: solver and `ConstraintInfo` walkthrough in [Day-19-1-Extra-ConstraintElimination-DeepDive.md](Day-19-1-Extra-ConstraintElimination-DeepDive.md); IR cases in [Day-19-2-Extra-ConstraintElimination-Examples.md](Day-19-2-Extra-ConstraintElimination-Examples.md).

### Mini exercise (paper)

Given facts `0 <= i` and `i + 1 < n` (unsigned), argue whether `i < n` holds. Then encode both facts as inequality rows with variables `i` and `n` and see that the implied row for `i < n` is a non-negative combination of existing rows. That is the solver’s job; CE’s job is building those rows from IR without lying about wrap.

### When CE and CVP disagree

On some IR, CVP constant-folds an icmp that CE leaves alone (or the reverse). That is normal: different fact languages. Prefer the pass that fired when reducing a bug; do not expect identical coverage. Combining `cvp,constraint-elimination,simplifycfg` is a reasonable experiment pipeline for bounds-check heavy code.

### Assume + branch together

If both a dominating branch and an `llvm.assume` encode related facts, the system should remain consistent (`mayHaveSolution`). Contradictory assumes indicate unreachable code; CE may fold later conditions aggressively in dominated regions—pair with SimplifyCFG to delete the dead CFG.
