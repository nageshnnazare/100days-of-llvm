# Day 40 Extra: TTI — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='print<cost-model>' -disable-output f.ll
```

Parent: [Day-40-0-TTI.md](Day-40-0-TTI.md). Where possible, compare with a target-aware clang invocation.

---

## Diagram

```text
   LoopVectorize
        |
        |  "what does this target want?"
        v
   +------------------+
   | TargetTransform  |---- X86TTI    (AVX widths, costs)
   | Info             |---- AArch64TTI
   +------------------+---- RISCVTTI
        |
        +-- getRegisterBitWidth(Fixed)
        +-- getArithmeticInstrCost(Add, v4i32)
        +-- getNumberOfRegisters
        +-- isLegalMaskedLoad
```

TTI is the question interface between target-independent passes and the backend. The vectorizer does not include `X86ISelLowering.h`. It asks TTI how wide a vector register is and what an add costs. Each target subclasses `BasicTTIImpl` and overrides the answers. A cost of 1 means "as cheap as a typical instruction," not "one cycle."

## Example 1: Cost model printer on scalar IR

### Input

```c
int add(int a, int b) { return a + b; }
```

### Commands

`print<cost-model>`.

### Expected result

Each instruction prints with a cost under the active TTI (often 1 on the generic model).

### Why it fires

CostModelPrinterPass queries `getInstructionCost` per instruction.

### Counterexample / contrast

On a build with X86 TM and AVX features enabled for a vectorized IR dump from clang -O3, the same printer shows non-uniform costs for vector adds vs shuffles—not all ones.

---

## Example 2: Unroll preferences differ by target (conceptual)

### Input

```c
void soften(int *a, int n) {
  for (int i = 0; i < n; ++i)
    a[i] = a[i] * 3 + 1;
}
```

### Commands

```bash
clang -O3 -Rpass=loop-unroll -Rpass-missed=loop-unroll -c soften.c -o /dev/null
# vs a different -mcpu / -Os
clang -Os -Rpass=loop-unroll -Rpass-missed=loop-unroll -c soften.c -o /dev/null
```

### Expected result

Different unroll decisions / remarks: `-Os` weighs `TCK_CodeSize` / size-sensitive preferences from `getUnrollingPreferences`.

### Why it fires

TTI unrolling preferences + optimization level.

### Counterexample

`#pragma clang loop unroll(disable)` / `llvm.loop.unroll.disable` metadata forces no unroll regardless of TTI enthusiasm.

---

## Example 3: Masked load legality

### Input

```c
void masked(int *a, int *b, int *m, int n) {
  for (int i = 0; i < n; ++i)
    if (m[i])
      a[i] = b[i] + 1;
}
```

### Commands

```bash
clang -O3 -Rpass=loop-vectorize -Rpass-missed=loop-vectorize -c masked.c
# Compare -march that has masked loads vs one that does not (target-dependent)
```

### Expected result

On targets where `isLegalMaskedLoad` is true for the type, LV may vectorize with masked loads. Elsewhere it may scalarize, use predication differently, or miss.

### Why it fires / not

TTI legality gate—not LAA alone.

### Counterexample

Unconditional loop (`a[i]=b[i]+1`) does not need masked loads—vectorizes under ordinary load legality.

---

## Example 4: Bare `opt` vs clang -O3 vectorize

### Input

```c
int sum(int *restrict a, int n) {
  int s = 0;
  for (int i = 0; i < n; ++i) s += a[i];
  return s;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sum.c -o sum.ll
opt -passes='loop-vectorize' -S sum.ll -o sum.opt.ll
clang -O3 -S -emit-llvm sum.c -o sum.clang.ll
```

### Expected result

`sum.clang.ll` more often contains vector IR / reduced loop; `sum.opt.ll` may remain scalar or choose differently when generic TTI + missing cpu attrs undervalue vector speedups.

### Why the gap

clang supplies TargetMachine/TTI/cpu features; bare opt often does not.

### Counterexample

```bash
# When your opt build accepts target flags matching clang:
opt -mtriple=x86_64-unknown-linux-gnu -mcpu=haswell -passes='loop-vectorize' -S sum.ll
```

Decisions move closer to clang (exact flags depend on your LLVM build).

---

## Example 5 (bonus): Code size cost kind

### Input

A large switch or a loop that could unroll a lot.

### Commands

Compare `-O3` vs `-Oz` remarks for unroll/vectorize.

### Expected result

`-Oz` uses size-oriented TTI costs / preferences → less aggressive unroll/vectorize.

### Why it fires

`TCK_CodeSize` / size preferences in TTI and pass configuration.

### Counterexample

`-O3` throughput-oriented costs favor wider vector factors and more unrolling.

---

## Table

| Experiment | Lesson |
|------------|--------|
| print\<cost-model\> | See TTI numbers |
| -O3 vs -Os unroll | Preferences differ |
| Predicated loop | `isLegalMaskedLoad` matters |
| opt vs clang -O3 | Generic TTI gap |
| -Oz | CodeSize costs win |

## TLI vs TTI one-liner

Day 39 asks “is memcpy a thing here?” Day 40 asks “how much does a `<8 x i32>` add cost on Haswell?”
