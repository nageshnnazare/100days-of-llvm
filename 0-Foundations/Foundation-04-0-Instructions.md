# Foundation 4: The instructions you will see every day

LangRef lists dozens of instructions. A typical function from Clang uses about fifteen of them. This is that set, grouped the way you should recognize them. Predicates, flags, and the long tail (`callbr`, `indirectbr`, `va_arg`) can wait.

Deep dive: [the opcode list in the source](Foundation-04-1-Extra-Instructions-DeepDive.md). Examples: [instruction examples](Foundation-04-2-Extra-Instructions-Examples.md).

## Diagram

```text
  a basic block is a list that ends in exactly one terminator

  arithmetic and compare          memory                    the rest
  +----------------------+        +----------------+        +------------------+
  | add sub mul          |        | alloca         |        | phi              |
  | udiv sdiv urem srem  |        | load           |        | select           |
  | shl lshr ashr        |        | store          |        | call             |
  | and or xor           |        | getelementptr  |        | zext sext trunc  |
  | icmp  fadd fcmp      |        +----------------+        | freeze           |
  +----------------------+                                  +------------------+
            \                        |                         /
             \                       |                        /
              +----------------------+-----------------------+
                                     |
                              terminator
                         ret   br   switch   unreachable
```

If you can put an instruction into one of those four boxes, you can read the function. Optimization passes delete, move, or rewrite instructions. They do not invent a fifth kind of program.

## Terminators

| Instruction | Meaning |
| --- | --- |
| `ret i32 %v` | Return `%v` to the caller. `ret void` returns nothing. |
| `br label %b` | Unconditional jump. |
| `br i1 %c, label %t, label %f` | If `%c` then `%t`, else `%f`. |
| `switch i32 %v, label %default [i32 1, label %one ...]` | Multi-way branch. |
| `unreachable` | Executing this is undefined behavior. Used after a call that never returns (`exit`, a noreturn function). |

A block has one terminator, and it is the last instruction.

## Integer arithmetic

`add`, `sub`, `mul` are neither signed nor unsigned. `udiv`, `sdiv`, `urem`, `srem` are. Shifts: `shl` shifts left, `lshr` shifts in zeros, `ashr` shifts in copies of the sign bit.

```llvm
%q = sdiv i32 %a, %b
%r = ashr i32 %a, 3
```

Flags `nsw` and `nuw` on `add` mean "signed overflow is poison" and "unsigned overflow is poison". They are not part of the opcode. Day 101. On a first reading, treat `add nsw` as `add` plus a promise.

Division by zero is undefined behavior for `udiv` and `sdiv`. The IR does not trap by itself.

## Compares

```llvm
%c = icmp slt i32 %a, %b    ; signed less-than, result i1
%d = icmp ult i32 %a, %b    ; unsigned less-than
%e = fcmp olt float %x, %y  ; ordered floating less-than
```

`icmp` predicates: `eq`, `ne`, `ugt`, `uge`, `ult`, `ule`, `sgt`, `sge`, `slt`, `sle`. The `u` and `s` prefixes are the signedness. `fcmp` has ordered and unordered forms because NaN compares are not true or false in the integer sense; `olt` is "ordered and less than".

## Memory

```llvm
%p = alloca i32
store i32 %v, ptr %p
%w = load i32, ptr %p
%q = getelementptr i32, ptr %base, i64 %i
```

- `alloca` reserves memory in the function's stack frame and yields a `ptr`. The type argument is what is allocated, not the type of `%p`.
- `store` writes a value. It does not define a `%` result.
- `load` reads a value and defines a result.
- `getelementptr` computes an address. It does not load. The first type (`i32` here) is the element type used for scaling: index `%i` means "byte offset `%i` times the size of `i32`". Day 102 is the full rule, including structs and `inbounds`.

`alloca` is how Clang represents a local variable before mem2reg (Day 5). It is not a heap allocation. Heap allocation is a call to `@malloc`.

## Conversions, select, call, phi, freeze

```llvm
%z = zext i1 %c to i32
%y = select i1 %c, i32 %a, i32 %b
%r = call i32 @f(i32 %a)
%x = phi i32 [ %a, %left ], [ %b, %right ]
%n = freeze i32 %m
```

- `zext` zero-extends, `sext` sign-extends, `trunc` narrows.
- `select` chooses between two values without a branch. Both options are evaluated in the sense that the instruction names both values; it does not skip an instruction in another block.
- `call` transfers to a function. The result type matches the callee's return type.
- `phi` is Foundation 5. It is "the value depends on which predecessor ran".
- `freeze` turns poison or undef into one concrete bit pattern (Day 101). You will see it appear after optimizations, not usually in raw Clang IR.

## Hands-on

Print IR for a function that contains an `if` and a local `int`. Mark each instruction with one of the four groups in the diagram. Confirm the last instruction of every block is a terminator.

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
```

## Pitfalls

- Reading `getelementptr` as a load. Nothing is read from memory until `load`.
- Reading `select` as a branch. It does not start a new block.
- Ignoring `nsw`. Later it explains a deleted overflow check. For reading the dataflow, the result is still the mathematical sum when there is no overflow.

## Check yourself

1. Which memory instruction defines a new value, `load` or `store`?
2. What is the type of `icmp eq i32 %a, %b`?
3. Does `getelementptr i32, ptr %p, i64 1` load `*%p`?

**Answers.** (1) `load`. `store` has no result name. (2) `i1`. (3) No. It is the address of the next `i32`.
