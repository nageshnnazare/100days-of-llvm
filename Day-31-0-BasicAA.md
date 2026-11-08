# Day 31: Basic Alias Analysis (`basic-aa`)

BasicAA is LLVM’s workhorse local alias analysis: it reasons about GEPs, allocas, arguments, globals, phis/selects, and capture/noalias metadata—without using type-based TBAA. Deep dive: [Day-31-1-Extra-BasicAA-DeepDive.md](Day-31-1-Extra-BasicAA-DeepDive.md). Examples: [Day-31-2-Extra-BasicAA-Examples.md](Day-31-2-Extra-BasicAA-Examples.md).

## Why this exists

Almost every memory optimization asks: “Can these two pointers refer to overlapping memory?” Wrong Yes destroys correctness; overly conservative MayAlias destroys optimization. BasicAA answers many common cases cheaply and forms the base of the **AA stack** aggregated by `AAResults`.

## Diagram

```text
   %p = alloca i32
   %q = alloca i32
   store 1, ptr %p
   %v = load ptr %q          BasicAA: different allocas
                             never alias -> load is not
                             clobbered by the store

   %g = getelementptr i32, ptr %base, i64 %i
   %h = getelementptr i32, ptr %base, i64 %j
   if %i and %j are not known equal,
   BasicAA says MayAlias
```

BasicAA answers "must", "no", or "may" from the IR shape: distinct allocas do not alias, a GEP into an object does not alias a different object, and two GEPs of the same base alias unless the offsets are provably different. It does not look at types. TBAA is a different analysis that adds type-based facts on top.

## The idea

Given two memory locations (pointer + size), return an `AliasResult`:

| Result | Meaning |
|--------|---------|
| `NoAlias` | Proven disjoint |
| `MayAlias` | Unknown; must assume overlap possible |
| `PartialAlias` | Overlap but not identical starting points / sizes |
| `MustAlias` | Same address (for the query’s purposes) |

For calls, BasicAA also contributes `ModRefInfo` (does this call Mod/Ref this location?).

It **decomposes** GEPs into a base plus variable indices plus constant offset (`DecomposedGEP`), then subtracts decompositions to see if offsets can overlap.

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| Analysis name | `basic-aa` |
| File | [llvm/lib/Analysis/BasicAliasAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/BasicAliasAnalysis.cpp) |
| Class | `BasicAAResult` |
| Header | [BasicAliasAnalysis.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/BasicAliasAnalysis.h) |
| Debug | `-debug-only=basicaa` |
| Phi recursion knob | `-basic-aa-recphi` |

AA is layered: `AAResults` queries BasicAA, TBAA, ScopedNoAlias, TypeBased, etc. Clients should use `AAResults`, not BasicAA alone.

Evaluate quality:

```bash
opt -passes='aa-eval' -aa-pipeline='basic-aa' -print-all-alias-modref-info -S f.ll
# Alias set printer (related):
opt -passes='print-alias-sets' -S f.ll
```

There is not always a `print<aa>` spelling; `aa-eval` and `print-alias-sets` are the practical printers. Confirm names in PassRegistry.def for your version.

## Core concepts

### Object identity

Distinct `alloca`s, distinct globals, and `noalias` arguments are different underlying objects → often NoAlias between them (subject to escapes/captures).

### GEP decomposition

`getelementptr` chains become:

- Base pointer
- List of variable index contributions
- Constant byte offset

If two queries share a base and constant offsets prove disjoint ranges for the access sizes → NoAlias. If bases differ and are known distinct objects → NoAlias.

### AssumptionCache / DominatorTree

`llvm.assume` and dominance help prove inequalities (e.g., nonnegative indices) that tighten GEP math.

### Captures and `nocapture`

If a pointer does not escape, it cannot alias an unrelated argument that arrived from outside—capture tracking feeds BasicAA.

### Metadata BasicAA *does* use

- `noalias` / `alias.scope` + `noalias` scoped metadata
- `invariant.group` (careful rules)
- Argument attributes (`noalias`, `nocapture`, …)

### Metadata BasicAA does **not** use

- **TBAA (`!tbaa`)** — separate TypeBasedAA (Day 32).

### Phi / select recursion

Alias queries through phi/select follow operands with a **depth limit** to avoid exponential blow-up / infinite recursion. `-basic-aa-recphi` toggles recursive phi analysis behavior.

### Conservativeness

Pointer provenance in LLVM is subtle. BasicAA must not overclaim NoAlias when PIECES of objects, inttoptr tricks, or unclear provenance appear. When unsure → MayAlias.

## The algorithm (alias query sketch)

1. Strip trivial casts; identify underlying objects.
2. Quick identity / distinct-object checks.
3. Decompose GEPs; subtract; analyze remaining variable indices.
4. Consult assumes/DT for constraints.
5. Recurse through phi/select within limits.
6. Return Must/Partial/No/May.

ModRef for calls walks arguments and known-readonly/readnone attributes, errno conventions, etc.

## Where to read the code

1. `BasicAAResult::alias` and `aliasGEP` in BasicAliasAnalysis.cpp.
2. `DecomposedGEP` struct and `DecomposeGEPExpression`.
3. AAResults aggregation in [AliasAnalysis.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/AliasAnalysis.h) / AAManager.
4. Day 32 for TBAA contrast.

## Worked before / after (query perspective)

```llvm
%a = alloca i32
%b = alloca i32
; BasicAA: %a NoAlias %b
```

```llvm
define void @f(ptr %p, ptr %q) {
  ; without noalias: MayAlias
}
define void @g(ptr noalias %p, ptr noalias %q) {
  ; NoAlias between %p and %q
}
```

## Reading order

1. AliasResult enum meanings.
2. `aliasGEP` + DecomposedGEP.
3. What is *not* TBAA.
4. DeepDive + Examples.

## Hands-on

```bash
cat > aa.c <<'EOF'
void f(int *a, int *b) { *a = 1; *b = 2; }
void g(int *restrict a, int *restrict b) { *a = 1; *b = 2; }
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes aa.c -o aa.ll
opt -passes='aa-eval' -aa-pipeline='basic-aa' -print-all-alias-modref-info -disable-output aa.ll
```

Compare `f` vs `g`.

## Pitfalls

- **Attributing TBAA wins to BasicAA.**
- **Assuming MustAlias ⇒ bitwise identical values** — it is about locations for AA queries.
- **inttoptr / ptrtoint** — usually MayAlias.
- **Overclaiming** across functions without nocapture/noalias.

## Neighbors

- TBAA (Day 32), MemDep/MSSA (Days 33/37), SCEV-based LAA (Day 36).

## Check yourself

1. **Q:** Four AliasResults?  
   **A:** NoAlias, MayAlias, PartialAlias, MustAlias.

2. **Q:** Does BasicAA read `!tbaa`?  
   **A:** No—TypeBasedAA does.

3. **Q:** What is `DecomposedGEP`?  
   **A:** BasicAA’s breakdown of a GEP into base, variable indices, constant offset.

4. **Q:** Analysis pass name?  
   **A:** `basic-aa`.

5. **Q:** Why MayAlias so often on two `ptr` args?  
   **A:** Without noalias/restrict, they may refer to the same object.

## PartialAlias and DSE

DSE and related passes must treat PartialAlias as “do not assume disjoint.” A store that PartialAlias a later store may still kill some bytes but not others—byte-level reasoning is specialized. BasicAA reporting PartialAlias is already a strong signal compared to MayAlias; clients still need size/offset math.

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

## Extended hands-on checklist (Day 31)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## PartialAlias and DSE

DSE and related passes must treat PartialAlias as “do not assume disjoint.” A store that PartialAlias a later store may still kill some bytes but not others—byte-level reasoning is specialized. BasicAA reporting PartialAlias is already a strong signal compared to MayAlias; clients still need size/offset math.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.
