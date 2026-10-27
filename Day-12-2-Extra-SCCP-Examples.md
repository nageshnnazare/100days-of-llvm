# Day 12 Extra: SCCP Examples

Opaque pointers throughout. Prepare IR with:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes FILE.c -o FILE.ll
opt -passes=sccp -S FILE.ll -o FILE.sccp.ll
opt -passes=sccp -debug-only=sccp -S FILE.ll
```

---

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

## Example 1: Constant branch kills false edge

### Goal
Prove `br` condition constant; only one successor executable; dead side disappears from live dataflow.

### Input (C)

```c
int only_then(int x) {
  if (1)
    return x + 2;
  return x - 2;
}
```

### Input (IR)

```llvm
define i32 @only_then(i32 %x) {
entry:
  br i1 true, label %then, label %else
then:
  %a = add i32 %x, 2
  ret i32 %a
else:
  %b = add i32 %x, -2
  ret i32 %b
}
```

### Commands

```bash
opt -passes=sccp -S only_then.ll -o only_then.sccp.ll
```

### Expected
`else` not executable; SCCP cleans infeasible edge / dead block when safe; live path keeps `add … 2`.

### Why it fires
Entry executable; terminator condition is constant true → only `then` marked via executable edge worklist.

### Counterexample
`br i1 %unknown` with non-constant `%unknown` → both successors may stay executable → both adds remain.

---

## Example 2: PHI collapses on executable predecessors only

### Goal
Show PHI meeting ignores dead incoming edges.

### Input (IR)

```llvm
define i32 @phi_const(i1 %c) {
entry:
  br i1 true, label %left, label %right
left:
  br label %merge
right:
  br label %merge
merge:
  %p = phi i32 [ 7, %left ], [ 9, %right ]
  ret i32 %p
}
```

### Commands

```bash
opt -passes=sccp -S phi_const.ll -o phi_const.sccp.ll
```

### Expected
`right` edge dead; `%p` becomes constant `7` (replace via `tryToReplaceWithConstant`); return constant.

### Why it fires
PHI meet only sees executable predecessor `%left` with constant 7.

### Counterexample
`br i1 %c` (argument) → both edges live → PHI overdefined / non-constant → no replace.

---

## Example 3: Arithmetic chain becomes constant

### Goal
Propagate constants through SSA users on the instruction worklist.

### Input (C)

```c
int chain(void) {
  int a = 2;
  int b = a + 3;
  int c = b * 4;
  return c;
}
```

### Input (IR)

```llvm
define i32 @chain() {
entry:
  %b = add i32 2, 3
  %c = mul i32 %b, 4
  ret i32 %c
}
```

### Commands

```bash
opt -passes=sccp -S chain.ll -o chain.sccp.ll
```

### Expected
`%b` → 5, `%c` → 20; return `20` after replacement (exact IR may fold stepwise).

### Why it fires
Each def’s lattice becomes constant; users pushed on InstWorkList; final `ret` operand replaceable.

### Counterexample
Replace `2` with argument `%x` → overdefined chain → no full fold.

---

## Example 4: Argument range attribute seeds lattice

### Goal
See `trackValueOfArgument` honor range metadata/attributes when present.

### Input (IR sketch)

```llvm
define i32 @ranged(i32 range(i32 0, 1) %x) {
entry:
  %c = icmp eq i32 %x, 0
  br i1 %c, label %z, label %o
z:
  ret i32 100
o:
  ret i32 200
}
```

### Commands

```bash
opt -passes=sccp -S ranged.ll -o ranged.sccp.ll
opt -passes=sccp -debug-only=sccp -S ranged.ll
```

### Expected
Depending on range encoding and solver state, `%x` may be constrained enough to fold the icmp / one edge. If the range is `{0}` effectively, only `%z` stays interesting.

### Why it fires
Argument tracking initializes lattice from attributes instead of immediately overdefined.

### Counterexample
Bare `i32 %x` with no range → icmp not constant → both blocks executable.

---

## Example 5 (bonus): Compare SCCP vs needing IPSCCP

### Goal
Show function SCCP cannot see callee constants across TUs / other functions without IPSCCP.

### Input (IR)

```llvm
define internal i32 @secret() {
  ret i32 42
}
define i32 @caller() {
  %v = call i32 @secret()
  ret i32 %v
}
```

### Commands

```bash
opt -passes=sccp -S call.ll -o call.sccp.ll
opt -passes=ipsccp -S call.ll -o call.ipsccp.ll
```

### Expected
Function `sccp` alone often leaves the `call` (return tracking may infer attrs but full zap is IPSCCP’s game). `ipsccp` can replace the call result with `42` for eligible internal callees.

### Why it fires / why not
Shared solver, different client: IPSCCP tracks functions/returns across the module.

### Counterexample
`define i32 @secret()` external linkage → IPSCCP must not assume the body is the only definition.

---

## Hands-on checklist

1. Constant `br` → one edge.
2. PHI with dead predecessor.
3. Integer arithmetic constant chain.
4. Float: try `fadd` of constants — observe conservative / limited folding vs integers.
5. Pipe `sccp,simplifycfg` and diff against `sccp` alone.
