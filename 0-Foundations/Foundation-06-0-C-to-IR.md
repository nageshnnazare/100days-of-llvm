# Foundation 6: A C function, as Clang first emits it

Optimized IR is a bad place to learn the mapping from C, because the mapping has already been optimized away. This note is the unoptimized shape: allocas, stores, loads, and branches. Day 5 and Day 6 exist to delete that shape. Day 92 is how Clang's CodeGen walks the AST. Here you only learn to recognize the result.

Deep dive: [what Clang is doing, one statement at a time](Foundation-06-1-Extra-Clang-IR-DeepDive.md). Examples: [C-to-IR examples](Foundation-06-2-Extra-C-to-IR-Examples.md).

## Diagram

```text
  int f(int c, int x) {
    int y;                 alloca i32, once, in the entry block
    if (c)                 icmp ne i32 %c, 0
      y = x;               store in the true block
    else
      y = 0;               store in the false block
    return y;              load, then ret
  }

  entry
    %y = alloca i32
    %cond = icmp ne i32 %c, 0
    br i1 %cond, label %then, label %else
  then:
    store i32 %x, ptr %y
    br label %join
  else:
    store i32 0, ptr %y
    br label %join
  join:
    %v = load i32, ptr %y
    ret i32 %v
```

The C name `y` became an address. Assignments became stores. The return became a load of that address. There is no phi yet. Mem2reg will turn the two stores and the load into the phi from Foundation 5.

## The command that produces this shape

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
```

You get Clang's CodeGen without the optimizer. The function should not be marked `optnone`. If it is, you used plain `-O0` and forgot `-disable-O0-optnone` (see [Setup and toolchain](Setup-and-Toolchain.md)).

## Locals

Every local that Clang cannot keep in a value becomes `alloca` in the **entry** block, even if the C declaration is inside a loop. The allocation happens once per call, not once per iteration. The loop body stores into that same pointer.

A local that is a struct or an array is one `alloca` of that aggregate type. SROA (Day 6) splits it. A local whose address is passed to another function often stays an `alloca`, because mem2reg will not promote an address that escapes.

## Conditions

C treats `0` as false and anything else as true. IR `br` wants `i1`. Clang inserts the compare:

```llvm
%cond = icmp ne i32 %c, 0
br i1 %cond, label %then, label %else
```

A C `&&` or `||` is usually two branches, not a single `and` of `i1`s, because the right-hand side must not run when the left-hand side decides. That is short-circuit evaluation, visible as blocks.

## Loops

```c
int sum(int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += i;
  return s;
}
```

The IR is a header block that loads `i`, compares it to `n`, and either goes to a body or to an exit. The body loads `s` and `i`, adds, stores both updates, and branches back. It looks long. It is the same loop. After mem2reg the loads and stores of `s` and `i` become phis, and that smaller form is what Day 21 and Day 23 expect.

## Calls and returns

```c
int g(int);
int h(int x) { return g(x) + 1; }
```

```llvm
%r = call i32 @g(i32 %x)
%s = add i32 %r, 1
ret i32 %s
```

A `declare i32 @g(i32)` line appears if `g`'s body is not in this file. `declare` means "someone else defines this". `define` means the body is here.

## Hands-on

Compile the `f` function from the diagram with the command above. Match each C line to one IR instruction. Then run:

```bash
opt -passes=mem2reg -S f.ll -o f.mem2reg.ll
```

The `alloca` should be gone, and `join` should contain a phi. That pair of files is the before and after of Day 5. Keep them.

## Pitfalls

- Reading an `alloca` as `malloc`. It is the stack slot for a local.
- Expecting one IR block per C line. An `if` is at least three blocks.
- Studying `clang -O2 -emit-llvm` while learning this mapping. `-O2` has already run mem2reg, inlining, and more. You will not see the stores.

## Check yourself

1. Why is the `alloca` in the entry block even when the C variable is declared inside the loop?
2. What instruction does Clang insert so that an `int` can control a `br`?
3. What does `declare` mean when there is no function body?

**Answers.** (1) Stack allocation is per call, not per iteration. The loop stores a new value into the one slot. (2) `icmp` against zero, producing `i1`. (3) The function is defined in another module. This file may call it and must not invent a body.
