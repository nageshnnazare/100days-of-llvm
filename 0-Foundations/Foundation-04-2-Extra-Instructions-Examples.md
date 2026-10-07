# Foundation 4 extra: Instruction examples

Companion to [Foundation 4](Foundation-04-0-Instructions.md). Deep dive: [opcode list](Foundation-04-1-Extra-Instructions-DeepDive.md).

## Diagram

```text
  C:  if (a < b) return a; else return b;

  entry:
    %c = icmp slt i32 %a, %b     compare
    br i1 %c, label %t, label %f terminator
  t:
    ret i32 %a                   terminator
  f:
    ret i32 %b                   terminator
```

## Example 1: Min with a branch

The diagram is the whole example. `slt` is signed. Replacing it with `ult` changes the meaning for negative numbers. `opt -passes=instcombine` may turn this into `llvm.smin` or a `select`. Both are the same function.

## Example 2: A local, before mem2reg

```llvm
define i32 @inc(i32 %x) {
entry:
  %p = alloca i32
  store i32 %x, ptr %p
  %v = load i32, ptr %p
  %s = add i32 %v, 1
  ret i32 %s
}
```

**Group them.** `alloca`, `store`, `load` are memory. `add` is arithmetic. `ret` is the terminator.

**Why it looks redundant.** This is Clang's shape for a local. Day 5 deletes the `alloca`, the `store`, and the `load`, and uses `%x` directly.

## Example 3: A pointer walk

```llvm
define i32 @idx(ptr %base, i64 %i) {
  %q = getelementptr i32, ptr %base, i64 %i
  %v = load i32, ptr %q
  ret i32 %v
}
```

**Read it.** `%q` is an address. `%v` is the integer loaded from that address. Deleting the `load` and returning `%q` would be a type error (`ptr` versus `i32`) and a different program.

## Example 4: `select` is not `br`

```llvm
define i32 @sel(i1 %c, i32 %a, i32 %b) {
  %r = select i1 %c, i32 %a, i32 %b
  ret i32 %r
}
```

**Read it.** One block, no labels besides the entry. Both `%a` and `%b` are operands. Nothing is "not executed" inside this function, because there is nothing left to skip.

## Example 5: What `unreachable` is for

```llvm
declare void @exit(i32) noreturn

define void @die() {
  call void @exit(i32 1)
  unreachable
}
```

**Read it.** If `@exit` returns, execution is undefined. The terminator is `unreachable`, not `ret`, because a `ret` would mean the function continues.

**Counterexample.** Putting `unreachable` at the top of a normal function that does return is a lie, and later passes may delete the real code as dead.

## Example 6: Shift direction

```llvm
define i32 @shifts(i32 %x) {
  %a = shl  i32 %x, 1
  %b = lshr i32 %x, 1
  %c = ashr i32 %x, 1
  ret i32 %a
}
```

**Read it.** `%a` is `%x * 2` when the shift does not overflow the width. `%b` is a logical shift (zero fill). `%c` is an arithmetic shift (sign fill). `lshr` of a negative `i32` is a large positive number. `ashr` of a negative `i32` stays negative.
