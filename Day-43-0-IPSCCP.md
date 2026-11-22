# Day 43: IPSCCP (Interprocedural Sparse Conditional Constant Propagation)

Extra notes: [DeepDive](Day-43-1-Extra-IPSCCP-DeepDive.md) · [Examples](Day-43-2-Extra-IPSCCP-Examples.md)

## Why

Intraprocedural SCCP (Day 12 territory) proves that values are constant *inside one function*. Real programs pass constants across calls: `foo(0)`, or a helper that always returns `1`. **IPSCCP** lifts the same lattice solver across the call graph so argument and return lattices meet at call sites. Dead arguments, dead returns, and unreachable callee blocks fall out as constants resolve.

## Diagram

```text
   define void @caller() {
     call void @callee(i32 7)
   }
   define void @callee(i32 %x) {
     ; SCCP inside @callee sees %x as overdefined,
     ; because a direct call could pass anything.
   }

   IPSCCP looks at the call edge:
     the only call passes 7
     so %x is Constant 7 in @callee
     and folds uses of %x
```

IPSCCP is SCCP plus the call graph. A parameter that is the same constant at every call site becomes that constant inside the function. A return value that is a constant flows back to the callers. This needs the whole SCC: a recursive function that sometimes passes a different value stays overdefined.

## The idea

Reuse **`SCCPSolver`** ([`llvm/lib/Transforms/Utils/SCCPSolver.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SCCPSolver.cpp)) but track:

- Lattice state for **formal arguments** and **return values** of each function.
- **Executable edges** between caller blocks and callee entries when a call is reachable.
- Propagation of constants into callees and of returned constants back into callers.

After the fixpoint, IPSCCP **rewrites IR**: replace uses with constants (`tryToReplaceWithConstant`), zap dead arg uses, simplify returns, delete unreachable internal code.

### Not the same as function specialization

**Constant propagation across calls** ≠ **cloning a function for a constant argument**.

| Pass | Name | What it does |
| --- | --- | --- |
| IPSCCP | `ipsccp` | Propagate constants; rewrite call sites / bodies in place |
| Function specialization | `func-specialization` | Clone a function specialized to a constant (or range) argument — [`FunctionSpecialization.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/FunctionSpecialization.cpp) |

IPSCCP may use constant ranges inside the solver and create opportunities that specialization later exploits, but **specialization is a separate pass**. Do not credit IPSCCP with inventing `@f.specialized.1` clones unless you also ran `func-specialization`.

## Pipeline position and `opt -passes=` name

```bash
opt -passes=ipsccp input.ll -S -o out.ll
# Debug (asserts build):
opt -passes=ipsccp -debug-only=sccp input.ll -S
```

Implementation entry: [`llvm/lib/Transforms/IPO/SCCP.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/SCCP.cpp) (IPSCCP pass uses shared `SCCPSolver`).

Emit IR without `optnone`:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
```

In `default<O#>` pipelines, IPSCCP runs early at module scope (near other IPO constant / attr passes)—see [`PassBuilderPipelines.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Passes/PassBuilderPipelines.cpp).

## Core concepts

### Lattice

Classic SCCP lattice: **undefined (top/unknown pending)** → **constant** → **overdefined (bottom)**. IPSCCP extends tracked values to include interprocedural return slots and arguments. Multiple call sites with different constants force an argument to overdefined unless all agree.

### Reachability

A call only pushes values into a callee if the call instruction is in a reachable block. Dead callers do not pollute callee lattices. Conversely, proving a callee return is constant can fold the caller’s compare and make more blocks unreachable—**sparse conditional** interplay across functions.

### Linkage and signatures

- **Internal / private**: IPSCCP can be aggressive about rewriting the function body and sometimes cleaning unused formals’ *uses* (actual signature deletion is often left to deadargelim).
- **External**: cannot change the ABI, but **call sites in this module** can still replace a call with a constant if the return lattice is constant *and* that is sound for the visible definition (careful with interposable linkage).

### What IPSCCP can do

- Replace call results with constants when the return is constant.
- Propagate constant arguments into internal callees and fold their bodies.
- Mark blocks unreachable inside functions once conditions fold.
- Remove uses of arguments proven unused / constant in limited ways (“zap” dead arg uses in the IPO SCCP pass).

### What it should not be blamed for

- Creating specialized clones → `func-specialization`.
- Removing unused formals from the type → primarily `deadargelim` (Day 44).
- Inferring `readonly` etc. → `function-attrs` / Attributor (Day 48).

## Algorithm

1. Initialize solver; seed tracked functions (typically all in the module with appropriate filters).
2. For each executable call: meet actual operands into formal argument lattices; mark callee entry executable.
3. Run intraprocedural SCCP rules inside executable blocks (folding binary ops, resolving br/switch).
4. Meet return values into the function’s return lattice; propagate to call results in callers.
5. Iterate to fixpoint.
6. Manifest: `tryToReplaceWithConstant`, delete dead blocks, clean dead arg uses where the pass is allowed to.

## Code map

| Piece | Link |
| --- | --- |
| IPSCCP pass | [`llvm/lib/Transforms/IPO/SCCP.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/SCCP.cpp) |
| Shared solver | [`llvm/lib/Transforms/Utils/SCCPSolver.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SCCPSolver.cpp) |
| Function specialization (separate) | [`llvm/lib/Transforms/IPO/FunctionSpecialization.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/FunctionSpecialization.cpp) |
| Intraprocedural SCCP pass | [`llvm/lib/Transforms/Scalar/SCCP.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SCCP.cpp) |

## Worked example

```c
static int scale(int x, int k) {
  if (k == 0) return 0;
  return x * k;
}

int only_k0(int x) { return scale(x, 0); }
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes s.c -o s.ll
opt -passes=ipsccp s.ll -S -o s.ipsccp.ll
```

IPSCCP sees the second argument is always `0` at the only call site (if `scale` is internal and this is the sole caller—or at least this site contributes `0` and no other site contributes a conflicting value). Inside `scale`, `k == 0` is true on that lattice → return `0`. Then `only_k0`’s call can become `0` without needing full inlining.

If another caller passes `k=2`, `k` becomes overdefined and the fold may not apply globally; specialization might clone a `k==0` variant later—that is **not** IPSCCP’s job.

## Reading order

1. Pass driver in IPO [`SCCP.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/SCCP.cpp).
2. Argument/return tracking in [`SCCPSolver.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SCCPSolver.cpp).
3. Skim [`FunctionSpecialization.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/FunctionSpecialization.cpp) only enough to contrast.
4. [Day-43-1-Extra-IPSCCP-DeepDive.md](Day-43-1-Extra-IPSCCP-DeepDive.md).

## Hands-on

1. Constant return through an internal helper; run only `ipsccp`.
2. Two call sites with different constants; observe overdefined.
3. External function that always returns a constant in *this* TU’s definition—note ABI limits.
4. Compare `opt -passes=ipsccp` vs `opt -passes=func-specialization` on a multi-constant-arg helper.
5. `-debug-only=sccp` and watch lattice transitions.

## Pitfalls

- Crediting IPSCCP for specialization clones.
- Expecting signature deletion (use deadargelim).
- Using `clang -O0` IR (`optnone`).
- Weak/interposable linkage: returning a constant in *this* copy does not mean every DSO agrees.
- Forgetting that overdefined is sticky: one unknown call site ruins an argument lattice.

## Neighbors

- **Day 12 / scalar SCCP** — same solver, one function.
- **Day 42 Inliner** — alternative way to expose constants; complementary.
- **Day 44 DeadArgElim** — removes formals whose values became useless.
- **Day 49 Linkage** — who you may rewrite.

## Lattice meet examples

| Call sites for arg `%k` | Lattice of `%k` |
| --- | --- |
| only `0` | constant `0` |
| `0` and `0` | constant `0` |
| `0` and `2` | overdefined |
| `0` and unknown `%x` | overdefined |

Return lattices work the same way across return instructions in reachable blocks.

## Check yourself

**Q1.** Which solver does IPSCCP share with function SCCP?  
**A1.** `SCCPSolver` in `Transforms/Utils/SCCPSolver.cpp`.

**Q2.** Pass name?  
**A2.** `ipsccp`.

**Q3.** Is function specialization part of IPSCCP?  
**A3.** No—`func-specialization` is separate (`FunctionSpecialization.cpp`).

**Q4.** Can IPSCCP rewrite call sites of external functions?  
**A4.** It can replace uses of a call’s *result* with a constant when sound; it must not change an external function’s signature.

**Q5.** Debug flag?  
**A5.** `-debug-only=sccp`.

## Lattice meet across call sites (worked IR)

```llvm
define internal i32 @pick(i32 %mode) {
  %c = icmp eq i32 %mode, 1
  br i1 %c, label %t, label %f
t:
  ret i32 10
f:
  ret i32 -1
}
define i32 @only() {
  %r = call i32 @pick(i32 1)
  ret i32 %r
}
```

`ipsccp` meets the sole actual `1` into `%mode`. The `icmp` becomes true, block `%f` is non-executable, and the return lattice is `10`. `@only` can then return `10` without inlining.

Add a second caller `call i32 @pick(i32 2)` and the meet of `1` and `2` is overdefined. Conditional folding inside `@pick` stops, and IPSCCP will not invent `@pick.specialized`. That clone, if it appears, comes from `func-specialization`.

## What “zap dead arguments” means in IR

IPSCCP may replace uses of a formal with a constant or with `undef`/`poison` when the formal is unused in executable code, but the function type string often still lists the parameter. Call sites still pass the operand until `deadargelim` rewrites the type. When reading diffs, look for:

- `call` results replaced by constants (`tryToReplaceWithConstant`)
- unreachable blocks deleted
- arguments whose uses vanished, but whose slots remain

Do not expect `@llvm.used` functions to lose ABI just because a local call looks constant.

## Debug log reading

```bash
opt -passes=ipsccp -debug-only=sccp t.ll -S 2>sccp.log
```

Look for transitions of argument and return values through constant → overdefined, and for blocks marked executable. If a block never becomes executable, a call inside it must not poison the callee’s lattice.

## Ordering relative to the inliner

IPSCCP is cheap when every call agrees. The inliner is the tool when calls disagree but one site has constants the callee’s CFG can fold. PassBuilder runs both; a lab that runs only `ipsccp` and is disappointed should try `func-specialization` next, not a higher inline threshold, if the goal was a clone.

## Internal linkage and executable edges

Only blocks that the solver marks executable contribute to argument and return lattices. A call in a dead block must not meet its operands into the callee. That is the same sparse rule as scalar SCCP, applied at call edges. External functions can still have a call in *this* module replaced by a constant when the return lattice is a constant and the linkage rules say the definition is the one being called. The signature of that external function stays put.

A constant range inside SCCPSolver can prove an icmp without a single ConstantInt.
tryToReplaceWithConstant only fires when the lattice is an exact constant the IR can hold.
Ranges themselves are not written back as range metadata by IPSCCP; other passes own that.
