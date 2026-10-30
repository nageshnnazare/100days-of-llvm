# Day 14 Extra: CVP Deep Dive

Companion to [Day-14-0-CVP.md](Day-14-0-CVP.md). Examples: [Day-14-2-Extra-CVP-Examples.md](Day-14-2-Extra-CVP-Examples.md).

## Diagram

```text
   %c = icmp ult i32 %x, 10
   br i1 %c, label %then, label %else

   then:                         else:
   LVI facts:                    LVI facts:
   %x in [0, 9]                  %x in [10, unsigned max]
   %d = icmp ult i32 %x, 10      %e = icmp ult i32 %x, 5
   --> true                      --> false
```

CVP uses LazyValueInfo, which intersects conditions along the path to the block. Inside `%then` the compiler already knows `%x` is below 10, so a second identical compare is the constant `true`. The information is a range or a predicate, not only a single constant, which is why this catches cases SCCP misses.

## 1. Files and entry points

- [llvm/lib/Transforms/Scalar/CorrelatedValuePropagation.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/CorrelatedValuePropagation.cpp)
- [llvm/include/llvm/Transforms/Scalar/CorrelatedValuePropagation.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/CorrelatedValuePropagation.h)
- LVI: [llvm/lib/Analysis/LazyValueInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyValueInfo.cpp)

`DEBUG_TYPE` is `"correlated-value-propagation"`.

`CorrelatedValuePropagationPass::run` obtains `LazyValueInfo` from `LazyValueAnalysis`, plus DominatorTree and other analyses the file’s `run` signature requires, then calls `runImpl`.

## 2. `runImpl` structure

The function walks the function’s blocks and instructions with a large `switch` on opcode / isa checks:

- Selects → `processSelect`
- PHIs → `processPHI` / `simplifyCommonValuePhi`
- ICmps / FCmps → `processCmp` → `processICmp` / `constantFoldCmp`
- Calls → `processCallSite` (intrinsics: abs, overflow, saturating, minmax, cmp, nonnull args, …)
- Binops → `processBinOp`, `processAnd`, div/rem handlers, `processAShr`
- Casts → `processSExt`, `processZExt`, `processTrunc`, sitofp/uitofp helpers
- Terminators: `processSwitch` on `SwitchInst`

Statistics at the top of the file (`NumCmps`, `NumDeadCases`, `NumNW`, …) tell you which rewrites fired in a build with stats enabled.

## 3. Compare path

`processICmp` fetches `ConstantRange` for both operands at their uses. It may:

- Constant-fold the compare via range predicates.
- Replace a signed predicate with an unsigned equivalent when ranges are sign-insensitive / non-negative (`ConstantRange` helpers like `areInsensitiveToSignednessOfICmpPredicate`, `getEquivalentPredWithFlippedSignedness`).

`constantFoldCmp` is the “LVI says this compare is constant at this point” hammer.

## 4. Switch path

`processSwitch`:

1. Range-analyze the condition at the switch.
2. Remove case values impossible under that range (`NumDeadCases`).
3. DominatorTree may be updated as cases disappear—read the DT touches next to the case-erasure logic.

Impossible cases become SimplifyCFG/ADCE food if the switch is not fully folded here.

## 5. Division and remainder

Handlers distinguish unsigned vs signed:

- `processUDivOrURem` — nonzero denominator ranges; narrowing; sometimes expand patterns.
- `processSDivOrSRem` / `processSDiv` / `processSRem` — sign-domain facts; convert to udiv/urem when both domains nonnegative; narrow widths when high bits unused.

Always require proofs that rewriting does not introduce UB (div-by-zero) or change poison behavior incorrectly. `UndefAllowed` is true only where the code carefully allows it (read each call site).

## 6. Nowrap inference

`computeNoWrapFlags` uses `ConstantRange::makeGuaranteedNoWrapRegion` for the opcode and checks whether the other operand’s range lies inside the safe region.

`processBinOp`:

1. Try flags at the definition using ranges at the binop’s operand uses.
2. If still missing flags, `inferNoWrapFromUses` re-queries ranges **at each user site** (capped use count, skip same-block users) and intersects the proven flags.

This is where poison discipline matters: a flag is only added when the guaranteed region contains the operand range with undef disallowed on the critical queries.

**Historical note:** `-cvp-dont-add-nowrap-flags` once disabled adding these flags. If your tree still has it, use it for A/B diffs; if not, reason about `NumNSW`/`NumNUW` statistics and IR flags directly. The teaching point remains: **ranges → nsw must be poison-correct**.

## 7. Select and PHI

`processSelect` uses LVI to see if the condition or an arm is known at the select. PHI handling looks for common incoming values and edge-specific replacements (`getValueOnEdge`). CVP can delete a PHI when all live incomings match (`NumPhiCommon`).

## 8. What CVP does NOT do

- Full Wegman–Zadeck sparse CCP fixed point (that is SCCP/IPSCCP).
- Rewrite arbitrary floating-point algebra (cmp path exists; ranges are integer-centric power).
- Guarantee CFG cleanup—it rewrites instructions; dead blocks need other passes.
- Blindly add nsw from “probably in range if we allow undef.”

## 9. Analyses

**Required:** LazyValueInfo; DominatorTree for several transforms.

**Related:** AssumptionCache feeds LVI; JumpThreading before CVP improves edge structure.

When CVP mutates IR, LVI caches can be invalidated—watch for `LVI->erase` / notify-style calls near erases in the .cpp file.

## 10. Debugging

```bash
opt -passes=correlated-propagation -debug-only=correlated-value-propagation -S in.ll
```

Enable statistics builds and look at `correlated-value-propagation.Num*` counters when comparing patches.

## 11. Source reading checklist

1. `run` / `runImpl` dispatch.
2. `processCmp` + `processSwitch`.
3. `processUDivOrURem` and `processSDivOrSRem`.
4. `computeNoWrapFlags` + `inferNoWrapFromUses`.
5. One LVI `getConstantRangeAtUse` definition.

## 12. Overflow and saturating intrinsics

`processOverflowIntrinsic` / `processSaturatingInst` use ranges to prove arithmetic cannot overflow, then expand to ordinary binops (often with nowrap flags). `willNotOverflow` for binary op intrinsics consults `makeGuaranteedNoWrapRegion`. Stats `NumOverflows` / `NumSaturating` track success.

These rewrites are high-value on code that came from checked arithmetic builtins once ranges prove the check redundant.

## 13. Abs / minmax / cmp intrinsics

`processAbsIntrinsic`, `processMinMaxIntrinsic`, `processCmpIntrinsic` fold or reshape language/runtime intrinsics when ranges make results obvious (e.g. abs of a nonnegative range). They illustrate CVP as a general “range consumer,” not only icmp/switch.

## 14. Narrowing divides

When both operands fit in a narrower width, CVP can truncate, divide in the narrow type, and extend back (`NumUDivURemsNarrowed`, `NumSDivSRemsNarrowed`). Expanded forms exist when bit tricks beat a wide div (`NumUDivURemsNarrowedExpanded`). Always check denom nonzero / sign constraints first.

## 15. Use-site nowrap vs def-site nowrap

A binop defined before a guard may still be used only after the guard. Ranges at the **definition** can be weak; ranges at **uses** can be strong. `inferNoWrapFromUses` exists for that pattern, with caps so compile time stays bounded. If any use is in the defining block, the helper bails—same-block uses do not give extra CFG correlation.

## 16. What “correlated” means operationally

Take SSA value `%x`. In block A after `x < 4`, LVI returns a tight range. In block B after `x > 100`, a different range. CVP queries at the **instruction being rewritten**, so two icmps on `%x` in A and B get different answers. SCCP’s single lattice element per value cannot express that split without predicate scaffolding; CVP leans on LVI’s positional queries instead.

## 17. Takeaway

CVP is a large opcode switch over LVI ranges: fold, narrow, and flag-annotate instructions with CFG correlation—and treat nowrap as a poison theorem, not a hint.

## 18. Return-value range union

Near the end of `runImpl`, CVP may union ranges of returned values to infer function attributes / return value facts (`NumReturns`). That is a mild function-level summary built from LVI at each `ret`, separate from IPSCCP’s tracked returns but philosophically related.

## 19. And-mask simplification

`processAnd` uses the LHS range to prove an `and` with a constant mask is a no-op (mask already implied by the range). Distinct from BDCE’s demand-based `and` deletion: CVP needs a proved subset relation on values, BDCE needs undemanded bits.

## 20. AShr to LShr

When the shifted value’s range is nonnegative, arithmetic shift equals logical shift (`processAShr`, `NumAShrsConverted`). Sometimes the shift can be removed entirely if the range is `{0}` or similar (`NumAShrsRemoved`). These are good FileCheck targets when learning.

## 21. Non-negative extensions

`processPossibleNonNeg` / `processZExt` / sitofp paths set `nneg` or convert signed casts when ranges prove nonnegative (`NumNNeg`, `NumSIToFP`). Watch for these flags in IR—they feed later InstCombine and codegen.

## 22. Reading stats in a release build

Many teaching builds disable statistics. An assertions + stats build makes `NumDeadCases` / `NumNW` visible via `opt -stats`. Use that when a rewrite is subtle in the textual IR (e.g. only a flag changed).

## 23. Select vs branch correlation

A `select` does not split the CFG, so LVI facts from a select’s condition are weaker than facts after a real branch. JumpThreading that turns selects into branches can unlock CVP. When a select “obviously” folds in your head but CVP refuses, check whether the knowledge only exists as a select condition without edge predicates.

## 24. Final CVP motto

Query LVI at the use, rewrite the instruction, respect poison when setting nsw/nuw.

## 25. See also

Main note and examples companions for this day.
