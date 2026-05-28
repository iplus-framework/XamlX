# InvalidCastException in VisitChildren when visitor returns skip nodes

## Description

When processing dynamic XAML, the XamlX compiler throws an `InvalidCastException` during the visitor pattern traversal. This occurs when a visitor returns `SkipXamlValueWithManipulationNode` or `SkipXamlAstNode`, but the `VisitChildren` methods attempt to cast these to specific interface types without validation.

**Exception:**
```
System.InvalidCastException: Unable to cast object of type 'XamlX.Ast.SkipXamlValueWithManipulationNode' to type 'XamlX.Ast.IXamlAstPropertyReference'.
   at XamlX.Ast.XamlAstXamlPropertyValueNode.VisitChildren(IXamlAstVisitor visitor)
```

## Root Cause

The visitor pattern in XamlX is designed to support skip nodes (`SkipXamlAstNode`, `SkipXamlValueWithManipulationNode`) to signal that certain nodes should be skipped during transformation. However, several `VisitChildren` methods in `Xaml.cs` perform unsafe casts on the results of `Visit(visitor)` without checking if the returned node actually implements the expected interface.

## Affected Methods

In `src/XamlX/Ast/Xaml.cs`:
- `XamlAstXamlPropertyValueNode.VisitChildren` - Line 59
- `XamlAstObjectNode.VisitChildren` - Line 87
- `XamlAstTextNode.VisitChildren` - Line 127
- `XamlAstNamePropertyReference.VisitChildren` - Lines 151-152

## Proposed Fix

Replace unsafe casts with safe type checks using pattern matching. Only update the property if the visitor returns a node implementing the expected interface.

### Example for `XamlAstXamlPropertyValueNode.VisitChildren`

**Before:**
```csharp
public override void VisitChildren(Visitor visitor)
{
    Property = (IXamlAstPropertyReference) Property.Visit(visitor);
    VisitList(Values, visitor);
}
```

**After:**
```csharp
public override void VisitChildren(Visitor visitor)
{
    var visitedProperty = Property.Visit(visitor);
    if (visitedProperty is IXamlAstPropertyReference propertyRef)
        Property = propertyRef;
    VisitList(Values, visitor);
}
```

### Apply the same pattern to

**`XamlAstObjectNode.VisitChildren`:**
```csharp
public override void VisitChildren(Visitor visitor)
{
    var visitedType = Type.Visit(visitor);
    if (visitedType is IXamlAstTypeReference typeRef)
        Type = typeRef;
    VisitList(Arguments, visitor);
    VisitList(Children, visitor);
}
```

**`XamlAstTextNode.VisitChildren`:**
```csharp
public override void VisitChildren(Visitor visitor)
{
    var visitedType = Type.Visit(visitor);
    if (visitedType is IXamlAstTypeReference typeRef)
        Type = typeRef;
}
```

**`XamlAstNamePropertyReference.VisitChildren`:**
```csharp
public override void VisitChildren(Visitor visitor)
{
    var visitedDeclaringType = DeclaringType.Visit(visitor);
    if (visitedDeclaringType is IXamlAstTypeReference declaringTypeRef)
        DeclaringType = declaringTypeRef;
    var visitedTargetType = TargetType.Visit(visitor);
    if (visitedTargetType is IXamlAstTypeReference targetTypeRef)
        TargetType = targetTypeRef;
}
```

## Impact

This fix allows visitors to properly return skip nodes without causing crashes, maintaining the intended visitor pattern behavior while preventing runtime exceptions during XAML compilation.
