# Foundation 5 extra: SSA examples

Companion to [Foundation 5](Foundation-05-0-SSA.md). Deep dive: [defs and uses](Foundation-05-1-Extra-SSA-DeepDive.md).

## Diagram

```text
        entry
        /    \
     then    else
        \    /
        join
         |
        %y = phi ...
```

## Example 1: The diamond, written out

```llvm
define i32 @pick(i1 %c) {
entry:
  br i1 %c, label %then, label %else
then:
  br label %join
else:
  br label %join
join:
  %y = phi i32 [ 1, %then ], [ 2, %else ]
  ret i32 %y
}
```

**Check.** `opt -S pick.ll` accepts it. When `%c` is true the result is 1.

**Counterexample.** Drop the `[ 2, %else ]` pair. The verifier reports that the phi does not match the predecessors.

## Example 2: No phi, because one definition dominates

```llvm
define i32 @both(i1 %c, i32 %x) {
entry:
  %y = add i32 %x, 1
  br i1 %c, label %then, label %else
then:
  ret i32 %y
else:
  ret i32 %y
}
```

**Read it.** `%y` is computed before the branch. Both returns see the same definition. A phi would be redundant.

## Example 3: A loop-carried value

```llvm
define i32 @count(i32 %n) {
entry:
  br label %header
header:
  %i = phi i32 [ 0, %entry ], [ %next, %header ]
  %next = add i32 %i, 1
  %c = icmp slt i32 %i, %n
  br i1 %c, label %header, label %exit
exit:
  %i.exit = phi i32 [ %i, %header ]
  ret i32 %i.exit
}
```

**Read it.** `%i` is 0 from `entry`, then `%next` from the backedge. The header is its own predecessor. The exit phi has one incoming value. That one-input phi is the shape LCSSA uses (Day 22): the value that leaves the loop has a name at the exit.

**Counterexample.** `ret i32 %next` in `exit` is illegal. `%next` is defined in `header`, and the exit's use is fine only if `header` dominates `exit`. Here it does, if the only entry to `exit` is from `header`. Using `%next` in `exit` is actually dominated. Prefer reasoning "does every path to this use pass the definition?" For `%next`, the path `entry -> header -> exit` does pass the add. So `ret i32 %next` in `exit` verifies. The interesting illegal case is Example 4.

## Example 4: A use that skips the definition

```llvm
define i32 @bad(i1 %c) {
entry:
  br i1 %c, label %t, label %join
t:
  %a = add i32 1, 1
  br label %join
join:
  ret i32 %a
}
```

**What happens.** `opt` rejects the function: `%a` does not dominate its use. One path never executes the add.

**The repair.**

```llvm
join:
  %v = phi i32 [ %a, %t ], [ 0, %entry ]
  ret i32 %v
```

The path that skipped the add now has an explicit value.

## Example 5: Memory is not a second definition of a name

```llvm
define i32 @mem(i32 %x) {
  %p = alloca i32
  store i32 %x, ptr %p
  store i32 0, ptr %p
  %v = load i32, ptr %p
  ret i32 %v
}
```

**Read it.** This is legal SSA. `%x` is defined once. The two stores overwrite memory, not `%x`. `%v` is a new name. After mem2reg the stores disappear and the returned value is the constant 0, because the second store overwrote the slot.
