# Day 41 Extra: Call Graph Examples

Companion to [Day-41-0-CallGraph.md](Day-41-0-CallGraph.md). Deep dive: [Day-41-1-Extra-CallGraph-DeepDive.md](Day-41-1-Extra-CallGraph-DeepDive.md).

Common setup:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='print<lazy-callgraph>' file.ll -disable-output
opt -passes='cgscc(inline)' file.ll -S -o out.ll
```

---

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

## Example 1: Acyclic call chain (bottom-up order)

**Input (C):**

```c
static int add1(int x) { return x + 1; }
static int add2(int x) { return add1(x) + 1; }
int entry(int x) { return add2(x); }
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes chain.c -o chain.ll
opt -passes='print<lazy-callgraph>' chain.ll -disable-output
opt -passes='cgscc(inline)' chain.ll -S -o chain.inlined.ll
```

**Expected result:**

- Call edges `entry→add2→add1`.
- Three singleton call-SCCs; post-order visits `add1`, then `add2`, then `entry`.
- After `cgscc(inline)`, `entry` typically contains the arithmetic directly (subject to cost thresholds); `add1`/`add2` may become dead if internal and unused.

**Why:** Bottom-up CGSCC visitation inlines leaves first so callers see already-simplified callees.

**Counterexample:** If `add1` is `noinline`, the edge `add2→add1` remains; the graph still has the same shape, but inlining does not remove that edge.

---

## Example 2: Mutual recursion → non-trivial SCC

**Input (C):**

```c
static int b(int x);
static int a(int x) {
  if (x <= 0) return 0;
  return b(x - 1);
}
static int b(int x) {
  if (x <= 0) return 1;
  return a(x - 1);
}
int start(int x) { return a(x); }
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes rec.c -o rec.ll
opt -passes='print<lazy-callgraph>' rec.ll -disable-output
```

**Expected result:**

- Call edges `a→b` and `b→a`.
- One call-SCC containing both `a` and `b` (plus `start` as a separate SCC that calls into it).
- Inliner will not freely inline `a` into `b` in a way that assumes a finished callee—recursion is handled specially (Day 42).

**Why:** Call-SCCs capture mutual reachability via call edges; recursive sets are processed as a unit.

**Counterexample:** Break the cycle by making `b` call a third non-recursive function only. Then `a` and `b` become separate SCCs again (or a trivial chain), and bottom-up inlining becomes ordinary.

---

## Example 3: Address-taken → reference edge / RefSCC

**Input (IR):**

```llvm
define internal i32 @secret(i32 %x) {
  ret i32 %x
}

define ptr @get_fp() {
  ret ptr @secret
}

define i32 @maybe(ptr %p, i32 %x) {
  %r = call i32 %p(i32 %x)
  ret i32 %r
}
```

**Commands:**

```bash
opt -passes='print<lazy-callgraph>' addr.ll -disable-output
```

**Expected result:**

- `@get_fp` has a **ref** to `@secret` (returned address), not necessarily a call edge.
- `@maybe` has an **indirect** call; no static edge to `@secret`.
- RefSCC analysis still treats `@secret` as referenced; GlobalDCE will not delete it while `@get_fp` can return it.

**Why:** Reference edges model “this function’s address escaped,” which is weaker than a call but stronger than “unused.”

**Counterexample:** Replace `ret ptr @secret` with `ret ptr null` and never mention `@secret`. Then `@secret` has no refs/calls and is DCE fodder (if internal).

---

## Example 4: `llvm.compiler.used` keeps a ref

**Input (IR):**

```llvm
define internal void @isr() {
  ret void
}

@llvm.compiler.used = appending global [1 x ptr] [ptr @isr], section "llvm.metadata"

define void @app() {
  ret void
}
```

**Commands:**

```bash
opt -passes='print<lazy-callgraph>' used.ll -disable-output
opt -passes=globaldce used.ll -S -o used.dce.ll
```

**Expected result:**

- Graph shows a reference involving `@isr` despite no `call @isr`.
- `globaldce` **keeps** `@isr` because `llvm.compiler.used` is a root.

**Why:** Environment-called symbols must remain live; the used lists are part of the module’s root set and create references.

**Counterexample:** Remove the `@llvm.compiler.used` entry. Then `globaldce` deletes internal `@isr`, and the call graph no longer contains that node after DCE.

---

## Example 5: External / library call conservatism

**Input (C):**

```c
int printf(const char *, ...);
int show(int x) { return printf("%d\n", x); }
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes ext.c -o ext.ll
opt -passes='print<lazy-callgraph>' ext.ll -disable-output
```

**Expected result:**

- `@show` calls `@printf`.
- `@printf` is external declarations; edges into / through external world are conservative. You cannot IPA-rewrite `printf`’s signature.

**Why:** The call graph must not pretend callees outside the module are fully known.

**Counterexample:** Provide an `internal` implementation of a similarly named helper in the same module—then IPO *can* see the body and may inline or propagate (subject to linkage).

---

## Quick drill

| Observation | Likely cause |
| --- | --- |
| Multi-node SCC | Mutual recursion or cycle of direct calls |
| Function in graph, never called | Address taken or `llvm.*.used` |
| No edge to obvious callee | Indirect call / opaque pointer callee |
| Edges vanish after `cgscc(inline)` | Successful inlining removed call sites |
