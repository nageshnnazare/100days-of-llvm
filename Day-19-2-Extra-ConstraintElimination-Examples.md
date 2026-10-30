# Day 19 Extra: Constraint Elimination — Examples

Overview: [Day-19-0-ConstraintElimination.md](Day-19-0-ConstraintElimination.md). Deep dive: [Day-19-1-Extra-ConstraintElimination-DeepDive.md](Day-19-1-Extra-ConstraintElimination-DeepDive.md).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=constraint-elimination -S file.ll -o file.out.ll
opt -passes=constraint-elimination -debug-only=constraint-elimination -S file.ll
```

---

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

## Example 1: Redundant unsigned bounds check

### Goal

Fold a second `icmp ult` dominated by an identical check.

### Input

```llvm
define i32 @twice(ptr %a, i32 %i, i32 %n) {
entry:
  %cmp = icmp ult i32 %i, %n
  br i1 %cmp, label %in.range, label %out
in.range:
  %cmp2 = icmp ult i32 %i, %n
  br i1 %cmp2, label %ok, label %out2
ok:
  %p = getelementptr inbounds i32, ptr %a, i32 %i
  %v = load i32, ptr %p, align 4
  ret i32 %v
out2:
  ret i32 -1
out:
  ret i32 0
}
```

### Commands

```bash
opt -passes=constraint-elimination -S twice.ll -o twice.out.ll
```

### Expected

`%cmp2` becomes `true` (or the branch to `ok` becomes unconditional). The dominating `%cmp` fact implies `%cmp2`.

### Why it fires

Same unsigned predicate and operands; fact holds throughout `in.range`.

### Counterexample

Move `%cmp2` into a block not dominated by the true edge of `%cmp` (e.g. also reachable from `out`). The fact does not apply → no fold.

---

## Example 2: `llvm.assume` supplies the fact

### Goal

Show assumes as first-class constraint sources.

### Input

```llvm
declare void @llvm.assume(i1)

define i32 @with_assume(i32 %i, i32 %n) {
entry:
  %ok = icmp ult i32 %i, %n
  call void @llvm.assume(i1 %ok)
  %chk = icmp ult i32 %i, %n
  %z = zext i1 %chk to i32
  ret i32 %z
}
```

### Commands

```bash
opt -passes=constraint-elimination -S with_assume.ll -o with_assume.out.ll
```

### Expected

`%chk` folds to true; `%z` becomes `1` after further folding (CE may only const-ify the icmp; InstCombine finishes).

### Why it fires

Assume adds `i < n` at `entry`; the later icmp is implied.

### Counterexample

Assume a **different** relation (`icmp eq i32 %i, 0`) while checking `i < n` with unrelated `%n` → generally no fold.

---

## Example 3: Implied tighter / related inequality

### Goal

Use a stronger dominating fact to prove a weaker later check when the linear system allows.

### Input

```llvm
define i1 @tighter(i32 %i, i32 %n) {
entry:
  ; i <= n-2  (as ult i, n-1)  — construct explicitly
  %nm1 = add i32 %n, -1
  %strong = icmp ult i32 %i, %nm1
  br i1 %strong, label %bb, label %no
bb:
  %weak = icmp ult i32 %i, %n
  ret i1 %weak
no:
  ret i1 false
}
```

### Commands

```bash
opt -passes=constraint-elimination -S tighter.ll -o tighter.out.ll
```

### Expected

Under a successful decomposition of `add`/`ult`, `%weak` may fold to true inside `bb` because `i < n-1` implies `i < n` for these unsigned encodings when wrap is handled correctly.

### Why it fires

Linear implication inside the unsigned system.

### Counterexample

Replace the strong check with a **signed** `slt` while keeping an `ult` query (or vice versa) without safe transfer—fold should not happen.

---

## Example 4: Signed vs unsigned mismatch (negative)

### Goal

Demonstrate that signed facts do not freely prove unsigned queries.

### Input

```llvm
define i1 @sign_mismatch(i32 %i, i32 %n) {
entry:
  %s = icmp slt i32 %i, %n
  br i1 %s, label %bb, label %no
bb:
  %u = icmp ult i32 %i, %n
  ret i1 %u
no:
  ret i1 false
}
```

### Commands

```bash
opt -passes=constraint-elimination -S sign_mismatch.ll -o sign_mismatch.out.ll
```

### Expected

`%u` typically **remains**. `i < n` signed does not mean `i < n` unsigned (negative `%i` is a classic counter-model).

### Why it does not fire

Separate signed/unsigned systems; unsound to transfer this pair.

### Counterexample (positive)

Use `icmp ult` for both dominating and dominated checks with the same operands → folds as in Example 1.

---

## Example 5: Nonlinear obstacle

### Goal

Show decomposition limits.

### Input

```llvm
define i1 @nonlinear(i32 %a, i32 %b, i32 %n) {
entry:
  %prod = mul i32 %a, %b
  %c = icmp ult i32 %prod, %n
  br i1 %c, label %bb, label %no
bb:
  %c2 = icmp ult i32 %prod, %n
  ret i1 %c2
no:
  ret i1 false
}
```

### Commands

```bash
opt -passes=constraint-elimination -S nonlinear.ll -o nonlinear.out.ll
```

### Expected

The **identical** second compare on the same `%prod` SSA value can still fold (same value, same predicate)—CE does not need to open the `mul`.  

Now change `%c2` to compare `%a` against `%n` and expect **no** proof from `%a*%b < n`.

```llvm
  %c2 = icmp ult i32 %a, %n   ; not implied by a*b < n
```

### Why the variant fails

Product of two variables is outside useful linear reasoning for that implication.

### Counterexample (linear mul)

`mul i32 %a, 3` (constant factor) can participate in rows; dominating facts about `%a` may transfer with coefficient 3 when nuw/wrap rules allow.

---

## Example 6: Fact does not cross the false edge

### Goal

Dominance scoping.

### Input

```llvm
define i32 @false_edge(i32 %i, i32 %n) {
entry:
  %c = icmp ult i32 %i, %n
  br i1 %c, label %yes, label %no
yes:
  ret i32 1
no:
  %c2 = icmp ult i32 %i, %n
  %z = zext i1 %c2 to i32
  ret i32 %z
}
```

### Commands

```bash
opt -passes=constraint-elimination -S false_edge.ll -o false_edge.out.ll
```

### Expected

`%c2` on the false edge may become **false** (because the edge implies `i >= n`), or remain if the negation encoding does not apply in your revision—check the output. It must **not** become true.

### Why

The false edge carries the complementary fact, not the true-edge fact.

### Counterexample

Place `%c2` in `yes` → becomes true.

---

## Aftercare

```bash
opt -passes='constraint-elimination,simplifycfg,dce' -S file.ll -o file.clean.ll
opt -passes=verify -disable-output file.out.ll
```
