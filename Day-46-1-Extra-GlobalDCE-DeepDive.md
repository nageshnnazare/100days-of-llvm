# Day 46 Extra: GlobalDCE Deep Dive

Companion to [Day-46-0-GlobalDCE.md](Day-46-0-GlobalDCE.md). Examples: [Day-46-2-Extra-GlobalDCE-Examples.md](Day-46-2-Extra-GlobalDCE-Examples.md).

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

## 1. Where the pass lives

[`llvm/lib/Transforms/IPO/GlobalDCE.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/GlobalDCE.cpp) implements a module pass used throughout optimization pipelines and as a cleanup after LTO internalization.

## 2. Root set construction

Typical roots:

- `GlobalValue` with externally visible linkage / non-discardable presence.
- Appended arrays `@llvm.used` and `@llvm.compiler.used`.
- Sometimes other special globals depending on version (read the seed function).

Declaration-only externals are not “bodies to keep” in the same sense, but references to them from live code are fine—they just aren’t deleted as definitions.

## 3. Marking transitively

When a function is live, scan its instructions for `Function*`, `GlobalVariable*`, aliases, blockaddresses, etc. When a global variable is live, scan its **initializer** constant. ConstantExprs / `ptr` references to functions (vtables, `@llvm.global_ctors` entries) are critical: they keep otherwise “uncalled” functions alive.

`@llvm.global_ctors` / `@llvm.global_dtors` arrays are classic roots: ctor functions stay live even without direct calls.

## 4. COMDAT behavior (high level)

COMDAT groups tell the linker to pick one copy of a group. LLVM IR models this with `comdat` on globals. GlobalDCE cannot delete members in a way that violates the group selection semantics. If one member is live, the pass keeps whatever the comdat kind requires so the remaining IR still forms a valid group for later codegen/linking.

Kinds you will see in IR (`any`, `exactmatch`, `largest`, `nodeduplicate`, `samesize`) differ in linker behavior; the pass’s job is to remain **compatible**, not to reinvent the linker.

## 5. Vtables and C++ without LTO

Clang emits vtables as globals containing function pointers. Those vtables often have `linkonce_odr` / exported-style linkage. GlobalDCE marks the vtable live as a root or via external visibility, then marks every method pointer in the initializer. **Dead virtual functions that still appear in the vtable stay.**

Whole Program Devirtualization / LTO visibility can prove more methods unused—outside GlobalDCE’s single-TU conservatism.

## 6. Aliases and ifuncs

Global aliases (`@a = alias …`) and ifunc resolvers add more edges in the reference graph. Marking must follow alias targets; deleting a target while leaving an alias is wrong. Read how GlobalDCE handles `GlobalAlias` when studying the source.

## 7. Interaction with internalization (Day 49)

LTO runs **internalize** to turn non-exported symbols into `internal`, *then* GlobalDCE can delete far more. Without internalization, most C++ template instantiations stay `linkonce_odr` visible and survive.

## 8. Debugging

```bash
opt -passes=globaldce -debug-only=globaldce m.ll -S 2>g.log
```

Useful to see why something unexpected survived: almost always a used-list, ctor array, or constant initializer reference.

## 9. Ordering tips

```
inline / IPSCCP / DAE / GlobalOpt
        ↓
    globaldce
```

Running GlobalDCE too early (before inlining deletes the last call) leaves more junk; running it periodically is normal.

## 10. What GlobalDCE does *not* do

- Drop dead *instructions* inside a live function (use DCE/ADCE).
- Change linkage (use internalize).
- Devirtualize (use WPD / type tests).
- Remove `llvm.used` entries themselves if they still list something—you must edit used-lists first if you truly want those symbols GC’d.

## 11. Source landmarks

1. Seed roots.
2. Worklist marking.
3. Erase loop.
4. Comdat cleanup helpers.
5. Tests in `llvm/test/Transforms/GlobalDCE/` when available.

## 12. Mental model

GlobalDCE is **GC for symbols**, keyed off IR references, not runtime reachability profiling. If the IR still mentions a function in any live constant, it lives.
