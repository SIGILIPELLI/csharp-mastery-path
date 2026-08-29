# 10 · Project — Console Task Manager App

This project pulls together everything from Level 1: classes, an `enum`,
collections, LINQ, exception-safe parsing, and file persistence, into one
working console app.

## What it does

A simple in-memory task list with commands to add, list, complete, remove,
filter by priority, show stats, and persist to/reload from a text file.

## The `TaskItem` class and `Priority` enum

```csharp
enum Priority { Low, Medium, High }

class TaskItem
{
    public int Id { get; }
    public string Title { get; }
    public Priority Priority { get; }
    public bool Done { get; set; }

    public TaskItem(int id, string title, Priority priority)
    {
        Id = id;
        Title = title;
        Priority = priority;
        Done = false;
    }

    public override string ToString()
    {
        var mark = Done ? "x" : " ";
        return $"[{mark}] #{Id} ({Priority}) {Title}";
    }
}
```

An `enum` gives named, type-safe constants (`Priority.High` instead of a
magic string or number) — the compiler catches typos that a raw `string`
priority field never would.

## Loading and saving — a tiny CSV-like format

```csharp
static List<TaskItem> LoadTasks(string path)
{
    var result = new List<TaskItem>();
    if (!File.Exists(path)) return result;

    foreach (var line in File.ReadAllLines(path))
    {
        if (string.IsNullOrWhiteSpace(line)) continue;
        var fields = line.Split(',');
        var id = int.Parse(fields[0]);
        var title = fields[1];
        var priority = Enum.Parse<Priority>(fields[2]);
        var done = bool.Parse(fields[3]);
        result.Add(new TaskItem(id, title, priority) { Done = done });
    }
    return result;
}

static void SaveTasks(string path, List<TaskItem> tasks)
{
    var lines = tasks.Select(t => $"{t.Id},{t.Title},{t.Priority},{t.Done}");
    File.WriteAllLines(path, lines);
}
```

`Enum.Parse<Priority>("High")` converts the stored string back into the
`enum` value — the generic type argument tells it which enum to target.
Note the object initializer `{ Done = done }` after the constructor call —
a compact way to set an additional property right after construction.

## The command dispatcher

```csharp
static void RunCommand(List<TaskItem> tasks, string command)
{
    var parts = command.Split('|');
    var action = parts[0];

    switch (action)
    {
        case "add":
            var title = parts[1];
            var priority = ParsePriority(parts[2]);
            int nextId = tasks.Count == 0 ? 1 : tasks.Max(t => t.Id) + 1;
            tasks.Add(new TaskItem(nextId, title, priority));
            Console.WriteLine($"Added #{nextId}: {title} [{priority}]");
            break;

        case "list":
            Console.WriteLine("--- Task List ---");
            if (tasks.Count == 0)
            {
                Console.WriteLine("(no tasks)");
                break;
            }
            foreach (var t in tasks.OrderBy(t => t.Done).ThenByDescending(t => t.Priority))
            {
                Console.WriteLine(t);
            }
            break;

        case "done":
            int doneId = int.Parse(parts[1]);
            var toComplete = tasks.FirstOrDefault(t => t.Id == doneId);
            if (toComplete == null)
            {
                Console.WriteLine($"No task with id {doneId}");
            }
            else
            {
                toComplete.Done = true;
                Console.WriteLine($"Marked #{doneId} done.");
            }
            break;

        case "remove":
            int removeId = int.Parse(parts[1]);
            int removed = tasks.RemoveAll(t => t.Id == removeId);
            Console.WriteLine(removed > 0 ? $"Removed #{removeId}" : $"No task with id {removeId}");
            break;

        case "filter":
            var wantedPriority = ParsePriority(parts[1]);
            Console.WriteLine($"--- Filter: {wantedPriority} ---");
            foreach (var t in tasks.Where(t => t.Priority == wantedPriority))
            {
                Console.WriteLine(t);
            }
            break;

        case "stats":
            int total = tasks.Count;
            int done = tasks.Count(t => t.Done);
            int pending = total - done;
            Console.WriteLine($"--- Stats: {total} total, {done} done, {pending} pending ---");
            break;

        default:
            Console.WriteLine($"Unknown command: {action}");
            break;
    }
}

static Priority ParsePriority(string value) => value.ToLower() switch
{
    "high" => Priority.High,
    "medium" => Priority.Medium,
    "low" => Priority.Low,
    _ => throw new ArgumentException($"Unknown priority: {value}")
};
```

`OrderBy(t => t.Done).ThenByDescending(t => t.Priority)` sorts pending tasks
(`Done == false` sorts first, since `false < true`) ahead of completed ones,
and within each group shows highest priority first — a two-level sort with
LINQ instead of a custom `IComparer`.

## Driving it end-to-end

```csharp
const string DataFile = "tasks.txt";

var tasks = LoadTasks(DataFile);

RunCommand(tasks, "add|Write Level 1 content|high");
RunCommand(tasks, "add|Buy groceries|low");
RunCommand(tasks, "add|Fix production bug|high");
RunCommand(tasks, "add|Read a book|low");
RunCommand(tasks, "list");
RunCommand(tasks, "done|1");
RunCommand(tasks, "done|3");
RunCommand(tasks, "list");
RunCommand(tasks, "filter|high");
RunCommand(tasks, "stats");
RunCommand(tasks, "remove|2");
RunCommand(tasks, "list");
RunCommand(tasks, "done|99");

SaveTasks(DataFile, tasks);
Console.WriteLine();
Console.WriteLine("--- reloaded from disk ---");
var reloaded = LoadTasks(DataFile);
foreach (var t in reloaded)
{
    Console.WriteLine(t);
}
```

Running this end-to-end (`dotnet run`) produces:

```
Added #1: Write Level 1 content [High]
Added #2: Buy groceries [Low]
Added #3: Fix production bug [High]
Added #4: Read a book [Low]
--- Task List ---
[ ] #1 (High) Write Level 1 content
[ ] #3 (High) Fix production bug
[ ] #2 (Low) Buy groceries
[ ] #4 (Low) Read a book
Marked #1 done.
Marked #3 done.
--- Task List ---
[ ] #2 (Low) Buy groceries
[ ] #4 (Low) Read a book
[x] #1 (High) Write Level 1 content
[x] #3 (High) Fix production bug
--- Filter: High ---
[x] #1 (High) Write Level 1 content
[x] #3 (High) Fix production bug
--- Stats: 4 total, 2 done, 2 pending ---
Removed #2
--- Task List ---
[ ] #4 (Low) Read a book
[x] #1 (High) Write Level 1 content
[x] #3 (High) Fix production bug
No task with id 99

--- reloaded from disk ---
[x] #1 (High) Write Level 1 content
[x] #3 (High) Fix production bug
[ ] #4 (Low) Read a book
```

Notice the reload after `SaveTasks`/`LoadTasks` reproduces the same three
remaining tasks with their completion state intact — proof the persistence
round-trips correctly, not just that the in-memory `List<TaskItem>` behaves.

## What this project exercised

| Level 1 module | Used here |
|---|---|
| 02 Variables & Types | `const string`, `int`, `bool` |
| 03 Control Flow | `switch` statement dispatch, `if`/`else` |
| 04 Methods & Functions | Static helper methods, expression-bodied `ParsePriority` |
| 05 Classes & Objects | `TaskItem` class, properties, constructor |
| 06 Collections | `List<TaskItem>`, `RemoveAll` |
| 07 Exception Handling | `ArgumentException` on unknown priority |
| 08 LINQ Basics | `Where`, `OrderBy`/`ThenByDescending`, `Max`, `Count`, `FirstOrDefault` |
| 09 File I/O | `File.ReadAllLines`/`WriteAllLines`, `File.Exists` |

## Exercise

Extend the task manager with an `"edit|<id>|<newTitle>"` command that
changes a task's title in place (you'll need to make `Title` settable, or add
a method on `TaskItem` that returns a new instance with the updated title).
Then add a `"clear-done"` command that removes every completed task at once
using `RemoveAll`, and verify with `"stats"` before and after.
