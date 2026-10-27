# Day 12 Extra: SCCP Deep Dive

Companion to [Day-12-0-SCCP.md](Day-12-0-SCCP.md). Examples: [Day-12-2-Extra-SCCP-Examples.md](Day-12-2-Extra-SCCP-Examples.md).

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

## 1. Files and entry points

Function pass:

- [llvm/lib/Transforms/Scalar/SCCP.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SCCP.cpp)
- Header: [llvm/include/llvm/Transforms/Scalar/SCCP.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/SCCP.h)

Shared engine:

- [llvm/lib/Transforms/Utils/SCCPSolver.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SCCPSolver.cpp)
- [llvm/include/llvm/Transforms/Utils/SCCPSolver.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/SCCPSolver.h)

Lattice:

- [llvm/include/llvm/Analysis/ValueLattice.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/ValueLattice.h)

`DEBUG_TYPE` is `"sccp"` in both the pass and the solver translation units that define it.

New PM: `SCCPPass::run(Function &F, FunctionAnalysisManager &AM)` fetches `TargetLibraryAnalysis`, `DominatorTreeAnalysis`, `AssumptionAnalysis`, then calls `runSCCP`.

## 2. `runSCCP` path

1. Construct `SCCPSolver(DL, TLIGetter, Ctx)`.
2. `Solver.addPredicateInfo(F, DT, AC)` — PredicateInfo copies help track edge predicates.
3. Optionally `addTrackedFunction` when returns can be tracked for attribute inference.
4. `markBlockExecutable(&F.front())`.
5. For each `Argument`, `trackValueOfArgument` (range attrs can seed tighter states).
6. Loop `solve` / `resolvedUndefsIn` until undef resolution returns false.
7. Walk all blocks: non-executable → collect for erase; executable → `simplifyInstsInBlock`.
8. `changeToUnreachable` on dead block contents; `removeNonFeasibleEdges` on all blocks; `DTU.deleteBB` for non-address-taken dead blocks.
9. `removeSSACopies`, `inferReturnAttributes`.

Preserve DominatorTree when changes occur (`PreservedAnalyses` keeps DT).

## 3. Worklists inside the visitor

The implementation class behind `SCCPSolver` (visitor) keeps roughly:

- `BBExecutable` — set of live blocks
- `InstWorkList` — instructions whose users must be revisited (`SmallSetVector`)
- `BBWorkList` — blocks newly marked executable

`markBlockExecutable` inserts into `BBExecutable` and pushes onto `BBWorkList`.

`markEdgeExecutable(Source, Dest)` marks `Dest` executable; if already executable, still pushes PHI nodes in `Dest` onto the instruction worklist so new incoming edges re-meet.

`solve()` drains: prefer processing instruction worklist visits, then pop blocks and visit their instructions until both lists empty. Only visit instructions whose parent is executable.

This is the concrete LLVM encoding of Wegman–Zadeck’s dual worklists (SSA edges and executable CFG edges).

## 4. Lattice operations you will see

Helpers on `SCCPSolver`:

- `getLatticeValueFor(Value*)`
- `markOverdefined(Value*)`
- `isConstant` / `isOverdefined` static helpers bridging classic LatticeVal names to `ValueLatticeElement`
- `getConstant` / `getConstantOrNull`
- `tryToReplaceWithConstant`

Integer values often live in `constantrange` rather than the bare `constant` tag; singleton ranges still replace. Multi-element ranges without collapsing to one constant leave the value non-replaceable for classic constant folding (IPSCCP leans harder on ranges).

Merges use `ValueLatticeElement::mergeIn` with `MergeOptions` (widen steps, may-include-undef). Repeated range widening can force overdefined—important for loops.

## 5. `tryToReplaceWithConstant`

Flow:

1. `getConstantOrNull(V)` — fail if not a replaceable constant / singleton range.
2. Special-case `CallBase`: refuse unsafe `musttail` result zapping and `clang.arc.attachedcall` bundles; may record must-preserve returns.
3. `V->replaceAllUsesWith(Const)`.

`simplifyInstsInBlock` then may erase trivially dead instructions, try signedness refinements, `refineInstruction`, or `simplifyInstruction` fallbacks. Replacement is the headline SCCP effect students look for in IR diffs.

## 6. Undef resolution

While solving, branches on undef may be treated so that successors are not all marked live (optimistic). That can be wrong. After a solve pass, `resolvedUndefsIn` / `resolvedUndef` force more conservative lattice updates and request another `solve()` if anything changed. `runSCCP` loops until stable.

Separate mental bucket: **poison**. NSW/NUW and other poison-generating annotations are not “undef lattice states.” Do not write notes that say undef ≡ poison.

## 7. What function SCCP does NOT do

- Full **interprocedural** constant propagation of formals/returns across a call graph — that is **IPSCCP**.
- Aggressive exploitation of arbitrary **forced constant ranges** as a teaching “classic SCCP” feature — ranges exist in the lattice, but IPSCCP / range-oriented IPO is where that story expands.
- Guaranteed deletion of **every** dead block (address-taken blocks; incomplete CFG surgery edge cases).
- Treating floating-point as freely foldable as integers.
- Ignoring `musttail` / ARC attached-call constraints on replacement.

## 8. PredicateInfo and assumes

`addPredicateInfo` ties DominatorTree and AssumptionCache into the solver so edge predicates and `llvm.assume` facts can refine values. When reading IR with assumes, expect tighter lattice states than a bare Wegman–Zadeck textbook sketch.

## 9. Debugging

```bash
opt -passes=sccp -debug-only=sccp -S in.ll 2>sccp.log
```

Look for:

- `SCCP on function '…'`
- `Marking Block Executable`
- `Marking Edge Executable`
- `RESOLVING UNDEFs`
- `Constant: … = …`
- `BasicBlock Dead:`

Pair with `opt -passes='sccp,verify'`.

## 10. Analyses

**Used:** DataLayout, TargetLibraryInfo, DominatorTree, AssumptionCache; PredicateInfo built internally.

**Related:** IPSCCP also tracks globals and argument-tracked functions via additional `SCCPSolver` APIs (`trackValueOfGlobalVariable`, `addArgumentTrackedFunction`).

## 11. Source reading checklist

1. `runSCCP` control flow.
2. `markBlockExecutable` / `markEdgeExecutable`.
3. `solve` worklist loop.
4. Visit handlers for `br`, PHI, binary ops, calls (in `SCCPSolver.cpp`).
5. `tryToReplaceWithConstant` + `removeNonFeasibleEdges`.
6. `ValueLatticeElement` state machine.

## 12. Visit handlers worth opening

Inside `SCCPSolver.cpp`, search for visitors on:

- Terminators (`br`, `switch`) — mark successor edges when the condition’s lattice is constant; otherwise mark all successors when forced.
- `PHINode` — meet only executable incomings.
- Binary / cast / cmp — fold when operands constant; otherwise merge toward overdefined or ranges.
- `CallBase` — `handleCallOverdefined` vs tracked-function paths.
- Load/store on tracked globals (IPSCCP-oriented APIs; function SCCP rarely tracks globals).

When a unit test fails to fold, set a breakpoint after `solve()` and print `getLatticeValueFor` for the stubborn value.

## 13. Feasible edges after solve

`removeNonFeasibleEdges` rewrites a terminator so successors that are not feasible become unreachable (often via an inserted unreachable block helper). That is the concrete “rewrite branches to constants / kill edges” step students see in IR diffs even when block deletion is partial.

`isEdgeFeasible(From, To)` is the query used both during solving and during cleanup.

## 14. PredicateInfo copies

PredicateInfo inserts SSA copies that carry edge predicates (`ssacopy`-like machinery). `addPredicateInfo` installs them; `removeSSACopies` cleans them after propagation. When reading IR mid-pass under a debugger, temporary predicate copies can look unfamiliar—they are scaffolding, not source-level values.

## 15. Widen and loops

Constant ranges in loops can grow each iteration. `MergeOptions::CheckWiden` / `MaxWidenSteps` stop infinite precision chasing by jumping to overdefined after too many extensions. That is why a value “almost constant” in a loop may still end overdefined under SCCP—by design.

## 16. Replaceability and provenance

`isReplaceableConstant` exists because pointer constants may carry provenance constraints; unconditionally replacing every use can be wrong. Integer singleton ranges are the easy case. When teaching, prefer integer examples before pointer-constant propagation.

## 17. Takeaway

Function SCCP is a dual-worklist sparse solver over `ValueLatticeElement` plus executable edges, shared with IPSCCP through `SCCPSolver`, finishing by constant replacement and infeasible-edge rewrite.
