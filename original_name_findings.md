# `Method#original_name` and alias chains

Reference notes for porting the fix (or verifying behavior) in JRuby.
Spec under test: https://github.com/ruby/spec/pull/1354 — extends the existing
two-hop alias chain to three hops (`foo → bar → baz → qux`) and asserts
`qux.original_name == :foo`.

## TruffleRuby: passes by construction

TruffleRuby already returns `:foo` for any depth of alias chain. There is no
walk at lookup time — the original name is captured once at parse time and
preserved structurally.

### Call path

`Method#original_name`
→ `MethodNodes.OriginalNameNode` (`src/main/java/org/truffleruby/core/method/MethodNodes.java:182`)
→ `InternalMethod.getOriginalName()` (`src/main/java/org/truffleruby/language/methods/InternalMethod.java:205-207`)
→ `sharedMethodInfo.getMethodName()`

### Why aliasing doesn't change it

`SharedMethodInfo` holds parse-time data and is documented as immutable w.r.t.
the method name:

> The original name of the method (of the surrounding method if this is a
> block). Does not change when aliased.
> — `src/main/java/org/truffleruby/language/methods/SharedMethodInfo.java:41-43`

`alias_method` (`src/main/java/org/truffleruby/core/module/ModuleNodes.java:355`)
calls `method.withName(newName)`. `InternalMethod.withName`
(`src/main/java/org/truffleruby/language/methods/InternalMethod.java:301-322`)
allocates a new `InternalMethod` but reuses the same `SharedMethodInfo`
instance. So:

```
def foo; end
alias bar foo   # InternalMethod{name="bar", sharedMethodInfo=SMI{methodName="foo"}}
alias baz bar   # InternalMethod{name="baz", sharedMethodInfo=SMI{methodName="foo"}}
alias qux baz   # InternalMethod{name="qux", sharedMethodInfo=SMI{methodName="foo"}}
```

All four point at the same `SharedMethodInfo`. `original_name` is O(1) and
depth-independent.

## JRuby: likely a small fix, not a refactor

I have not read JRuby source in this session — what follows is conceptual
based on architectural similarity. Verify against the actual code.

### Analogous types already exist

| TruffleRuby                  | JRuby (approximate)             |
|------------------------------|---------------------------------|
| `InternalMethod`             | `DynamicMethod`                 |
| `SharedMethodInfo`           | `IRScope` / `IRMethod`          |
| `InternalMethod.withName`    | `AliasMethod` wrapping delegate |

`IRMethod` already carries the parse-time name, source position, and arity
descriptor, so a parallel "SharedMethodInfo" largely exists.

### Recommended fix (proportional to the spec)

Two options for `AliasMethod#original_name`:

1. **Walk the delegate chain** until a non-alias method is reached, return its
   name. Simple, but O(depth) per lookup.
2. **Snapshot at construction**: `AliasMethod` records the original name of
   its target when built, by reading the target's own `originalName` (which
   for an `AliasMethod` is already resolved). Three-hop chains then resolve
   in O(1) and match the TruffleRuby invariant.

Option 2 is preferable: matches "original name is fixed at definition time,
copied through, never re-resolved" and is robust if an inner alias's target
is later redefined.

### What would be a real refactor (probably not worth it)

A full SharedMethodInfo-style split in JRuby would mean separating parse-time
state (name, scope, arity, source) from runtime state (visibility, owner,
declaring module) across every `DynamicMethod` subclass and every site that
constructs one. That is a large change and not required to pass spec #1354.

## TL;DR

- TruffleRuby passes spec PR #1354 with no change. Confirmed by code reading;
  not run.
- JRuby fix is likely a few lines in `AliasMethod` (or wherever aliases are
  built). No new `SharedMethodInfo` type needed — the parse-time data already
  lives in `IRMethod`/`IRScope`.
