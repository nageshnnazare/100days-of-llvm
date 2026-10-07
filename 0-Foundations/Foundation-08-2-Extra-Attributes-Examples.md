# Foundation 8 extra: Attribute examples

Companion to [Foundation 8](Foundation-08-0-Attributes.md). Deep dive: [attribute classes](Foundation-08-1-Extra-Attributes-DeepDive.md).

## Diagram

```text
  without readonly                         with readonly

  define i32 @f(ptr %p) {                  define i32 @f(ptr readonly %p) {
    %a = load i32, ptr %p                    %a = load i32, ptr %p
    call void @g()                           call void @g()
    %b = load i32, ptr %p                    ; a pass may reuse %a for %b
    ret i32 %b                               ; only if @g cannot write %p
  }                                          }
```

The attribute is what makes the second load redundant. Without it, `@g` might store through a global that aliases `%p`.

## Example 1: See `noundef` and `#0`

```c
int id(int x) { return x; }
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes id.c -o id.ll
```

**What you should see.** `noundef` on the argument, and an `attributes #0` line. You should not see `optnone`.

## Example 2: `optnone` blocks the pass

```llvm
define i32 @f(i32 %x) optnone noinline {
  %y = add i32 %x, 0
  ret i32 %y
}
```

```bash
opt -passes=instcombine -S t.ll
```

**What you should see.** The `add` of zero remains. InstCombine would normally delete it.

**The repair.** Remove `optnone noinline` and run the command again. The function should `ret i32 %x`.

## Example 3: `noreturn` and `unreachable`

```llvm
declare void @exit(i32) noreturn

define void @die() {
  call void @exit(i32 1)
  unreachable
}
```

**Read it.** `noreturn` on the declaration matches `unreachable` in the caller. If you delete `noreturn` and keep `unreachable`, you have told the optimizer that returning is undefined even if a future `@exit` returns. Keep them consistent.

## Example 4: `signext` is a widening, not a sign test

```llvm
define signext i8 @neg(i8 signext %x) {
  %y = sub i8 0, %x
  ret i8 %y
}
```

**Read it.** The parameter and the return are `i8` values that the ABI widened. The `sub` is still an 8-bit subtract. `signext` does not change the opcode.

**Counterexample of a misreading.** Treating `signext` as "branch if negative". The branch would be `icmp slt i8 %x, 0`.

## Example 5: `memory(none)` versus a store

```llvm
define void @bad(ptr %p) memory(none) {
  store i32 1, ptr %p
  ret void
}
```

**What happens.** The verifier rejects a function that claims `memory(none)` and contains a store. Attributes are checked, not merely trusted blindly, when the contradiction is local.

A `readonly` function that calls an unknown function is also suspicious: the verifier does not always see the callee's body. Do not add `readonly` by hand unless the body only loads.
