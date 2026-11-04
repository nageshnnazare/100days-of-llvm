# Day 25: Loop Deletion (`loop-deletion`)

LoopDeletion removes loops that do not affect observable behavior (aside from time), or that are required to make progress but do not. Implementation: [LoopDeletion.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopDeletion.cpp). Extra: [Day-25-1-Extra-LoopDeletion-DeepDive.md](Day-25-1-Extra-LoopDeletion-DeepDive.md). Examples: [Day-25-2-Extra-LoopDeletion-Examples.md](Day-25-2-Extra-LoopDeletion-Examples.md).

## 1. Why this exists

Frontends and earlier opts leave loops that compute nothing useful:

```c
for (int i = 0; i < n; ++i)
  /* empty / dead math */;
```

Or loops proven never entered. Keeping them wastes code size and blocks CFG simplify. Deletion also implements language rules around **progress**: under `mustprogress`, an infinite side-effect-free loop is undefined / non-conforming and may be removed.

## Diagram

```text
   header:
     %c = icmp slt i32 %i, %n
     br i1 %c, label %body, label %exit
   body:
     ; no side effects, no volatile, no escape
     br label %header
   exit:
     ret i32 0

   mustprogress is set, trip count may be infinite
   only if the loop never exits -- but a side-effect-free
   loop that the language says must make progress
   can be deleted:

   exit:
     ret i32 0
```

LoopDeletion removes a loop that does nothing observable. A finite loop whose body has no side effects disappears and control goes to the exit. An infinite loop is deleted only when the function is `mustprogress` (C++ forward-progress). Without that attribute the infinite loop stays, because deleting it would let later code run when the source never did.

## 2. The idea

Given LoopSimplify + LCSSA, try in order (see `LoopDeletionPass::run`):

1. **`deleteLoopIfDead`** — remove the whole loop when never executed or when the body is invariant/dead per `isLoopDead`.
2. Else **`breakBackedgeIfNotTaken`** — if the backedge is never taken (SCEV zero BTC or first-iteration symbolic proof), break the latch; the “loop” collapses without necessarily deleting every block up front.

`deleteDeadLoop` (in LoopUtils) does the CFG surgery: redirect preheader to the exit, poison or rewrite exit PHIs, drop loop blocks from LoopInfo.

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `loop-deletion` |
| File | [llvm/lib/Transforms/Scalar/LoopDeletion.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopDeletion.cpp) |
| Debug | `-debug-only=loop-delete` (`DEBUG_TYPE "loop-delete"`) |
| Needs | Preheader, dedicated exits, LCSSA |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa,loop-deletion' -S file.ll -o file.del.ll
```

## 4. Core concepts

### Never-executed loops

`isLoopNeverExecuted`: every predecessor of the preheader is a conditional branch with a **constant** condition that does **not** take the edge into the preheader. Then the loop is dead on all paths; exit PHIs are filled with poison and the loop is deleted.

### Finite side-effect-free loops

`isLoopDead` requires:

- At most one exit block (or no exits only in special “no exit blocks” cases handled carefully).
- All LCSSA exit PHI incoming values from the loop are the same across exiting blocks and can be made loop-invariant (`makeLoopInvariant`).
- No instruction in the loop `mayHaveSideEffects()` (unless droppable)—no volatile, no escaping writes, no general calls with side effects. Calls that are truly side-effect-free (readnone-like / willreturn+nounwind appropriately reflected in `mayHaveSideEffects`) can remain.

Then the loop only exists to burn time or produce invariant outs already known—safe to delete for **finite** cases.

### `mustprogress` and infinite SEF loops (accurate)

LLVM allows a side-effect-free loop to run forever **unless** progress is required.

From `isLoopDead` / `deleteLoopIfDead` comments:

- If the **function** has `mustprogress` (`Function::mustProgress()`), a SEF loop may be deleted even when SCEV cannot prove a finite trip count—an infinite SEF loop would violate the progress requirement (C++ `std::is_trivially_*` / Clang often emit `mustprogress` on functions; loops may also carry `llvm.loop.mustprogress` via `hasMustProgress`).
- If the function is **not** mustprogress, every (sub)loop must either have `mustprogress` metadata/attr **or** a known SCEV constant-max backedge taken count. If SCEV returns “could not compute” for a non-mustprogress loop, **deletion is refused**—preserving a potentially infinite SEF loop.

Summary:

| Loop | mustprogress? | Delete SEF? |
|------|---------------|-------------|
| Never executed | n/a | Yes |
| Finite SEF, exit vals OK | either | Yes |
| Infinite SEF | no | **No** |
| Infinite SEF | yes (fn or loop) | **Yes** |

Do not invert this table: without mustprogress, infinite empty loops are intentionally kept.

### SCEV’s role

- Trip / backedge counts decide finiteness and the backedge-break path (`getConstantMaxBackedgeTakenCount`, `getBackedgeTakenCount`).
- Exit values must be invariant; often later indvars already rewrote LCSSA phis to closed forms, making deletion easier.
- Before deleting a never-executed loop, `SE.forgetLoop` runs so exit PHIs set to poison do not leave stale SCEVs.

### Backedge break

Even when the whole loop is not “dead invariant,” if BTC is zero or first-iteration symbolic execution proves the latch never returns to the header (`canProveExitOnFirstIteration`), `breakLoopBackedge` destroys the cycle. Statistic `NumBackedgesBroken`.

## 5. The algorithm

`deleteLoopIfDead`:

1. Require preheader + dedicated exits; reject EH-pad unique exits.
2. If unique exit && never executed → poison exit PHIs → `deleteDeadLoop`.
3. Else require unique exit (or no-exit special case); run `isLoopDead`.
4. On success → remark “invariant” → `deleteDeadLoop`.

`isLoopDead` checks invariant exit PHIs, scans for side effects, then applies the mustprogress / SCEV finiteness rules above (including irreducible CFG rejection when progress is not required).

`LoopDeletionPass::run` merges results; on deletion notifies `LPMUpdater::markLoopAsDeleted`.

## 6. Where to read the code

1. [LoopDeletion.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopDeletion.cpp) — `isLoopDead`, `isLoopNeverExecuted`, `deleteLoopIfDead`, `breakBackedgeIfNotTaken`, `canProveExitOnFirstIteration`.
2. `deleteDeadLoop` / `breakLoopBackedge` in [LoopUtils](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopUtils.h).
3. Tests: `llvm/test/Transforms/LoopDeletion/`.

## 7. Worked before/after

```c
void fool(int n) {
  for (int i = 0; i < n; ++i) {
    /* empty */
  }
}
```

With `mustprogress` on the function (typical for Clang C++ / many C modes) or a known trip count, the empty loop can disappear entirely:

```llvm
define void @fool(i32 %n) {
  ret void
}
```

Without mustprogress and with unknown `n` that might never exit a non-empty-but-SEF construct, deletion must not invent termination.

## 8. Reading order for today

1. Never-executed vs invariant-SEF vs mustprogress table in this note.
2. `isLoopDead` in LoopDeletion.cpp end-to-end.
3. Day 25 DeepDive (symbolic first iteration, irreducible CFG).
4. Day 25 Examples.
5. Revisit LCSSA (Day 22)—exit PHIs are the live-out contract deletion rewrites.

## 9. Hands-on commands

```bash
cat > del.c <<'EOF'
void empty(int n) {
  for (int i = 0; i < n; ++i) {
  }
}
int never(int x) {
  if (0)
    for (int i = 0; i < x; ++i) { }
  return x;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes del.c -o del.ll
opt -passes='loop-simplify,lcssa,loop-deletion' -S del.ll -o del.out.ll
# opt -debug-only=loop-delete -passes='loop-simplify,lcssa,loop-deletion' -S del.ll
```

Inspect attributes for `mustprogress`. Compare a language mode / flags that omit it if you want to see retention of infinite SEF loops (hand-written IR is clearer for that experiment).

## Side effects in practice

`mayHaveSideEffects()` is the gate—not a hand-rolled list. Still, when reading IR, these almost always block deletion:

- `store` to memory that escapes / is observable
- `load volatile` / `store volatile`
- `call` without attributes proving no side effects / always returns
- Synchronization / atomics (as modeled)

Pure arithmetic, invariant GEPs, and non-volatile loads of non-escaping locals may survive inside a still-deletable loop if outs are invariant.

## Backedge break vs full delete

Full delete: preheader branches to the exit; loop blocks removed from LI; statistics `NumDeleted`.

Backedge break: latch no longer targets header; CFG may still contain the old body as dead or single-pass code; `NumBackedgesBroken`. Later SimplifyCFG/ADCE finish the job.

Knowing which path fired matters when `-debug-only=loop-delete` shows “invariant” vs backedge messages.

## 10. Pitfalls and invariants

- Deleting infinite SEF loops **without** mustprogress — incorrect; LLVM preserves them.
- Ignoring side effects: a single volatile or opaque call blocks `isLoopDead`.
- Multiple exits — usually refuse full deletion.
- Missing LCSSA — assertions / unsafe exit-value reasoning.
- Debug type is `loop-delete`, not `loop-deletion`.
- Assuming C `for(;;);` is always deleted — depends on mustprogress emission for that language mode.

## 11. Neighbors

- **LCSSA / LoopSimplify** — hard requirements.
- **SCEV** — finiteness and BTC.
- **ADCE / SimplifyCFG** — clean after deletion.
- **IndVarSimplify** — often makes exit values invariant first.
- **mustprogress** attrs / `llvm.loop.mustprogress` metadata.

## 12. Check yourself

1. **Q:** Three deletion themes?  
   **A:** Never executed; finite SEF with invariant outs; SEF infinite only when mustprogress requires progress.

2. **Q:** Infinite empty loop, no mustprogress — delete?  
   **A:** No.

3. **Q:** Same loop with function `mustprogress` — delete?  
   **A:** Yes, if otherwise SEF / dead per `isLoopDead`.

4. **Q:** Debug flag?  
   **A:** `-debug-only=loop-delete`.

5. **Q:** What if only the backedge is proven untaken?  
   **A:** `breakBackedgeIfNotTaken` / `breakLoopBackedge` — destroy the cycle without the full “invariant loop” path.

6. **Q:** Why LCSSA?  
   **A:** Exit PHIs are the only live-outs that must be invariant / rewritable.
## C vs C++ mental note

Clang’s `mustprogress` emission depends on language and flags. When an empty loop survives in C IR, check attributes before blaming LoopDeletion. Hand-written IR (extras Example 4) removes that ambiguity: infinite SEF stays without mustprogress and may go with it.

## Attribute checklist when deletion “should have” fired

1. Is the body free of `mayHaveSideEffects` ops?
2. Is there a unique dedicated exit + preheader?
3. Are exit PHI incomings invariant and consistent?
4. Is trip count finite via SCEV **or** is `mustprogress` present on the function/loop?

If any answer is no, LoopDeletion is doing the right thing by leaving the loop alone.
Miss one checkbox → expect the loop to remain.

> Tip: Deep dive in `Day-25-1-Extra-LoopDeletion-DeepDive.md`. Examples in `Day-25-2-Extra-LoopDeletion-Examples.md`.
