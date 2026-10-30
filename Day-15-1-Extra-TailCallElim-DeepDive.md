# Day 15 Extra: Tail Call Elimination Deep Dive

Companion to [Day-15-0-TailCallElim.md](Day-15-0-TailCallElim.md). Examples: [Day-15-2-Extra-TailCallElim-Examples.md](Day-15-2-Extra-TailCallElim-Examples.md).

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

## 1. Files and entry points

- [llvm/lib/Transforms/Scalar/TailRecursionElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/TailRecursionElimination.cpp)
- [llvm/include/llvm/Transforms/Scalar/TailRecursionElimination.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/TailRecursionElimination.h)

`DEBUG_TYPE` is `"tailcallelim"`.

New PM: `TailCallElimPass::run` gathers TTI, AA, ORE, optional BFI (when entry counts exist), ProfileSummaryInfo via module proxy, cached DT/PDT, builds `DomTreeUpdater`, calls `TailRecursionEliminator::eliminate`.

## 2. `eliminate` control flow

```text
disable-tail-calls? → abort
markTails(...)
varargs? → return (marking may have run)
canTRE? (no dynamic alloca)
for each BB: processBlock → findTRECandidate → eliminateCall
cleanupAndFinalize
```

TRE is skipped independently of marking when dynamic allocas or varargs intervene.

## 3. `markTails` and alloca tracking

`AllocaDerivedValueTracker::walk` starts from static allocas and byval arguments. It classifies:

- **AllocaUsers** — calls that receive alloca-derived values
- **EscapePoints** — stores of the pointer, capturing calls, etc.

Byval at a callee argument is **not** treated as escape (copy semantics). `nocapture` avoids treating the call as capturing for further propagation.

`markTails` refuses functions that call `setjmp`-like returns-twice callees. It uses AA and optional PSI/BFI for remarks / cold callsite behavior when deciding aggressiveness of marking. Successful marks set the `tail` flag on `CallInst`.

## 4. `canTRE`

```cpp
// Dynamic allocas → false
return all allocas are static (isStaticAlloca)
```

Comment in-tree: dynamic stack allocation would need deallocation before the next iteration—unimplemented.

## 5. Finding and eliminating candidates

`findTRECandidate(BasicBlock *BB)` looks for a recursive call in a form `processBlock` can eliminate (tail `ret` patterns, including some `select` of recursive results depending on current code).

`eliminateCall`:

1. Optionally create byval temp allocas/copies when updating arguments that need memory.
2. Detect accumulator recursion via `canTransformAccumulatorRecursion`.
3. Emit optimization remarks (`tailcall-recursion`).
4. Rewrite CFG: branch to entry-like loop header; PHI argument updates; remove the call; fix returns to use accumulator materialization helpers when needed.
5. DomTreeUpdater deletes empty successors when safe.

Accumulator init values and PHI insertion live in `insertAccumulator` / `MaterializeAccumulator` lambdas near finalize.

## 6. Accumulator eligibility

`canTransformAccumulatorRecursion` / `isUnaryAccumulatorRecurrence` decide whether an instruction combining the recursive call’s result with another value can become a running PHI update. Associative ops with identities (0 for add, 1 for mul, etc.—as implemented) qualify. If the op is not suitable, TRE still may rewrite **accumulator-passing** style recursion without inventing a new op.

## 7. What is NOT handled

- TRE across **different** functions (only self-recursion).
- TRE with **dynamic allocas**.
- TRE on **varargs**.
- Guaranteeing codegen sibling calls—only marks `tail`.
- Lowering `musttail` (already a verifier-level contract).
- Arbitrary escaping frame pointers.

## 8. Debugging and remarks

```bash
opt -passes=tailcallelim -debug-only=tailcallelim -S in.ll
```

Optimization remarks use `DEBUG_TYPE` (`tailcall-readnone`, `tailcall-recursion`, …). Enable remarks through the usual `-pass-remarks=tailcallelim` style flags when testing.

## 9. Analyses and preservation

Uses: TargetTransformInfo, AliasAnalysis, OptimizationRemarkEmitter, DomTreeUpdater; optional BlockFrequencyInfo + ProfileSummaryInfo.

Preserves DominatorTree / PostDominatorTree when updated correctly (`PreservedAnalyses` in the pass `run`).

## 10. Byval pitfalls during TRE

When a recursive call passes byval aggregates, updating the “argument” for the next iteration may require copying into a temporary alloca. Readonly byval parameters complicate whether the callee may have mutated the copy—the file comments discuss forcing careful updates. Misreading this is a common source of TRE bugs historically.

## 11. Source reading checklist

1. `TailCallElimPass::run`.
2. `markTails` + tracker switch on opcodes.
3. `canTRE`.
4. `findTRECandidate` / `eliminateCall`.
5. Accumulator insert + finalize returns.

## 12. `processBlock` cases

`processBlock` looks at the block terminator:

- `ReturnInst` path: find a TRE candidate call that feeds the return (directly or through limited patterns).
- Other terminators / `select`-shaped returns: additional candidate search as implemented.

If a successor becomes empty after eliminating a call, `DTU.deleteBB` may remove it. Read `processBlock` fully once—TRE is not only “call then ret” in one basic block; PHIs and multiple returns appear in real functions.

## 13. `cleanupAndFinalize`

After all blocks are processed, finalize rewrites returns to materialize accumulators, clean temporary state, and update function entry counts when `UpdateFunctionEntryCount` is set (profile-aware builds). Skipping finalize mentally when reading diffs will make accumulator IR look half-finished.

## 14. AliasAnalysis role

AA helps decide whether calls capture or clobber memory in ways that block `tail` marking. Combined with the alloca tracker’s syntactic walk, AA covers harder cases (may-alias stores). When a call looks tail-safe but is not marked, dump AA facts around escaping pointers.

## 15. TargetTransformInfo

TTI can influence profitability / legality edges of TRE (target preferences). Most teaching examples do not need TTI detail; still note it is a required analysis in `run`.

## 16. Comparison table (mid-end vs codegen)

| Question | Answer owner |
|----------|----------------|
| Recursion → loop? | `tailcallelim` TRE |
| Call gets `tail` flag? | `markTails` |
| Emit sibling JMP? | Backend ISel / lowering |
| Guaranteed matching tail call? | `musttail` + verifier |

Keep this table next to the Day 15 main note.

## 17. Takeaway

Read the pass as two serial phases—**mark** then **TRE**—sharing escape analysis ideas but different success criteria: `tail` for backends, loops for self-recursion.

## 18. `eliminateCall` argument PHI update

For each function argument, TRE creates (or reuses) a PHI in the loop header. The recursive call’s actual operands become the new incoming values from the recursive edge; the original entry operands come from the pre-header/entry edge. Getting operand order wrong is a classic bug when reading or modifying this code—match formals to call operands by index carefully, including attributes that move with them.

## 19. Readonly byval and copies

Comments near byval lowering note that a byval argument is a copy; if the parameter is also marked readonly, the callee promises not to write through it, which constrains how TRE may reuse buffers across iterations. When TRE inserts temporary allocas for byval updates, it must honor those semantics. Prefer scalar i32 examples before byval TRE.

## 20. Function entry counts

When profile entry counts exist, TRE may update them as recursion becomes a loop (different dynamic call count interpretation). Controlled by `UpdateFunctionEntryCount` in the eliminator. Relevant for PGO consistency; ignore on expect-only Day 16-style IR without counts.

## 21. Remarks strings

Search the cpp for `OptimizationRemark(DEBUG_TYPE, …)` to see remark identifiers used in `-pass-remarks=tailcallelim` output. Matching a remark to a source call site is often faster than eyeballing PHI-heavy IR.

## 22. Verifier after TRE

Always run `opt -passes='tailcallelim,verify'`. PHI arity bugs and broken `musttail` lookalikes show up immediately. TRE should leave well-formed loop IR.

## 23. Final TRE motto

Mark `tail` for the backend; loopify self-recursion when the frame stays static and the call is truly tail.

## 24. See also

Main note and examples companions for this day.
