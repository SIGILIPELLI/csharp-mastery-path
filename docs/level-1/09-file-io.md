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

## Exercise

Write a program that writes a list of at least five numbers to
`scores.txt`, one per line, then reads the file back line by line, parses
each line to an `int` (skipping and reporting any line that fails to
parse), and prints the sum and average of the successfully parsed numbers.
