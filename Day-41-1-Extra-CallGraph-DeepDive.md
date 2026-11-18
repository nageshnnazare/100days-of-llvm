# Day 41 Extra: Call Graph Deep Dive

Companion to [Day-41-0-CallGraph.md](Day-41-0-CallGraph.md). Examples: [Day-41-2-Extra-CallGraph-Examples.md](Day-41-2-Extra-CallGraph-Examples.md).

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

## 1. Legacy CallGraph anatomy

[`llvm/lib/Analysis/CallGraph.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/CallGraph.cpp) defines:

- **`CallGraph`** — owns a map from `Function*` to `CallGraphNode*`, plus special nodes.
- **`CallGraphNode`** — one function (or a synthetic node) and a list of called nodes (`CallRecord` pairs of call-site and callee node).
- **`CallGraphSCC`** — one strongly connected component used by the legacy `CallGraphSCCPass` adaptor.

Typical special nodes:

- **External** — stands for calls that leave the module or enter from outside.
- Sometimes a **calls-external** style sink for indirect / unknown callees (exact naming is in the source; the idea is one conservative sink).

The legacy graph is eager: construct once, optionally update when IPO mutates IR. The old PM scheduled `CallGraphSCCPass` instances over Tarjan SCCs. That design still exists for legacy clients, but **PassBuilder’s default pipelines do not use it as the CGSCC driver**.

## 2. LazyCallGraph: deferred work

[`LazyCallGraph`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyCallGraph.cpp) exists because full eager SCC recomputation after every inline is expensive, and because reference edges need a first-class representation.

### Nodes and edges

Each function becomes a `LazyCallGraph::Node`. When the analysis scans a function body it records:

| Edge kind | Typical IR | Meaning |
| --- | --- | --- |
| Call | `call void @f()`, `invoke ... @f` | May transfer control to `f` now |
| Ref | `store ptr @f, ptr %slot`, operand of `ptrtoint`, etc. | Address / value use; may call later |

A function that is **only** referenced but never directly called still participates in RefSCCs. That affects which functions can be assumed dead, and which sets must be treated as potentially mutually reachable through unknown indirect calls.

### Building SCCs

High-level flow (see the large comment block at the top of `LazyCallGraph.h`):

1. Populate nodes and edges (lazily as functions are visited).
2. Compute **RefSCCs** on the undirected/strongly-connected sense of the combined graph.
3. Restrict to call edges inside a RefSCC to form **call-SCCs**.
4. Expose iterators so the CGSCC pass manager can walk RefSCCs and the SCCs within them in a post-order compatible with bottom-up IPO.

Incremental APIs exist to **switch** an edge from ref to call or remove edges when IR changes—needed when the inliner turns an indirect pattern into a direct call or deletes a call entirely.

## 3. CGSCCPassManager walk

[`CGSCCPassManager`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/CGSCCPassManager.h) is to SCCs what `FunctionPassManager` is to functions.

Conceptual schedule:

```
for each RefSCC in post-order:
  for each call-SCC in that RefSCC (bottom-up):
    run CGSCC pipeline on SCC
    apply CGSCCUpdateResult
    maybe revisit newly created SCCs
```

Bottom-up matters: when processing caller `C`, callees outside `C`’s SCC should already have been simplified, attributed, and possibly inlined into *their* callers. Inside a recursive SCC `{f,g}`, you cannot fully “finish” `f` before `g` in a strict sense; the inliner uses heuristics and may iterate with care.

## 4. Why inlining makes SCCs delicate

Suppose a recursive SCC:

```
f → g → f
```

If the inliner inlines `g` into `f` and the only call `g→f` disappears (or becomes dead), the SCC may **split** into smaller SCCs or singletons. Conversely, promoting an indirect call to a direct call can **merge** components by adding a call edge.

The manager keeps:

- Worklists of SCCs / RefSCCs to revisit (`CWorklist`, related RefSCC worklists — see `CGSCCUpdateResult` fields in the header).
- `InvalidatedSCCs` so stale SCC objects are not reused.
- `UpdatedC` when the “current” SCC object is replaced mid-pass.

The inliner is the stress test for this machinery: every successful inline is a micro graph edit. That is why `InlinerPass::run` takes `LazyCallGraph &`, `LazyCallGraph::SCC &`, and `CGSCCUpdateResult &` together.

## 5. Indirect calls, `llvm.compiler.used`, and function pointers

### Function pointers

```llvm
@fp = internal global ptr @target

define internal i32 @target(i32 %x) {
  ret i32 %x
}

define i32 @call_fp(i32 %x) {
  %p = load ptr, ptr @fp
  %r = call i32 %p(i32 %x)
  ret i32 %r
}
```

- `@call_fp` has a call of a loaded pointer: **no static call edge** to `@target`.
- Storing `@target` into `@fp` creates a **ref** involving `@target`.
- Without ICP or constant propagation of `@fp`, the inliner will not inline `@target` into `@call_fp`.

### `llvm.compiler.used`

```llvm
@llvm.compiler.used = appending global [1 x ptr] [ptr @keep_me], section "llvm.metadata"

define internal void @keep_me() {
  ret void
}
```

Even if nothing calls `@keep_me`, the compiler-used list references it. GlobalDCE must keep it; the call graph sees a reference. This pattern appears for interrupt handlers, ctor registration, and other “called by the environment” symbols.

## 6. Printers

- `LazyCallGraphPrinterPass` — textual dump of RefSCCs / SCCs / nodes ([`LazyCallGraph.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyCallGraph.cpp)).
- `LazyCallGraphDOTPrinterPass` — Graphviz-oriented output.

On a typical opt binary:

```bash
opt -passes='print<lazy-callgraph>' m.ll -disable-output 2>&1 | less
```

(Exact pipeline spelling is registered next to the pass in PassBuilder / PassRegistry; if your build rejects the name, search the tree for `LazyCallGraphPrinterPass`.)

## 7. Relating legacy SCC iteration to NPM

| Concern | Legacy | NPM |
| --- | --- | --- |
| Graph class | `CallGraph` | `LazyCallGraph` |
| SCC object | `CallGraphSCC` | `LazyCallGraph::SCC` + `RefSCC` |
| Pass base | `CallGraphSCCPass` | `PassInfoMixin` CGSCC passes |
| Updates | manual CG updates | `CGSCCUpdateResult` |
| Refs | weak / external-node approximations | first-class ref edges |

When reading older blog posts, mentally translate “CallGraphSCCPass” → “CGSCC pass on LazyCallGraph.”

## 8. Interaction with analyses

CGSCC passes may still request function analyses for each function in the SCC. The analysis manager’s SCC-aware proxies ensure that when you invalidate one function, dependent analyses in the SCC are refreshed. Inlining invalidates a lot: DT, LI, SE for the caller, and call-graph edges for everyone involved.

## 9. Practical debugging checklist

1. Print the lazy call graph before IPO.
2. Run a single CGSCC pass: `opt -passes='cgscc(inline)'`.
3. Print again; confirm call edges removed for inlined sites.
4. For recursive tests, ensure you still see a multi-node SCC when expected.
5. On asserts builds, `-debug-only=inline` shows decisions; graph bugs often show up as crashes in SCC update, not as silent wrong IR.

## 10. What to read in the source (ordered)

1. File-level comment in [`LazyCallGraph.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/LazyCallGraph.h).
2. `struct CGSCCUpdateResult` in [`CGSCCPassManager.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/CGSCCPassManager.h).
3. `printRefSCC` / printer pass at the bottom of [`LazyCallGraph.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyCallGraph.cpp).
4. Call sites of graph update helpers from [`Inliner.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/Inliner.cpp).
5. Optional contrast: [`CallGraph.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/CallGraph.cpp) constructor and external node setup.

## 11. Mental model summary

Think of **RefSCCs as “could be connected through pointers”** and **call-SCCs as “are connected through visible calls.”** IPO that needs soundness w.r.t. function pointers respects RefSCCs; IPO that wants bottom-up call optimization walks call-SCCs. The inliner sits on the call-SCC walk but must refresh both layers when edges change.

That separation is the main intellectual jump from the old single `CallGraph` picture—and the reason Day 42’s inliner feels “graph-aware” rather than “just replace call with body.”
