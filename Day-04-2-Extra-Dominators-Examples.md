# Day 4 Extra: Dominator Examples

Use [Day-04-0-Dominator-Trees.md](Day-04-0-Dominator-Trees.md) for the definitions. These examples are graphs you should be able to label from memory.

```bash
opt -passes='print<domtree>' -disable-output ex.ll
opt -passes='print<postdomtree>' -disable-output ex.ll
```

---

## Diagram

```text
   CFG                         dominator tree

        A                         A
       / \                        |
      B   C                       +-- B
      |   |                       |   |
      D   |                       |   +-- D
       \ /                        |
        E                         +-- C
                                  |
                                  +-- E

   A dominates everyone.
   B dominates D.
   E's dominators are only A and E:
   neither B nor C is on every path to E.

   dominance frontier of D is {E}:
   D dominates a predecessor of E,
   and D does not strictly dominate E.
   Phis for a definition in D go at E.
```

A block A dominates B when every path from the entry to B passes through A. The dominator tree stores the immediate dominator of each block, and the DFS in/out numbers answer "does A dominate B?" in constant time. The dominance frontier of a definition is exactly the set of blocks that need a phi for it. That is the set mem2reg uses.

## Example 1: Straight line

```llvm
define i32 @line(i32 %x) {
entry:
  %a = add i32 %x, 1
  br label %mid
mid:
  %b = mul i32 %a, 2
  br label %end
end:
  ret i32 %b
}
```

**Tree:** `entry` dominates `mid` dominates `end`. Immediate dominators: `idom(mid)=entry`, `idom(end)=mid`.

**Frontiers:** all empty. No block has two predecessors, so dominance never "stops" at a join. Mem2Reg inserts no phis for a variable defined and used along this line.

**Post-dom:** `end` post-dominates `mid` post-dominates `entry`.

**Counterexample:** add a second return in `mid` (`ret i32 0`) and delete the branch to `end`. Then `end` is unreachable, `getNode(end)` is null, and `end` is not in the tree. Do not iterate all blocks and assume `DT.dominates(entry, BB)` is true for each.

---

## Example 2: Diamond frontier

```llvm
define i32 @diamond(i1 %c, i32 %x) {
entry:
  br i1 %c, label %L, label %R
L:
  %a = add i32 %x, 1
  br label %J
R:
  br label %J
J:
  %p = phi i32 [ %a, %L ], [ %x, %R ]
  ret i32 %p
}
```

| Block | Dominated by | idom | DF |
| --- | --- | --- | --- |
| entry | entry | — | empty |
| L | entry, L | entry | {J} |
| R | entry, R | entry | {J} |
| J | entry, J | entry | empty |

**Check:** `%a` is used only as the incoming value for predecessor `L`. Legal. Move `%a` into the `ret` (`ret i32 %a`) and verification fails, because `L` does not dominate `J`.

```bash
opt -passes=verify -disable-output diamond.ll
```

---

## Example 3: Loop header is the frontier of the body

```llvm
define i32 @loop(i32 %n) {
entry:
  br label %H
H:
  %i = phi i32 [ 0, %entry ], [ %i2, %B ]
  %c = icmp slt i32 %i, %n
  br i1 %c, label %B, label %E
B:
  %i2 = add i32 %i, 1
  br label %H
E:
  ret i32 %i
}
```

**Backedge:** `B -> H`, and `H` dominates `B`. That is the definition of a backedge used by `LoopInfo`.

**idom:** `idom(H)=entry`, `idom(B)=H`, `idom(E)=H`.

**Why the phi is in `H`:** `B` defines `%i2` and `B` does not dominate `H`. The iterated dominance frontier of `{B}` includes `H`, because `H` is a predecessor-join of a block `B` dominates (the predecessor is `B` itself) and `B` does not strictly dominate `H`.

**Post-dom twist:** `E` post-dominates `H` only if every path from `H` reaches `E`. If `%n` is such that the compare is always true, that is a *dynamic* fact. The post-dom tree is static: the branch *might* go to `E`, so `E` does not post-dominate `B`. Confirm with `print<postdomtree>`. `B` does not post-dominate `H`.

**Counterexample:** an irreducible loop (two headers that jump to each other, neither dominating the other) still has a dominator tree, but `LoopInfo` will not treat it as a single natural loop. Day 21's LoopSimplify exists because natural-loop passes refuse this shape. You can build one with computed gotos; you will not get it from ordinary C `for` loops.

---

## Example 4: Nested conditionals, iterated frontier

```c
int f(int x) {
  int y = 0;
  if (x > 0) {
    if (x > 10)
      y = 1;
    else
      y = 2;
  }
  return y;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes nest.c -o nest.ll
opt -passes=mem2reg -S nest.ll -o nest.m2r.ll
```

**What to look for:** two phis, not one.

- The inner join needs a phi merging `1` and `2`.
- The outer join needs a phi merging "that inner phi" and the incoming `0` from the entry path.

The inner phi is itself a definition, so it contributes to the iterated frontier. A single phi at the function's last block would be wrong when the inner join has a use that is still inside the outer `if` — and it would also be wrong as a general algorithm even if in this particular function you could theoretically sink everything. Mem2Reg places phis at the frontier, then a later SimplifyCFG may fold them into selects.

**Counterexample:** if `y` is stored only on paths that the function never joins (both arms `return`), there is no phi at all. Dominance frontiers are about joins that exist in the CFG, not about lexical nesting in C.

---

## Example 5: Same-block dominance is not the tree

```llvm
define i32 @order(i32 %x) {
entry:
  %b = add i32 %a, 1
  %a = add i32 %x, 1
  ret i32 %b
}
```

```bash
opt -passes=verify -disable-output order.ll
```

**Expected:** verifier failure. `entry` dominates itself, and the dominator tree has one node. The instruction `%a` still does not dominate `%b`, because it appears later in the block. Tree dominance is necessary but not sufficient inside a block.

**Counterexample that is legal:** swap the two adds. Verification passes. The tree did not change; the instruction order did.

---

## Example 6: Critical edge split changes the tree

Start from Example 2 and split the critical edge if you add a second successor to `L` (see Day 3 Example 3). After `break-crit-edges`:

- The new edge block's immediate dominator is `L` (it has one predecessor).
- `J`'s predecessors change, so any pass holding a stale `DominatorTree` now has the wrong children lists.
- Phis in `J` name the new block.

```bash
opt -passes=break-crit-edges -S crit.ll -o crit2.ll
opt -passes='print<domtree>' -disable-output crit2.ll
```

Compare the two dumps side by side. The new block should be a child of its single predecessor.

---

## A 10-minute drill

For a function you care about, print the dominator tree and answer:

1. Which blocks does the header of the hottest loop dominate?
2. Which blocks are in its dominance frontier? Those are the only places a value defined inside the loop can be observed outside without an LCSSA phi (Day 22).
3. Does the preheader dominate the header? After LoopSimplify (Day 21) it should, and it should have the header as its only successor.

If you cannot answer (2) from the dump, re-read the frontier definition with Example 2 in front of you. That definition is the whole of Day 5's phi placement.
