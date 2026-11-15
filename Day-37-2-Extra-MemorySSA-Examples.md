# Day 37 Extra: MemorySSA — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='print<memoryssa>' -disable-output f.ll
```

Parent: [Day-37-0-MemorySSA.md](Day-37-0-MemorySSA.md).

---

## Diagram

```text
   MemorySSA

   entry:
     liveOnEntry
   block:
     1 = MemoryDef(liveOnEntry)    store %p
     MemoryUse(1)                  load  %p   -- uses def 1
     2 = MemoryDef(1)              call @g    -- may clobber
     MemoryUse(2)                  load  %p   -- uses def 2, not def 1
```

MemorySSA is a small SSA form for memory. Stores and calls that may write are `MemoryDef`s. Loads are `MemoryUse`s that point at the def that reaches them. A clobber is the nearest def, not a search backward through the block. Passes such as DSE, LICM, and EarlyCSE-with-MemorySSA walk this graph instead of asking MemDep from scratch.

## Example 1: Trivial store–load

### Input

```c
int f(int *p) {
  *p = 7;
  return *p;
}
```

### Commands

`print<memoryssa>`.

### Expected result

Store is MemoryDef(liveOnEntry); load is MemoryUse of that def. Clients can forward 7.

### Why it fires

MustAlias with immediately dominating store; AA proves same location.

### Counterexample

Insert `opaque(p)` between store and load → new MemoryDef from the call; load’s use clobbers at the call.

---

## Example 2: NoAlias lets use skip a store

### Input

```c
int g(int *restrict p, int *restrict q) {
  *q = 1;
  *p = 2;
  return *q;
}
```

### Commands

`print<memoryssa>`; compare without `restrict`.

### Expected result

With restrict/noalias, the load of `*q` can optimize past `*p`’s def to the `*q` store. Without restrict, MayAlias ⇒ load clobbered by `*p` store.

### Why it fires

optimizeUses + BasicAA NoAlias.

### Counterexample

Non-restrict parameters (MayAlias).

---

## Example 3: MemoryPhi at a diamond

### Input IR

```llvm
define i32 @diamond(i1 %c, ptr %p) {
entry:
  store i32 1, ptr %p
  br i1 %c, label %a, label %b
a:
  br label %join
b:
  store i32 2, ptr %p
  br label %join
join:
  %v = load i32, ptr %p
  ret i32 %v
}
```

### Commands

```bash
opt -passes='print<memoryssa>' -disable-output diamond.ll
```

### Expected result

MemoryPhi at `join` merging the entry store’s memory version with `%b`’s store. Load uses the phi. Cannot CSE to a single constant.

### Why it fires

Two reaching MemoryDefs.

### Counterexample

Delete store in `%b` → both edges share one def → load uses that def (value 1).

---

## Example 4: LICM client view

### Input

```c
int sum(int *a, int n, int *p) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i] + *p; /* *p invariant if noalias with a writes */
  return s;
}
```

### Commands

```bash
opt -passes='loop-mssa(licm)' -S sum.ll
# Inspect MemorySSA before with print<memoryssa>
```

### Expected result

If MSSA/AA prove loop stores to `a[i]` NoAlias `*p`, LICM hoists `*p`. If MayAlias, load stays in-loop.

### Why it fires

Walker finds clobber outside the loop (or liveOnEntry) for `*p`.

### Counterexample

`p` aliases `a` (same pointer) → store to `a[i]` may clobber `*p`.

---

## Example 5 (bonus): DSE with MemorySSA

### Input

```c
void dse(int *p) {
  *p = 1;
  *p = 2;
}
```

### Commands

```bash
opt -passes='dse' -S dse.ll
```

### Expected result

First store deleted; second kept. MSSA shows the first def has no intervening MemoryUse before being killed by the second def to the same location.

### Why it fires

MustAlias kill with no read between.

### Counterexample

```c
void dse2(int *p, int *q) {
  *p = 1;
  sink(*q);
  *p = 2;
}
```

If `q` MayAlias `p`, the call may read `*p` → first store must stay.

---

## Table

| Case | MSSA picture |
|------|----------------|
| store; load same | Use → that Def |
| store q; store p; load q (noalias) | Use skips p |
| diamond two stores | MemoryPhi |
| invariant load in loop | Clobber outside loop |
| store; store same | First dead for DSE |
