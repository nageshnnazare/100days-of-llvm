# Day 9 Extra: Deep Dive into Reassociate.cpp

Companion to [Day-09-0-Reassociate.md](Day-09-0-Reassociate.md). File: [llvm/lib/Transforms/Scalar/Reassociate.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/Reassociate.cpp).

---

## Diagram

```text
   before                         after reassociate

   (%a + 1) + %b                  (%a + %b) + 1
        ^                              ^
   rank(const) is lowest          constants are pulled
   so they sink to the right      to the right end

   (%x * 2) * 3                   %x * 6
```

Reassociate ranks operands (constants lowest, arguments next, instructions by their order) and rewrites a chain into one canonical order. Two expressions that were algebraically the same but parenthesized differently then hash to the same thing for CSE. Integer reassociation drops `nsw`: the rewritten form must not invent overflow poison.

## 1. The pass driver

`ReassociatePass::run` builds a rank map for the function, then walks instructions. The rank map is filled by a reverse-postorder scan: each instruction gets a rank higher than its operands' ranks, with arguments and constants fixed at the bottom. `BuildRankMap` (search for `RankMap`) is that scan.

Doing rank before rewriting means the sort key does not depend on the order the pass happens to rewrite instructions. Two runs on the same function produce the same shape.

Changed instructions are revisited. The pass keeps a worklist or re-scans the block. Look at the loop in `run` to see which. The important property is that a rewrite which creates a longer chain gets another look.

## 2. `getRank`

Constants return a rank below every instruction so they cluster. Arguments get ranks from their position in the argument list so `f(a, b)` and a similar expression written with the arguments swapped still canonicalize the same way only if the sort is on rank — argument 0 sorts before argument 1. That means canonical form *depends on argument order*, which is what you want for CSE inside one function and not a semantic problem.

Instructions return the rank assigned during the scan. A phi's rank is the phi's own rank, not the ranks of its incoming values. The pass does not look through phis. If it did, it would reassociate across control flow and might lengthen live ranges badly.

## 3. Linearize

Search for `LinearizeExpr`. It takes an instruction and a vector of factors.

For addition it recursively linearizes operands that are themselves adds with a single use (or within the use limit). Each leaf is appended with weight `+1`. A `sub` contributes its first operand with positive weight and its second with negative weight, which may itself be an expression to negate.

Negating a product or a sum flips every weight. That is how `-(x + y)` becomes a contribution of `-x` and `-y` inside a surrounding add. The implementation has to avoid double-negating when the same value appears through two paths; single-use checks are what keep this a tree walk rather than a DAG walk.

XOR linearization uses weight modulo 2: `x ^ x` has weight 0 and disappears. AND/OR are similar but weights do not cancel the same way; they mostly sort and flatten. Read the opcode switch rather than assuming every opcode supports negative weights. Only `add` has a meaningful negative weight. `xor` has parity. `mul` has positive exponents.

## 4. Constant folding inside the list

After linearization, constant factors are combined with `APInt` arithmetic. For `add`, constants sum. For `mul`, constants multiply. For `xor`, constants xor. This is not a call to the IR constant folder; it happens on the factor list before any instruction is rebuilt. The result is one constant factor, which then sorts to the constant end of the chain.

Overflow: combining the constants is exact in `APInt` of the right bit width, with wraparound matching two's complement. That part is always safe. The unsafe part is the *re-association of the non-constants*, which is where `nsw` gets dropped.

## 5. `RewriteExprTree`

This function is worth the time. It walks the old tree and the new factor list together and morphs existing `BinaryOperator` nodes in place when the opcode matches, instead of creating a parallel chain and deleting the old one. In-place rewrite:

- keeps the instruction that other uses might point at, when the root is reused,
- reduces allocator traffic,
- preserves more debug locations.

When the new chain is shorter than the old one (weights cancelled), leftover nodes are dead and get erased. When the new chain needs a node the old tree does not have, it creates one.

The leaves of the new chain are the sorted factors. Internal nodes are fresh or recycled operators of the same opcode. The root replaces the original instruction's uses if the root is a different object, via `replaceAllUsesWith`.

## 6. The use limit

Search for a check on `hasOneUse` or a small integer limit. An intermediate add that has five users cannot be pulled into one user's chain without either duplicating the add (code size) or rewriting all five users at once (compile time and live ranges). The pass linearizes through single-use intermediates freely and stops at multi-use intermediates.

This is the reason a "obviously associative" expression in a large function sometimes only partially canonicalizes. Look at the use count of the intermediate before you blame the ranker. `opt -S` and reading `uselistorder` is unnecessary; count the uses in the IR.

## 7. Flag dropping

Search for `dropPoisonGeneratingFlags` or `setHasNoSignedWrap(false)`. On any operator the pass rewrites, wrapping flags are cleared. Fast-math flags on floating-point are *kept* when they are what made the rewrite legal (`reassoc` must remain, and `nsz` / `ninf` are generally preserved so later folds still see them). There is a difference between "flags that justify the transform" and "flags the transform invalidates". Poison flags are the second kind for reassociation. `reassoc` is the first kind.

If a comment in the source cites a missed optimization around preserving `nsw` when both sides provably do not overflow, believe the comment for the version you are reading. The conservative behavior (drop the flag) is always a correct fallback.

## 8. XOR and AND/OR

`OptimizeXor` (or the shared path with opcode `Xor`) is the clearest cancellation demo:

```text
factors: x (weight 1), y (weight 1), x (weight 1)
xor weights mod 2: y (weight 1)
result: y
```

AND and OR flatten and sort but do not cancel. `(a & b) & a` becomes `a & b` because of idempotence, which may be implemented as a weight cap of 1. Look at how the factor is recorded for `And`. Idempotence is not the same as XOR parity, but the factor list can express both.

## 9. Interaction with `InstCombine`

Reassociate deliberately does not implement the full InstCombine repertoire. After it creates `(x + 3)` from `(x + 1) + 2`, the constants are one immediate, and InstCombine's `visitAdd` is what has the pattern "add of constant is just a constant on the RHS in canonical form". Sometimes reassociate emits an already-folded constant because it combined the factor list itself. Either way the *pipeline* contract is that these passes run as a group. A test that runs only `reassociate` should assert the shape (operand order, constants clustered), not the fully folded program.

## 10. Debug

```bash
opt -passes=reassociate -debug-only=reassociate -S in.ll
```

The debug output prints the expression before and after. Ranks show up in those dumps. When the pass skips an instruction, the debug line usually says it is not a candidate or that the use count blocked linearization.

No asserts build: `-print-changed=quiet` shows the rewritten function and nothing else.

## 11. Tests to copy from

`llvm/test/Transforms/Reassociate/add.ll` and `xor.ll` and `fast-ReassociateVector.ll` (names drift; list the directory). The fast-math file is the one that shows `fadd reassoc` moving and plain `fadd` staying. Those two checks are the spec for section 4 of the main notes.
