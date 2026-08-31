# 09 · NuGet & Project Structure

Real .NET applications are rarely a single `.csproj`. This module covers how
NuGet packages are referenced and restored, and how to lay out a multi-project
solution the way production codebases do.

## Anatomy of a `.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>

</Project>
```

This is the entire project file for a modern SDK-style console app — no
`<Compile Include="*.cs" />` needed, because the SDK includes every `.cs`
file under the project directory automatically.

## Adding and removing packages

```bash
dotnet add package Newtonsoft.Json
dotnet add package Newtonsoft.Json --version 13.0.3
dotnet remove package Newtonsoft.Json
dotnet list package
dotnet list package --outdated
```

`dotnet add package` edits the `.csproj`'s `<ItemGroup>` for you and triggers
a restore. `dotnet list package --outdated` checks NuGet.org for newer
versions of everything currently referenced.

## `PackageReference` vs. lockfiles

By default, `PackageReference` entries resolve to the latest version
satisfying the range at restore time (or the pinned version if one is given).
For reproducible builds across machines and CI, generate a lockfile:

```xml
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

```bash
dotnet restore
# generates packages.lock.json — commit this file to source control
```

## Central Package Management (one version list for a whole solution)

When several projects in a solution reference the same package, keeping
versions in sync by hand is error-prone. `Directory.Packages.props` at the
solution root centralizes them:

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageVersion Include="xunit" Version="2.6.2" />
  </ItemGroup>
</Project>
```

Each project's `.csproj` then references packages *without* a version:

```xml
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" />
</ItemGroup>
```

## Multi-project solution layout

A typical layered application separates the domain/library code from the
executable that hosts it, and keeps tests in their own project:

```
MyApp.sln
src/
  MyApp.Core/
    MyApp.Core.csproj
    Models/
    Services/
  MyApp.Cli/
    MyApp.Cli.csproj
    Program.cs
tests/
  MyApp.Core.Tests/
    MyApp.Core.Tests.csproj
```

Wiring it together:

```bash
dotnet new sln -n MyApp
dotnet new classlib -o src/MyApp.Core
dotnet new console -o src/MyApp.Cli
dotnet new xunit -o tests/MyApp.Core.Tests

dotnet sln add src/MyApp.Core/MyApp.Core.csproj
dotnet sln add src/MyApp.Cli/MyApp.Cli.csproj
dotnet sln add tests/MyApp.Core.Tests/MyApp.Core.Tests.csproj

# project references (not NuGet packages) link projects to each other
dotnet add src/MyApp.Cli reference src/MyApp.Core
dotnet add tests/MyApp.Core.Tests reference src/MyApp.Core
```

A project reference in the `.csproj` looks like this:

```xml
<ItemGroup>
  <ProjectReference Include="..\MyApp.Core\MyApp.Core.csproj" />
</ItemGroup>
```

Now `dotnet build` / `dotnet test` / `dotnet run --project src/MyApp.Cli` all
operate against the whole graph, and `MyApp.Cli` and the test project can both
use types from `MyApp.Core` (e.g. `using MyApp.Core.Services;`).

## `Directory.Build.props`: shared settings across projects

Rather than repeating `<Nullable>enable</Nullable>` and the target framework
in every `.csproj`, put shared settings in one file at (or above) the
solution root, and every project under it inherits them automatically:

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
</Project>
```

## `global.json`: pinning the SDK version

To make sure everyone on a team (and CI) builds with the same .NET SDK
version, `global.json` at the repo root pins it:

```json
{
  "sdk": {
    "version": "8.0.100",
    "rollForward": "latestMinor"
  }
}
```

`dotnet --version` respects this file when one is present — if the installed
SDK doesn't satisfy it, `dotnet build` fails fast with a clear error instead
of silently using whatever happens to be on `PATH`.

## Where packages actually live

Restored packages are cached once per machine (not per project) under
`~/.nuget/packages`, keyed by package name and version — so ten projects that
all reference `Newtonsoft.Json 13.0.3` share a single copy on disk instead of
duplicating it.

```bash
dotnet nuget locals global-packages --list
# global-packages: /Users/you/.nuget/packages
```

## Exercise

Build a two-project solution: `MyLib.Core` (a class library with a
`StringUtils` static class exposing `SlugifyTitle(string)` that lowercases
and replaces spaces with hyphens) and `MyLib.Cli` (a console app that
references `MyLib.Core` and prints slugs for a few hardcoded titles). Wire it
up with `dotnet new sln`, `dotnet sln add`, and `dotnet add reference`, then
confirm `dotnet run --project MyLib.Cli` works end to end.
