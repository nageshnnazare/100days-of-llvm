# Day 3 Extra: Deep Dive into the IR and CFG Classes

Companion to [Day-03-0-CFG-and-Basic-Blocks.md](Day-03-0-CFG-and-Basic-Blocks.md). This is the C++ you are actually stepping through when a pass walks a function.

Primary headers:

- [llvm/include/llvm/IR/Value.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Value.h)
- [llvm/include/llvm/IR/User.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/User.h)
- [llvm/include/llvm/IR/Instruction.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Instruction.h)
- [llvm/include/llvm/IR/BasicBlock.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/BasicBlock.h)
- [llvm/include/llvm/IR/CFG.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/CFG.h)
- [llvm/include/llvm/IR/Function.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Function.h)

---

## Diagram

```text
        entry
        /    \
     then    else
        \    /
        join
         |
       return

   join:
     %x = phi i32 [ 1, %then ], [ 2, %else ]
```

A function is a graph of basic blocks. Each block runs straight through to its terminator, and the terminator's edges are the only way into another block. `%x` is one SSA value. The phi says which definition arrives from which predecessor. A critical edge is an edge from a block with several successors to a block with several predecessors: both `then` and `else` would have that problem if `entry` also branched elsewhere into `join`. Passes split such edges by inserting an empty block.

## 1. `Value`, `Use`, and `User`

Every IR entity that can appear as an operand is a `Value`: instructions, arguments, constants, basic blocks (a block is a value of label type, which is why a branch's operand is the destination block), and functions.

`Use` is the edge from a user to a value. It is not a pointer you allocate yourself. Each operand slot in an instruction is a `Use`.

`User` is a `Value` that has operands. `Instruction` and `ConstantExpr` are users. The operand list is how pattern matching (Day 2) sees inside an instruction.

The methods you will call constantly:

- `Value::users()` / `use_begin()` — everyone who mentions this value. InstCombine's worklist pushes users after a rewrite because their patterns may have changed.
- `Value::replaceAllUsesWith(New)` — the primitive behind "this instruction folded to a constant".
- `Value::hasOneUse()` / `hasNUses` — profitability checks. A rewrite that makes code bigger is often refused when the value has many users.
- `Use::set(New)` — retarget one edge. Prefer `replaceAllUsesWith` unless you mean one user.

`Use` chains are intrusive linked lists hanging off the `Value`. That makes "find all users" proportional to the number of users, not the size of the function. It also means deleting a `Value` while a `Use` still points at it is a use-after-free. The destructor asserts in a debug build if uses remain. The usual sequence is: rewrite uses, then `eraseFromParent()`.

## 2. `Instruction`

[Instruction.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Instruction.h) is the common base. Opcodes are an enum generated from `llvm/include/llvm/IR/Instruction.def`. That `.def` file is worth ten minutes: it is the complete list of real instructions, grouped into terminator, unary, binary, memory, cast, and so on.

Subclasses you should be able to recognize on sight:

| Class | IR |
| --- | --- |
| `BinaryOperator` | `add`, `sub`, `mul`, `and`, `shl`, … |
| `ICmpInst` / `FCmpInst` | `icmp`, `fcmp` |
| `CastInst` | `zext`, `sext`, `trunc`, `bitcast`, `ptrtoint`, `inttoptr` |
| `LoadInst` / `StoreInst` | memory |
| `GetElementPtrInst` | address arithmetic |
| `AllocaInst` | stack slot |
| `PHINode` | `phi` |
| `BranchInst` | `br` |
| `CallInst` / `InvokeInst` | calls |
| `ReturnInst` | `ret` |
| `SelectInst` | `select` (this is **not** a terminator) |

`dyn_cast<LoadInst>(&I)` is the normal type test. It returns null on failure. `cast<LoadInst>` asserts.

Flags live on the instruction, not in the opcode: `nsw` / `nuw` on overflowing binary ops, `exact` on div, `inbounds` on GEP, `fast` math flags on FP, `volatile` and `atomic` on memory ops, alignment on loads and stores. A pass that drops `nsw` is allowed to (it is a weakening). A pass that *adds* `nsw` is claiming that poison cannot newly appear. That claim is a frequent source of bugs.

`Instruction::isTerminator()`, `isEHPad()`, `mayHaveSideEffects()`, `mayReadOrWriteMemory()`, `isSafeToSpeculativelyExecute()` are the predicates later passes use instead of re-deriving them. Read `isSafeToSpeculativelyExecute` in [llvm/lib/IR/Instruction.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/Instruction.cpp) before you hoist anything on Day 23.

Insertion: `IRBuilder` (Day 1) is the frontend-friendly wrapper. Passes often use `Instruction::insertBefore` / `insertAfter` or `IRBuilder` guarded by a folder so constants collapse immediately.

`eraseFromParent()` unlinks the instruction from its block and deletes it. `removeFromParent()` unlinks but does not delete; you then have to insert it somewhere else or you leak it.

## 3. `BasicBlock`

A `BasicBlock` is both:

- an `ilist` node in its parent `Function`, and
- a `Value` of type `LabelTy`, so it can be a branch operand.

The instruction list is an `iplist<Instruction>` with the terminator kept last by convention and checked by the verifier. Helpers:

- `begin()` / `end()` walk every instruction, phis included.
- `phis()` walks only the phi range. The phi range is a contiguous prefix; the moment you see a non-phi, phis are over.
- `getFirstInsertionPt()` returns an iterator after the phis (and after any EH pad). Inserting before that point corrupts the block.
- `getTerminator()` is `back()` of the list, with a cast. An incomplete block under construction may not have one yet; the verifier will reject it if you try to run a pass on it.
- `splitBasicBlock(Iterator, Name)` splits at an instruction. Instructions before the split stay. The split point and everything after it move to a new block. The old block gets an unconditional `br` to the new one. Phis in successors that mentioned the old block are updated when you use the utility in `BasicBlockUtils` rather than the raw method — check which overload you are calling. The method on `BasicBlock` itself updates successor phis to refer to the new block for the moved terminator.

`BasicBlock::removePredecessor(Pred)` fixes phis when an incoming edge disappears. Forgetting this is the classic "phi has 3 inputs but the block has 2 predecessors" verifier crash.

`moveAfter` / `moveBefore` reorder blocks in the function's list. The list order is **not** execution order. It is just the order IR prints in. Passes must not assume the block list is topological. Loop headers often appear before their bodies because Clang emitted them that way, and then SimplifyCFG shuffles them.

## 4. `CFG.h`: edges without an edge object

LLVM does not allocate an `Edge` object for ordinary control flow. The edge is implicit in the terminator's operands.

`succ_iterator` reads the terminator and yields destination blocks. `pred_iterator` is harder: predecessors are not stored on the block. The iterator walks every use of the block-as-value and keeps the ones that are terminators. That is correct and usually cheap (blocks are not used by much besides branches and phis), but it means "how many predecessors?" is not a stored integer you can trust across a rewrite unless you just computed it.

```cpp
for (BasicBlock *Pred : predecessors(&BB)) {
  // Pred terminates with something that can land here.
}
```

`llvm::inverse_children<BasicBlock *>` and `Inverse<Function *>` expose the reversed CFG, which is what post-dominator construction walks.

`isCriticalEdge(Branch, SuccIdx)` in `CFG.h` implements the definition from the main notes: predecessor has multiple successors, successor has multiple predecessors.

## 5. `Function` and the blocks it owns

- `Function::getEntryBlock()` — only valid if `!F.empty()`. Declarations have no blocks.
- `F.isDeclaration()` is true when there is no body. IPO passes must not try to walk blocks of a declaration.
- Arguments are values: `F.args()`. They dominate the entire function.
- `F.getAttributes()` is the function-level attribute set. `optnone` is why your pass silently did nothing (see [Setup-and-Toolchain.md](Setup-and-Toolchain.md)).
- The block list is intrusive. Iterating `for (BasicBlock &BB : F)` while deleting arbitrary blocks is unsafe. The usual pattern is a worklist of pointers, or `make_early_inc_range`.

`GraphTraits<Function *>` defines the entry node as `&F.getEntryBlock()` and the children of a block as its successors. Any algorithm written against `GraphTraits` (depth first, dominators, SCCs of the CFG) then works on functions for free.

## 6. The verifier is the spec

[llvm/lib/IR/Verifier.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/Verifier.cpp) runs between passes when assertions are on, and whenever you pass `-verify-each` to `opt`.

Search for these visit functions:

- `visitBr` — conditional branch operand 0 is `i1`; operands 1 and 2 are blocks.
- `visitPHINode` — number of incoming values equals number of predecessors; incoming blocks are exactly the predecessors; types match.
- `visitCall` — callee type matches the call type (modern LLVM uses opaque pointers, so the interesting checks are argument counts and parameter types).
- The dominance check: each use of a value must be dominated by its definition. This uses `DominatorTree` logic local to the verifier. When it fails you see `Instruction does not dominate all uses`.

Run it on purpose:

```bash
opt -passes=verify -disable-output file.ll
```

A broken phi fails here, not inside your pass's "real" algorithm.

## 7. Utilities that mutate the CFG safely

Prefer these over hand-rolled branch editing. They live in [llvm/lib/Transforms/Utils/BasicBlockUtils.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/BasicBlockUtils.cpp) and [Local.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/Local.cpp).

| Function | What it does |
| --- | --- |
| `SplitBlock(BB, I, DT, LI, MSSAU)` | Split, and update the analyses you pass in |
| `SplitCriticalEdge` | Insert a block on a critical edge |
| `MergeBlockIntoPredecessor` | The inverse of a useless split; used by SimplifyCFG |
| `DeleteDeadBlock` | Remove a block and fix phis in successors |
| `SplitBlockAndInsertIfThen` | Insert `if (cond) { ... }` at a point, with correct phis |
| `removeUnreachableBlocks` | Delete blocks not reachable from the entry |

If you update the CFG and you were given a `DominatorTree`, you must either update it (`DT->applyUpdates(...)`) or tell the pass manager you did not preserve `DominatorTreeAnalysis`. Returning `PreservedAnalyses::all()` after deleting an edge is a real bug: later LICM will trust a stale tree.

## 8. What to ignore on a first read

- The EH pad instructions (`CatchPad`, `CleanupPad`, `LandingPad`). They are blocks with extra "unwind edge" rules.
- `CatchSwitch` and Windows funclet CFG.
- `IndirectBr` and blockaddress. Rare in C, used by computed gotos.
- `CallBr`, which is how Clang lowers `asm goto`.

## 9. Debug flags

```bash
opt -passes=verify -disable-output in.ll
opt -passes=dot-cfg -disable-output in.ll
opt -passes='print<domtree>' -disable-output in.ll   # Day 4
```

There is no `-debug-only=cfg`. The IR dump *is* the debug output. `F.dump()` and `BB.dump()` from a debugger call the same printer as `opt -S`.
