# Day 39 Extra: TLI — Deep Dive

Sources: [TargetLibraryInfo.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/TargetLibraryInfo.h), [TargetLibraryInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/TargetLibraryInfo.cpp), [TargetLibraryInfo.def](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/TargetLibraryInfo.def), client [SimplifyLibCalls.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SimplifyLibCalls.cpp). Parent: [Day-39-0-TargetLibraryInfo.md](Day-39-0-TargetLibraryInfo.md).

## Diagram

```text
   source:  memcpy(d, s, n);

   IR:      call void @memcpy(ptr %d, ptr %s, i64 %n)
            ; recognized because TargetLibraryInfo
            ; says this libcall is available and
            ; has the standard signature

   if -fno-builtin-memcpy, TLI marks memcpy
   unavailable, and the call stays an opaque
   call: no memcpy->load/store fold, no DSE
   across it from the libcall summary
```

TargetLibraryInfo knows which C library functions this module is allowed to treat as the standard ones. `memcpy` has a known set of read and write locations; an unknown function does not. Compiling with `-fno-builtin` clears those entries, and every pass that special-cases the name has to stop. The pass does not emit the library call; it answers questions about one that is already in the IR.

## `.def` table mechanics

The `.def` file is included multiple times with different macros to:

1. Declare `enum LibFunc { ... }`
2. Build name tables
3. Attach standard prototypes / properties

When adding a library function upstream, you extend this file—not random string compares in InstCombine.

## Availability per triple

`TargetLibraryInfoImpl` (or equivalent impl class) sets a bitvector of which LibFuncs are valid for a `Triple`. Examples of variation:

- Windows vs Linux naming (`_memcpy` vs `memcpy` in some eras—follow current .cpp)
- Darwin availability of certain math routines
- Embedded triples with partial libC

`disableAllFunctions` clears availability—used for `-fno-builtin` style configurations and some freestanding setups. Individual functions can also be disabled via attributes / command-line in some pipelines.

## `getLibFunc` details

Overloads:

- By `Function &` — name + type check
- By `StringRef` — name lookup then type check when a Function is available
- By opcode/type for some intrinsic-like mappings

Failure modes:

1. Name unknown
2. Name known but unavailable on triple
3. Type mismatch (e.g., `strlen` returning `i32` on LP64 where it must be `i64`)

Teaching tip: demangle confusion is rare; **prototype mismatch** is the common footgun when hand-writing IR tests.

## `LibCallSimplifier`

Class in SimplifyLibCalls.cpp used by InstCombine:

- Switch on LibFunc
- Apply constant folding, strength reduction, neighbor call merges
- Respect TLI `has` and fast-math flags for FP

Study three paths:

1. **strlen** of a constant string global
2. **mem* family** ↔ `llvm.mem*` intrinsic
3. **printf → puts** when format is `"...\n"` with no other format specifiers

Also note fortified chk variants get different cases.

## Vector library descriptors

`VecDesc` stores:

- Scalar function name
- Vector function name
- VF / scaled VF info
- Calling convention / VABI kind fields (see struct in the header)

`addVectorizableFunctions` fills tables from `-fveclib` presets (SVML, libmvec, Accelerate, …). `isFunctionVectorizable(name, VF)` is the LV query.

Without a veclib, scalar `sin` stays scalar even if vector ISAs exist—hardware `sin` is not automatic.

## Interaction with LoopIdiom

LoopIdiom asks TLI whether memset/memcpy exist before emitting libcalls (intrinsics may still be OK). Connecting Day 27 to Day 39 closes that loop.

## `opt` vs `clang` discrepancies

clang -O3 sets:

- DataLayout
- TargetTriple
- Builtin attributes / TLI feature flags
- Sometimes veclib

Bare `opt -passes=instcombine` on IR missing a triple uses defaults that may disable or alter libfuncs. Always pass `-mtriple=` (and `-data-layout=` when relevant) in experiments.

## Reading path

1. Ten entries in the `.def` file (mem*, printf, pow, sin).
2. `getLibFunc` type-checking path.
3. One LibCallSimplifier case fully.
4. `VecDesc` + `isFunctionVectorizable`.
5. Examples.

## Fortified libcalls

`__memcpy_chk` and friends carry an extra object-size argument. They are distinct LibFuncs. Simplifying them requires correct size args; blindly treating them as memcpy can break fortification. When IR comes from hardened builds, expect chk variants and look them up in TargetLibraryInfo.def.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.

## Reading upstream tests

For any pass, open `llvm/test/Transforms/<PassName>/` or `llvm/test/Analysis/<Name>/` and skim the smallest lit test. The `RUN:` lines show the canonical `opt` invocation, including required simplify/lcssa partners. Copying a RUN line into your shell is the fastest way to get a known-good baseline before editing IR by hand.

## Debug and remarks together

Combine `-debug-only=<DEBUG_TYPE>` with `-pass-remarks` / `-pass-remarks-missed` for the same pass. Debug logs show internal decisions; remarks show the user-facing summary. When they disagree with your mental model, trust the code path that emits the remark—jump to that string in the .cpp file.

## How this day connects forward and back

Loop and AA topics form a dependency chain: simplify/LCSSA → rotate/unswitch → SCEV/indvars → idiom/interchange → vectorize. Alias analyses (BasicAA, TBAA) and MemorySSA sit underneath almost every memory transform. TLI/TTI decide availability and profitability at the edges of the mid-end. Re-read neighboring day notes when a failure looks like “wrong pass” but is actually a missing analysis fact.

## Source navigation checklist (Day 39 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Fortified libcalls

`__memcpy_chk` and friends carry an extra object-size argument. They are distinct LibFuncs. Simplifying them requires correct size args; blindly treating them as memcpy can break fortification. When IR comes from hardened builds, expect chk variants and look them up in TargetLibraryInfo.def.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.

## Reading upstream tests

For any pass, open `llvm/test/Transforms/<PassName>/` or `llvm/test/Analysis/<Name>/` and skim the smallest lit test. The `RUN:` lines show the canonical `opt` invocation, including required simplify/lcssa partners. Copying a RUN line into your shell is the fastest way to get a known-good baseline before editing IR by hand.

## Debug and remarks together

Combine `-debug-only=<DEBUG_TYPE>` with `-pass-remarks` / `-pass-remarks-missed` for the same pass. Debug logs show internal decisions; remarks show the user-facing summary. When they disagree with your mental model, trust the code path that emits the remark—jump to that string in the .cpp file.

## How this day connects forward and back

Loop and AA topics form a dependency chain: simplify/LCSSA → rotate/unswitch → SCEV/indvars → idiom/interchange → vectorize. Alias analyses (BasicAA, TBAA) and MemorySSA sit underneath almost every memory transform. TLI/TTI decide availability and profitability at the edges of the mid-end. Re-read neighboring day notes when a failure looks like “wrong pass” but is actually a missing analysis fact.

## Source navigation checklist (Day 39 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.
