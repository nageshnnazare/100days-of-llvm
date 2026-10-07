# Foundation 3 extra: Type examples

Companion to [Foundation 3](Foundation-03-0-Types.md). Deep dive: [type representation](Foundation-03-1-Extra-Types-DeepDive.md).

## Diagram

```text
  C                        IR type of the value

  int x;                   i32
  char c;                  i8
  int *p;                  ptr          (the load is i32)
  int cond;                i1 after icmp, i32 if it is the variable itself
  float f;                 float
  int a[4];                [4 x i32] in an alloca, or four i32 values after SROA
```

## Example 1: Widths

```llvm
define i64 @widen(i32 %x) {
  %w = zext i32 %x to i64
  ret i64 %w
}
```

**Read it.** `%x` is 32 bits. `%w` is 64 bits. `zext` zero-extends. `sext` would sign-extend. The `add` instruction cannot do this; the widths must already match.

## Example 2: A condition is `i1`

```llvm
define i32 @is_neg(i32 %x) {
  %c = icmp slt i32 %x, 0
  br i1 %c, label %yes, label %no
yes:
  ret i32 1
no:
  ret i32 0
}
```

**Read it.** `icmp` produces `i1`. `br` consumes `i1`.

**Counterexample.** `br i32 %x, label %yes, label %no` is rejected. Compare against zero first.

## Example 3: Opaque pointer

```llvm
define i32 @load(ptr %p) {
  %v = load i32, ptr %p
  ret i32 %v
}
```

**Read it.** `%p` does not say "pointer to i32". The `load` says it reads an `i32`.

**Older spelling, do not write this.** `load i32, i32* %p`. If you see it in an old test, the address operand is `ptr` in current IR.

## Example 4: Array versus vector

```llvm
define <4 x i32> @vadd(<4 x i32> %a, <4 x i32> %b) {
  %c = add <4 x i32> %a, %b
  ret <4 x i32> %c
}
```

**Read it.** One instruction, four lanes.

**Counterexample.** `add [4 x i32] %a, %b` is not legal. Arrays are not arithmetic operands. A loop that adds array elements is several scalar `add`s, or a vector `add` after the vectorizer.

## Example 5: A named struct and a field load

```llvm
%struct.S = type { i32, i32 }

define i32 @field(ptr %p) {
  %q = getelementptr %struct.S, ptr %p, i32 0, i32 1
  %v = load i32, ptr %q
  ret i32 %v
}
```

**Read it.** `%struct.S` names a type. The `getelementptr` computes the address of the second field (index 1). The `load` reads an `i32`. Whether that field is at byte offset 4 is a DataLayout question (Day 102), and for `{i32, i32}` it is.

## Example 6: `void`

```llvm
declare void @g()

define void @f() {
  call void @g()
  ret void
}
```

**Read it.** The call's type is `void` because `@g` returns nothing. There is no `%t = call void @g()`.
