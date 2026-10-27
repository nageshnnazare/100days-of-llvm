# Day 12: Sparse Conditional Constant Propagation (SCCP)

SCCP is LLVM’s Wegman–Zadeck sparse conditional constant propagator for a single function. It assumes values are constant and blocks are dead until proven otherwise, then replaces proven constants and kills infeasible control-flow edges. The production pass is `sccp` (`SCCPPass`). The shared engine lives in `SCCPSolver` and is also used by IPSCCP.

Deep dive: [Day-12-1-Extra-SCCP-DeepDive.md](Day-12-1-Extra-SCCP-DeepDive.md). Examples: [Day-12-2-Extra-SCCP-Examples.md](Day-12-2-Extra-SCCP-Examples.md).

## 1. Why this exists

Ordinary constant folding walks instructions once and only folds when operands are already `Constant`. That misses cases where a value becomes constant only after a branch is proven unrealizable, or where a PHI merges the same constant on every *executable* predecessor. SCCP solves a sparse dataflow problem over two lattices at once: value constants and executable CFG edges. Dead edges never feed PHIs, so more constants appear; those constants prove more branches constant; the worklists iterate to a fixed point.

Without SCCP (and IPSCCP at module scope), middle-end IR keeps many `br i1` edges and PHIs that a later SimplifyCFG or InstCombine must rediscover piece by piece. SCCP is the pass that systematically couples constant lattice meets with reachability.

## Diagram

```text
        lattice for one SSA value

            top  (unknown, not yet seen)
             |
         Constant 17     or     overdefined
             |                     |
             +----------+----------+
                        |
                   never goes back up

   %c = icmp eq i32 %x, 0
   br i1 %c, label %t, label %f

   in %t, %x is Constant 0
   in %f, %x is overdefined (anything but 0,
   which this lattice cannot say)
```

SCCP's lattice has three interesting states: unknown, a single constant, or overdefined. A value only moves downward. On a branch, the true successor can learn that the compared value is that constant. The false successor usually learns nothing, because "not 17" is not a constant. That is the hole IPSCCP and CVP work around in different ways.

## 2. The idea

Every SSA value sits in a lattice element (`ValueLatticeElement`). Classic teaching lattice:

- **unknown / undef** — bottom-ish: nothing known yet, or known undef
- **constant** — a concrete `Constant` (or a singleton integer constant range)
- **overdefined** — top: not a single constant; stop refining

Wegman–Zadeck SCCP maintains **two worklists**:

1. **SSA / instruction worklist** — a value’s lattice state changed; revisit users.
2. **Executable CFG / block worklist** — a block or edge became live; visit its instructions and successor edges.

Only instructions in executable blocks are visited. A conditional branch whose condition is constant marks only the taken successor executable. PHIs only meet incoming values from executable predecessors. That is the “conditional” in SCCP.

Forced multi-element constant ranges and heavy interprocedural tracking are primarily **IPSCCP** / richer lattice work; function SCCP still uses `ValueLatticeElement` (including ranges for integers) but stays intra-procedural aside from light return-attribute inference.

## 3. Where it sits in the -O2/-O3 pipeline

In `PassBuilderPipelines.cpp`’s default function simplification pipeline, `SCCPPass` typically runs **after** the loop nest of passes and **immediately before** `BDCEPass`, then InstCombine. Nearby later: JumpThreading, `CorrelatedValuePropagationPass`, ADCE. Module-level `ipsccp` runs earlier in the IPO pipeline and shares `SCCPSolver`.

| Item | Value |
|------|--------|
| Pass name for `opt` | `sccp` |
| Class | `SCCPPass` |
| Solver | `SCCPSolver` in `SCCPSolver.cpp` |
| Debug | `-debug-only=sccp` |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=sccp -S file.ll -o file.sccp.ll
```

## 4. Core concepts

### `ValueLatticeElement`

Defined in [llvm/include/llvm/Analysis/ValueLattice.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/ValueLattice.h). States include `unknown`, `undef`, `constant`, `notconstant`, `constantrange`, `constantrange_including_undef`, and `overdefined`. `SCCPSolver::isConstant` / `isOverdefined` treat singleton ranges like classic constants for replacement.

### Executable blocks and edges

`SCCPSolver::markBlockExecutable` / `markEdgeExecutable` seed and grow the live CFG. Entry is marked executable at the start of `runSCCP`. Feasibility queries (`isEdgeFeasible`, `isBlockExecutable`) gate PHI meets and instruction visits.

### Replacement API

After `Solver.solve()` (and undef resolution loops), the pass walks executable blocks and calls `Solver.simplifyInstsInBlock`, which uses `tryToReplaceWithConstant` when the lattice proves a replaceable constant. (Some study materials say `Solver.solved` then replace—there is no public `solved` member on current `SCCPSolver`; the solved state is the lattice map after `solve()`, queried via `getLatticeValueFor` / `getConstantOrNull`.) Dead / infeasible edges are cleaned with `removeNonFeasibleEdges` and related helpers.

### Undef vs poison

**Do not treat undef and poison as the same.** Lattice `undef` models LLVM `UndefValue` / undef-producing states. Poison is a separate IR concept (flags like `nsw`, freeze, etc.). SCCP’s undef-resolution pass (`resolvedUndefsIn`) exists because temporarily assuming branches on undef never take successors is unsafe; the solver may need a second round. Poison-generating flags are a different contract—see CVP/InstCombine for nsw/nuw care.

### Floats

Floating-point lattice behavior is **conservative**. Do not expect aggressive FP constant folding through SCCP the way you do for integers; when in doubt, overdefined wins.

## 5. The algorithm

1. Construct `SCCPSolver` with `DataLayout`, a TLI getter, and context (`runSCCP` in `SCCP.cpp`).
2. Optionally attach PredicateInfo via `addPredicateInfo` (DominatorTree + AssumptionCache).
3. `markBlockExecutable(&F.front())`; `trackValueOfArgument` for each argument (range attributes matter).
4. Loop: `Solver.solve()` then `resolvedUndefsIn(F)` until undef resolution stabilizes.
5. For each block: if not executable, record for cleanup; else `simplifyInstsInBlock` (replace constants, refine, simplify).
6. Turn dead block bodies unreachable; `removeNonFeasibleEdges` rewrites branches that cannot take some successors; delete non-address-taken dead blocks via `DomTreeUpdater` when safe.
7. `removeSSACopies`, `inferReturnAttributes`.

Worklist solve (inside the visitor): while instruction or BB worklist nonempty, visit instructions in executable blocks and mark successor edges when terminators become resolvable.

## 6. Where to read the code

- [llvm/lib/Transforms/Scalar/SCCP.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SCCP.cpp) — `runSCCP`, `SCCPPass::run`
- [llvm/lib/Transforms/Utils/SCCPSolver.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SCCPSolver.cpp) — solver worklists, visits, `tryToReplaceWithConstant`
- [llvm/include/llvm/Transforms/Utils/SCCPSolver.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/SCCPSolver.h) — public solver API
- [llvm/include/llvm/Analysis/ValueLattice.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/ValueLattice.h) — `ValueLatticeElement`
- IPSCCP: [llvm/lib/Transforms/IPO/SCCP.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/SCCP.cpp) (module pass, shared solver)

Key symbols: `SCCPPass::run`, `runSCCP`, `SCCPSolver::solve`, `markBlockExecutable`, `markEdgeExecutable`, `tryToReplaceWithConstant`, `simplifyInstsInBlock`, `removeNonFeasibleEdges`, `resolvedUndefsIn`.

## 7. Worked before/after

**C** (`sccp_br.c`):

```c
int pick(int x) {
  int c = 1;
  if (c)
    return x + 1;
  return x - 1; /* unreachable once c folds */
}
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sccp_br.c -o sccp_br.ll
opt -passes=sccp -S sccp_br.ll -o sccp_br.out.ll
```

**Before (sketch):**

```llvm
define i32 @pick(i32 %x) {
entry:
  %c = add i32 0, 1
  %tobool = icmp ne i32 %c, 0
  br i1 %tobool, label %if.then, label %if.else
if.then:
  %a = add i32 %x, 1
  br label %return
if.else:
  %b = add i32 %x, -1
  br label %return
return:
  %r = phi i32 [ %a, %if.then ], [ %b, %if.else ]
  ret i32 %r
}
```

**After SCCP (shape):** `%c` / icmp fold; only the true edge stays feasible; the false side becomes unreachable / deleted; PHI collapses toward `%a` (exact cleanup depends on `removeNonFeasibleEdges` + block deletion). Follow with `simplifycfg` if you want a single straight-line block.

## 8. Reading order for today

1. Header comment in `SCCP.cpp` (what SCCP claims to do).
2. `runSCCP` end-to-end.
3. `ValueLatticeElement` tags in `ValueLattice.h`.
4. Worklists in `SCCPSolver.cpp` (`InstWorkList`, `BBWorkList`, `markEdgeExecutable`).
5. `tryToReplaceWithConstant` and `simplifyInstsInBlock`.
6. Deep dive + examples extras.

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sccp_br.c -o sccp_br.ll
opt -passes=sccp -S sccp_br.ll -o out.ll
opt -passes=sccp -debug-only=sccp -S sccp_br.ll 2>&1 | less
opt -passes='sccp,simplifycfg' -S sccp_br.ll -o out.clean.ll
opt -passes=ipsccp -S module.ll -o module.ipsccp.ll   # Day 43 territory
```

Use an assertions build for denser `-debug-only=sccp` logs (“Marking Block Executable”, “Constant: …”).

## 10. Pitfalls and invariants

- `clang -O0` adds `optnone`; `opt` may skip the function. Prefer `-O1 -Xclang -disable-llvm-passes`.
- SCCP is **function-scoped**; cross-function constants need `ipsccp`.
- Address-taken dead blocks may survive as shells; not every dead block is always erased.
- Floats and many calls go **overdefined** quickly.
- Undef branch handling can force re-solve; do not equate undef with poison.
- Replacing `musttail` call results is restricted inside `tryToReplaceWithConstant`.
- Opaque pointers: lattice tracks `ptr` constants carefully (provenance / replaceability via `isReplaceableConstant`).

## 11. Neighbors

- **Before:** loop passes, InstCombine-ish cleanup that expose constants.
- **After:** `bdce` (Day 13), InstCombine, JumpThreading, `correlated-propagation` (Day 14).
- **IPO cousin:** `ipsccp` shares `SCCPSolver`.
- **CFG cleanup:** SimplifyCFG / ADCE consume unreachable edges and dead code SCCP exposes.

## 12. Check yourself

**Q1.** `opt` pass name for function SCCP?
**A1.** `sccp`.

**Q2.** Name the two worklist kinds in Wegman–Zadeck SCCP.
**A2.** SSA/instruction edges and executable CFG/block edges.

**Q3.** Where does the shared solver live?
**A3.** `llvm/lib/Transforms/Utils/SCCPSolver.cpp` (`SCCPSolver`).

**Q4.** What lattice type stores states?
**A4.** `ValueLatticeElement` in `ValueLattice.h`.

**Q5.** Does SCCP always delete every dead block itself?
**A5.** No—not always; it rewrites infeasible edges / unreachable contents and deletes when safe (e.g. non-address-taken), with SimplifyCFG still helping.

**Q6.** Are undef and poison the same in this mental model?
**A6.** No.

> Tip: Implementation walk-through is in `Day-12-1-Extra-SCCP-DeepDive.md`. Runnable cases are in `Day-12-2-Extra-SCCP-Examples.md`.

## Appendix A: Why “sparse”?

Classic CCP evaluates every instruction in every live block repeatedly. Sparse CCP only re-evaluates users of values whose lattice state *changed*, and only walks newly executable blocks. The SSA use-def graph is the sparse dependence structure; CFG executability is the second sparse structure. That is why two worklists appear in the solver.

## Appendix B: PHI meeting rule

A PHI’s lattice state is the meet of incoming values from predecessors whose edges are executable. An incoming from a dead edge is ignored. If all live incomings are the same constant, the PHI becomes that constant even if a dead predecessor would have brought a different value. That interaction is the entire point of coupling the lattices.

## Appendix C: Struct returns and multi-value lattices

Struct-typed SSA values carry a vector of lattice elements (`getStructLatticeValueFor`). Replacement builds a `ConstantStruct` only when every element is replaceable. IPSCCP leans on this for aggregate returns; function SCCP still goes through the same helpers when structs appear. When studying IR with `{i32, i32}` returns, dump lattice state per element rather than expecting a single scalar tag.

## Appendix D: Calls and overdefined

Most general calls force the returned value overdefined unless the solver has a tracked callee (IPSCCP) or a foldable known library call via TLI. That is why toy examples with `call i32 @opaque()` never constant-fold under `sccp` alone. Read the call visitor in `SCCPSolver.cpp` when a surprising call result fails to fold—usually the lattice went overdefined on purpose.

## Appendix E: Comparing `sccp` and `ipsccp` on one module

```bash
opt -passes=sccp -S m.ll -o m.sccp.ll
opt -passes=ipsccp -S m.ll -o m.ipsccp.ll
diff -u m.sccp.ll m.ipsccp.ll | less
```

Look for zapped call results, inferred `returned` / range-like attributes, and constant global loads. Shared solver, different client drivers and tracked sets.

## Appendix F: One-line summary

SCCP = sparse constant lattice meet + executable CFG edges, replacing via `tryToReplaceWithConstant` and killing infeasible branches for later CFG cleanup.
