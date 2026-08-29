# 01 · What Is C# & .NET?

C# is a modern, statically-typed, object-oriented language created by
Microsoft. It compiles to an intermediate language (IL) that runs on **.NET**
— a cross-platform runtime and standard library (works on Windows, macOS, and
Linux). ".NET" is the platform; "C#" is the language you write against it.
There are other .NET languages (F#, VB.NET) but C# is by far the most common.

## Installing the .NET SDK

```bash
# macOS (Homebrew)
brew install --cask dotnet-sdk

# Ubuntu/Debian
sudo apt install dotnet-sdk-8.0

# Windows: use the installer from https://dotnet.microsoft.com/download
```

Verify the install:

```bash
dotnet --version
# 8.0.x  (or newer -- these lessons work on .NET 8+)
```

`dotnet` is the CLI you'll use for almost everything: creating projects,
restoring packages, building, running, testing, and publishing.

## Creating your first project

Unlike Java or C, a C# project isn't a loose file — it's a **project**
described by a `.csproj` file, usually inside a folder. The `dotnet new`
command scaffolds one for you:

```bash
dotnet new console -o HelloWorld
cd HelloWorld
```

This creates:

```
HelloWorld/
├── HelloWorld.csproj   # project file: target framework, package refs, etc.
└── Program.cs          # your code
```

Modern C# (since .NET 6) uses **top-level statements** — no boilerplate
`class`/`Main` wrapper needed for a simple program. `Program.cs` starts out as:

```csharp
// See https://aka.ms/new-console-template for more information
Console.WriteLine("Hello, World!");
```

Run it:

```bash
dotnet run
# Hello, World!
```

`dotnet run` restores packages, compiles, and executes in one step — perfect
while learning. For a distributable binary you'd use `dotnet build` or
`dotnet publish`, covered later.

## The classic (explicit) form

Every top-level-statements file is really shorthand for a class with a `Main`
method. Understanding the expanded form matters because you'll see it in
older code, tutorials, and whenever you need multiple classes with an explicit
entry point:

```csharp
using System;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

- `namespace` groups related types and avoids naming collisions across
  libraries.
- `class Program` — in C#, executable code always lives inside a class.
- `static void Main(string[] args)` — the entry point: `static` because it
  runs without an instance, `void` because it returns nothing here (it can
  also return `int` as an exit code), `string[] args` holds command-line
  arguments.

## Command-line arguments

```csharp
// Program.cs (top-level statements)
if (args.Length == 0)
{
    Console.WriteLine("No arguments passed.");
}
else
{
    foreach (var arg in args)
    {
        Console.WriteLine($"Arg: {arg}");
    }
}
```

```bash
dotnet run -- Alice Bob
# Arg: Alice
# Arg: Bob
```

Note the `--` — it tells `dotnet run` "everything after this belongs to the
program, not to the `dotnet` CLI itself."

## What "compiling" looks like in .NET

- `dotnet build` compiles your project into **IL (Intermediate Language)**,
  stored in a `.dll` inside `bin/Debug/net8.0/`.
- The **CLR** (Common Language Runtime) JIT-compiles that IL to native machine
  code at run time.
- `dotnet run` does build + execute together — the loop you'll live in while
  learning.

```bash
dotnet build
# Build succeeded.
# HelloWorld -> /path/to/HelloWorld/bin/Debug/net8.0/HelloWorld.dll

dotnet bin/Debug/net8.0/HelloWorld.dll
# Hello, World!
```

## Choosing an editor

**Visual Studio Code** with the C# Dev Kit extension (free, cross-platform,
lightweight) is the most common choice for learning and day-to-day work.
**Visual Studio** (Windows-only, free Community edition) has the deepest
tooling for large enterprise/ASP.NET projects. **JetBrains Rider** is a
strong paid cross-platform alternative. Any of these gives you IntelliSense,
debugging, and refactoring — pick one and move on.

| Term | Meaning |
|------|---------|
| .NET | The cross-platform runtime + standard library C# runs on |
| C# | The language itself |
| SDK | Software Development Kit — compiler + CLI + runtime, what you install |
| CLR | Common Language Runtime — executes compiled IL, handles GC, JIT |
| IL | Intermediate Language — what C# compiles to before JIT |
| `dotnet new` | Scaffold a new project from a template |
| `dotnet run` | Build and execute in one step |
| `dotnet build` | Compile without running |
| Top-level statements | Modern shorthand — no explicit `class`/`Main` needed |

## Exercise

Create a new console project called `Greeter`. Write a program using
top-level statements that reads command-line arguments and prints
`"Hello, <name>!"` for each name passed in, or `"Hello, stranger!"` if no
arguments were given. Run it with `dotnet run -- Alice Bob` and with no
arguments to confirm both paths work.
