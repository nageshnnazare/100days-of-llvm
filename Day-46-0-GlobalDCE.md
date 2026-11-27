# Day 46: GlobalDCE

Extra notes: [DeepDive](Day-46-1-Extra-GlobalDCE-DeepDive.md) · [Examples](Day-46-2-Extra-GlobalDCE-Examples.md)

## Why

IPO and the frontend leave many **`internal` functions and globals** that nothing references. **GlobalDCE** is mark-and-sweep for global values: start from roots that must stay, transitively mark what they reference, delete the rest.

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

## The idea

Roots include:

- Externally visible globals / functions (exported linkage).
- Entries in **`llvm.used`** and **`llvm.compiler.used`**.
- Anything those roots **reference** (operands of constants, initializers, personalities, vtables, etc.).

Then delete unmarked internal (and other dead) GlobalValues.

## Pipeline position and `opt -passes=` name

```bash
opt -passes=globaldce input.ll -S -o out.ll
opt -passes=globaldce -debug-only=globaldce input.ll -S
```

Code: [`llvm/lib/Transforms/IPO/GlobalDCE.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/GlobalDCE.cpp).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
```

## Core concepts

### Mark-and-sweep

1. Seed live set with roots.
2. Walk constant initializers, function bodies of live functions, aliases, ifuncs, etc., marking referenced GlobalValues.
3. Erase unmarked ones (and clean comdats as required).

### `llvm.used` vs `llvm.compiler.used`

Both append lists of pointers to globals that must remain. `llvm.used` is more “linker / user visible preserve”; `llvm.compiler.used` preserves for the compiler (metadata section) even if the linker might not see a symbol reference—exact section semantics are documented in LangRef. For GlobalDCE, **both are roots**.

### COMDAT

Globals in a comdat group have group semantics (`any`, `exactmatch`, `largest`, …). If any member is live, LLVM must keep the group consistent with the comdat kind—typically a live member keeps the required companions so the linker’s comdat rules stay valid. At a high level: **liveness is not always per-symbol independent inside a comdat**.

### Vtables without LTO

A vtable symbol is often **externally visible**. Virtual methods pointed to by the vtable are referenced from that constant initializer → marked live—even if no direct call exists in this TU. Without LTO/WPD, GlobalDCE cannot delete “unused” virtual functions that appear in exported vtables.

## Algorithm

```
live = roots
while worklist not empty:
  v = pop
  for each GlobalValue referenced by v:
    mark live
delete all GlobalValues not live (subject to comdat rules)
```

## Code map

| Item | Link |
| --- | --- |
| Pass | [`GlobalDCE.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/GlobalDCE.cpp) |
| LangRef used lists | [`llvm/docs/LangRef.rst`](https://github.com/llvm/llvm-project/blob/main/llvm/docs/LangRef.rst) (search `llvm.used`) |

## Worked example

```llvm
define internal void @dead() { ret void }
define internal void @live_helper() { ret void }
define void @api() {
  call void @live_helper()
  ret void
}
```

`@api` is a root → `@live_helper` live → `@dead` deleted by `globaldce`.

## Reading order

1. `GlobalDCE.cpp` root seeding.
2. Constant walking / Marking.
3. Comdat handling.
4. [DeepDive](Day-46-1-Extra-GlobalDCE-DeepDive.md).

## Hands-on

1. Internal unused function — deleted.
2. Add to `llvm.compiler.used` — kept.
3. Internal only referenced from dead function — both deleted.
4. Exported vtable pointing at methods — methods stay.
5. `-debug-only=globaldce`.

## Pitfalls

- Thinking CFI/WPD happens here without LTO.
- Forgetting used-lists as roots.
- Expecting external `linkonce_odr` methods to vanish when vtables stay exported.
- Comdat surprises when one member is live.

## Neighbors

- **Day 41** — used-lists create refs.
- **Day 49** — internalization creates more deletable internals.
- **Day 47 GlobalOpt** — may delete stores / fold ctors before DCE.

## Module roots vs function DCE

Function-level DCE/ADCE delete dead *instructions*. GlobalDCE deletes dead *GlobalValues*. A function can be entirely empty of useful code yet still live if exported or listed in `llvm.used`. Conversely, a huge internal function with no callers and no refs is deleted wholesale—its body never needs instruction-level DCE first, though emptying a function via other passes can remove the last reference to a callee and *then* GlobalDCE removes the callee.

Run GlobalDCE after IPO waves that delete call edges (inliner) or constant references.

## Check yourself

**Q1.** Pass name?  
**A1.** `globaldce`.

**Q2.** Name three root kinds.  
**A2.** Externally visible GVs, `llvm.used`, `llvm.compiler.used` (and what they reference).

**Q3.** Why do virtual methods survive without LTO?  
**A3.** Exported vtables reference them in initializers → mark live.

**Q4.** Debug flag?  
**A4.** `-debug-only=globaldce`.

**Q5.** Is this the same as ADCE?  
**A5.** No—ADCE is function-local instruction DCE; GlobalDCE is module-level GV sweep.

## Marking through constants, not just calls

```llvm
define internal void @method(ptr %this) { ret void }
@vt = constant [1 x ptr] [ptr @method]
define ptr @getvt() { ret ptr @vt }
```

`@getvt` is externally visible, so it is a root. Its body references `@vt`. `@vt`’s initializer references `@method`. All three stay. Delete `@getvt` and make `@vt` `internal` and unreferenced, and both `@vt` and `@method` can go.

## `llvm.global_ctors` is a root edge

```llvm
define internal void @ctor() { ret void }
@llvm.global_ctors = appending global [1 x { i32, ptr, ptr }] [
  { i32, ptr, ptr } { i32 65535, ptr @ctor, ptr null }
]
```

Nothing calls `@ctor` directly, but the ctor array references it, so GlobalDCE keeps it. GlobalOpt may later fold the ctor and drop the array entry; a subsequent GlobalDCE can then delete `@ctor` if nothing else references it.

## Comdat, high level

```llvm
$g = comdat any
@a = internal global i32 0, comdat($g)
@b = internal global i32 1, comdat($g)
```

If any member of the group is live, the pass must leave a group the linker can still select according to the comdat kind (`any`, `exactmatch`, and the others listed in LangRef). Do not assume “delete the unused member and keep the live one” is always legal for every kind. Read the comdat handling in [`GlobalDCE.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/GlobalDCE.cpp) rather than inventing per-kind deletion rules.

## Used lists vs ordinary refs

```llvm
define internal void @isr() { ret void }
@llvm.compiler.used = appending global [1 x ptr] [ptr @isr], section "llvm.metadata"
```

`llvm.compiler.used` and `llvm.used` are roots even when no instruction mentions `@isr`. Removing the used entry is what makes `@isr` eligible for deletion, not changing its linkage alone.

## Aliases and ifuncs

```llvm
@target = internal alias void (), ptr @impl
define internal void @impl() { ret void }
define void @api() { call void @target() ret void }
```

Marking follows the alias to `@impl`. Deleting `@impl` while leaving `@target` would be invalid IR. The same idea applies to ifunc resolvers: the resolver function is referenced by the ifunc and stays live while the ifunc is live.

## What a “live external” is not

A `declare` of `printf` is not a definition GlobalDCE is trying to delete. A `define` with `external` linkage is a root even if this file never calls it, because another object file might. That is why LTO’s `internalize` pass (Day 49) is what makes GlobalDCE effective on C++ inline/`linkonce_odr` symbols: it removes the “someone else might reference this” assumption for symbols that are not exported.

## A full root walk on a small module

```llvm
@llvm.used = appending global [1 x ptr] [ptr @keep], section "llvm.metadata"
define internal void @keep() { ret void }
define internal void @dead() { ret void }
define void @main() { ret void }
```

Roots: `@main` (external) and the `@llvm.used` array. Mark `@keep` from the array. `@dead` is never marked. After the sweep only `@dead` is removed. If `@main` called `@dead`, the call edge would mark it during the walk over `@main`’s instructions. GlobalDCE does not delete dead instructions inside `@main`; that is DCE/ADCE. It deletes dead GlobalValues.

## Debug

```bash
opt -passes=globaldce -debug-only=globaldce m.ll -S
```

If a symbol you expected to die is still there, search the IR for `ptr @that_symbol` in initializers, used lists, and ctor arrays before reading the pass for a bug.

GlobalDCE is mark-and-sweep over GlobalValues, not over instructions.
Externally visible definitions are roots even when this file never calls them.
llvm.used and llvm.compiler.used are roots; both appear as appending globals in llvm.metadata.
An initializer that contains ptr @f marks @f live once the initializer’s global is live.
@llvm.global_ctors marks each constructor function it references.
A vtable that is externally visible keeps every virtual function pointer in its initializer.
Without LTO those virtual functions are not dead, even if no direct call exists in the TU.
Comdat groups stay consistent with the comdat kind when any member is live.
Do not delete one comdat member and assume the linker will accept the rest for every kind.
Aliases are followed to their targets; the target stays if the alias stays.
A function that is only called from a dead internal function is dead too.
Run globaldce after the inliner so that removed call edges can make callees unreachable.
-debug-only=globaldce shows the live set decisions.
Source: llvm/lib/Transforms/IPO/GlobalDCE.cpp.
Internalize first if you want linkonce_odr symbols to become eligible for deletion.
