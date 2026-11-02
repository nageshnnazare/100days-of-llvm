# Day 23 Extra: LICM — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa,loop-mssa(licm)' -S file.ll -o file.out.ll
```

See [Day-23-0-LICM.md](Day-23-0-LICM.md).

---

## Diagram

```text
   preheader                         preheader
     |                                 %inv = add %a, %b   <-- hoisted
     v                                 |
   header                              v
     %i = phi ...                    header
     %inv = add %a, %b                 %i = phi ...
     %use = ... %inv                   %use = ... %inv
     br latch                          br latch

   the add does not depend on %i,
   and nothing in the loop writes
   the memory it would touch
```

LICM moves an instruction that is invariant and safe to the preheader, so it runs once instead of once per iteration. A load is invariant only when alias analysis says the loop does not write that memory. A store sinks to the exit only when the address is loop-invariant and no later iteration reads the old value.

## Example 1: Hoist invariant load

### Input

```c
int sum_scaled(int *a, int n, int *scale) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i] * (*scale);
  return s;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-mssa(licm)' -S sum_scaled.ll -o out.ll
```

### Expected result

`load` from `%scale` appears in the preheader (or guard/preheader region); the loop multiplies by that SSA value.

### Why it fires

`*scale` is not stored in the loop; AA+MSSA show no clobber; the load is invariant and safe enough to hoist (especially after simplify/rotate in richer pipelines).

### Counterexample

```c
int sum_aliased(int *a, int n, int *scale) {
  int s = 0;
  for (int i = 0; i < n; ++i) {
    s += a[i] * (*scale);
    a[i] = s; /* may alias scale if caller passes overlapping pointers */
  }
  return s;
}
```

If `scale` may alias `a[i]`, the load of `*scale` may be clobbered → stay in-loop.

---

## Example 2: Sink computation used only at exit

### Input

```c
int sinkish(int n) {
  int i, last = 0;
  for (i = 0; i < n; ++i)
    last = i * 2 + 7; /* only last iteration’s value matters outside */
  return last;
}
```

### Commands

Same LICM pipeline; compare whether the mul/add stays in-loop or is rewritten with exit-side materialization / promotion-style patterns. (Indvars/other passes may also rewrite; isolate with `loop-mssa(licm)` after simplify+lcssa.)

### Expected result

LICM prefers sinking instructions whose users are all outside the loop. You may see the heavy ops cloned toward the exit, with the loop carrying a simpler recurrence—or the values closed via LCSSA with sunk ops in the exit block.

### Why it fires

`isNotUsedOrFoldableInLoop` style reasoning: outside-only users → sink candidate.

### Counterexample

Use `last` inside the loop (e.g. call `sink(last)` every iteration) → cannot sink.

---

## Example 3: Scalar promotion of a must-alias cell

### Input

```c
void accumulate(int *p, int n) {
  for (int i = 0; i < n; ++i)
    *p += i;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-mssa(licm)' -S accumulate.ll -o acc.out.ll
```

### Expected result

Theme: load `*p` in preheader into a scalar PHI; loop updates the PHI; store once at the exit (LCSSA of the scalar). In-loop load/store to `p` disappear when promotion succeeds.

### Why it fires

All accesses are must-alias to one location; MSSA+AST promotion set is clean; dedicated exit can host the store.

### Counterexample

```c
void accumulate2(int *p, int *q, int n) {
  for (int i = 0; i < n; ++i) {
    *p += i;
    *q += 1; /* may alias p */
  }
}
```

May-alias between `p` and `q` blocks a single must-alias promotion set (or forces conservative behavior).

---

## Example 4: Speculative hoist refused (conditional load)

### Input

```c
int cond_load(int *p, int n, int flag) {
  int s = 0;
  for (int i = 0; i < n; ++i) {
    if (flag)
      s += *p; /* p may be invalid when flag is false */
  }
  return s;
}
```

### Commands

LICM pipeline; inspect whether `load` from `p` moves to preheader.

### Expected result

Typically **not** hoisted to the preheader: that would speculate a potentially faulting load when `flag` is false or when the loop is zero-trip without safety proof.

### Why it must not fire carelessly

`isSafeToExecuteUnconditionally` / guaranteed-to-execute checks exist exactly for this class of bug.

### Counterexample / twin

If `flag` is loop-invariant and the frontend/`llvm.assume` proves `p` is dereferenceable, or the load is proven non-faulting, hoist may become legal—do not assume either without metadata/attrs.

---

## Example 5: LICM ≠ unswitch

### Input

```c
void both(int *a, int n, int mode) {
  for (int i = 0; i < n; ++i) {
    if (mode)
      a[i] += 1;
    else
      a[i] -= 1;
  }
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-mssa(licm)' -S both.ll -o both.licm.ll
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -S both.ll -o both.unswitch.ll
```

### Expected result

LICM alone does not duplicate the loop for `mode`. Unswitch does (subject to cost). LICM might still hoist trivial invariants but leaves the branch.

### Why it matters

Students often expect LICM to “pull the if out.” That is unswitch’s job.

---

## Example 6 (bonus): Debug

```bash
opt -debug-only=licm -passes='loop-simplify,lcssa,loop-mssa(licm)' -S sum_scaled.ll -o /dev/null
```

---

## Table

| Case | Likely LICM action |
|------|--------------------|
| Invariant non-aliasing load | Hoist to preheader |
| Outside-only pure compute | Sink toward exits |
| Must-alias `*p +=` loop | Scalar promote |
| Conditional possibly-faulting load | Stay (no bad speculate) |
| Invariant branch | Not unswitched by LICM |
