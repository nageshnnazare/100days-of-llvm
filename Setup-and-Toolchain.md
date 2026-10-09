# Setup and Toolchain

Every later day assumes you can dump LLVM IR, run one pass, and read the result. Do this once before [Foundation 6](Foundation-06-0-C-to-IR.md). Foundations 1–5 explain what the dumped file means.

The study notes point at the upstream tree [llvm/llvm-project](https://github.com/llvm/llvm-project). You do not need a full from-source build to start. A packaged `clang`, `opt`, and `llc` is enough until Day 93 (out-of-tree passes) and Day 94 (TableGen).

---

## 1. Check what you already have

```bash
clang --version
opt --version
llc --version
```

The three version lines should be the same major version. Mixing a Homebrew `clang` with an `opt` from a different LLVM install produces confusing IR (`ptr` vs typed pointers, different pass names, different intrinsics).

If a tool is missing:

| Platform | Install |
| --- | --- |
| macOS (Homebrew) | `brew install llvm` then put `$(brew --prefix llvm)/bin` on `PATH` |
| Debian / Ubuntu | `sudo apt install clang llvm` |
| Fedora | `sudo dnf install clang llvm` |
| From source | See [Getting Started](https://llvm.org/docs/GettingStarted.html). A useful minimum is `-DLLVM_ENABLE_PROJECTS="clang;lld" -DCMAKE_BUILD_TYPE=Release` and targets `clang opt llc llvm-dis FileCheck`. |

Confirm opaque pointers are the default (LLVM 15 and later). This IR is what the notes use:

```llvm
define i32 @add(i32 %a, i32 %b) {
  %s = add i32 %a, %b
  ret i32 %s
}
```

Pointers are written `ptr`, not `i32*`.

---

## 2. The `optnone` trap

`clang -O0` stamps `optnone` on every function. `opt` then **skips** that function, so a pass looks like it did nothing.

Produce unoptimized IR that passes are still allowed to change:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes \
  file.c -o file.ll
```

What that combination means:

- `-O1` selects the optimization *attribute* set (no `optnone`, frame pointer and inline decisions closer to a real -O1 compile).
- `-Xclang -disable-llvm-passes` tells Clang not to run the LLVM pipeline, so the IR is still in alloca / load / store form.
- `-S -emit-llvm` writes textual IR instead of an object file.

When you specifically want the raw `-O0` allocas **and** you plan to run `opt` yourself, strip the attribute:

```bash
clang -S -emit-llvm -O0 -Xclang -disable-O0-optnone file.c -o file.ll
```

Inspect attributes before you blame the pass:

```bash
grep -n "optnone\|noinline" file.ll
```

---

## 3. Run one pass

```bash
opt -passes=instcombine -S file.ll -o file.instcombine.ll
```

`-S` prints textual IR. Without it, `opt` writes bitcode.

Useful variants:

```bash
# The whole -O2 pipeline, as a pass list.
opt -passes='default<O2>' -S file.ll

# Print the pipeline without running it.
opt -passes='default<O2>' -print-pipeline-passes < /dev/null

# Trace which pass runs and which analysis it pulls in.
opt -passes=instcombine -debug-pass-manager -disable-output file.ll

# Show only the IR of functions a pass actually changed.
opt -passes=sroa -S -print-changed=quiet file.ll
```

`-debug-only=sroa` (and similar) needs an LLVM built with assertions (`-DLLVM_ENABLE_ASSERTIONS=ON`). Release packages often omit it. `-print-changed` and `-debug-pass-manager` work on release builds.

---

## 4. Diff IR without drowning

```bash
opt -passes=mem2reg -S file.ll -o after.ll
diff -u file.ll after.ll
```

For a full Clang pipeline, `-mllvm -print-changed` is noisy. Prefer one pass at a time until you know what you are looking at.

Optimization remarks name the pass that fired:

```bash
clang -O2 -Rpass=inline -Rpass-missed=inline -Rpass-analysis=loop-vectorize \
  file.c -c -o /dev/null
```

---

## 5. Codegen, not just IR

```bash
llc -mtriple=x86_64-unknown-linux-gnu -mcpu=skylake file.ll -o file.s
llc -mtriple=aarch64-unknown-linux-gnu file.ll -o file.aarch64.s

# Stop after instruction selection and print Machine IR.
llc -mtriple=x86_64-unknown-linux-gnu -stop-after=finalize-isel \
  -simplify-mir file.ll
```

---

## 6. How a day is laid out

Each day from Day 3 onward is three files:

| File | Role |
| --- | --- |
| `Day-NN-0-....md` | The lesson: idea, pipeline position, algorithm, where to read, one worked example |
| `Day-NN-1-Extra-....md` | A walk through the C++ that implements it |
| `Day-NN-2-Extra-....md` | Several examples, including a case the transform must refuse |

Days 0–2 use the same idea with slightly different suffixes. The glossary is the short version of the same passes.

Suggested rhythm:

1. Read the main file once, without the compiler.
2. Open the two or three source files it names in [llvm-project](https://github.com/llvm/llvm-project).
3. Paste an example into `/tmp`, run the commands, and write down one thing that surprised you.
4. Read the deep dive only after you have seen the pass change IR.

---

## 7. Tests you will meet on Day 91

Upstream regression tests live in `llvm/test`. A typical FileCheck test is an `.ll` file whose first lines are:

```llvm
; RUN: opt -passes=instcombine -S < %s | FileCheck %s
```

You can run one test, once you have a build, with `llvm-lit -v path/to/test.ll`. Day 91 covers how to write these.
