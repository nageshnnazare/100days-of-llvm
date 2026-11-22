# Day 43 Extra: IPSCCP Deep Dive

Companion to [Day-43-0-IPSCCP.md](Day-43-0-IPSCCP.md). Examples: [Day-43-2-Extra-IPSCCP-Examples.md](Day-43-2-Extra-IPSCCP-Examples.md).

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

## 1. Two entry points, one solver

LLVM splits “run SCCP on a function” and “run IPSCCP on a module,” but both funnel into [`SCCPSolver`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SCCPSolver.cpp).

- Scalar pass: [`llvm/lib/Transforms/Scalar/SCCP.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SCCP.cpp)
- IPO pass: [`llvm/lib/Transforms/IPO/SCCP.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/SCCP.cpp)

The IPO driver registers tracked functions, wires call-site value transfer, and applies module-level cleanups after the solver reaches fixpoint. When reading, start at the IPO `run` method, then jump into solver methods named for arguments, returns, and resolved undefs.

## 2. Interprocedural state

Beyond ordinary SSA value lattices, the solver keeps structures conceptually like:

- Formal argument lattice per function argument.
- Return value lattice per function (and possibly multiple returns / struct returns handled carefully).
- Executable flag per basic block (sparse: only visit reachable ones).

Transfer rules at a call:

```
if call block executable:
  for each arg i:
    meet(formal_i, lattice(actual_i))
  mark callee entry executable
```

At a return:

```
if return block executable:
  meet(function_return_lattice, lattice(returned_value))
```

At the call’s result in the caller:

```
meet(call_value, callee_return_lattice)
```

Indirect calls and unknown callees contribute little useful precision (often overdefined).

## 3. Constant ranges and `tryToReplaceWithConstant`

Modern SCCPSolver is stronger than “only exact `ConstantInt`.” It can track **constant ranges** for integers, enabling folds like proving an icmp true without a single constant. Manifestation still goes through replacement helpers such as `tryToReplaceWithConstant` when a value is exactly replaceable by a `Constant`.

Ranges help *inside* the solver; the IR rewrite is still “replace with constant or delete dead code,” not “leave range metadata everywhere” (other passes own range metadata).

## 4. Zapping dead arguments and returns

After fixpoint, IPO SCCP can:

- Replace uses of an argument that is unused or always a known constant with that constant / undef as appropriate, simplifying the body.
- Fold callers that ignore nothing but receive a constant return.

Full **deletion of the argument from the function type** is the job of **DeadArgumentElimination** (Day 44). IPSCCP prepares the battlefield by making the value useless or constant; DAE rewrites the ABI for internal functions.

## 5. Unreachable code deletion

When a branch condition becomes constant, the false edge is non-executable. IPSCCP / solver machinery can delete non-executable blocks in functions it is allowed to modify (typically with internal linkage constraints for aggressive edits). This is the same sparse idea as scalar SCCP, extended so that *calls* in dead blocks never taint callees.

## 6. Function specialization contrasted

[`FunctionSpecialization.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/FunctionSpecialization.cpp) looks for call sites where specializing on a constant (or sometimes a range) is profitable: it **clones** the callee, substitutes the constant, and redirects selected calls.

IPSCCP would instead try to prove facts that hold for *all* calls. If calls disagree, IPSCCP goes overdefined; specialization can still create a clone for the interesting subset.

Pipeline-wise both may appear in aggressive configurations; always check PassBuilder for your version rather than assuming both run at O2.

## 7. Linkage corner cases

| Linkage | Body rewrite | Signature change | Call-site replace return w/ const |
| --- | --- | --- | --- |
| `internal` / `private` | yes | via later DAE | yes |
| `available_externally` | limited / careful | usually no emit | often yes for local opts |
| `linkonce_odr` / `weak_odr` | must respect ODR/interposition rules | no | careful |
| `external` | only if def in module and non-interposable locally (`dso_local` helps) | no | sometimes |

When unsure, treat “external ABI locked” as the default mental model and verify with tests.

## 8. Debugging session recipe

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes t.c -o t.ll
opt -passes=ipsccp -debug-only=sccp t.ll -S -o t.out.ll 2>sccp.log
grep -E 'constant|overdefined|executable' sccp.log | head
```

Then compare:

```bash
opt -passes=func-specialization t.ll -S -o t.spec.ll
diff -u t.out.ll t.spec.ll
```

If `t.spec.ll` invents new function symbols and `t.out.ll` does not, you have successfully distinguished the passes.

## 9. Interaction with inlining

Inlining *also* propagates constants—but by copying bodies. IPSCCP is cheaper when a small lattice fact suffices (always-return-1 helpers). Inlining is better when you need the callee CFG exposed for loop opts. PassBuilder runs both in an order where each feeds the other across iterations of the inliner simplification cycle.

## 10. Source landmarks to find

In `SCCPSolver.cpp` / IPO `SCCP.cpp` (names evolve; search):

- Methods that set argument lattices from call sites.
- Return value merge.
- `tryToReplaceWithConstant`.
- Dead argument zap helpers.
- Restrictions gating which functions are tracked.

Do not invent line numbers; use project search.

## 11. Failure modes worth memorizing

1. **One bad call site** → overdefined argument → no fold.
2. **Funnel through function pointer** → no IP transfer.
3. **`optnone`** → skipped.
4. **Misattributing specialization** → wrong file when debugging.
5. **Expecting DAE effects from ipsccp alone** → signature still has dead params.

## 12. Reading order inside deep dive sources

1. IPO `SCCP.cpp` runOnModule / NewPM `run`.
2. Solver initialization and tracked values.
3. Call/return transfer.
4. Manifestation / IR rewrite.
5. Ten-minute skim of `FunctionSpecialization.cpp` introduction comment only.
