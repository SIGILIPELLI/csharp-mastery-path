---
description: "File I/O & Working with Text — System.IO provides both simple one-shot helpers (File.ReadAllText, File.WriteAllText) and stream-based classes…"
---

# 09 · File I/O & Working with Text

`System.IO` provides both simple one-shot helpers (`File.ReadAllText`,
`File.WriteAllText`) and stream-based classes (`StreamReader`,
`StreamWriter`) for larger or line-by-line work.

## Writing and reading a whole file at once

```csharp
using System.IO;

string path = "notes.txt";

File.WriteAllText(path, "Line one\nLine two\nLine three");
string content = File.ReadAllText(path);
Console.WriteLine(content);
// Line one
// Line two
// Line three
```

`File.WriteAllText` creates the file if it doesn't exist, or **overwrites**
it completely if it does.

## Reading line by line

```csharp
string[] lines = File.ReadAllLines(path);
Console.WriteLine(lines.Length);   // 3
foreach (var line in lines)
{
    Console.WriteLine("> " + line);
}
// > Line one
// > Line two
// > Line three
```

`File.ReadAllLines` splits on line breaks and hands back an array — simplest
option when the whole file comfortably fits in memory.

## Appending

```csharp
File.AppendAllText(path, "\nLine four");
Console.WriteLine(File.ReadAllLines(path).Length);
// 4
```

Unlike `WriteAllText`, `AppendAllText` adds to the end of the existing file
instead of replacing it.

## StreamWriter / StreamReader with `using`

For more control (or very large files where you don't want everything in
memory at once), use `StreamWriter`/`StreamReader` inside a `using` block so
the underlying file handle is always closed, even if an exception occurs:

```csharp
using (StreamWriter writer = new StreamWriter(path))
{
    writer.WriteLine("Rewritten line one");
    writer.WriteLine("Rewritten line two");
}
Console.WriteLine(File.ReadAllText(path));
// Rewritten line one
// Rewritten line two
// (trailing blank line, since WriteLine ends each line with a newline)

using (StreamReader reader = new StreamReader(path))
{
    string? line;
    int count = 0;
    while ((line = reader.ReadLine()) != null)
    {
        count++;
        Console.WriteLine($"Read #{count}: {line}");
    }
}
// Read #1: Rewritten line one
// Read #2: Rewritten line two
```

`new StreamWriter(path)` (no append flag) also overwrites, same as
`WriteAllText`. `reader.ReadLine()` returns `null` at end-of-file, which is
why the `while` loop's condition is `(line = reader.ReadLine()) != null` —
assignment and null-check in one expression, a very common C# idiom.

## Checking existence and handling missing files

```csharp
Console.WriteLine(File.Exists(path));               // True
Console.WriteLine(File.Exists("nonexistent.txt"));  // False

try
{
    File.ReadAllText("nonexistent.txt");
}
catch (FileNotFoundException e)
{
    Console.WriteLine("Not found: " + e.Message);
}
// Not found: Could not find file '<full path>/nonexistent.txt'.
```

Always prefer checking `File.Exists` first for expected-missing cases, and
reserve the `try/catch` for genuinely exceptional I/O failures (permissions,
disk errors) — the same "expected vs exceptional" judgment call from
Module 7.

## Working with directories and paths

```csharp
string dir = "notes_dir";
Directory.CreateDirectory(dir);
Console.WriteLine(Directory.Exists(dir));   // True

File.WriteAllText(Path.Combine(dir, "a.txt"), "hello");
Console.WriteLine(string.Join(", ", Directory.GetFiles(dir)));
// notes_dir/a.txt

File.Delete(path);
Directory.Delete(dir, true);   // true = recursive, deletes contents too
Console.WriteLine(File.Exists(path));   // False
```

`Path.Combine` builds paths using the correct separator for the current OS
(`/` on macOS/Linux, `\` on Windows) — always prefer it over manual string
concatenation of path segments.

| Method | Behavior |
|--------|----------|
| `File.WriteAllText` | Overwrite (or create) the whole file at once |
| `File.AppendAllText` | Add to the end of an existing (or new) file |
| `File.ReadAllText` | Read the whole file as one string |
| `File.ReadAllLines` | Read the whole file as a `string[]`, split on newlines |
| `StreamWriter` / `StreamReader` | Line-by-line or buffered access, wrap in `using` |
| `File.Exists` / `Directory.Exists` | Check before acting, avoid exceptions for expected cases |
| `Path.Combine` | Build cross-platform-correct file paths |

## How It Actually Works

- **`StreamWriter`/`StreamReader` wrap a `FileStream`, which wraps an OS file
  descriptor — and buffer in user space to reduce syscalls.** Every read or
  write ultimately becomes a system call into the OS kernel, which is orders
  of magnitude slower than in-memory work. `StreamReader`/`StreamWriter`
  keep an internal character buffer (default 1KB, tunable via a
  constructor overload) so that calling `ReadLine()` repeatedly doesn't
  issue a syscall per line — it issues one syscall to fill the buffer, then
  serves many `ReadLine()` calls out of memory until the buffer is
  exhausted. `File.ReadAllText`/`ReadAllLines` do the same buffering
  internally but hide it, at the cost of holding the *entire* decoded file
  in memory at once — fine for small files, a real problem for anything
  approaching available RAM.
- **`Dispose()` on a stream flushes buffered writes before releasing the OS
  handle.** This is the mechanism-level reason `using` matters so much for
  I/O specifically: if you write via `StreamWriter` and the process crashes
  (or an exception propagates) before `Dispose()` runs, buffered-but-not-yet-
  flushed bytes never reach disk — the file can look truncated or missing
  your last few writes. `using` guarantees `Dispose()` — and therefore the
  flush — runs via the compiler-generated `finally` block from Module 7,
  even on the exception path.
- **Text encoding happens at the stream boundary, not in your strings.** A
  C# `string` is always UTF-16 in memory; `StreamWriter`/`StreamReader`
  default to UTF-8 on disk (configurable via a constructor overload) and
  transcode on every read/write. This conversion is why a file written by
  one encoding and read assuming another silently corrupts non-ASCII
  characters — the bytes on disk and the `char`s in memory are never the
  same representation.
- **`File.ReadAllText` internally opens, reads, and disposes a `FileStream`
  in one call** — it's not a different I/O mechanism, just a convenience
  wrapper that saves you writing the `using` block yourself for the common
  "read it all now" case.

## 🔀 See this in another language

- [Go — Packages & Modules](https://sigilipelli.github.io/go-mastery-path/level-1/09-packages-modules/)
- [Scala — Traits Basics](https://sigilipelli.github.io/scala-mastery-path/level-1/09-traits-basics/)
- [PowerShell — Modules Basics](https://sigilipelli.github.io/powershell-mastery-path/level-1/09-modules-basics/)

## Exercise

Write a program that writes a list of at least five numbers to
`scores.txt`, one per line, then reads the file back line by line, parses
each line to an `int` (skipping and reporting any line that fails to
parse), and prints the sum and average of the successfully parsed numbers.
