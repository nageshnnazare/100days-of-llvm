# Day 3 Extra: CFG and IR Examples

Paste these into files and run them. They are about the shape of the IR, not about a single optimization.

The command that produces IR `opt` is willing to modify:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
```

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

## Example 1: A diamond and its phi

**Goal:** See predecessors, successors, and a phi that mem2reg inserts.

```c
int pick(int a, int b, int c) {
  int r;
  if (a)
    r = b;
  else
    r = c;
  return r;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes pick.c -o pick.ll
opt -passes=mem2reg -S pick.ll -o pick.m2r.ll
```

**What you should see** in `pick.m2r.ll`: an `entry` block that branches, two blocks that each compute a value, and a join whose first instruction is

```llvm
%r = phi i32 [ %b, %then ], [ %c, %else ]
```

(The block names will not be exactly `then` / `else`.)

**Why:** `r` is one SSA value with two reaching definitions. The phi is the join.

**Counterexample:** if both arms store the same constant, InstCombine + SimplifyCFG often delete the phi and the branch. The CFG is allowed to get simpler; the phi is not sacred. Run:

```bash
opt -passes=mem2reg,instcombine,simplifycfg -S pick.ll
```

with `b` and `c` replaced by the constants `1` and `1` in the source. The diamond collapses.

---

## Example 2: A loop backedge

**Goal:** Identify the header, the latch, and the backedge by reading `br` only.

```llvm
define i32 @sum_to_n(i32 %n) {
entry:
  %cmp0 = icmp sgt i32 %n, 0
  br i1 %cmp0, label %header, label %exit

header:
  %i = phi i32 [ 0, %entry ], [ %inext, %latch ]
  %acc = phi i32 [ 0, %entry ], [ %accnext, %latch ]
  %accnext = add i32 %acc, %i
  %inext = add i32 %i, 1
  br label %latch

latch:
  %cont = icmp slt i32 %inext, %n
  br i1 %cont, label %header, label %exit

exit:
  %result = phi i32 [ 0, %entry ], [ %accnext, %latch ]
  ret i32 %result
}
```

```bash
opt -passes=verify -disable-output loop.ll
opt -passes=dot-cfg -disable-output loop.ll
```

**Walk:**

- `entry` successors: `header`, `exit`.
- `header` successors: `latch` only.
- `latch` successors: `header` (the backedge) and `exit`.
- `exit` successors: none.

`header` has two predecessors (`entry` and `latch`), so both phis need two incoming values. `%i` and `%acc` are loop-carried. `%accnext` is not a phi; it is an ordinary add that the latch's outgoing phi (in `exit`) and the next iteration's `%acc` both use.

**Counterexample:** delete the incoming value `[ %inext, %latch ]` from `%i` and `opt -passes=verify` must fail. A phi's incoming set is part of the CFG, not a comment.

---

## Example 3: Critical edge

**Goal:** Recognize an edge you cannot insert onto.

```llvm
define i32 @crit(i1 %c1, i1 %c2, i32 %x) {
entry:
  br i1 %c1, label %left, label %right

left:
  br i1 %c2, label %join, label %other

right:
  br label %join

join:
  %p = phi i32 [ %x, %left ], [ 0, %right ]
  ret i32 %p

other:
  ret i32 1
}
```

The edge `left -> join` is critical: `left` has two successors (`join` and `other`), and `join` has two predecessors (`left` and `right`).

The edge `right -> join` is **not** critical: `right` has only one successor. You may append instructions at the end of `right` (before its branch) and they execute only on that edge.

```bash
opt -passes=break-crit-edges -S crit.ll -o crit.split.ll
```

**Expected:** a new block, often named `left.join_crit_edge`, sitting between `left` and `join`. The phi in `join` now names that new block, not `left`.

**Counterexample:** a straight-line function with one block has no critical edges. `break-crit-edges` should leave it unchanged. An edge is critical only when the source has multiple successors **and** the destination has multiple predecessors. `entry -> left` in this example is not critical, because `left`'s only predecessor is `entry`.

---

## Example 4: `getelementptr` does not touch memory

**Goal:** Separate address arithmetic from loads.

```llvm
define i32 @idx(ptr %base, i64 %i, i64 %j) {
  %p = getelementptr inbounds i32, ptr %base, i64 %i
  %q = getelementptr inbounds i32, ptr %p, i64 %j
  %v = load i32, ptr %q, align 4
  ret i32 %v
}
```

```bash
opt -passes=instcombine -S gep.ll -o gep.opt.ll
```

**Expected:** one `getelementptr` whose index is `%i + %j`, then one load. InstCombine folded the address math and did not invent or delete a memory operation.

**Counterexample:**

```llvm
define i32 @vol(ptr %p) {
  %a = load volatile i32, ptr %p, align 4
  %b = load volatile i32, ptr %p, align 4
  %s = add i32 %a, %b
  ret i32 %s
}
```

InstCombine must keep both loads. `volatile` is a side effect. The GEP fold in the previous example was legal only because those instructions were not loads.

---

## Example 5: The entry block is special

**Goal:** See why allocas live in the entry.

```c
int scoped(int n) {
  int s = 0;
  if (n) {
    int t = n * 3;
    s = t;
  }
  return s;
}
```

```bash
clang -S -emit-llvm -O0 -Xclang -disable-O0-optnone scoped.c -o scoped.ll
grep alloca scoped.ll
```

**Expected:** both `s` and `t` are `alloca`s in the entry block, even though `t` is lexically inside the `if`. Stack slots are created on function entry so that StackColoring and the backend see a fixed frame.

**Counterexample / illegal IR:** an `alloca` that is not in the entry and is not a dynamic `alloca` (variable size) is still *syntactically* allowed in some cases, but many passes (`mem2reg` in particular) refuse to promote allocas that are not in the entry. Put an alloca in a non-entry block and `opt -passes=mem2reg` leaves it as memory.

Dynamic alloca (`alloca i8, i64 %n`) is a different instruction shape: the size is not constant. It must not be promoted by mem2reg.

---

## Example 6: `unreachable` vs a dead block

```llvm
define i32 @noreturn_path(i1 %c) {
entry:
  br i1 %c, label %live, label %dead
live:
  ret i32 1
dead:
  unreachable
}
```

```bash
opt -passes=simplifycfg -S unr.ll
```

**Expected:** SimplifyCFG deletes `dead` and turns the branch into `ret i32 1` when it can prove the `dead` target is unreachable *and* the branch has no other purpose. If your version keeps a conditional branch to `unreachable`, look at the successor: that edge is still in the CFG until a cleanup pass folds it.

A block that is textually present but has no predecessors is different from `unreachable`. `unreachable` means "if control arrives here, the program has undefined behavior". A disconnected block never arrives. `removeUnreachableBlocks` deletes the disconnected kind. Both show up in real IR after other passes rewrite branches and forget to delete the old target.

---

## What to write down

For one function you compiled yourself, answer:

1. How many basic blocks?
2. Which edges are backedges (target dominates source — you will define this properly on Day 4; for now, "jumps upward to a header" is enough)?
3. Which phis have an incoming value from a backedge?
4. Which edges are critical?

That picture is the one GVN, LICM, and the vectorizer all start from.
