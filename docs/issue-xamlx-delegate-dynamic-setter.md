# Runtime XAML can fail for delegate-overload dynamic setters (InvalidProgramException / InvalidCastException)

## Summary

Runtime XAML loading can fail when a property/event assignment has multiple candidate setters and all candidate last-parameter types are delegates.

Observed failures:

- `System.InvalidProgramException`
- `System.InvalidCastException`

These failures originate from generated dynamic setter methods (for example `<>XamlDynamicSetter_*`) and surface during runtime XAML population.

## Reproduction Pattern

1. Load XAML at runtime (dynamic path).
2. Assign an event/property value via a markup extension returning a delegate.
3. Resolver finds multiple setter candidates for the last argument.
4. All candidate last-parameter types are delegate-compatible.
5. Dynamic setter dispatch is generated and executed.
6. Runtime throws `InvalidProgramException` or `InvalidCastException`.

## Expected Behavior

Runtime XAML should resolve/set delegate-based assignments without generating invalid IL or invalid delegate casts.

## Actual Behavior

The delegate-overload dynamic dispatch path may generate invalid IL/casts and fails at runtime.

## Root Cause (analysis)

`PropertyAssignmentEmitter` allows multi-setter dynamic dispatch for the final argument. In delegate-only overload groups, the generic runtime check/cast path is not robust for all delegate combinations, which can result in invalid IL and/or incompatible casts.

## Proposed Fix

In `RemoveRedundantSetters`, if all candidate last-parameter types are delegates, keep only the first candidate and skip dynamic delegate-overload dispatch generation.

This avoids the unstable runtime code path while preserving existing behavior for non-delegate cases.

## Validation

- With the change: runtime XAML path succeeds.
- Without the change: exception is reproducible again.
- Build remains successful.

## Environment

- Avalonia fork with custom runtime XAML usage
- XamlX dynamic setter emission path
