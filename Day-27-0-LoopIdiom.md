# Day 27: Loop Idiom Recognition (`loop-idiom`)

Compilers should not emit a byte-store loop when the target has `memset`. Loop Idiom Recognition finds loops that are really memory fills, copies, bit counts, or shift scans, and replaces them with intrinsics or library calls. Deep implementation notes: [Day-27-1-Extra-LoopIdiom-DeepDive.md](Day-27-1-Extra-LoopIdiom-DeepDive.md). Worked IR: [Day-27-2-Extra-LoopIdiom-Examples.md](Day-27-2-Extra-LoopIdiom-Examples.md).

## Why this exists

Hand-written or frontend-lowered loops of the form:

```c
for (size_t i = 0; i < n; ++i)
  p[i] = 0;
```

are semantically `memset(p, 0, n * sizeof *p)`. Targets implement memset in hand-tuned assembly; leaving a scalar store loop blocks that and also confuses vectorizers with an already-optimal idiom. The same story applies to memcpy-shaped load/store loops, popcount loops, and “shift until zero” / bit-scan loops that map to `ctz`/`clz`-like operations.

## Diagram

```text
   for (i = 0; i < n; ++i)
     a[i] = 0;

   loop-idiom recognizes a store of a
   loop-invariant byte across a unit stride

              |
              v
   memset(a, 0, n * sizeof(a[0]))

   the loop is gone; the library call remains
```

Loop idiom recognition looks for a loop that is really a memory primitive: a zeroing store becomes `llvm.memset`, a copy becomes `llvm.memcpy`, and a popcount-style shift loop can become a bit-count intrinsic. The replacement is profitable because the library call lowers to a tuned sequence. A loop that writes a non-invariant value, or that has a carried dependence, is not a memset.

## The idea

`LoopIdiomRecognize` pattern-matches the loop body against known idioms using ScalarEvolution (pointer AddRecs), alias analysis, and TargetLibraryInfo / TargetTransformInfo:

| Idiom | Typical replacement |
|-------|---------------------|
| Contiguous stores of zero / byte | `llvm.memset` (or libcall if preferred) |
| Load from src[i], store to dst[i] | `llvm.memcpy` / `llvm.memmove` |
| Bit-counting loop | `llvm.ctpop` |
| Shift-until-zero / bit tests | `cttz`/`ctlz`-related forms, shift idioms |
| Count leading/trailing zeros style shifts | Bit-counting intrinsics |

Recognition is **opt-in per target capabilities**: TLI must say the libc function exists; TTI may say an intrinsic is cheap enough.

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| Pass name | `loop-idiom` |
| File | [llvm/lib/Transforms/Scalar/LoopIdiomRecognize.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopIdiomRecognize.cpp) |
| Class | `LoopIdiomRecognize` / `LoopIdiomRecognizePass` |
| Debug | `-debug-only=loop-idiom` |
| Depends on | LoopInfo, SCEV, AA, DT, TLI, TTI, often MSSA in modern pipelines |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes zero.c -o zero.ll
opt -passes='loop-simplify,lcssa,loop-idiom' -S zero.ll -o zero.idiom.ll
```

## Core concepts

### SCEV AddRec on pointers

The store pointer must be an affine induction: something SCEV can write as `{p,+,sizeof(elem)}<%loop>`. Non-affine indexing (`p[i*i]`) is not a memset. The trip count times the stride must form a contiguous region with no holes.

### Memset-style stores

Requirements in spirit:

- Store value is a splat/zero (or byte-wise constant the intrinsic can express).
- Stores are not `volatile`.
- No other memory ops in the loop that alias the region in a way that breaks the transformation.
- The loop’s only meaningful memory effect is that fill (plus allowed IV arithmetic).

### Memcpy-style copies

- One load and one store per iteration with matching sizes.
- Source and destination regions must not overlap **unless** the replacement is `memmove` and overlap is handled correctly; classic memcpy recognition demands NoAlias between src and dst for the whole iteration space.
- Again, SCEV must see affine AddRecs for both pointers.

### Popcount and shift idioms

Separate matchers look for loops that clear the lowest set bit each iteration and count steps (`ctpop`), or that shift until the value becomes zero (`shift-until-zero`). These are narrower and more pattern-fragile than memset.

### TLI / TTI gate

Even a perfect memset loop is left alone if the triple has no memset (`-fno-builtin-memset`, freestanding quirks, or TLI `disableAllFunctions`). Intrinsics like `llvm.memset` may still be emitted when the backend lowers them without a libc dependency—follow the code path for your version.

## The algorithm (memset path sketch)

1. Walk candidate loops (often innermost, simplified).
2. Find a strided store; ask SCEV for the pointer expression and store size.
3. Prove the loop covers a contiguous byte range of computable length.
4. Query AA: nothing else in the loop may read/write that range unsafely; volatility kills the idiom.
5. Ask TLI/TTI whether to emit `llvm.memset` / libcall.
6. Replace the loop with a single intrinsic/call, delete dead IV code, update analyses.

Memcpy recognition is analogous but tracks a load–store pair and overlap.

## Where to read the code

1. [LoopIdiomRecognize.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopIdiomRecognize.cpp) — start at the pass run method, then memset/memcpy recognizers, then popcount / shift-until-zero sections (search for those strings in DEBUG messages).
2. [TargetLibraryInfo](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/TargetLibraryInfo.h) — `has` / `getLibFunc` for memset/memcpy.
3. SCEV (Day 34) — why AddRec is the gateway drug.

Disable knobs in-tree include flags like `disable-loop-idiom-memset` / `disable-loop-idiom-memcpy` (names follow `DEBUG_TYPE`).

## Worked before / after

```c
void clear(char *p, unsigned n) {
  for (unsigned i = 0; i < n; ++i)
    p[i] = 0;
}
```

Before (shape): loop with `store i8 0, ptr %gep`.

After: roughly

```llvm
call void @llvm.memset.p0.i64(ptr align 1 %p, i8 0, i64 %zext.n, i1 false)
```

(exact type of length depends on data layout and extensions).

## Reading order

1. This overview.
2. Memset recognizer only in LoopIdiomRecognize.cpp.
3. Memcpy / memmove path.
4. One popcount or shift idiom for variety.
5. Day 27 DeepDive + Examples.

## Hands-on

```bash
cat > idiom.c <<'EOF'
void zero(int *p, int n) {
  for (int i = 0; i < n; ++i) p[i] = 0;
}
void copy(int *d, int *s, int n) {
  for (int i = 0; i < n; ++i) d[i] = s[i];
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes idiom.c -o idiom.ll
opt -passes='loop-simplify,lcssa,loop-idiom' -S idiom.ll -o idiom.out.ll
opt -debug-only=loop-idiom -passes='loop-simplify,lcssa,loop-idiom' -S idiom.ll -o /dev/null
```

## Pitfalls

- **Volatile stores** never become memset.
- **Overlapping copy** must not become `memcpy`.
- **Non-affine pointers** defeat SCEV AddRec matching.
- **Missing TLI** (`opt` without triple) may change whether libcalls appear.
- **Partial element stores** / struct fields with gaps are not a contiguous memset.

## Neighbors

- MemCpyOpt (Day 33 adjacency) — merges memsets/memcpys after the fact.
- InstCombine / SimplifyLibCalls — further fold libcalls.
- LoopIdiom is *not* the vectorizer; it replaces the loop entirely.

## Check yourself

1. **Q:** Which SCEV shape is required for the pointer?  
   **A:** An affine `scAddRecExpr` (AddRec) with a constant stride covering contiguous memory.

2. **Q:** Why consult TLI?  
   **A:** So you do not emit a libcall the target/runtime does not provide; builtins may be disabled.

3. **Q:** Debug flag?  
   **A:** `-debug-only=loop-idiom`.

4. **Q:** Does may-alias between src and dst allow memcpy idiom?  
   **A:** No for memcpy; overlap needs memmove or rejection.

5. **Q:** Pass name?  
   **A:** `loop-idiom`.

## Contiguous-range proof in more detail

For memset, SCEV must show that the set of addresses touched equals a single interval `[base, base+len)`. That fails when:

- The stride is not equal to the store size (unless a wider idiom store path applies).
- The trip count is unknown and cannot be expressed for the length operand.
- The start pointer is not loop-invariant plus an AddRec of the loop.

LoopIdiom therefore leans on the same trip-count machinery as Day 35. If `print<scalar-evolution>` cannot see an AddRec for the pointer, idiom recognition will not either.

## Memmove versus memcpy

Overlap is the difference. Memcpy contracts require disjoint regions. If AA says MayAlias, LoopIdiom must not emit `llvm.memcpy`. Some code paths may consider `memmove` when overlap is possible but the transformation is still valid—verify the specific recognizer comments in LoopIdiomRecognize.cpp rather than assuming. Teaching default: **MayAlias ⇒ no memcpy idiom**.

## Hash / other idioms

Depending on revision, additional recognizers (hash-related, more bit manipulation) appear behind disable flags like `disable-loop-idiom-hashrecognize`. They are optional reading after memset/memcpy/popcount/shift-until-zero.

## Remarks as a teaching UI

```bash
opt -passes='loop-idiom' -pass-remarks=loop-idiom -pass-remarks-missed=loop-idiom -S in.ll
```

Success remarks name the idiom; missed remarks name the failed check (`SizeStrideUnequal`, `LoopMayAccessStore`, …). That is faster than stepping through C++ the first time.

## Pipeline placement

LoopIdiom typically runs after loops are simplified and often after indvars-ish cleanup in the full pipeline. Running it alone on `-disable-llvm-passes` IR still works for classic zeroing loops. If it fails, try `loop-simplify,lcssa,indvars,loop-idiom` to see if IV canonicalization unlocks SCEV.

### Study note addendum 1


- The stride is not equal to the store size (unless a wider idiom store path applies).
- The trip count is unknown and cannot be expressed for the length operand.
- The start pointer is not loop-invariant plus an AddRec of the loop.

LoopIdiom therefore leans on the same trip-count machinery as Day 35. If `print<scalar-evolution>` cannot see an AddRec for the pointer, idiom recognition will not either.

## Memmove versus memcpy

Overlap is the difference. Memcpy contracts require disjoint regions. If AA says MayAlias, LoopIdiom must not emit `llvm.memcpy`. Some code paths may consider `memmove` when overlap is possible but the transformation is still valid—verify the specific recognizer comments in LoopIdiomRecognize.cpp rather than assuming. Teaching default: **MayAlias ⇒ no memcpy idiom**.

## Hash / other idioms

Depending on revision, additional recognizers (hash-related, more bit manipulation) appear behind disable flags like `disable-loop-idiom-hashrecognize`. They are optional reading after memset/memcpy/popcount/shift-until-zero.

## Remarks as a teaching UI

```bash
opt -passes='loop-idiom' -pass-remarks=loop-idiom -pass-remarks-missed=loop-idiom -S in.ll
```

### Study note addendum 2

- The start pointer is not loop-invariant plus an AddRec of the loop.

LoopIdiom therefore leans on the same trip-count machinery as Day 35. If `print<scalar-evolution>` cannot see an AddRec for the pointer, idiom recognition will not either.

## Memmove versus memcpy

Overlap is the difference. Memcpy contracts require disjoint regions. If AA says MayAlias, LoopIdiom must not emit `llvm.memcpy`. Some code paths may consider `memmove` when overlap is possible but the transformation is still valid—verify the specific recognizer comments in LoopIdiomRecognize.cpp rather than assuming. Teaching default: **MayAlias ⇒ no memcpy idiom**.

## Hash / other idioms

Depending on revision, additional recognizers (hash-related, more bit manipulation) appear behind disable flags like `disable-loop-idiom-hashrecognize`. They are optional reading after memset/memcpy/popcount/shift-until-zero.

## Remarks as a teaching UI

```bash
opt -passes='loop-idiom' -pass-remarks=loop-idiom -pass-remarks-missed=loop-idiom -S in.ll
```

Success remarks name the idiom; missed remarks name the failed check (`SizeStrideUnequal`, `LoopMayAccessStore`, …). That is faster than stepping through C++ the first time.


