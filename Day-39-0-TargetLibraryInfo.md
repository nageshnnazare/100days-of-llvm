# Day 39: TargetLibraryInfo (TLI)

TargetLibraryInfo tells the optimizer which C library functions exist for the current triple, what their names/signatures are, and how they map to vector variants. Without TLI, InstCombine cannot turn `printf("hi\n")` into `puts`, and vectorizers cannot find `sin` → `__svml_sin4`. Deep dive: [Day-39-1-Extra-TLI-DeepDive.md](Day-39-1-Extra-TLI-DeepDive.md). Examples: [Day-39-2-Extra-TLI-Examples.md](Day-39-2-Extra-TLI-Examples.md).

## Why this exists

LLVM IR is triple-agnostic until someone says “this is `x86_64-linux-gnu` with glibc.” Freestanding targets, Windows, Darwin, and `-fno-builtin` each expose different subsets of libfuncs. TLI is the single source of truth for “is this call really memcpy?” and “may I emit memset?”

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

## The idea

[TargetLibraryInfo.def](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/TargetLibraryInfo.def) is a giant table expanding into the `LibFunc` enum (memset, memcpy, strlen, printf, pow, … including fortified `__memcpy_chk`, wide-char variants, etc.).

Runtime object [TargetLibraryInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/TargetLibraryInfo.cpp) / header [TargetLibraryInfo.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/TargetLibraryInfo.h):

- `getLibFunc(Function &)` / `getLibFunc(StringRef, LibFunc &)` — recognize a declaration
- `has(LibFunc)` — available on this triple / config
- `disableAllFunctions()` — like freestanding / `-fno-builtin`
- Vector mappings via `VecDesc`, `isFunctionVectorizable`, `addVectorizableFunctions`

**Signature matching is strict:** wrong prototype ⇒ not recognized as that LibFunc (so InstCombine will not treat a random `strlen` with a weird type as the real one).

## Where it sits

| Item | Value |
|------|--------|
| Analysis | `target-lib-info` (`TargetLibraryAnalysis`) |
| Clients | InstCombine / [SimplifyLibCalls.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SimplifyLibCalls.cpp) (`LibCallSimplifier`), LoopIdiom, vectorizers |
| Vector libs | `-fveclib=` → libmvec, SVML, Accelerate, … |

```bash
# Triple matters:
opt -mtriple=x86_64-apple-darwin -passes=instcombine -S f.ll
opt -mtriple=x86_64-unknown-linux-gnu -passes=instcombine -S f.ll
```

## Core concepts

### LibCallSimplifier examples

- `strlen("hello")` → constant 5
- `memcpy` IR call → `llvm.memcpy` intrinsic (or vice versa depending on phase)
- `printf("hello\n")` → `puts("hello")` when TLI says puts exists
- `pow(x, 0.5)` → `sqrt` under fast-math / availability

### Fortified / wchar / alternate names

`.def` lists many cousins. A call named `memcpy` with a mismatched type is **not** `LibFunc_memcpy`. `__memcpy_chk` is a separate LibFunc.

### Vectorization mapping

`VecDesc` records scalar name → vector name for a given VF / VABI. TLI/`isFunctionVectorizable` answers LoopVectorize when replacing scalar libm calls with vector library calls under `-fveclib=`.

### `-fno-builtin` / disableAllFunctions

Disables recognition/optimization of builtins so the compiler will not assume libc semantics (important for some kernels / custom runtimes).

## The algorithm (recognition)

1. Look up callee name in the triple’s available set.
2. Check function type against the expected prototype for that LibFunc.
3. If match → return LibFunc enum; else fail.
4. Clients query `has` before transforming.

## Where to read the code

1. Skim TargetLibraryInfo.def — appreciate the scale.
2. `getLibFunc` in TargetLibraryInfo.cpp.
3. LibCallSimplifier in SimplifyLibCalls.cpp — pick strlen/printf/pow.
4. VecDesc in the header.

## Worked example

```c
#include <string.h>
int f(void) { return strlen("abc"); }
```

After InstCombine with proper TLI: `ret i32 3` (or i64 depending on ABI).

Without TLI / with builtins disabled: call remains.

## Reading order

1. Why triple matters.
2. `.def` + `getLibFunc`.
3. Two LibCallSimplifier folds.
4. VecDesc glance.
5. Extras.

## Hands-on

```bash
cat > tli.c <<'EOF'
int printf(const char *, ...);
void f(void) { printf("hi\n"); }
double pow(double, double);
double g(double x) { return pow(x, 0.5); }
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes tli.c -o tli.ll
opt -mtriple=x86_64-unknown-linux-gnu -passes=instcombine -S tli.ll -o tli.out.ll
```

Inspect puts/sqrt folding (fast-math may be required for pow→sqrt—add flags if needed).

## Pitfalls

- Running `opt` with no `-mtriple` / datalayout and wondering why libcalls differ from clang -O3.
- Custom `void *memcpy(...)` declarations that fail signature checks.
- Assuming vector lib mappings exist without `-fveclib`.

## Neighbors

- LoopIdiom (Day 27), TTI (Day 40), InstCombine, LoopVectorize.

## Check yourself

1. **Q:** Where is the LibFunc table?  
   **A:** `TargetLibraryInfo.def`.

2. **Q:** APIs to recognize / availability-check?  
   **A:** `getLibFunc`, `has`; `disableAllFunctions` clears them.

3. **Q:** Famous client file?  
   **A:** `SimplifyLibCalls.cpp` / `LibCallSimplifier`.

4. **Q:** Why does signature matter?  
   **A:** Wrong prototype ⇒ not a LibFunc; no folding.

5. **Q:** Analysis name in the pass registry?  
   **A:** `target-lib-info`.

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

## Extended hands-on checklist (Day 39)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

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

## Extended hands-on checklist (Day 39)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## Fortified libcalls

`__memcpy_chk` and friends carry an extra object-size argument. They are distinct LibFuncs. Simplifying them requires correct size args; blindly treating them as memcpy can break fortification. When IR comes from hardened builds, expect chk variants and look them up in TargetLibraryInfo.def.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.

## Reading upstream tests

For any pass, open `llvm/test/Transforms/<PassName>/` or `llvm/test/Analysis/<Name>/` and skim the smallest lit test. The `RUN:` lines show the canonical `opt` invocation, including required simplify/lcssa partners. Copying a RUN line into your shell is the fastest way to get a known-good baseline before editing IR by hand.
