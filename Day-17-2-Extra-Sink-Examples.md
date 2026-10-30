# Day 17 Extra: Sink — Examples

Overview: [Day-17-0-Sink.md](Day-17-0-Sink.md). Deep dive: [Day-17-1-Extra-Sink-DeepDive.md](Day-17-1-Extra-Sink-DeepDive.md).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=sink -S file.ll -o file.out.ll
opt -passes=sink -debug-only=sink -S file.ll -o /dev/null
```

Hand-written IR below is often clearer than Clang output for isolating Sink.

---

## Diagram

```text
        entry
        %s = add %a, %b          used only in %cold
          |                      |
          +----------+           |  Sink moves %s
                     v           v
                   hot         cold:
                   (no use)    %s = add %a, %b
                               use %s

   %a and %b must be available in %cold
   (they dominate it, or they are arguments)
```

Sink walks uses backward. If every use of an instruction is in a successor, and the operands are available there, the instruction moves to the use. That shortens its live range. It will not sink into a block where the operands would have to be recomputed, and it will not sink a load across a possible clobber.

## Example 1: Arithmetic used only on the true arm

### Goal

Show Sink moving an `add` into the successor that dominates its only real use.

### Input

```llvm
define i32 @add_true(i1 %c, i32 %x, i32 %y) {
entry:
  %t = add i32 %x, %y
  br i1 %c, label %T, label %F
T:
  ret i32 %t
F:
  ret i32 0
}
```

### Commands

```bash
opt -passes=sink -S add_true.ll -o add_true.out.ll
```

### Expected

`%t = add` moves into `T`. `F` no longer sits below a live add that it never uses.

### Why it fires

All uses of `%t` are in `T` (the `ret`). `T` dominates that use. The add is side-effect free. Loop depth unchanged.

### Counterexample

Use `%t` on both arms (`ret i32 %t` in `F` as well). No single successor dominates all uses → Sink leaves the add in `entry`.

---

## Example 2: PHI incoming restricts the sink target

### Goal

Understand PHI uses: the value is “used” along one predecessor edge.

### Input

```llvm
define i32 @phi_use(i1 %c, i32 %x, i32 %y) {
entry:
  %t = add i32 %x, %y
  br i1 %c, label %T, label %F
T:
  br label %J
F:
  br label %J
J:
  %p = phi i32 [ %t, %T ], [ 0, %F ]
  ret i32 %p
}
```

### Commands

```bash
opt -passes=sink -S phi_use.ll -o phi_use.out.ll
```

### Expected

`%t` sinks into `T` (the predecessor that feeds `%t` into the PHI), not into `J`. Sinking into `J` would place the def after the PHI use.

### Why it fires

The meaningful use context is the `T → J` edge. `T` dominates that context; `F` does not use `%t`.

### Counterexample

```llvm
%p = phi i32 [ %t, %T ], [ %t, %F ]
```

Both edges need `%t` → stays in `entry` (or cannot sink into only one successor).

---

## Example 3: Load blocked by a may-alias store

### Goal

Show AA + store tracking preventing an unsafe sink.

### Input

```llvm
declare void @use(i32)

define void @load_store(ptr %p, ptr %q, i1 %c) {
entry:
  %v = load i32, ptr %p, align 4
  store i32 1, ptr %q, align 4
  br i1 %c, label %T, label %F
T:
  call void @use(i32 %v)
  ret void
F:
  ret void
}
```

### Commands

```bash
opt -passes=sink -S load_store.ll -o load_store.out.ll
# Optional: tighten AA
opt -passes=sink -aa-pipeline='basic-aa' -S load_store.ll
```

### Expected

With `%p` and `%q` as distinct pointer arguments, BasicAA often returns MayAlias → the load does **not** sink past the store into `T`. The load remains above the store in `entry`.

### Why it (does not) fire

`isSafeToMove` refuses to move a load across a store that may alias. Sinking the load into `T` would reorder it after the store in dynamic execution of the true path.

### Counterexample (legal sink)

Use proven NoAlias (different `noalias` params, or disjoint constant GEPs the AA understands):

```llvm
define void @load_store_noalias(ptr noalias %p, ptr noalias %q, i1 %c) {
entry:
  %v = load i32, ptr %p, align 4
  store i32 1, ptr %q, align 4
  br i1 %c, label %T, label %F
T:
  call void @use(i32 %v)
  ret void
F:
  ret void
}
```

With NoAlias, Sink may move the load into `T` (still after verifying other rules). If your build still refuses, check volatility and call attributes on `@use`.

---

## Example 4: Refuse sinking into a deeper loop

### Goal

Demonstrate LoopInfo guarding against anti-LICM motion.

### Input

```llvm
define void @outer_inner(i32 %n, i32 %m, ptr %p) {
entry:
  %v = add i32 %n, 1
  br label %outer
outer:
  %i = phi i32 [ 0, %entry ], [ %i.next, %outer.latch ]
  %oc = icmp slt i32 %i, %n
  br i1 %oc, label %inner.ph, label %exit
inner.ph:
  br label %inner
inner:
  %j = phi i32 [ 0, %inner.ph ], [ %j.next, %inner ]
  %ic = icmp slt i32 %j, %m
  br i1 %ic, label %inner.body, label %outer.latch
inner.body:
  ; only use of %v is here
  store i32 %v, ptr %p
  %j.next = add i32 %j, 1
  br label %inner
outer.latch:
  %i.next = add i32 %i, 1
  br label %outer
exit:
  ret void
}
```

### Commands

```bash
opt -passes='loop-simplify,sink' -S outer_inner.ll -o outer_inner.out.ll
opt -passes='print<loops>' -disable-output outer_inner.ll
```

### Expected

`%v = add` does **not** sink into `inner.body` / the inner loop. Doing so would execute the add once per inner iteration instead of once (or once per outer structure) outside.

### Why it refuses

`IsAcceptableTarget` sees a deeper loop nest at the use block than at `entry` / outer level where `%v` lives.

### Counterexample (same-depth sink)

Place the only use in a sibling non-loop successor at the same depth as the def’s block. Same-depth (or shallower) targets remain eligible.

---

## Example 5: Volatile load never sinks

### Goal

Hard stop on volatility.

### Input

```llvm
define i32 @vol(ptr %p, i1 %c) {
entry:
  %v = load volatile i32, ptr %p
  br i1 %c, label %T, label %F
T:
  ret i32 %v
F:
  ret i32 0
}
```

### Commands

```bash
opt -passes=sink -S vol.ll -o vol.out.ll
```

### Expected

No change: volatile load stays in `entry`.

### Why it refuses

Sinking would change the number of observable volatile accesses on the false path (zero vs one).

### Counterexample

Drop `volatile`. Non-volatile load with a single use in `T` and no conflicting stores can sink.

---

## Quick checklist after each example

1. Diff IR; confirm only intended moves.
2. `opt -passes=verify` on the output.
3. With `-debug-only=sink`, confirm iteration count matches your mental model for dependent chains.
