# Day 8 Extra: Deep Dive into EarlyCSE.cpp

Companion to [Day-08-0-EarlyCSE.md](Day-08-0-EarlyCSE.md). File: [llvm/lib/Transforms/Scalar/EarlyCSE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/EarlyCSE.cpp).

---

## Diagram

```text
   entry
     %x = add %a, %b          hash(%a,%b,add) -> %x
     |
     +-- then --+
     |  %y = add %a, %b       same hash -> reuse %x
     |                        delete %y
     +-- else --+
        %z = add %a, %b       NOT the same scope if this
                              block is not dominated by
                              the first add... it is, via
                              entry, so %z folds too

   a store to %p followed by a load of %p, with no
   intervening write, forwards the stored value
```

EarlyCSE keeps a scoped hash table that pushes a scope when it enters a block and pops when it leaves. A value computed in a block is available in blocks it dominates. The MemorySSA mode can forward a load across a call that does not alias; the plain mode is more conservative.

## 1. Two passes, one class

`EarlyCSEPass::run` looks at a boolean, `UseMemorySSA`, set by the pass constructor. The pass registry exposes both names:

- `early-cse` constructs `EarlyCSEPass(/*UseMemorySSA=*/false)`
- `early-cse-memssa` constructs it with `true`

The run method requests analyses accordingly. MemorySSA mode also requests `AAResults` because MemorySSA's walker uses alias analysis. If you schedule `early-cse-memssa` by hand you do not need to schedule AA yourself; the analysis manager pulls it in. You will see that in `-debug-pass-manager` output.

## 2. `SimpleValue`

A small struct wrapping an `Instruction *` or a `Value *`, with `getHashValue` and `isEqual` as required by `DenseMapInfo` / the scoped hash table.

What goes into the hash:

- opcode,
- type,
- operands, commuted into a stable order when `Instruction::isCommutative()` is true,
- predicate, for compares,
- the flags that change the value: wrapping flags, exact flag, fast-math flags.

What does not:

- the name,
- the basic block,
- debug location.

`isEqual` must agree with the hash. If two values compare equal they must have the same hash, or the table silently drops CSE opportunities. If two values have the same hash but `isEqual` is false, that is only a collision and the table probes onward. Read `isEqual` when a CSE you expected did not happen: usually the flags differ.

`CallValue` is a sibling hasher for calls. It includes the called function and the arguments. A call is eligible only when it does not access memory (or, in some cases, when it is `readonly` and the memory generation matches) and will return. Search for `canHandle` near the call logic.

## 3. `ScopedHashTable`

[llvm/include/llvm/ADT/ScopedHashTable.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/ADT/ScopedHashTable.h).

`ScopedHashTableScope` is an RAII object. Constructing it pushes a scope. Destroying it pops every insertion made while it was top. Insertions in an outer scope remain visible. This is why `StackNode` holds a `ScopedHashTableScope` by value: the node's lifetime is the scope's lifetime.

The table maps `SimpleValue` to the leader `Value *`. Lookup during the walk finds the nearest dominating leader.

## 4. `StackNode` and the DFS

`StackNode` holds:

- the block,
- an iterator over the dominator-tree children,
- the hash scope,
- the memory generation at the start of the block,
- the last store, for forwarding.

`processNode` handles one block's instructions. The DFS driver pushes a node for the entry, and when a block finishes, pushes nodes for its dominator-tree children. Using an explicit stack rather than recursion matters on huge functions; the logic is the same as recursion.

Dominance invariant: the hash scope chain at the start of `processNode` contains exactly the contributions of the proper dominators. The first instruction of the block has not been inserted yet.

## 5. Processing one instruction

The order inside `processNode` is the whole pass:

1. **Match an available value.** If the instruction is simple and the table has a leader, replace all uses and delete. Continue to the next instruction. Do not insert the deleted one.
2. **Try memory CSE / forwarding** if it is a load. On success, replace and delete.
3. **Try trivial DSE** if it is a store of a value that was just stored to the same pointer.
4. **Insert** the instruction into the table if it defines a simple value.
5. **Update memory state** if the instruction may write memory.

Step 5 must happen even when the instruction was also inserted as a simple value. A `readnone` call does not bump the generation. An unknown call does.

Load forwarding: if the last store's pointer is the load's pointer (pointer equality, not alias analysis, in the simple mode) and the generation matches, the load becomes the stored value. Pointer equality means bit-identical `Value *`, so two different GEPs that compute the same address do not forward in simple mode unless InstCombine already merged the GEPs. That is another reason this pass runs *after* InstCombine, and another reason MemorySSA mode is stronger: AA can see through different GEP instructions.

## 6. Generation counter versus MemorySSA

Simple mode fields to look for: `CurrentGeneration`, `LastStore`. A may-write bumps `CurrentGeneration`. Loads stash the generation in the hash so a load from generation 3 does not match a load from generation 4.

MemorySSA mode: the load's hash includes `MSSA->getSkipSelfWalker()->getClobberingMemoryAccess(Load)`. Two loads with the same pointer and the same clobbering access match. A store to an unrelated pointer produces a MemoryDef that does not clobber the load, so the walker steps over it, and the generation-counter bug does not exist.

The walker is not free. EarlyCSE-with-MemorySSA is still much cheaper than GVN because it does not do PRE and it takes one dominator-tree walk.

## 7. Assumes

Search for `assume`. When the pass sees `llvm.assume`, it records the assumed condition in the current scope. Later, `match` against that condition can fold a compare to true or false, or treat two values as equal for the hasher. Assumptions are scoped, so an assume inside a branch does not leak into the other branch. That is correct: the other branch is exactly where the condition may be false.

`AssumptionCache` is still requested because other queries in the file ask "is there an assume that dominates this instruction?" The cache is the standard way to avoid scanning the function for assumes at every query.

## 8. Deletion and the worklist

The pass does not use InstCombine's worklist. One walk is the design. After a deletion it may call `RecursivelyDeleteTriviallyDeadInstructions` on the now-unused operands so a chain `t = mul; u = add t, 0` collapses in the same pass when the `add` was the only user and it got CSE'd away. If you do not see that helper, the pipeline relies on a following DCE. Check the file: calling the helper is the modern behavior.

## 9. Preservation

EarlyCSE invalidates instruction-level analyses when it deletes instructions. It preserves the CFG, so the dominator tree remains valid. MemorySSA mode updates MemorySSA or claims not to preserve it; look at the `PreservedAnalyses` at the end of `run`. Getting this wrong causes GVN later in the same pipeline to see a stale MemorySSA. The upstream pass does this carefully; a copy-pasted pass often does not.

## 10. Debug

```bash
opt -passes=early-cse -print-changed=quiet -S in.ll
```

Asserts build: `-debug-only=early-cse` prints hashes and replacements. When a replacement does not happen, the debug line showing the hash miss is more useful than reading IR.

## 11. What EarlyCSE leaves for GVN

- Partial redundancy (the value is available on some but not all incoming edges).
- Loads that are equal only after phi translation ("the load of `%p` in the predecessor is the load of `%phi` here").
- Store-to-load forwarding across basic blocks in the simple mode, when the store does not dominate in a way the single last-store slot can see. The simple mode's "last store" is per scope, so across blocks it works only along the dominator stack, and only for the latest store. GVN handles a lattice of memory definitions.

If you are deciding which pass should implement a new memory optimization, EarlyCSE is the right home only when a dominator-scoped hash is the entire story.
