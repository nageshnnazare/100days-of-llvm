# Foundation 5: SSA, in the form you need to read IR

Static single assignment means each value is defined once. The name `%s` is that definition. Uses of `%s` are the later instructions that name it. When two control-flow paths would have written the same C variable, IR does not write it twice. It adds a `phi` at the join.

Day 5 is the algorithm that inserts those phis. Day 4 is why they sit in particular blocks. This note is only how to read them.

Deep dive: [defs, uses, and the verifier](Foundation-05-1-Extra-SSA-DeepDive.md). Examples: [SSA examples](Foundation-05-2-Extra-SSA-Examples.md).

## Diagram

```text
  C                              SSA

  int y;
  if (c) y = 1;                  then:
  else   y = 2;                    br label %join
  return y;                      else:
                                   br label %join
                                 join:
                                   %y = phi i32 [ 1, %then ], [ 2, %else ]
                                   ret i32 %y

  one C variable, two writes     one phi, two incoming values
  the reads see "whatever        each incoming value is tied
  was written last"              to the predecessor it came from
```

Read a phi left to right: result name, type, then a list of `[value, predecessor]` pairs. The pair `[1, %then]` means "if we arrived from `then`, the result is 1".

## Straight-line code needs no phi

```llvm
define i32 @chain(i32 %a) {
  %b = add i32 %a, 1
  %c = mul i32 %b, 2
  ret i32 %c
}
```

`%a` is defined by the argument. `%b` is defined by the add. `%c` is defined by the mul. Each name appears on the left of `=` once. This is already SSA. Nothing about phis is required until two definitions can reach one use.

## Where a phi is allowed to look

A phi operand's value is defined on the path through that predecessor. The text of the phi sits at the start of the join block, but the value was computed before the jump.

In a loop the predecessor is the latch, which appears later in the file:

```llvm
define i32 @sum(i32 %n) {
entry:
  br label %header
header:
  %i = phi i32 [ 0, %entry ], [ %i.next, %latch ]
  %i.next = add i32 %i, 1
  %c = icmp slt i32 %i, %n
  br i1 %c, label %latch, label %exit
latch:
  br label %header
exit:
  ret i32 %i
}
```

`%i` on the first visit comes from `entry` and is 0. On later visits it comes from `latch` and is `%i.next`. `%i.next` is defined below the phi in the file. That forward reference is legal because the latch runs only after `%i.next` has been computed on the previous iteration. A non-phi instruction cannot use a value that is defined later in the block.

Phis are grouped at the top of the block, before ordinary instructions.

## What SSA is not

- It is not "no mutation of memory". A `store` still overwrites bytes. SSA applies to the values named with `%`, not to the contents of an `alloca`. That is why unoptimized Clang IR can look non-SSA at the C level (many stores to one slot) while still being SSA at the instruction level (each `load` is a new name).
- It is not "one name per source variable" after optimization. Mem2reg gives you one phi per join. Later passes invent more names (`%i.next`).
- It is not a machine register. `%i` may end up in a register, spilled, or folded away.

## How to answer "where does this value come from?"

1. Find the line that defines it: `%name = ...`, or the argument list.
2. If the opcode is `phi`, follow the predecessor you care about and repeat.
3. If the opcode is `load`, the value came from memory. The SSA name is the load's result, not the stored variable. Finding the store is dependence analysis (Day 33), not the phi rule.

## Hands-on

Write a three-block function on paper: entry branches to `left` or `right`, both branch to `join`, `join` returns a different constant from each side. Then write the phi. Check it with `opt -S`.

## Pitfalls

- Reading `[ %a, %left ]` as "the value `%left`". The second item is a block label. The first item is the value.
- Putting a phi in the middle of a block. The verifier rejects it.
- Forgetting a predecessor. If three blocks branch to `join`, the phi needs three pairs.

## Check yourself

1. How many times may `%s` be defined in one function?
2. In `%y = phi i32 [ 1, %then ], [ 2, %else ]`, what is `%y` when control came from `else`?
3. Why can a loop phi name a value that is printed later in the function?

**Answers.** (1) Once. (2) 2. (3) The value is produced on the previous iteration, before the branch back to the header. The phi's incoming pair names that predecessor.
