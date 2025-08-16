## Advanced C# Concepts – Guide & Exercises

This guide explains the requested C# concepts clearly and precisely, step by step, with runnable examples. Use it as a mini-handbook and practice with the exercises.

### How to use this guide
- Read each section in order
- Run/modify the examples
- Do the exercises at the end of sections
- Check your answers in the provided solutions

## Table of Contents

1. [Properties (auto, init-only)](#properties-auto-init-only)
2. [Generics & Constraints](#generics--constraints)
3. [Delegates & Events](#delegates--events)
4. [LINQ](#linq)
5. [Async & Await](#async--await)
6. [Reflection](#reflection)
7. [Attributes](#attributes)
8. [Pattern Matching](#pattern-matching)
9. [Memory Management](#memory-management)
10. [Exercises](#exercises)
11. [Theory Check](#theory-check)
12. [Quick Reference](#quick-reference)

---

## Properties (auto, init-only)

Properties expose data with getters/setters while encapsulating fields.

### Auto-properties
- Automatically generate a backing field
- Optional access modifiers per accessor

```csharp
public class User
{
    // Read-write
    public string Name { get; set; } = "Unknown";

    // Read-only (no set accessor)
    public int Age { get; }

    // Public getter, private setter
    public string Email { get; private set; }

    public User(int age, string email)
    {
        Age = age;           // settable inside constructor for get-only auto-property
        Email = email;       // private setter usable inside the class
    }
}
```

### Init-only properties (immutable after construction)
- C# 9+: `init` allows setting a property during object creation (constructor or object initializer) only
- Great for immutable records or DTOs

```csharp
public class Product
{
    public string Id { get; init; }
    public string Name { get; init; }
    public decimal Price { get; init; }
}

var p = new Product { Id = "P-1", Name = "Keyboard", Price = 49.99m };
// p.Price = 59.99m; // Compile error: init-only
```

---

## Generics & Constraints

Generics let you write type-safe, reusable code.

```csharp
public class Box<T>
{
    public T Value { get; set; }
    public Box(T value) => Value = value;
}
```

### Generic methods
```csharp
public static T FirstOrDefaultSafe<T>(IEnumerable<T> source)
{
    foreach (var item in source) return item;
    return default!; // default for T (null for refs, zero/false for value types)
}
```

### Constraints (`where`)
- `where T : class` reference types only
- `where T : struct` value types only
- `where T : new()` must have public parameterless constructor
- `where T : SomeBaseClass` must inherit base class
- `where T : ISomeInterface` must implement interface
- `where T : unmanaged` blittable value types (no refs)
- `where T : notnull` disallow null
- `where T : Enum` enum types only
- `where T : Delegate` delegate types only

```csharp
public interface IRepository<T> where T : class, new()
{
    T Add(T entity);
}

public static T CreateInstance<T>() where T : new() => new T();

public static bool IsDefaultValue<T>(T value) where T : struct => value.Equals(default);
```

When to use `where T : class`?
- You need reference-type semantics (nullable, identity)
- You plan to assign or compare to `null`
- Your API is only meaningful for reference types (e.g., tracking entities)

---

## Delegates & Events

Delegates are type-safe function pointers. Events are a restricted publish/subscribe mechanism built on delegates.

### Declaring and using a delegate
```csharp
public delegate int MathOp(int a, int b);

int Add(int x, int y) => x + y;
int Multiply(int x, int y) => x * y;

var op = new MathOp(Add);
Console.WriteLine(op(2, 3)); // 5

// Multicast (invocation list). Best with void-return delegates.
op += Multiply; // both Add and Multiply will run; return value from last delegate
```

### Built-in delegate types
- `Action` – no return value (void). Up to 16 parameters, e.g., `Action<int, string>`
- `Func<TResult>` / `Func<T1,...,TResult>` – returns TResult
- `Predicate<T>` – returns bool; equivalent to `Func<T, bool>`

```csharp
Action<string> log = Console.WriteLine;
Func<int, int, int> combine = (a, b) => a + b;
Predicate<string> isNullOrEmpty = string.IsNullOrEmpty;
```

### Anonymous methods & lambdas
```csharp
// Anonymous method
Func<int, int> square1 = delegate(int n) { return n * n; };

// Lambda expressions
Func<int, int> square2 = n => n * n;           // expression-bodied
Func<int, int> square3 = (int n) => { return n * n; }; // statement-bodied
```

### Events and the `event` keyword
- Encapsulates `add/remove` accessors for a delegate
- Only the declaring type can raise the event
- Subscribers use `+=`/`-=` to attach/detach handlers

```csharp
public class BalanceLowEventArgs : EventArgs
{
    public decimal Balance { get; }
    public BalanceLowEventArgs(decimal balance) => Balance = balance;
}

public class BankAccount
{
    public decimal Balance { get; private set; }
    public decimal LowBalanceThreshold { get; set; } = 50m;

    public event EventHandler<BalanceLowEventArgs>? BalanceLow;

    public BankAccount(decimal initial) => Balance = initial;

    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        Balance -= amount;
        if (Balance <= LowBalanceThreshold)
        {
            BalanceLow?.Invoke(this, new BalanceLowEventArgs(Balance));
        }
    }
}

// Usage
var acct = new BankAccount(120m) { LowBalanceThreshold = 75m };
acct.BalanceLow += (s, e) => Console.WriteLine($"Warning: Low balance {e.Balance:C}");
acct.Withdraw(30m); // no event
acct.Withdraw(20m); // event fires
```

---

## LINQ

Language Integrated Query for querying objects, XML, DB, etc.

### Query syntax vs method syntax
```csharp
var numbers = new[] { 1, 2, 3, 4, 5, 6 };

// Query syntax
var evensQuery = from n in numbers
                 where n % 2 == 0
                 select n * n;

// Method syntax (equivalent)
var evensMethod = numbers
    .Where(n => n % 2 == 0)
    .Select(n => n * n);
```

### Common operators
```csharp
var people = new[]
{
    new { Id = 1, Name = "Ana", Age = 30, City = "NY" },
    new { Id = 2, Name = "Ben", Age = 41, City = "SF" },
    new { Id = 3, Name = "Cara", Age = 41, City = "NY" },
};

// Where
var older = people.Where(p => p.Age > 35);

// Select
var names = people.Select(p => p.Name);

// GroupBy
var byAge = people.GroupBy(p => p.Age);
foreach (var g in byAge)
{
    Console.WriteLine($"Age: {g.Key}, Count: {g.Count()}");
}

// Join
var cities = new[]
{
    new { City = "NY", Country = "USA" },
    new { City = "SF", Country = "USA" },
};
var joined = people.Join(cities,
    p => p.City,
    c => c.City,
    (p, c) => new { p.Name, c.Country });

// Aggregate
int sum = people.Select(p => p.Age).Aggregate(0, (acc, n) => acc + n);

// Any / All
bool anyInNY = people.Any(p => p.City == "NY");
bool allAdults = people.All(p => p.Age >= 18);
```

---

## Async & Await

Asynchronous programming with `Task`/`ValueTask` and `await`.

### Basics
- `async` marks a method containing `await`
- `await` pauses until the awaited task completes without blocking the thread

```csharp
public static async Task<string> DownloadAsync(HttpClient http, string url)
{
    using var response = await http.GetAsync(url);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync();
}
```

### Task vs ValueTask
- Prefer `Task` by default (simpler, widely supported)
- `ValueTask` can reduce allocations when a method often completes synchronously
- A `ValueTask` instance should be awaited only once

```csharp
public static ValueTask<int> MaybeSyncAsync(bool fast)
{
    if (fast) return new ValueTask<int>(42); // already completed
    return new ValueTask<int>(Task.Run(() => 42));
}
```

### Exception handling in async methods
- Exceptions are captured in the `Task` and rethrown on `await`
- Use `try`/`catch` around `await`
- With `Task.WhenAll`, catch `AggregateException` or inspect `Task.Exception`

```csharp
try
{
    var html = await DownloadAsync(new HttpClient(), "https://example.com");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"Network error: {ex.Message}");
}

// Parallel tasks
var urls = new[] { "https://a", "https://b" };
var tasks = urls.Select(u => DownloadAsync(new HttpClient(), u));
try
{
    var results = await Task.WhenAll(tasks);
}
catch
{
    foreach (var t in tasks)
    {
        if (t.IsFaulted)
            Console.WriteLine(t.Exception);
    }
}
```

Note: Use `async void` only for event handlers. Otherwise prefer `async Task`.

---

## Reflection

Inspect metadata and interact with objects/types at runtime.

### Loading types at runtime
```csharp
var current = Assembly.GetExecutingAssembly();
foreach (var type in current.GetTypes())
{
    Console.WriteLine(type.FullName);
}

// Load by name (assembly must be available)
var asm = Assembly.Load("MyLibrary");
var t = asm.GetType("MyLibrary.Services.EmailService");
```

### `Activator.CreateInstance`, `PropertyInfo`
```csharp
Type personType = typeof(Person);
object person = Activator.CreateInstance(personType)!; // calls parameterless ctor

PropertyInfo? nameProp = personType.GetProperty("Name");
nameProp!.SetValue(person, "Alice");
var name = (string?)nameProp.GetValue(person);
```

---

## Attributes

Metadata you can attach to program elements (types, members, params, etc.).

### Built-in examples: `[Obsolete]`, `[Serializable]`
```csharp
public class Demo
{
    [Obsolete("Use NewMethod instead.")]
    public void OldMethod() { }

    public void NewMethod() { }
}

[Serializable]
public class LegacySerializable
{
    public int Id { get; set; }
}
```

### Custom attributes
```csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, Inherited = false)]
public sealed class AuditAttribute : Attribute
{
    public string Actor { get; }
    public AuditAttribute(string actor) => Actor = actor;
}

[Audit("system")]
public class PaymentService { }

// Reading attributes via reflection
var attr = typeof(PaymentService).GetCustomAttribute<AuditAttribute>();
Console.WriteLine(attr?.Actor); // "system"
```

---

## Pattern Matching

Concise type/shape tests and transformations.

### Type and property patterns
```csharp
object o = "hello";
if (o is string s && s.Length > 3)
{
    Console.WriteLine(s.ToUpperInvariant());
}
```

### Relational and logical patterns
```csharp
int n = 42;
var category = n switch
{
    < 0 => "negative",
    0 => "zero",
    > 0 and < 100 => "small",
    _ => "large"
};
```

### Switch expressions (with types)
```csharp
abstract record Shape;
record Circle(double Radius) : Shape;
record Rectangle(double Width, double Height) : Shape;

double Area(Shape s) => s switch
{
    Circle { Radius: var r } => Math.PI * r * r,
    Rectangle { Width: var w, Height: var h } => w * h,
    _ => throw new NotSupportedException()
};
```

---

## Memory Management

### Garbage collection basics
- Managed heap with generations (0, 1, 2) for efficiency
- GC reclaims unreachable objects automatically
- Finalizers (`~TypeName()`) are costly and non-deterministic; avoid unless necessary
- Large Object Heap (LOH) for large allocations

### `IDisposable` & `using`
- Deterministically release unmanaged resources (files, sockets, handles)

```csharp
public sealed class FileWriter : IDisposable
{
    private readonly StreamWriter _writer;
    public FileWriter(string path) => _writer = new StreamWriter(path);

    public void Write(string text) => _writer.WriteLine(text);

    public void Dispose()
    {
        _writer.Dispose();
        GC.SuppressFinalize(this);
    }
}

// using statement (scope)
using (var fw = new FileWriter("log.txt"))
{
    fw.Write("Hello");
}

// using declaration (C# 8+)
using var fw2 = new FileWriter("log2.txt");
fw2.Write("World");
```

For async resources, use `IAsyncDisposable` and `await using`.

---

## Exercises

### 1) Delegate with `Func`
- Create a method `Transform<TIn, TOut>(IEnumerable<TIn>, Func<TIn, TOut>)` that applies a transform and returns the results
- Use it to convert an array of integers to their string representations with a prefix

Solution:
```csharp
IEnumerable<TOut> Transform<TIn, TOut>(IEnumerable<TIn> source, Func<TIn, TOut> projector)
{
    foreach (var item in source)
        yield return projector(item);
}

var input = new[] { 1, 2, 3 };
var output = Transform(input, n => $"No.{n}");
Console.WriteLine(string.Join(", ", output)); // No.1, No.2, No.3
```

### 2) BankAccount event on low balance
- Implement `BankAccount` that raises `BalanceLow` when `Balance` <= threshold after withdrawal
- Subscribe to the event and print a warning

Solution: see Delegates & Events section example.

---

## Theory Check

- Difference between delegate & event?
  - A delegate is a type-safe function pointer. An event is a restricted delegate exposure used for pub/sub; only the declaring type can raise it, others can only subscribe/unsubscribe.

- When to use `where T : class` constraint?
  - When your API must accept only reference types: you rely on `null` semantics, identity/reference behavior, or the logic only makes sense for reference types (e.g., entity tracking, caching by reference).

---

## Quick Reference
- Properties: use `init` for immutable setup
- Generics: constrain types to express requirements
- Delegates: `Action`/`Func`/`Predicate`; Events for pub/sub
- LINQ: method and query syntax are equivalent; choose by readability
- Async: prefer `Task`; use `try/catch` around `await`; `WhenAll` aggregates
- Reflection: use `Activator`, `PropertyInfo` for dynamic scenarios
- Attributes: `[Obsolete]`, `[Serializable]`, plus custom attributes
- Patterns: switch expressions, relational/type patterns
- Memory: GC is automatic; free resources via `IDisposable` and `using`