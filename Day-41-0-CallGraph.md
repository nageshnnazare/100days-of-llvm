# Day 41: Call Graph (LazyCallGraph & CGSCC)

Extra notes: [DeepDive](Day-41-1-Extra-CallGraph-DeepDive.md) · [Examples](Day-41-2-Extra-CallGraph-Examples.md)

## Why

Almost every interesting interprocedural optimization walks the **call graph**: which functions call which others, and which sets of functions can call each other recursively. The inliner, IPSCCP, dead-argument elimination, argument promotion, and Attributor all need a consistent view of that graph—and they need to **mutate** it when they inline or delete calls.

LLVM historically shipped a classic `CallGraph` analysis. The New Pass Manager does **not** drive IPO with that class. It uses `LazyCallGraph` plus the CGSCC (Call-Graph SCC) pass manager, which support lazy construction, reference edges, and in-place SCC updates during inlining.

## Diagram

```text
        @main
        /    \
      @a      @b
        \    /
         @c ----+
         ^      |  (recursive)
         +------+

   call graph SCCs, bottom-up:
     1. {@c}          c is recursive, one SCC
     2. {@a}  {@b}    neither calls the other
     3. {@main}
```

The call graph has an edge from a function to each function it calls. A strongly connected component is a set of functions that can all reach each other, which is exactly a recursive group. The inliner runs bottom-up on these SCCs so callees are simplified before callers. An indirect call is an edge to every possible callee the analysis can see, or to an unknown external node.

## The idea

A call graph is a directed graph whose nodes are functions (plus a couple of special nodes). An edge `A → B` means “A may call B.”

Two refinements matter in LLVM today:

1. **Call edges** vs **reference edges**. A direct `call @f` is a call edge. Taking `@f`’s address (`store ptr @f, …`) is a reference edge: `f` might be called later through that pointer, but the compiler cannot see a static callee.
2. **SCCs** vs **RefSCCs**. An SCC is a maximal set of nodes mutually reachable via **call** edges. A RefSCC is the same idea over **call ∪ reference** edges. Inside one RefSCC you may have several call-SCCs.

Bottom-up visitation of call-SCCs is how the inliner gets a chance to inline callees before their callers: when you finish an SCC, every callee outside that SCC has already been (re)optimized.

## Pipeline position and `opt -passes=` name

- Analysis: `LazyCallGraph` (module analysis).
- Driver: CGSCC pass manager (`CGSCCPassManager`), declared with the graph machinery in [`llvm/include/llvm/Analysis/CGSCCPassManager.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/CGSCCPassManager.h) and used from PassBuilder pipelines.
- Legacy (still in-tree, not the NPM driver): `CallGraph` / `CallGraphNode` / `CallGraphSCC` in [`llvm/lib/Analysis/CallGraph.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/CallGraph.cpp).

Useful print passes (New PM):

```bash
opt -passes='print<lazy-callgraph>' input.ll -disable-output
# DOT variant also exists as LazyCallGraphDOTPrinterPass
```

There is no single “callgraph” transform to run for learning; you run CGSCC passes such as `inline` *on* the lazy call graph.

Prepare IR without `optnone`:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
```

## Core concepts

### Legacy `CallGraph`

[`CallGraph`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/CallGraph.cpp) builds nodes for every function, plus an **external** node representing the outside world. Indirect calls and calls to external functions become conservative edges to that external node. `CallGraphSCC` iterates Tarjan SCCs for the old PM’s `CallGraphSCCPass`.

### `LazyCallGraph`

[`LazyCallGraph`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyCallGraph.cpp) builds the graph on demand. Nodes track:

- **Call edges** — from `call` / `invoke` with a known callee.
- **Ref edges** — from any other use of a function as a value (address-taken).

`llvm.compiler.used` / `llvm.used` keep symbols alive for the linker; they also force **reference** edges (the function’s address is “used”), which affects RefSCC structure even when nothing calls the function directly.

### RefSCC vs SCC

- **RefSCC**: strongly connected component using reference *and* call edges. Address-taken mutual reachability lives here.
- **SCC** (call-SCC): SCC using only call edges, **partitioned inside** a RefSCC.

Why both? Inlining and function attrs care about *actual calls*. Reference edges stop you from assuming a function is never reachable, but you still want to process call-SCCs bottom-up for inlining.

### Indirect calls

A call of a function pointer with no known set of targets is modeled conservatively (legacy: edge to the external node; lazy graph: treat as potentially calling anything the analysis cannot prove otherwise). That is why IPO is weak on heavy function-pointer code without LTO + whole-program visibility or specialized ICP (indirect call promotion).

### `CGSCCUpdateResult`

When the inliner deletes a call edge or creates a new one, SCCs can **split or merge**. The CGSCC manager cannot assume the graph is frozen. Passes that mutate the graph fill a [`CGSCCUpdateResult`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/CGSCCPassManager.h) so the manager can re-queue SCCs / RefSCCs and continue the bottom-up walk correctly. Getting this wrong causes missed inlines or use-after-free of SCC objects.

## Algorithm (LazyCallGraph + CGSCC walk)

1. **Discover nodes** for functions in the module; scan instructions for call and ref uses.
2. **Build RefSCCs** over the combined edge set (Tarjan-style / incremental).
3. **Within each RefSCC**, refine **call-SCCs**.
4. **Visit call-SCCs in post-order** (callees before callers for acyclic edges; within a recursive SCC, visit the SCC as a unit).
5. Run the CGSCC pass pipeline on that SCC (e.g. inliner, function-attrs).
6. If the pass reports updates via `CGSCCUpdateResult`, **invalidate / recreate** affected SCCs and possibly re-run.

## Code map

| Piece | Path |
| --- | --- |
| Lazy call graph | [`llvm/lib/Analysis/LazyCallGraph.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyCallGraph.cpp), [`llvm/include/llvm/Analysis/LazyCallGraph.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/LazyCallGraph.h) |
| CGSCC PM + `CGSCCUpdateResult` | [`llvm/include/llvm/Analysis/CGSCCPassManager.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/CGSCCPassManager.h) |
| Legacy call graph | [`llvm/lib/Analysis/CallGraph.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/CallGraph.cpp), [`llvm/include/llvm/Analysis/CallGraph.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/CallGraph.h) |
| Pipeline wiring | [`llvm/lib/Passes/PassBuilderPipelines.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Passes/PassBuilderPipelines.cpp) |

## Worked example

```c
static int leaf(int x) { return x + 1; }

static int mid(int x) { return leaf(x) * 2; }

int root(int x) { return mid(x) + leaf(x); }

void take_addr(void) {
  int (*fp)(int) = &leaf; /* reference edge to leaf */
  (void)fp;
}
```

Emit IR:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes callgraph.c -o callgraph.ll
```

Sketch of the lazy graph:

- Call edges: `root → mid`, `root → leaf`, `mid → leaf`.
- Ref edge: `take_addr → leaf` (address taken).
- Call-SCCs are trivial singletons `{leaf}`, `{mid}`, `{root}`, `{take_addr}` (no recursion).
- Post-order for inlining visits `leaf` before `mid` before `root`, so `leaf` can be inlined into `mid`, then the enlarged `mid` into `root`.

Indirect call:

```llvm
define i32 @via_ptr(ptr %p, i32 %x) {
  %r = call i32 %p(i32 %x)
  ret i32 %r
}
```

There is no static callee; IPO must be conservative about what `%p` may reach.

## Reading order

1. Header comments in [`LazyCallGraph.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/LazyCallGraph.h) — RefSCC vs SCC rationale.
2. `CGSCCUpdateResult` documentation in [`CGSCCPassManager.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/CGSCCPassManager.h).
3. Skim how `InlinerPass` accepts `LazyCallGraph::SCC &` and `CGSCCUpdateResult &` in [`Inliner.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/Inliner.cpp).
4. Optional: legacy [`CallGraph.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/CallGraph.cpp) for the older mental model.

Then read [Day-41-1-Extra-CallGraph-DeepDive.md](Day-41-1-Extra-CallGraph-DeepDive.md).

## Hands-on

1. Build the C example above; print the lazy call graph.
2. Add mutual recursion `a→b→a` and observe a non-trivial call-SCC.
3. Take the address of `a` from another function; observe RefSCC growth even if no new call edge appears.
4. Run `opt -passes='cgscc(inline)'` and watch how edges disappear as calls are inlined (use `-debug-only=inline` on an asserts build).

## Pitfalls

- Confusing legacy `CallGraph` with `LazyCallGraph` — tutorials and old papers talk about the former; NPM uses the latter.
- Assuming address-taken functions are “called” — they are only **referenced** until a call through that pointer is proven.
- Ignoring `llvm.compiler.used` — keeps symbols and creates refs that change RefSCCs.
- Expecting precise targets for every indirect call without type tests / ICP / LTO.
- Mutating calls inside a CGSCC pass **without** updating `CGSCCUpdateResult` — graph and manager disagree.

## Neighbors

- **Day 42 Inliner** — primary consumer of bottom-up CGSCC order and SCC updates.
- **Day 43 IPSCCP** — propagates constants over the call graph.
- **Day 48 Attributor** — CGSCC and module flavors walk related structure.
- **Day 49 Linkage** — only `internal`/`private` (after internalize) unlock signature-changing IPO.

## IR patterns cheat sheet

Direct call (call edge):

```llvm
define i32 @caller(i32 %x) {
  %y = call i32 @callee(i32 %x)
  ret i32 %y
}
```

Address taken (ref edge), no call edge from this function to `@callee`:

```llvm
define ptr @escape() {
  ret ptr @callee
}
```

Indirect call (conservative; no static callee edge):

```llvm
define i32 @apply(ptr %fp, i32 %x) {
  %y = call i32 %fp(i32 %x)
  ret i32 %y
}
```

Compiler-used reference root:

```llvm
@llvm.compiler.used = appending global [1 x ptr] [ptr @callee], section "llvm.metadata"
```

When you print the lazy call graph, match each of these patterns to the edge kinds you see. If a function is unexpectedly in a large RefSCC, search for `ptr @that_function` uses and for used-lists before assuming a mystery call edge.

## How this shows up in PassBuilder

In [`PassBuilderPipelines.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Passes/PassBuilderPipelines.cpp), the CGSCC pipeline is assembled roughly as: optional OpenMP/Attributor CGSCC pieces, inliner wrapper, function-attrs, then function simplification nested under the SCC. You never “run LazyCallGraph” as a transform; you run passes that *require* it. Invalidating the module’s call graph analysis forces rebuild on the next CGSCC run—another reason IPO passes must declare preservations carefully.

Legacy PM code paths may still construct `CallGraph` for specific tools or outdated examples; if you grep the monorepo, treat NPM LazyCallGraph hits under `Transforms/IPO` and `Passes/` as authoritative for modern pipelines.

## Check yourself

**Q1.** Why does the New Pass Manager use `LazyCallGraph` instead of legacy `CallGraph` as the IPO driver?  
**A1.** Lazy construction, explicit ref vs call edges, RefSCC/SCC layering, and first-class support for SCC mutation during passes like the inliner via `CGSCCUpdateResult`.

**Q2.** What is the difference between a RefSCC and a call-SCC?  
**A2.** RefSCC uses call∪reference edges; call-SCC uses only call edges and lives inside a RefSCC.

**Q3.** How are indirect calls modeled conservatively?  
**A3.** Without a known callee set, the graph cannot add precise edges; analysis treats the call as potentially reaching external / unknown targets (legacy: external node).

**Q4.** Why is SCC mutation during inlining delicate?  
**A4.** Inlining removes call edges and can split recursive SCCs or move functions between SCCs. The CGSCC manager’s walk must be told via `CGSCCUpdateResult` which SCCs are invalid or need reprocessing.

**Q5.** What do `llvm.used` / `llvm.compiler.used` do to the graph?  
**A5.** They mark functions/globals as referenced (often creating ref edges / liveness), so RefSCCs and DCE roots change even without direct calls.

## One more distinction: calls vs refs in the printer

When `print<lazy-callgraph>` lists a function inside a RefSCC but its call-SCC is a singleton, you are looking at address-taken reachability without a direct call cycle. A multi-function call-SCC means mutual calls (`a` calls `b` and `b` calls `a`). Keep those two observations separate when you explain an inliner decision.
