Title:
Avoid dynamic delegate-overload setters in XamlX runtime emission

Body:
Runtime XAML loading could fail when property/event assignment resolved to
multiple setters whose last-parameter types were all delegates.

In this case the generated dynamic setter path could produce invalid IL or
incompatible runtime casts, leading to InvalidProgramException or
InvalidCastException.

This change updates RemoveRedundantSetters in PropertyAssignmentEmitter:
- detect delegate-only last-parameter candidate groups
- keep only the first candidate
- skip dynamic delegate-overload dispatch generation

Also adds IsDelegateType helper for delegate hierarchy detection.

Result:
- fixes runtime crashes in delegate-overload dynamic assignment scenarios
- keeps existing behavior for non-delegate and value-type paths
- build remains successful
