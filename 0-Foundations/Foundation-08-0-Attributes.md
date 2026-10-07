# Foundation 8: Attributes, a field guide

Attributes are facts attached to a function, a parameter, a return value, or a call site. They are not instructions. They do not run. Passes read them as permission: "this value is not poison", "this function does not write memory", "do not optimize this function".

This note is the dozen attributes that show up in ordinary Clang IR. The full list is [LangRef's attribute section](https://llvm.org/docs/LangRef.html#function-attributes). Day 48 (Attributor) is how LLVM infers more of them. Day 101 is `noundef` as a poison rule.

Deep dive: [where attributes sit in the API](Foundation-08-1-Extra-Attributes-DeepDive.md). Examples: [attribute examples](Foundation-08-2-Extra-Attributes-Examples.md).

## Diagram

```text
  define noundef i32 @f(ptr nocapture noundef readonly %p) nounwind {
         |            |     |         |        |             |
         |            |     |         |        |             +-- function
         |            |     |         |        +-- parameter: no write through %p
         |            |     |         +-- parameter: not poison
         |            |     +-- parameter: the function does not store this pointer
         |            +-- return value: not poison
         +-- return type

  call void @g(i32 %x) nounwind
                       |
                       +-- this call site only
```

Read the words nearest the thing they describe. A word between the type and the name of a parameter belongs to that parameter. A word after the argument list, or before `define`'s return type when it describes the function as a whole (`nounwind`), belongs to the function. Clang also gathers a set of function attributes into a group `#0` and prints `define void @f() #0`, with the list at the bottom of the file as `attributes #0 = { nounwind ... }`.

## The ones you should recognize

| Attribute | On | Meaning |
| --- | --- | --- |
| `noundef` | value | Not undef and not poison. Passing poison here is undefined behavior. |
| `nonnull` | pointer | Not a null pointer. |
| `dereferenceable(n)` | pointer | The next `n` bytes may be loaded. |
| `readonly` | function or parameter | No writes through this pointer, or no writes in the function. |
| `readnone` | function | No memory reads or writes. A newer spelling is `memory(none)`. |
| `nocapture` | pointer parameter | The function does not store the pointer into memory that outlives the call. |
| `nounwind` | function or call | The call will not throw an exception. |
| `willreturn` | function | The call returns to its caller (it does not loop forever or exit the process). |
| `signext` / `zeroext` | integer parameter or return | The caller widened a small integer, and the high bits are a sign or zero copy. |
| `optnone` | function | `opt` must skip this function. Combined with `noinline`. |
| `noinline` / `alwaysinline` | function | Directions to the inliner (Day 42). |
| `noreturn` | function | Does not return. Callers are followed by `unreachable`. |

You can ignore an attribute and still understand the instructions. You cannot ignore `optnone` and expect a pass to do anything. You cannot ignore `readonly` when you are asking why a later pass moved a load across a call.

## `memory(...)`

Newer IR spells function memory effects as one attribute:

```llvm
define void @f(ptr %p) memory(argmem: read) {
  %v = load i32, ptr %p
  ret void
}
```

`memory(none)` is the old `readnone`. `memory(read)` is the old `readonly`. You will see both spellings. They are the same family of fact.

## `#0` groups

```llvm
define i32 @f(i32 %a) #0 {
  ret i32 %a
}

attributes #0 = { nounwind memory(none) }
```

`#0` is not a type and not a metadata node. It is a shorthand so every function does not repeat a long list. More than one group can appear (`#0 #1`).

## Hands-on

Generate IR for a tiny C file with the Foundation 6 command. Find `noundef` on the arguments and the `attributes #0` line at the bottom. Then find whether the function is `optnone`. It should not be, with that command.

Generate the same file with `clang -S -emit-llvm -O0` and no other flag. `optnone` should be present. That is the trap from the setup note, now visible as an attribute.

## Pitfalls

- Treating attributes as comments. The verifier checks some of them, and passes trust the rest. A wrong `readonly` is a miscompile waiting to happen.
- Confusing `!nonnull` metadata with the `nonnull` attribute. Metadata can be dropped. The attribute cannot be dropped as casually. Day 113.
- Reading `signext` as "this value is negative". It means the high bits, if the argument was widened, match the sign. The value can still be positive.

## Check yourself

1. Which attribute makes `opt` skip a function?
2. Why does `nocapture` on `%p` matter to a pass that wants to turn a pointer argument into an integer value?
3. Where do you look up an attribute that this table does not list?

**Answers.** (1) `optnone`. (2) The pointer does not escape, so the integer it points at can sometimes be passed instead (Day 45). If the pointer is stored into a global, that rewrite is illegal. (3) LangRef's attribute section, then the pass that queried it.
