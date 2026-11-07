# Day 28 Extra: Simple Loop Unswitch — Deep Dive

Source: [SimpleLoopUnswitch.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SimpleLoopUnswitch.cpp). Overview: [Day-28-0-SimpleLoopUnswitch.md](Day-28-0-SimpleLoopUnswitch.md).

## Diagram

```text
   for (...) {                         if (cond) {
     if (cond)          unswitch         for (...) { A }
       A;              -------->       } else {
     else                               for (...) { B }
       B;                              }
   }

   `cond` does not change in the loop,
   so the branch is invariant
```

SimpleLoopUnswitch hoists an invariant branch out of the loop and duplicates the loop into the two arms. Each copy then has straight-line control where the branch used to be, so LICM and the vectorizer see a simpler body. The pass will not unswitch a branch that is trivial to fold another way, and it refuses when duplicating the body would blow the size threshold. Non-trivial unswitching freezes a possibly poison condition rather than branching on poison.

## Historical context

Classic `LoopUnswitch` lived in the old pass manager era and predates MemorySSA. `SimpleLoopUnswitch` reimplements the transform with:

- Explicit trivial / non-trivial split.
- MemorySSA updates for modern LICM/MSSA pipelines.
- Freeze insertion for poison safety.
- Richer knobs for guards, implicit null checks, and profile.

When reading old blog posts, map “LoopUnswitch” → this pass.

## Options worth knowing

Near the top of the file (`DEBUG_TYPE "simple-loop-unswitch"`):

| Option | Role |
|--------|------|
| `unswitch-threshold` | Max cost for non-trivial duplication (default ~50) |
| `freeze-loop-unswitch-cond` | Insert `freeze` on unswitched conditions (default true) |
| `simple-loop-unswitch-memoryssa-threshold` | Cap MSSA work |
| `simple-loop-unswitch-guards` | Interact with `llvm.experimental.guard` / makes.guard style |
| `simple-loop-unswitch-inject-invariant-conditions` | Advanced: inject conditions to create invariants |
| `simple-loop-unswitch-estimate-profile` | Use estimated profile for profitability |

Do not memorize every flag; know that **threshold + freeze + MSSA** are the teaching core.

## Trivial unswitch path

Trivial cases include invariant branches where one edge exits the loop. The rewrite:

1. Split the preheader / insert a branch on the invariant condition.
2. Connect the exit successor without cloning the whole body.
3. Keep a single loop for the continuing edge.

This is cheap and often enables dead-code removal of the unused side. Read the functions that rewrite branches and set unknown branch weights when profiled (`setExplicitlyUnknownBranchWeightsIfProfiled`)—unswitching can invalidate previous edge weights.

## Non-trivial unswitch path

### Finding candidates

Scan terminators in the loop for `BranchInst` / `SwitchInst` whose condition is invariant. For switches, unswitching may duplicate per case or per group depending on implementation limits—start with binary branches when learning.

### Costing

Cost is roughly “size of instructions that would be duplicated,” scaled by how many times nested unswitch has already multiplied the loop. Comments in the file discuss exponential behavior; the threshold comparison is the principal kill switch.

### Cloning

Uses loop cloning utilities (CloneLoop / CloneBasicBlock patterns shared with other transforms). After cloning:

- Remap instructions via `ValueToValueMapTy`.
- Fix exit block LCSSA phis for both copies.
- Dominator tree and LoopInfo must describe two sibling loops under the same parent (or top-level).

### Preheader branch

The outer selector branch uses the (possibly frozen) condition. Each successor’s preheader is the preheader of one specialized loop. The original invariant inner branch is simplified to an unconditional branch in each copy (InstSimplify / fold), or deleted as dead.

## Freeze insertion policy

Comments near the freeze helper state the conditions (paraphrased):

1. `freeze-loop-unswitch-cond` is true.
2. The condition might be poison / continuing without freeze could introduce divergent UB.
3. Insert `freeze` before the selecting branch; often **without** stealing a debug location from the original (see comments about debug locations on new freezes).

Also watch selects that feed branches: branching on poison is UB in LLVM; freeze may be placed on select conditions involved in unswitch.

Mental model for students:

```text
%cond     ; may be poison
%cond.fr  = freeze i1 %cond
br i1 %cond.fr, label %A, label %B
```

## MemorySSA updates

Non-trivial cloning duplicates memory ops. MemorySSA must gain duplicated MemoryDefs/Uses and new MemoryPhis at merge points. The pass uses MemorySSAUpdater rather than full rebuild when possible. If MSSA verification is enabled (`-verify-memoryssa`), failures after unswitch usually mean the updater missed an edge—goldmine for learning MSSA (Day 37).

## Profile-guided behavior

With branch weights:

- A nearly-always-true invariant condition still costs code size for the cold clone; PGO may still allow it if the loop is hot enough, or may skip if size-sensitive.
- After unswitch, cloned edges may get `explicitly unknown` weights to avoid lying to later passes.

## Partial invariance / injection (optional reading)

Flags under `simple-loop-unswitch-inject-invariant-conditions` try to rewrite conditions so more branches become unswitchable. Treat as advanced; not required for the Day 28 check-yourself set.

## Suggested reading order inside the .cpp

1. cl::opt block + file header comments.
2. Trivial unswitch function(s) — smaller CFG edits.
3. Cost estimation for non-trivial.
4. Clone + preheader splice for non-trivial.
5. Freeze helper.
6. One test from `llvm/test/Transforms/SimpleLoopUnswitch/` with freeze in the CHECK lines.

## Relationship to other days

- **Day 26 LoopRotate** — often runs before; unswitch does not require rotated form but likes simplify/LCSSA.
- **Day 37 MemorySSA** — explains why updater calls matter.
- **Day 40 TTI** — some cost hooks may consult TTI; the primary knob remains `unswitch-threshold`.

## Cloning checklist while reading the code

When following non-trivial unswitch:

1. Identify the invariant terminator.
2. See cost estimate vs threshold.
3. Find freeze insertion on the condition.
4. Observe loop clone and Value maps.
5. Observe preheader split and outer branch.
6. Observe MSSA updater calls.
7. Observe LCSSA repair for exits.

If any step is missing in a patch you review, be suspicious.

## Profile metadata after unswitch

`setExplicitlyUnknownBranchWeightsIfProfiled` appears when old weights no longer apply to new edges. Downstream BlockPlacement (later days in the backend) should not trust stale weights. For mid-end study, just note that unswitch is profile-aware and may refuse cold clones.

## Exponential unswitch guardrails

Comments about exponential behavior describe limiting repeated unswitch on the same nest. Even with a high threshold, internal caps prevent compile-time bombs. That is why a hand-crafted torture test with many invariant branches may not produce `2^n` clones.

## Trivial unswitch vs SimplifyCFG

Some trivial exit hoists look like SimplifyCFG work. The difference is loop awareness: SimpleLoopUnswitch updates LoopInfo/LCSSA/MSSA for the remaining loop. Doing the same rewrite only with SimplifyCFG can break loop pass preconditions. Prefer the loop pass when the condition is a loop invariant exit test.

### Additional study points (1)

- When following non-trivial unswitch:
- 1. Identify the invariant terminator.
- 2. See cost estimate vs threshold.
- 3. Find freeze insertion on the condition.
- 4. Observe loop clone and Value maps.
- 5. Observe preheader split and outer branch.
- 6. Observe MSSA updater calls.
- 7. Observe LCSSA repair for exits.
- If any step is missing in a patch you review, be suspicious.
`setExplicitlyUnknownBranchWeightsIfProfiled` appears when old weights no longer apply to new edges. Downstream BlockPlacement (later days in the backend) should not trust stale weights. For mid-end study, just note that unswitch is profile-aware and may refuse cold clones.
Comments about exponential behavior describe limiting repeated unswitch on the same nest. Even with a high threshold, internal caps prevent compile-time bombs. That is why a hand-crafted torture test with many invariant branches may not produce `2^n` clones.
Some trivial exit hoists look like SimplifyCFG work. The difference is loop awareness: SimpleLoopUnswitch updates LoopInfo/LCSSA/MSSA for the remaining loop. Doing the same rewrite only with SimplifyCFG can break loop pass preconditions. Prefer the loop pass when the condition is a loop invariant exit test.

```bash
# revisit with debug (session 1)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

## More on trivial vs non-trivial decision points

Trivial unswitch is attempted aggressively because it is cheap. Non-trivial unswitch asks cost first. A branch that looks “almost trivial” (one edge exits a child loop but not the parent) can fall into non-trivial cloning of a subset of the nest. When IR surprises you, map each edge as “exits L?” for the loop L being processed—SimpleLoopUnswitch is loop-scoped, not function-scoped.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
