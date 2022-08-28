# MSBuild and .csproj

## Target Framework(s)

One target framework:

```xml
<PropertyGroup>
  <TargetFramework>net6.0</TargetFramework>
</PropertyGroup>
```

Multiple target frameworks (notice the property changes from "TargetFramework" to "TargetFramework**s**"):

```xml
<PropertyGroup>
  <TargetFrameworks>net6.0;netstandard2.0</TargetFrameworks>
</PropertyGroup>
```

Possible target framework names: <https://docs.microsoft.com/en-us/dotnet/standard/frameworks#latest-versions>
