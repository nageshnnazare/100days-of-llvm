# Day 33 Extra: Memory Dependence / MemCpyOpt — Examples

Exercise primarily through **MemCpyOpt** and mental MemDep/MSSA dependence reasoning.

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='memcpyopt' -S f.ll -o out.ll
```

Parent: [Day-33-0-MemoryDependence.md](Day-33-0-MemoryDependence.md).

---

## Diagram

```text
   store 1, ptr %p          <-- clobber of a load of %p
   call void @g()           <-- may clobber, unless AA says no
   store 2, ptr %q          <-- not a clobber of %p if NoAlias
   %v = load i32, ptr %p    <-- query starts here and walks up

   MemDep reports:
     Def    a store that produces the loaded value
     Clobber a write that may overlap
     NonFuncLocal  the answer is outside this block
```

MemoryDependenceAnalysis walks backward from a load or store until it finds the nearest memory instruction that must define the value or might overlap it. EarlyCSE and GVN use that answer to forward a store into a load. The walk is local and stops at calls it cannot see through, which is why MemorySSA later replaced it for many passes: one upfront graph instead of a walk per query.

## Example 1: Byte stores → memcpy / wider op

### Input

```c
void copy4(char *restrict d, char *restrict s) {
  d[0] = s[0];
  d[1] = s[1];
  d[2] = s[2];
  d[3] = s[3];
}
```

### Commands

`memcpyopt` as above (sometimes needs instcombine first in real pipelines).

### Expected result

A `llvm.memcpy` of 4 bytes, or a single wider load/store, when dependences show a clean s→d transfer without clobbers.

### Why it fires

Nearest dependence of each store is the corresponding load (or chained), AA says regions disjoint thanks to restrict, lengths form a contiguous copy.

### Counterexample

```c
void copy4_overlap(char *p) {
  p[0] = p[1];
  p[1] = p[2];
  p[2] = p[3];
  p[3] = 0;
}
```

Overlapping / chained dependences — must not become a naive memcpy from `p+1` to `p`.

---

## Example 2: Store–store merging / memset

### Input

```c
void zap(char *p) {
  p[0] = p[1] = p[2] = p[3] = 0;
}
```

### Commands

`memcpyopt` (+ optionally `instcombine`).

### Expected result

Often a single `llvm.memset` of 4 bytes or an `i32` zero store.

### Why it fires

Dependences show consecutive stores of the same value with no intervening reads of those bytes.

### Counterexample

```c
int sink;
void zap_read(char *p) {
  p[0] = 0;
  sink = p[0]; /* read between */
  p[1] = 0; p[2] = 0; p[3] = 0;
}
```

The read depends on the first store; merging all four into one memset before the read would change the read’s value if mis-ordered—optimizer must respect that dependence (may still memset the last three only).

---

## Example 3: Load CSE mindset (nearest def)

### Input IR

```llvm
define i32 @fwd(ptr %p) {
  store i32 7, ptr %p
  %v = load i32, ptr %p
  ret i32 %v
}
```

### Commands

```bash
opt -passes='gvn' -S fwd.ll   # or early-cse / memssa variants
```

### Expected result

Load forwards to `7`; nearest dependence of the load is the store (MustAlias def).

### Why it fires

Local MemDep/MSSA clobber walk finds the store immediately.

### Counterexample

```llvm
declare void @opaque(ptr)
define i32 @nofwd(ptr %p) {
  store i32 7, ptr %p
  call void @opaque(ptr %p)
  %v = load i32, ptr %p
  ret i32 %v
}
```

Call may clobber → dependence is the call (clobber), not the store; must not forward `7`.

---

## Example 4: Non-local dependence (CFG)

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

### Expected result

Non-local query: from `%a` path dependence is store `1`; from `%b` store `2`. Load cannot CSE to a single constant without a phi. MemDep reports non-local deps; MSSA inserts a MemoryPhi at `join`.

### Why it matters

Shows why non-local MemDep walks predecessors and why MSSA MemoryPhis exist.

### Counterexample

Remove the store in `%b` → both paths see store `1` → load can forward `1`.

---

## Example 5 (bonus): Volatile blocks merging

### Input

```c
void vzap(volatile char *p) {
  p[0] = p[1] = p[2] = p[3] = 0;
}
```

### Expected result

Must not collapse to a single non-volatile memset. Volatile dependences forbid the usual MemCpyOpt merging.

### Why it must not fire

Volatility is observable; MemDep/AA treat volatile specially.

### Positive twin

Non-volatile `zap` (Example 2).

---

## Table

| Case | Dependence lesson |
|------|-------------------|
| restrict copy4 | Clean load→store chain → memcpy |
| Overlapping shifts | Cannot naive memcpy |
| Zero stores | Merge if no intervening read |
| Store; load | Local def |
| Store; call; load | Clobber at call |
| Diamond CFG | Non-local / MemoryPhi |
| Volatile | No merge |
