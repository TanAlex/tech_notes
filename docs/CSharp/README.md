# C# Note

## Flagged enum
```csharp
[Flags]
public enum Tag {
  None   =0x0,
  Tip    =0x1,
  Example=0x2
}

snippet.Tag = Tag.Tip | Tag.Example
```

## Cast vs As

| CAST   |      DESCRIPTION  |
|----------|:-------------|
| Tree tree = (Tree)obj |  Use this when you expect that obj will only ever be of type Tree. If obj is not a Tree, an InvalidCast exception will be raised. |
| Tree tree = obj as Tree |    Use this when you anticipate that obj may or may not be a Tree. **If obj is not a Tree, a null value will be assigned to tree.** Always follow an “as” cast with conditional logic to properly handle the case where null is returned. Only use this style of conversion when necessary, since it necessitates conditional handling of the return value. This extra code creates opportunities for more bugs and makes the code more difficult to read and debug.   | 

