# Foundation 7 extra: Module examples

Companion to [Foundation 7](Foundation-07-0-Modules-and-Globals.md). Deep dive: [the Module class](Foundation-07-1-Extra-Module-DeepDive.md).

## Diagram

```text
  @limit = constant i32 4

  define i32 @get() {
    %v = load i32, ptr @limit
    ret i32 %v
  }

  after instcombine, often:

  define i32 @get() {
    ret i32 4
  }
```

## Example 1: Fold a constant global

Use the diagram as `t.ll`. Include a triple so the file is explicit:

```llvm
target triple = "x86_64-unknown-linux-gnu"

@limit = constant i32 4

define i32 @get() {
  %v = load i32, ptr @limit
  ret i32 %v
}
```

```bash
opt -passes=instcombine -S t.ll
```

**What you should see.** `ret i32 4`, or the load already gone.

## Example 2: A mutable external global does not fold

```llvm
@limit = global i32 4

define i32 @get() {
  %v = load i32, ptr @limit
  ret i32 %v
}
```

**What you should see.** The load remains. Another module could store to `@limit` before the call.

**The variant that does fold.** `@limit = internal global i32 4` and no store anywhere in the module. GlobalOpt (Day 47) may then treat it as a constant. InstCombine alone might still leave the load; if so, run `opt -passes=globalopt -S`.

## Example 3: `declare` versus a missing name

```llvm
declare i32 @abs(i32)

define i32 @f(i32 %x) {
  %y = call i32 @abs(i32 %x)
  ret i32 %y
}
```

**What you should see.** `opt -S` accepts this. There is no body.

**Counterexample.** Delete the `declare` line but keep the call. The parser reports an undefined reference. A call operand must name a function the module knows about.

## Example 4: A private string

```llvm
@msg = private unnamed_addr constant [6 x i8] c"hello\00"

declare i32 @puts(ptr)

define i32 @hello() {
  %r = call i32 @puts(ptr @msg)
  ret i32 %r
}
```

**Read it.** `@msg` has six bytes. `@puts` has no body. The call's argument type is `ptr`.

**Length mismatch.** `[5 x i8] c"hello\00"` is rejected: the initializer is 6 bytes and the type says 5.

## Example 5: Two functions, one global

```llvm
@g = internal global i32 0

define void @set(i32 %v) {
  store i32 %v, ptr @g
  ret void
}

define i32 @get() {
  %v = load i32, ptr @g
  ret i32 %v
}
```

**Read it.** `internal` means other modules cannot name `@g`. The store in `@set` is why `@get`'s load cannot be folded to 0. If you delete `@set` and nothing else stores, Day 47's global optimization has something to prove.
