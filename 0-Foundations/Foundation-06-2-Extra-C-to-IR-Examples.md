# Foundation 6 extra: C-to-IR examples

Companion to [Foundation 6](Foundation-06-0-C-to-IR.md). Deep dive: [statement lowering](Foundation-06-1-Extra-Clang-IR-DeepDive.md).

Use this command for every example:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
```

## Diagram

```text
  C source line            you should be able to point at

  a local declaration  --> one alloca in the entry block
  an assignment        --> one store
  a use of a local     --> one load
  an if                --> icmp, br, two labels
  a return             --> ret
```

## Example 1: Add two parameters

```c
int add(int a, int b) { return a + b; }
```

**What you should see.** With `-disable-llvm-passes`, often two allocas, two stores of the arguments, two loads, an `add`, and a `ret`. Some Clang versions keep scalar parameters in values when optimizations are not fully off; if you see a direct `add` of the arguments, that is the same program, one step closer to mem2reg. Either form is worth reading line by line.

**After mem2reg.** `opt -passes=mem2reg -S` should leave an `add` of the two arguments and a `ret`.

## Example 2: A conditional store

```c
int pick(int c, int x) {
  int y;
  if (c)
    y = x;
  else
    y = 0;
  return y;
}
```

**What you should see.** The diagram in the main note: `icmp`, two stores, one load.

**Counterexample to study.** `clang -O2 -S -emit-llvm` on the same file. The allocas are gone. You are looking at the output of many days at once. Keep this file only as a "later" picture.

## Example 3: A loop

```c
int sum_n(int n) {
  int s = 0;
  for (int i = 0; i < n; i = i + 1)
    s = s + i;
  return s;
}
```

**What you should see.** A block that branches to itself, compares `i` with `n`, and stores updated `s` and `i`.

**After mem2reg.** Two phis in the header, one for `s` and one for `i`. That is the form Day 29 (induction variables) and Day 23 (LICM) discuss.

## Example 4: Address taken, promotion refused

```c
int escape(int x) {
  int y = x;
  g(&y);
  return y;
}
```

```c
void g(int *);
```

**What you should see after mem2reg.** The `alloca` remains, because `g` receives its address. The return is still a load. This is the case Day 5's "is this alloca promotable?" question answers with no.

## Example 5: Short-circuit

```c
int both(int *p) {
  if (p && *p)
    return 1;
  return 0;
}
```

**What you should see.** A branch on `p` being non-null (`icmp ne ptr %p, null`) and, only in the true block, a `load i32`. The load is not in the entry block.

**Wrong lowering, for contrast.** A load of `*p` before the null check would be a different program. If you see the load in the entry, you are not looking at this source.
