# Day 0 Extra Notes: Pass Manager Examples

This file provides practical examples of how to interact with the LLVM Pass Manager, print its pipeline, and understand its scheduling.

## Example 1: Viewing the Optimization Pipeline
The best way to understand the LLVM Pass Manager is to watch it work. You can ask `opt` (the LLVM optimizer) to print out the exact sequence of passes it intends to run for a given optimization level.

*   **Command:** 
    `opt -O1 -print-pipeline-passes < /dev/null`
*   **What this does:** Tells the New Pass Manager to print the pipeline it constructed for `-O1` without actually running it on any real code.
*   **What to look for:** Notice how it wraps passes in `module(...)`, `cgscc(...)`, or `function(...)`. This indicates the level of the IR that the pass operates on (Module, CallGraph Strongly Connected Component, or Function).

## Example 2: Running a Specific Pass
If you are developing or studying a specific pass (like InstCombine), you often want to run *only* that pass and observe the output.

*   **Create a file `test.ll`:**
    ```llvm
    define i32 @test(i32 %x) {
      %add = add i32 %x, 0
      ret i32 %add
    }
    ```
*   **Command:**
    `opt -passes=instcombine test.ll -S -o optimized.ll`
*   **Result:** The output `optimized.ll` will have the `add` instruction completely removed, replacing the return value with just `%x`.

## Example 3: Debugging the Pass Manager (Tracing)
When compiling a very complex C program, you might want to see which passes run and in what order they modify the IR.

*   **Command:**
    `clang -O2 -fdebug-pass-manager test.c -c`
*   **What this does:** It outputs a trace of the Pass Manager's execution. You will see lines like:
    ```
    Running pass: InstCombinePass on test
    Running analysis: TargetLibraryAnalysis on test
    ```
*   **Why it's useful:** This allows you to see how Analyses (like TargetLibraryAnalysis) are run dynamically as prerequisites for Transformation passes (like InstCombinePass).
