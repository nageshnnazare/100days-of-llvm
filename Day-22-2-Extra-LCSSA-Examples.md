# Day 22 Extra: LCSSA — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa' -S file.ll -o file.out.ll
```

Concepts: [Day-22-0-LCSSA.md](Day-22-0-LCSSA.md).

---

## Diagram

```text
   header
     |
    body:  %i = phi ...
     |
     +----> exit:  use %i

   %i is defined inside the loop and used
   outside. That use is not in LCSSA form.

   after lcssa:

   header
     |
    body:  %i = phi ...
     |
     +----> loopexit:  %i.lcssa = phi i32 [ %i, %body ]
                          |
                          v
                        exit: use %i.lcssa
```

LCSSA adds a phi at each loop exit for values that live out. The phi has one incoming value, so it does not change the program. It gives later passes a single place to update when the loop body is rewritten: they edit the live-out phi instead of hunting for uses outside the loop.

## Example 1: Single exit, live-out induction

### Input

```c
int last(int n) {
  int x = 0;
  for (int i = 0; i < n; ++i)
    x = i;
  return x;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes last.c -o last.ll
opt -passes='loop-simplify,lcssa' -S last.ll -o last.lcssa.ll
```

### Expected result

Exit block contains a PHI such as `%x.lcssa = phi i32 [ %x, %for.body ]` (names vary). `ret` uses `%x.lcssa`, not the in-loop `%x`.

### Why it fires

`%x` is defined in the loop and used outside → LCSSA inserts an exit PHI and rewrites the use.

### Counterexample

If `x` is never used after the loop, no LCSSA PHI is required for it (stores/`ret` of something else). Dead values do not get decorative PHIs.

---

## Example 2: Live-out of a non-PHI instruction

### Input

```c
int add_last(int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += i;
  return s + 1; /* use of s outside */
}
```

### Commands

Same `loop-simplify,lcssa` pipeline; inspect the exit.

### Expected result

The add that forms `s` (or the PHI for `s`) is closed with an `.lcssa` PHI before the outside `+ 1`.

### Why it fires

Any outside user counts—binary operators after the loop are classic LCSSA clients.

### Counterexample

Move `return s + 1` into a form where `s` is recomputed outside from invariants only; if the loop def is unused externally, no PHI.

---

## Example 3: Early `break` — multiple exits

### Input

```c
int find(int *a, int n, int k) {
  int i;
  for (i = 0; i < n; ++i) {
    if (a[i] == k)
      break;
  }
  return i;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa' -S find.ll -o find.lcssa.ll
# Look for multiple exit blocks and .lcssa PHIs feeding a merge
```

### Expected result

LoopSimplify tends to create dedicated exits. LCSSA places PHIs on exits dominated by `%i` / related values. A merge block after the loop may PHI together `.lcssa` values from the break exit and the fallthrough exit.

### Why it fires

`i` is live out on more than one exit path; each exiting edge needs a closed SSA value.

### Counterexample / twin

If the function returns a constant on both paths and never returns `i`, LCSSA has nothing to close for `i`.

---

## Example 4: Nested loops — close inner first

### Input

```c
int nest(int n, int m) {
  int last = 0;
  for (int i = 0; i < n; ++i) {
    for (int j = 0; j < m; ++j)
      last = j;
  }
  return last;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa' -S nest.ll -o nest.lcssa.ll
```

### Expected result

Inner loop gets a `.lcssa` for `last`/`j` at the inner exit used by the outer body. Outer loop then closes whatever escapes the outer region toward `ret`. Names show inner then outer closing.

### Why it fires

`formLCSSARecursively` processes subloops before parents so the outer pass does not see raw inner defs escaping incorrectly.

### Counterexample

Hand-running a mental “outer only” close without inner LCSSA leaves illegal external uses of inner defs—exactly what recursive form prevents.

---

## Example 5: Trivial PHI cleaned by InstCombine

### Input

Same as Example 1.

### Commands

```bash
opt -passes='loop-simplify,lcssa' -S last.ll -o last.lcssa.ll
opt -passes=instcombine -S last.lcssa.ll -o last.ic.ll
```

### Expected result

After LCSSA alone, `%x.lcssa` exists. After further instcombine (especially if the loop is later deleted or the PHI is trivial and legal to fold in context), the redundant copy may disappear. For a live loop with a real exit edge, the PHI often remains until a later pass removes the need for the form.

### Why it matters

LCSSA PHIs are an **invariant for loop passes**, not sacred forever. Teaching IR that still shows `.lcssa` after a full `-O2` pipeline may have already cleaned them—inspect right after `lcssa`.

### Counterexample

Expecting `-passes=lcssa,instcombine` to always delete every `.lcssa` while the loop still exists and the PHI is the true live-out edge—often they stay until the loop structure changes.

---

## Example 6 (bonus): Debug with `-debug-only=lcssa`

```bash
opt -debug-only=lcssa -passes='loop-simplify,lcssa' -S last.ll -o /dev/null
```

Look for worklist activity / statistics (`NumLCSSA` via `-stats` if enabled in your build).

---

## Table

| Situation | LCSSA PHI? |
|-----------|------------|
| In-loop def used after exit | Yes |
| Def never used outside | No |
| Multiple exits, live-out | PHI per dominated exit (+ merges) |
| Nested live-out | Inner close, then outer |
| Token / lifetime cross-loop | Skip tokens; may drop lifetime markers |
