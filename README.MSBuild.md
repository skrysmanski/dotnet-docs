# MSBuild and .csproj

## Project Types

Official names: **SDK-style** (modern), **non-SDK-style** (legacy)

Documentation: <https://docs.microsoft.com/en-us/nuget/resources/check-project-format>

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

## Create Link to Multiple Files

Use `LinkBase`:

```xml
<EmbeddedResource Include="..\..\..\Demo\**\*.cs" LinkBase="Resources" />
```

## Import other MSBuild File

Imports `Project.Shared.props` from any parent folder:

```xml
<Import Project="$([MSBuild]::GetPathOfFileAbove('Project.Shared.props', '$(MSBuildThisFileDirectory)'))" />
```

## Common MSBuild Properties

For a full list, see:

* <https://docs.microsoft.com/en-us/visualstudio/msbuild/common-msbuild-project-properties>

### General

* `$(MSBuildThisFileDirectory)`: The directory of the current file (.csproj, .targets, .props, ...)

* `$(OutputPath)`: The relative output path (e.g. `bin\Debug`) - same as `$(OutDir)`
* `$(TargetPath)`: The fully qualified path to the output file.
* `$(TargetName)`: The name of the target file without file extension (and without path).
* `$(TargetFileName)`: The name of the target file with file extension (but without path).

### For NuGet Packages

* `$(PackageId)`: Name of the NuGet package
* `$(PackageVersion)`: Version of the NuGet package

## List All Targets, Properties, And Environment Variables

To list all MSBuild targets, properties, and environment variables for a build, build with logging **Diagnostic** (`/v:d`).

*Targets* will be listed in the **Target Performance Summary**:

```
1>Target Performance Summary:
1>        * ms  GetCopyToOutputDirectoryItems              1 calls
1>        * ms  _GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences   1 calls
1>        0 ms  CreateCustomManifestResourceNames          1 calls
1>        0 ms  SetWin32ManifestProperties                 1 calls
1>        0 ms  PrepareResources                           1 calls
1>        0 ms  ResGen                                     1 calls
1>        0 ms  AfterResGen                                1 calls
...
```

*Properties* and their values will be listed under **Initial Properties**:

```
1>Initial Properties:
...
1>AppDesignerFolder              = Properties
1>AppDesignerFolderContentsVisibleOnlyInShowAllFiles = false
1>AppendRuntimeIdentifierToOutputPath = true
1>AppendTargetFrameworkToOutputPath = false
1>AssemblyFoldersConfigFile      = C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\AssemblyFolders.config
1>AssemblyFoldersConfigFileSearchPath = {AssemblyFoldersFromConfig:C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\AssemblyFolders.config,v6.0};
1>AssemblyFoldersSuffix          = AssemblyFoldersEx
...
```

*Environment variables* are listed under **Environment at start of build**:

```
1>Environment at start of build:
1>ALLUSERSPROFILE                = C:\ProgramData
1>CommonProgramFiles             = C:\Program Files\Common Files
1>CommonProgramFiles(x86)        = C:\Program Files (x86)\Common Files
1>CommonProgramW6432             = C:\Program Files\Common Files
1>ComSpec                        = C:\WINDOWS\system32\cmd.exe
1>DriverData                     = C:\Windows\System32\Drivers\DriverData
...
```
