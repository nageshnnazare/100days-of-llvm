# Day 42: Inliner

Extra notes: [DeepDive](Day-42-1-Extra-Inliner-DeepDive.md) · [Examples](Day-42-2-Extra-Inliner-Examples.md)

## Why

Function boundaries hide constants, block SROA of arguments, and duplicate prologue/epilogue costs at runtime. The **inliner** copies a callee’s body into a caller at a call site, deleting the call. It is the highest-leverage IPO pass—and the most heuristic-heavy—because inlining the wrong function blows up code size and compile time.

In LLVM’s New Pass Manager, inlining is a **CGSCC** pass: it runs bottom-up on `LazyCallGraph` SCCs (Day 41) so callees are optimized before callers consider inlining them.

## Diagram

```text
   define i32 @caller() {
     %r = call i32 @callee(i32 4)
     ret i32 %r
   }
   define i32 @callee(i32 %x) {
     %y = shl i32 %x, 1
     ret i32 %y
   }

   after inline, @callee's body is
   copied and the argument is substituted:

   define i32 @caller() {
     %y = shl i32 4, 1
     ret i32 %y
   }
   ; InstCombine then folds shl 4, 1 -> 8
```

Inlining replaces a call with a copy of the callee, substituting arguments for parameters. The point is not the copy itself but what later passes can do with the constants that are now local. The cost model refuses when the callee is large, has `noinline`, or is in a different module that this compilation cannot see. ThinLTO's job is to make the profitable cross-module cases visible.

## The idea

For a call `call T @callee(args)` inside `@caller`:

1. Ask an **InlineAdvisor** / cost model whether the site is profitable.
2. If yes, **`InlineFunction`** splices the callee IR into the caller, remaps values, fixes PHIs/returns, and updates metadata/attributes.
3. Update the call graph (`CGSCCUpdateResult`) so the SCC walk stays valid.

Cost is not “instruction count only”: constants that fold after inlining get bonuses; calls, dynamic allocas, and other patterns get penalties. Attributes can force or forbid inlining regardless of cost.

## Pipeline position and `opt -passes=` name

- Pass name: **`inline`** (CGSCC).
- Typical invocation:

```bash
opt -passes='cgscc(inline)' input.ll -S -o out.ll
# or as part of a pipeline:
opt -passes='default<O2>' input.ll -S
```

Core files:

- Orchestration: [`llvm/lib/Transforms/IPO/Inliner.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/Inliner.cpp)
- Cost: [`llvm/lib/Analysis/InlineCost.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineCost.cpp) (`getInlineCost`)
- IR splice: [`llvm/lib/Transforms/Utils/InlineFunction.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/InlineFunction.cpp) (`InlineFunction`)
- Advisor: [`llvm/lib/Analysis/InlineAdvisor.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineAdvisor.cpp) (`InlineAdvisor`, `DefaultInlineAdvisor`; ML / replay advisors live alongside)

Prepare IR:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
```

## Core concepts

### Always / never / hint

| Attribute / flag | Effect |
| --- | --- |
| `alwaysinline` | Inline if legally possible (still blocked by some constraints). |
| `noinline` | Do not inline. |
| `optnone` | Implies nooptimize; treated like noinline for inlining purposes. |
| `inlinehint` | Lower threshold / preference—**not** the same as `alwaysinline`. |
| `cold` | Higher bar / less eager inlining into cold paths. |

Profile entry counts and hot/cold information further adjust thresholds.

### Thresholds

Command-line knobs (names are stable in spirit; check `--help-hidden` on your build):

- `-inline-threshold=N` — baseline budget for ordinary sites.
- Hot/cold variants interact with BFI / profile data and attributes like `inlinehint` / `cold`.

If estimated cost ≤ threshold (after bonuses/penalties), the site is accepted.

### Cost model sketch (`getInlineCost`)

Rough accounting in [`InlineCost.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineCost.cpp):

- Each IR instruction contributes some cost (TTI-informed where relevant).
- **Bonus** when a parameter is a constant that will fold loads/branches in the callee.
- **Penalty** for nested calls, dynamic `alloca`, complex control flow, etc.
- **Invalid / never** when recursion would be inlined unsafely, or attributes forbid it.
- By default **recursive inlining is not done** unless forced (`alwaysinline` / dedicated options).

### Hard constraints

- `musttail` call chains must preserve tail-call requirements after inlining.
- `noduplicate` call sites / functions restrict duplication of the call.
- Indirect calls without a known callee cannot be inlined until promoted/devirtualized.
- Interposable linkage may limit deleting the callee even after all local sites are inlined (Day 49).

### Advisor layer

`InlineAdvisor` abstracts “should we inline?”:

- **`DefaultInlineAdvisor`** — classical threshold + `getInlineCost`.
- **ML advisor** — optional learned policy (when enabled in the build/pipeline).
- **Replay advisor** — replay recorded decisions for debugging / research.

The pass asks the advisor; the advisor consults cost analysis; only then does `InlineFunction` mutate IR.

## Algorithm

Per SCC (simplified):

1. Collect call sites in functions of the SCC (often favoring colder-to-hotter or bottom-up order inside the SCC).
2. For each site, compute `InlineResult` / cost via advisor.
3. If inline:
   - Call `llvm::InlineFunction(...)`.
   - Update LazyCallGraph edges; fill `CGSCCUpdateResult`.
   - Simplify caller (instsimplify-ish cleanups may run as part of the inliner wrapper).
4. Repeat while progress / worklist non-empty within budget.

## Code map

| Role | Link |
| --- | --- |
| CGSCC inliner pass | [`Inliner.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/Inliner.cpp) |
| Cost model | [`InlineCost.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineCost.cpp) |
| Body clone/splice | [`InlineFunction.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/InlineFunction.cpp) |
| Advisors | [`InlineAdvisor.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineAdvisor.cpp) |
| Call graph updates | [`CGSCCPassManager.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/CGSCCPassManager.h) |

## Worked example

```c
static inline int clamp(int x, int lo, int hi) {
  if (x < lo) return lo;
  if (x > hi) return hi;
  return x;
}

int apply(int x) {
  return clamp(x, 0, 255);
}
```

Emit and inline:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes clamp.c -o clamp.ll
opt -passes='cgscc(inline)' clamp.ll -S -o clamp.in.ll
```

Before: `apply` contains `call i32 @clamp(...)`.  
After: `apply` contains the compares; with follow-on instcombine/SCCP the constants `0` and `255` fold the branches further. Cost model gives a **constant-arg bonus** for `lo`/`hi`.

Never-inline contrast:

```c
__attribute__((noinline)) static int clamp(...);
```

Cost is irrelevant; the call remains.

## Reading order

1. High-level loop in [`Inliner.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/Inliner.cpp).
2. `getInlineCost` entry and threshold logic in [`InlineCost.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineCost.cpp).
3. `InlineFunction` in [`InlineFunction.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/InlineFunction.cpp)—focus on how returns become branches to a merge block.
4. Advisor class comments in [`InlineAdvisor.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InlineAdvisor.cpp).

Then [Day-42-1-Extra-Inliner-DeepDive.md](Day-42-1-Extra-Inliner-DeepDive.md).

## Hands-on

```bash
# Cost dump / remarks (asserts build for -debug-only)
opt -passes='cgscc(inline)' -debug-only=inline in.ll -S 2>inline.log
opt -passes='cgscc(inline)' -pass-remarks=inline -pass-remarks-missed=inline in.ll -S
# Clang-style remarks:
clang -O2 -Rpass=inline -Rpass-missed=inline file.c
# Print inline cost analysis helpers when available:
opt -passes='cgscc(inline)' -print-inline-cost in.ll -S
```

Vary `-inline-threshold=10` vs `1000` on a medium callee and watch accept/reject flip.

## Pitfalls

- `clang -O0` IR has `optnone` → inliner skips those functions; use `-O1 -Xclang -disable-llvm-passes`.
- `inlinehint` ≠ `alwaysinline`.
- Recursive functions usually **not** inlined into themselves.
- Successful inline can explode size if thresholds are raised blindly.
- Forgetting opaque pointers: callee signature uses `ptr`; examples must match.

## Neighbors

- **Day 41** — LazyCallGraph / SCC updates.
- **Day 43 IPSCCP** — often runs in module pipeline; constants enable better inline cost bonuses when ordered well.
- **Day 49** — linkage limits deleting inlined callees.
- **Day 48** — attributes from Attributor / function-attrs change inline eligibility (`norecurse`, memory effects, etc.).

## Minimal IR before / after

Before:

```llvm
define internal i32 @add(i32 %a, i32 %b) {
  %t = add i32 %a, %b
  ret i32 %t
}
define i32 @sum(i32 %x) {
  %r = call i32 @add(i32 %x, i32 1)
  ret i32 %r
}
```

After `opt -passes='cgscc(inline)'` (typical):

```llvm
define i32 @sum(i32 %x) {
  %t = add i32 %x, 1
  ret i32 %t
}
; @add may remain if other uses/refs exist, or be deleted later by globaldce
```

Opaque pointers only: arguments and callees use `ptr` when dealing with pointers; integer examples stay `i32`. Do not emit typed pointer IR (`i32*`) in new notes.

## Threshold intuition

There is no single magic number. Treat `-inline-threshold` as a **budget in cost units**, not “max instructions.” A 20-instruction callee with two constant args that fold away half the body may cost less than an 8-instruction callee that calls `malloc` and uses a VLA. Always pair threshold experiments with `-pass-remarks=inline` and `-pass-remarks-missed=inline` so you see the scored cost, not just the IR diff.

## Check yourself

**Q1.** What are the three layers of the inliner stack?  
**A1.** Advisor/cost (`InlineCost` / `InlineAdvisor`), CGSCC orchestration (`Inliner.cpp`), IR mutation (`InlineFunction`).

**Q2.** Why run inlining as a CGSCC pass?  
**A2.** Bottom-up SCC order optimizes callees first; graph updates keep the walk correct when edges disappear.

**Q3.** Does `inlinehint` force inlining?  
**A3.** No—it biases thresholds. Only `alwaysinline` (when legal) forces attempts regardless of ordinary cost.

**Q4.** What debug flags help?  
**A4.** `-debug-only=inline`, `-print-inline-cost` (when present), remarks `-Rpass=inline` / `-Rpass-missed=inline`.

**Q5.** Name two blockers besides high cost.  
**A5.** `noinline`/`optnone`, recursion policy, `musttail`/`noduplicate` constraints, unknown indirect callee.
