# Foundation 2 extra: Syntax examples

Companion to [Foundation 2](Foundation-02-0-Reading-IR.md). Deep dive: [syntax rules](Foundation-02-1-Extra-IR-Syntax-DeepDive.md).

## Diagram

```text
  well formed                         rejected

  %s = add i32 %a, %b                 %s = add i32 %a, %b
  ret i32 %s                          %t = add i32 %s, 1
                                      ret i32 %t
                                      ^ illegal if this add is
                                        after a terminator
```

Save snippets as `t.ll` and run `opt -passes=verify -S t.ll` (or just `opt -S t.ll`, which verifies too).

## Example 1: Name every part

```llvm
define i32 @mul(i32 %a, i32 %b) {
entry:
  %p = mul i32 %a, %b
  ret i32 %p
}
```

**Read it.** Global function `@mul`. Locals `%a`, `%b`, `%p`. Block `entry`. Opcode `mul`. Terminator `ret`.

**Check.** `opt -S t.ll` prints the function back. No error.

## Example 2: The skipped number

```llvm
define i32 @add(i32 %0, i32 %1) {
  %3 = add i32 %0, %1
  ret i32 %3
}
```

**Read it.** `%2` is the unnamed entry block, not a missing instruction.

**Variant.** Name the block `entry:` and reassemble. The add's number can change. Behavior does not.

## Example 3: A comment is not an instruction

```llvm
define i32 @id(i32 %a) {
  ; %a = add i32 %a, 1
  ret i32 %a
}
```

**Read it.** The function returns its argument. The commented line is not a second definition of `%a`.

## Example 4: Two definitions of one name

```llvm
define i32 @bad(i32 %a) {
  %a = add i32 %a, 1
  ret i32 %a
}
```

**What happens.** The parser or the verifier rejects this. `%a` already names the argument. SSA allows one definition (Foundation 5). The legal spelling is `%b = add i32 %a, 1`.

## Example 5: A type mismatch

```llvm
define i64 @bad(i32 %a, i64 %b) {
  %s = add i32 %a, %b
  ret i64 %s
}
```

**What happens.** `%b` is `i64` and the `add` demands `i32`. This does not silently truncate. Truncation is a separate instruction, `trunc` (Foundation 4).

## Example 6: `br` targets

```llvm
define i32 @f(i1 %c) {
entry:
  br i1 %c, label %yes, label %no
yes:
  ret i32 1
no:
  ret i32 0
}
```

**Read it.** Two successors. Each label exists. `%c` must be `i1`, not `i32`. A C `int` used as a condition becomes an `icmp` against zero in the IR; the `br` itself only takes `i1`.
