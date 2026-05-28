# XamlX runtime failure with delegate-based dynamic setters

## Summary

In our Avalonia fork, runtime XAML loading can fail when a property/event assignment is resolved through multiple candidate setters whose last argument types are all delegate types.

Observed failures include:

- `System.InvalidProgramException`
- `System.InvalidCastException`

The errors happen inside generated dynamic setter methods (for example `<>XamlDynamicSetter_*`) and bubble up through runtime XAML population (`__AvaloniaXamlIlPopulate`).

## Where it happens

File:

- `external/XamlX/src/XamlX/IL/Emitters/PropertyAssignmentEmitter.cs`

Method:

- `RemoveRedundantSetters(IXamlType valueType, List<IXamlPropertySetter> setters)`

## Reproduction pattern

The problem is reproducible with runtime-loaded XAML that binds events via markup extensions returning delegates, where XamlX sees multiple matching setter candidates for the last argument and tries to generate a dynamic dispatcher.

Typical scenario:

1. Runtime XAML load (`AvaloniaXamlIlRuntimeCompiler.LoadOrPopulate`).
2. Event property assignment resolves to more than one candidate setter.
3. Last parameter types of those setters are delegate-compatible.
4. Generated dynamic setter path is taken.
5. Runtime throws `InvalidProgramException` or `InvalidCastException`.

## Root cause

`PropertyAssignmentEmitter` allows multi-setter dynamic dispatch for the last argument type. This works for many reference/value type combinations, but delegate overload groups are a special case:

- the generated dynamic checks/casts are not reliably valid for all delegate-type combinations,
- runtime can end up emitting invalid IL or performing an incompatible cast between delegate generic argument types.

In short: the dynamic "pick one of many delegate setters" path is unstable in this case.

## Fix

Short-circuit delegate-only setter groups in `RemoveRedundantSetters`.

If all last-parameter candidate types are delegates, keep only the first candidate and skip dynamic dispatch generation:

```csharp
// Dynamic setter IL for multiple delegate-compatible event setters can become invalid
// on some runtimes. Keeping the first delegate setter avoids generating that path.
if (lastParameterTypes.All(IsDelegateType))
{
    setters.RemoveRange(1, setters.Count - 1);
    return;
}
```

Supporting helper:

```csharp
private static bool IsDelegateType(IXamlType? type)
{
    while (type != null)
    {
        if (type.FullName == "System.MulticastDelegate" || type.FullName == "System.Delegate")
            return true;

        type = type.BaseType;
    }

    return false;
}
```

## Why this is necessary

Removing this change reintroduces the runtime exception in our fork. With this change in place, the failing XAML path runs successfully.

## Behavioral impact

- Delegate-only ambiguous setter groups are now resolved deterministically to the first candidate.
- This avoids unstable runtime IL generation for this edge case.
- Non-delegate and existing value-type handling logic remains unchanged.

## Validation

- Build succeeds after the change.
- Runtime XAML path that previously failed now executes successfully.
- Reverting this block causes the exception to return.

## Notes for maintainers

This is a targeted stability fix for runtime XAML emission in delegate-overload scenarios. If a broader long-term solution is preferred, dynamic delegate overload resolution in `EmitDynamicSetterMethod` may need dedicated handling that does not rely on the current generic runtime cast/check sequence.
