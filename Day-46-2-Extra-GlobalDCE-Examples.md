# Day 46 Extra: GlobalDCE Examples

Companion to [Day-46-0-GlobalDCE.md](Day-46-0-GlobalDCE.md). Deep dive: [Day-46-1-Extra-GlobalDCE-DeepDive.md](Day-46-1-Extra-GlobalDCE-DeepDive.md).

```bash
opt -passes=globaldce in.ll -S -o out.ll
opt -passes=globaldce -debug-only=globaldce in.ll -S
```

---

## Diagram

```text
   @main --> @used
   @unused          (internal, nothing calls it)
   @gv = global i32  (internal, no users)

   GlobalDCE marks from external roots:
     @main is external, so it is live
     @used is live because @main calls it
     @unused is not reachable -> deleted
     @gv has no live user -> deleted
```

GlobalDCE deletes functions and globals that nothing live can reach. The roots are externally visible symbols: `main`, anything with external linkage that another module might call, and globals that are not internal. An internal function with no callers is dead even if it contains stores. A global whose only user was a dead function goes away with it.

## Example 1: Delete unused internal

**Input:**

```llvm
define internal void @dead() { ret void }
define void @main() { ret void }
```

**Commands:** `opt -passes=globaldce e1.ll -S`

**Expected result:** `@dead` removed; `@main` kept.

**Why:** `@main` external root; `@dead` unmarked.

**Counterexample:** `call void @dead()` from `@main` — both live.

---

## Example 2: `llvm.compiler.used` root

**Input:**

```llvm
define internal void @isr() { ret void }
@llvm.compiler.used = appending global [1 x ptr] [ptr @isr], section "llvm.metadata"
define void @main() { ret void }
```

**Commands:** `opt -passes=globaldce e2.ll -S`

**Expected result:** `@isr` remains.

**Why:** used-list is a root referencing `@isr`.

**Counterexample:** Remove the used entry — `@isr` deleted.

---

## Example 3: Reference from dead function

**Input:**

```llvm
define internal void @leaf() { ret void }
define internal void @mid() { call void @leaf() ret void }
define void @main() { ret void }
```

**Commands:** `opt -passes=globaldce e3.ll -S`

**Expected result:** `@mid` and `@leaf` both deleted.

**Why:** Neither reachable from roots; marking never reaches them.

**Counterexample:** Call `@mid` from `@main` — both kept.

---

## Example 4: Vtable keeps method

**Input (simplified):**

```llvm
define void @virt_method(ptr %this) { ret void }
@vtable = constant [1 x ptr] [ptr @virt_method]
define ptr @get_vt() { ret ptr @vtable }
```

**Commands:** `opt -passes=globaldce e4.ll -S`

**Expected result:** `@virt_method` stays because `@vtable` stays (exported) and references it.

**Why:** Initializer edges mark methods live.

**Counterexample:** Make `@vtable` `internal` and unreferenced — vtable and method can both go.

---

## Example 5: global_ctors root

**Input:**

```llvm
define internal void @ctor() { ret void }
@llvm.global_ctors = appending global [1 x { i32, ptr, ptr }] [
  { i32, ptr, ptr } { i32 65535, ptr @ctor, ptr null }
]
```

**Commands:** `opt -passes=globaldce e5.ll -S`

**Expected result:** `@ctor` kept.

**Why:** Ctor list references it.

**Counterexample:** Empty ctors array / remove entry — `@ctor` deletable if otherwise unused.

---

## Example 6: Internalize then DCE (preview of Day 49)

**Input:** many `linkonce_odr` functions only used internally after LTO.

**Commands (illustrative):**

```bash
opt -passes='internalize,globaldce' -internalize-public-api-list=main m.ll -S
```

**Expected result:** Non-listed symbols become internal; unused ones disappear.

**Why:** GlobalDCE becomes effective once linkage stops forcing external roots.

**Counterexample:** Without `internalize`, `linkonce_odr` copies often remain as visible discardable-but-not-deleted-by-GlobalDCE in the same way—or still act as live depending on visibility; internalization is what widens the garbage set.
