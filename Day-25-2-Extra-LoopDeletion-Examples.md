# Day 25 Extra: Loop Deletion — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa,loop-deletion' -S file.ll -o file.out.ll
```

See [Day-25-0-LoopDeletion.md](Day-25-0-LoopDeletion.md). Prefer checking `mustprogress` on the function in the IR.

---

## Diagram

```text
   header:
     %c = icmp slt i32 %i, %n
     br i1 %c, label %body, label %exit
   body:
     ; no side effects, no volatile, no escape
     br label %header
   exit:
     ret i32 0

   mustprogress is set, trip count may be infinite
   only if the loop never exits -- but a side-effect-free
   loop that the language says must make progress
   can be deleted:

   exit:
     ret i32 0
```

LoopDeletion removes a loop that does nothing observable. A finite loop whose body has no side effects disappears and control goes to the exit. An infinite loop is deleted only when the function is `mustprogress` (C++ forward-progress). Without that attribute the infinite loop stays, because deleting it would let later code run when the source never did.

## Example 1: Empty counted loop

### Input

```c
void empty(int n) {
  for (int i = 0; i < n; ++i) {
  }
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes empty.c -o empty.ll
grep mustprogress empty.ll
opt -passes='loop-simplify,lcssa,loop-deletion' -S empty.ll -o empty.out.ll
```

### Expected result

If the function/loop requires mustprogress (common) **or** SCEV proves a finite max BTC compatible with `isLoopDead`, the loop is removed → function roughly `ret void`.

### Why it fires

SEF body + progress/finiteness rules in `isLoopDead`.

### Counterexample

Hand-written IR: same empty loop, **no** `mustprogress` on function/loop, and trip count not SCEV-known → loop **kept** (may be infinite if `n` adversary / wrapping models).

---

## Example 2: Never-executed loop

### Input

```c
int never(int x) {
  if (0) {
    for (int i = 0; i < x; ++i)
      x += i;
  }
  return x;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-deletion' -S never.ll -o never.out.ll
```

### Expected result

After constants fold onto preheader predecessors, `isLoopNeverExecuted` triggers; loop deleted; exit PHIs poisoned if any. Function returns `%x` unchanged.

### Why it fires

Constant false branch never enters the preheader.

### Counterexample

`if (opaque())` with unknown condition — predecessors not constant → never-executed path does not fire (loop may still die later via other reasons if SEF+finite).

---

## Example 3: Side effect blocks deletion

### Input

```c
void use(int);
void side(int n) {
  for (int i = 0; i < n; ++i)
    use(i);
}
```

### Commands

Same deletion pipeline.

### Expected result

Loop **not** deleted: `use` is a side-effecting call (`mayHaveSideEffects`).

### Why it must not fire

Observable behavior includes calls.

### Counterexample / twin

Replace body with pure `i * 2` unused → deletable under finiteness/mustprogress rules.

---

## Example 4: Infinite SEF without mustprogress (keep)

### Input (hand-written theme)

```llvm
define void @spin() {
entry:
  br label %loop
loop:
  br label %loop
}
```

No `mustprogress`, no exit.

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-deletion' -S spin.ll -o spin.out.ll
```

### Expected result

Loop remains. Deleting it would turn a non-terminating function into an immediate `ret`/`unreachable` change of termination behavior without a progress obligation.

### Why it must not fire

`isLoopDead` refuses when progress is not required and SCEV cannot give a finite max BTC (here: no exit / infinite).

### Counterexample

```llvm
define void @spin_mp() mustprogress {
entry:
  br label %loop
loop:
  br label %loop
}
```

With `mustprogress`, deletion **may** remove the loop (infinite SEF violates progress).

---

## Example 5: Finite SEF producing an invariant out

### Input

```c
int always_five(int n) {
  int x = 5;
  for (int i = 0; i < n; ++i) {
    /* x unchanged */
  }
  return x;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-deletion' -S always_five.ll -o out.ll
```

### Expected result

Loop deletable; `return 5` (possibly after LCSSA trivial PHIs cleaned). Exit PHI incoming is invariant.

### Why it fires

SEF + invariant live-outs + finiteness/mustprogress.

### Counterexample

`x += i` and `return x` with unknown `n` — live-out depends on the trip; not invariant → no delete (unless other passes rewrite the exit value first, e.g. closed form via SCEV/indvars, which is a different story).

---

## Example 6 (bonus): Debug + backedge break

```bash
opt -debug-only=loop-delete -passes='loop-simplify,lcssa,loop-deletion' -S f.ll -o /dev/null
```

Craft a loop whose latch condition is constant-false on the first iteration; watch `NumBackedgesBroken` / debug lines about breaking the backedge even when the full invariant-delete path does not run.

---

## Table

| Case | Deleted? |
|------|----------|
| Empty finite / mustprogress | Yes |
| Never entered (const preds) | Yes |
| Call/volatile in body | No |
| Infinite SEF, no mustprogress | No |
| Infinite SEF + mustprogress | Yes |
| Variant live-out through LCSSA | No |
