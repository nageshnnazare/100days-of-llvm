# Day 9: Reassociate

Reassociate rewrites chains of associative operators into a canonical shape: constants gathered on one side, operands sorted into a stable order. The rewrite is not usually a speedup by itself. It is a setup pass. After it runs, EarlyCSE and InstCombine see duplicates and foldable constants that were previously written in different orders.

---

## 1. Why this exists

```c
int f(int x, int y) {
  int a = (x + 1) + y;
  int b = (y + x) + 1;
  return a - b;
}
```

Both expressions are `x + y + 1`. Written as two different trees, EarlyCSE does not match them, and the subtract does not fold to zero. Reassociate turns both into the same ranked chain. InstCombine then folds `a - b` to 0.

The same rewrite groups constants: `(x + 1) + 2` becomes `x + 3` after reassociate and instcombine, even when the constants were not adjacent in the source.

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

## 2. The idea

For an operator that is associative and commutative (`add`, `mul`, `and`, `or`, `xor`, and floating-point `fadd` / `fmul` only when the `reassoc` fast-math flag is present):

1. Flatten the tree into a list of leaves with integer weights. `x + x` is `x` with weight 2. `x - y` is `x` with weight 1 and `y` with weight -1, because `x - y` is `x + (-y)`.
2. Combine constants in that list.
3. Sort the list by **rank**, a number that reflects where the value is defined in the function. Sorting by rank puts operands in a deterministic order so two expressions with the same leaves become the same tree.
4. Rebuild the tree as a left- or right-associated chain in that order.

The rebuilt tree computes the same value only when the operator really is associative. Signed overflow and floating-point rounding are why the pass is careful about flags.

## 3. Where it sits in the pipeline

Pass name: `reassociate`. File: [llvm/lib/Transforms/Scalar/Reassociate.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/Reassociate.cpp). Class `ReassociatePass`.

It runs in the function simplification pipeline next to InstCombine and EarlyCSE, typically once per simplification cycle. It does not need alias analysis. It uses `AssumptionCache` in some versions and always walks the function's instructions.

```bash
opt -passes=reassociate -S in.ll
opt -passes=reassociate,instcombine,early-cse -S in.ll
```

Judging the pass from its own diff is misleading. The interesting diff appears after the InstCombine that follows it.

## 4. Core concepts

### Rank

`getRank(Value *)` assigns:

- constants the lowest rank, so they sort to one end and combine,
- arguments a rank based on their argument number,
- instructions a rank based on their order in a reverse postorder numbering of the function, so an operand defined earlier sorts before an operand defined later.

The absolute numbers do not matter. The order does. Two identical leaves get the same rank, and the sort is stable enough that the rebuilt expression is unique.

Rank is recomputed if the function changes. The pass numbers instructions once per run.

### Linearization

`LinearizeExpr` / the function that builds `std::pair<Value *, APInt>` factors (search for `Linearize` or `RewriteExprTree` in the file) walks a chain of the same opcode and produces a vector of `(leaf, weight)`.

For `add`:

- `add X, Y` contributes the leaves of X and the leaves of Y.
- `sub X, Y` is treated as adding X and adding the negation of Y.

Negation weight is how `x + y - x` becomes `y` with the two `x` weights cancelling.

For `mul`, weights are exponents: `x * x * y` is `x^2 * y`. The pass can turn that into `x * x * y` in a canonical order, which is what a later `InstCombine` strength-reduces. Reassociate itself is not the pass that turns `x * 8` into `shl`.

For `xor`, the same flattening implements `x ^ y ^ x -> y`.

### Rebuilding

`RewriteExprTree` takes the linearized, sorted factors and reuses the existing instruction nodes when it can, to avoid allocating a brand-new chain and to keep debug info slightly more stable. The shape of the result is a spine of the same opcode.

### Flags and poison

`add nsw` is not associative in LLVM's poison model. `(a +nsw b) +nsw c` can be poison when `a +nsw (b +nsw c)` is not, and the reverse. Reassociate drops `nsw` and `nuw` on expressions it rewrites, unless it can prove the rewrite does not create new poison. Dropping the flags is the conservative default: the new expression is defined in more cases, never fewer. A later pass may re-infer `nsw` if it can.

If you see `nsw` disappear after `reassociate` and a microbenchmark changes, this is why. The pass traded a poison-on-overflow promise for a canonical shape.

### Floating point

`fadd` without `reassoc` is not associative. `0.1 + (x + -0.1)` is not something LLVM will turn into `x` unless the instruction has the `reassoc` flag (Clang `-ffast-math` or the more specific `-fassociative-math` family sets these flags). Reassociate checks `I->hasAllowReassoc()` (the fast-math flag) before touching floating-point operators. Do not expect this pass to fire on ordinary `float` code.

### What it refuses

- Operators that are not associative: `sub` is handled only by being rewritten into `add` of a negation, not by sorting a `sub` chain as if subtraction were commutative.
- `shl`, `div`, comparisons, calls.
- Expressions with more than one use of an intermediate, when rewriting would duplicate work. The pass uses a heuristic (`-reassociate-use-limit` or the internal use-count check) so it does not explode compile time or code size by pulling a shared subexpression into two different chains. Search for the use-count bailout in the file.

## 5. The algorithm

For each basic block, for each instruction that is a candidate opcode, bottom-up or in an order that sees leaves first:

1. If the instruction is already in canonical form, skip it.
2. Linearize the chain into weighted leaves.
3. Fold constant leaves together.
4. Cancel leaves whose weights sum to zero (`x` and `-x` in an add, `x ^ x` in an xor).
5. Sort by rank.
6. Rewrite the tree.
7. If the whole expression collapsed to a single leaf or a constant, replace the instruction.

The pass repeats within the block when a rewrite creates a new opportunity. It is a worklist, not a single scan, but it is local to chains of one opcode.

## 6. Where to read the code

1. [llvm/lib/Transforms/Scalar/Reassociate.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/Reassociate.cpp) — `ReassociatePass::run`, `getRank`, `ReassociateExpression`, `LinearizeExpr`, `RewriteExprTree`.
2. The tests in `llvm/test/Transforms/Reassociate/` — small and they show the canonical form directly.

Read `getRank` first. Then one of `OptimizeAdd`, `OptimizeXor`, or the unified optimizer if the version you are reading has merged them. Upstream has gone through both shapes: per-opcode helpers and a shared factor list. The factor-list explanation in section 4 matches either structure.

## 7. Worked example

```llvm
define i32 @same(i32 %x, i32 %y) {
  %a1 = add i32 %x, 1
  %a = add i32 %a1, %y
  %b1 = add i32 %y, %x
  %b = add i32 %b1, 1
  %d = sub i32 %a, %b
  ret i32 %d
}
```

```bash
opt -passes=reassociate,instcombine -S same.ll
```

**Expected:** `ret i32 0`.

Run the passes one at a time and look at the IR after `reassociate` alone. You should see both `%a` and `%b` turned into the same shape, roughly `(%x + %y) + 1` with the same association. InstCombine, or EarlyCSE plus InstCombine, does the subtraction.

## 8. Reading order

1. Section 4, rank and weight.
2. `getRank` in the source.
3. One optimize function, with the worked example IR next to it.
4. The floating-point counterexample in the examples file, so you do not "fix" it.

## 9. Hands-on

```bash
opt -passes=reassociate -S same.ll -o step1.ll
opt -passes=instcombine -S step1.ll -o step2.ll
opt -passes=reassociate -debug-only=reassociate -S same.ll   # asserts build
```

Try `-ffast-math` on a C version of `(x + 1.0) + y` versus `y + (x + 1.0)` and dump IR:

```bash
clang -O1 -ffast-math -S -emit-llvm -Xclang -disable-llvm-passes fp.c -o fp.ll
opt -passes=reassociate -S fp.ll
```

You should see `reassoc` on the `fadd` instructions in `fp.ll`, and the pass will reorder them. Remove `-ffast-math` and the pass should leave them alone.

## 10. Pitfalls and invariants

- **Reassociate is allowed to drop `nsw`.** If your optimization depends on `nsw` surviving this pass, you are depending on a non-guarantee. Re-infer flags after canonicalization if you need them.
- **Intermediates with many uses are left alone.** A chain that does not look fully sorted may be a shared subexpression the pass refused to duplicate.
- **Rank order is not source order and not alphabetical.** Do not write a FileCheck that pins the exact operand order unless you have just run the pass and used `update_test_checks.py` (Day 91).
- **It does not reassociate across selects or phis.** Each chain stops at a non-add. A phi of two adds is not one chain.
- **Canonical form is a means.** If the following InstCombine did not fold anything, reassociate may still have "worked". Look at whether the two trees became the same tree.

## 11. Neighbors

| Day | Relationship |
| --- | --- |
| Day 1 Constant folding | Folds the constants reassociate pushed together |
| Day 2 InstCombine | The main consumer; also does smaller associative folds of its own |
| Day 8 EarlyCSE | Matches the newly identical trees |
| Day 13 BDCE / demanded bits | Orthogonal: bits versus expression shape |
| Day 29 IndVarSimplify | A different, SCEV-based canonicalization for induction variables |

## 12. Check yourself

**Why doesn't EarlyCSE alone fold the worked example?**
The two adds are different trees. EarlyCSE matches opcodes and operands, not algebraic equivalence.

**What is a rank?**
A sort key derived from where a value is defined, plus a special low rank for constants, used to give each expression one shape.

**Why doesn't reassociate reorder `fadd` in default C?**
IEEE `fadd` is not associative. The `reassoc` fast-math flag is required.

**What happens to `nsw` on an add the pass rewrites?**
It is dropped unless the rewrite is proven not to introduce poison.

**`(x ^ y) ^ x`. What should come out?**
`y`, after reassociate or instcombine. XOR's weight cancellation removes `x ^ x`.

> [!TIP]
> Ranks, weights, and the rewrite limit are in [Day-09-1-Extra-Reassociate-DeepDive.md](Day-09-1-Extra-Reassociate-DeepDive.md). More chains, including the ones that must not move, are in [Day-09-2-Extra-Reassociate-Examples.md](Day-09-2-Extra-Reassociate-Examples.md).
