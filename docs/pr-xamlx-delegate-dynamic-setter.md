# Fix XamlX dynamic delegate-overload setter instability

## Problem

Runtime XAML assignment can fail when multiple candidate setters are resolved and all last-parameter candidate types are delegates. In this case, generated dynamic setter logic may throw:

- `InvalidProgramException`
- `InvalidCastException`

## Root Cause

Dynamic setter dispatch for delegate-only overload groups is unstable in some runtime scenarios. The generated cast/check sequence is not reliable for all delegate combinations.

## Change

File changed:

- `external/XamlX/src/XamlX/IL/Emitters/PropertyAssignmentEmitter.cs`

In `RemoveRedundantSetters`:

- Detect when all candidate last-parameter types are delegates.
- Keep only the first candidate.
- Return early to avoid generating dynamic delegate-overload dispatch code.

Added helper:

- `IsDelegateType(IXamlType? type)`

## Why this approach

This is a targeted stability fix: it avoids a known-failing runtime path while keeping existing non-delegate resolution behavior intact.

## Validation

- Reproduced failure before change.
- Confirmed runtime success after change.
- Confirmed exception returns when this block is removed.
- Build succeeds.

## Risk / Compatibility

- Delegate-only ambiguous setter groups become deterministic (first candidate wins).
- This may alter overload selection behavior for ambiguous delegate-only cases, but avoids runtime crashes.
- No behavior change for non-delegate or value-type paths.

## Follow-up

A broader upstream solution could add dedicated delegate-overload handling in dynamic setter emission instead of relying on generic cast/check dispatch.
