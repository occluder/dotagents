---
name: csharp-code
description: >-
  Defines the user's preferred C# code style. Use when writing, reviewing, or
  refactoring C# in any project.
version: 2.0.0
license: MIT
tags:
  - csharp
  - dotnet
  - style
  - conventions
  - code-review
---

# C# house style

This skill dictates the *default* way to write C# in any codebase. It describes how code should look, not how any particular project currently looks — do not assume any specific repository, framework, or vendor API is present.

Use it unless the user explicitly asks otherwise, or the project ships its own guidelines. Precedence, highest first: the user's current instruction, the project's explicit guidelines (`CONTRIBUTING`, `AGENTS.md`, repo-wide conventions), this skill, the style of the file you are editing. Do not conform to the surrounding file's style just because it is there; do not reformat code you are not otherwise touching, and do not perform churn-only refactors unless asked.

Where this skill and the project's `.editorconfig` disagree about pure formatting (whitespace, using order, brace placement), the `.editorconfig` wins — it is the formatting source of truth.

Everything below assumes modern .NET (C# 12+). Check the project's target framework and `LangVersion` before reaching for the newest feature, and use the newest form that is available rather than an older idiom.

# Omit the `private` keyword

`private` is the default accessibility level, so it is boilerplate the reader can infer. Omit it on fields, methods, properties, nested types and static members alike.

```csharp
public class CacheStore
{
    readonly SemaphoreSlim _gate = new(1, 1);
    static readonly TimeSpan _defaultTtl = TimeSpan.FromMinutes(5);

    bool TryGetFresh(string key, out Entry entry) { ... }
    static string Normalize(string key) => key.Trim().ToLowerInvariant();
}
```

Keep an explicit modifier when it is not `private` (`public`, `internal`, `protected`, `file`), when the language requires it (explicit interface implementation, `protected virtual`), or when the project's analyzer rules demand it. Never churn existing `private` in a file you are not otherwise editing.

# Prefer primary constructors

Assign-only constructors become primary constructors. Parameters are camelCase with no underscore and are referenced directly by the class body.

<bad_example>
```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }
}
```
</bad_example>

<good_example>
```csharp
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger) : IOrderService
{
    // use repository / logger directly
}
```
</good_example>

This is the default for dependency-injected services, handlers, controllers, commands and tools. Put one parameter per line once there are roughly four or more, with `)` and then ` : BaseType` on separate lines. Keep an explicit constructor when it does real work: validation, mapping, reflection, computed state.

# Do not create single-use helper methods

Do not extract a method that is called from exactly one place and holds simple logic (roughly 1-10 lines).

<bad_example>
```csharp
static string GetLocalIconPath(ItemRecord item)
{
    return Path.Combine(IconCacheDirectory, Path.GetFileName(item.Icon));
}
```
</bad_example>

Write that code at the caller's location instead. Methods like this clutter classes and add a navigation cost: it is easier to deduce what is happening from one slightly bloated method than from several scattered ones.

Define a method when it is called from two or more places, is long enough that inlining would bury the caller's intent, implements an interface or overridden member, or is a genuinely reusable transformation. The counter-pressure is size: once a method passes roughly 80-100 lines, split it by responsibility instead of letting it grow.

# Prefer `ValueTask` when async is conditional

Use `ValueTask` / `ValueTask<T>` when the method can return without doing async work; use `Task` / `Task<T>` when it is always asynchronous.

<good_example>
```csharp
async ValueTask<Item[]> GetItemsAsync()
{
    if (_items is not null)
    {
        return _items;
    }

    _items = await FetchItemsAsync();
    return _items;
}
```
</good_example>

`ValueTask` is also right for handler-style methods that sometimes complete synchronously; `return default;` is the normal synchronous completion. Do not add `.ConfigureAwait(false)` unless the project already does.

# Instance method vs. extension method

A method belongs *on* the type if it is central to the type's responsibility or requires non-public state. It belongs in an extension method if it is a convenience operation that could be written entirely against the type's public surface.

Decision order:

1. Does it need private/protected members? -> instance method (no choice).
2. Is it core to the type's identity? Would a consumer be surprised *not* to find it on the type? -> instance method.
3. Does it need to be virtual/overridable? -> instance method.
4. Is it a transformation, query, or convenience built on public API? -> extension method.
5. Would putting it on the class bloat it with a tangential concern (one method per output format)? -> extension method.
6. Do you not own the type? -> extension method (no choice).

When in doubt ask: *"If someone reads the class name, would they expect this method to be there?"*

Group extensions into one broad static class per concern, not one file per type: `NumericExtensions` (broad) over `Int32Extensions` (narrow). Use C# 14 `extension(...)` blocks when available; otherwise classic `this`-parameter extension methods.

# Prefer modern language features

Reach for the newest form of an idiom. Existing older code in the project is not a reason to write old C#.

- Collection expressions: `List<Item> items = [];`, `[.. source]`, `Method([first, .. rest])`.
- `field` for cached or lazy properties instead of a hand-written backing field: `public string Name => field ??= GetType().Name;`
- Target-typed `new()` when the type is already stated (fields, or a type you keep deliberately): `readonly SemaphoreSlim _gate = new(1, 1);`, `Dictionary<string, int> counts = new();`, `new(0x5865f2)`.
- Pattern matching over casts and equality checks: `x is null`, `x is not null`, `status is Status.Pending or Status.Retrying`, switch expressions ending in `_ => throw new ArgumentOutOfRangeException(nameof(x), ...)`. A null guard combined with member checks collapses into a single pattern — see *Collapse null checks into patterns* below.
- `record` / `record struct` for data carriers, positional when the shape is the point: `public record Money(decimal Amount, string Currency);`
- `required` and object initializers for input DTOs; `init` for immutable-after-construction data.
- File-scoped types for helpers used by one file only: `file readonly record struct Point(int X, int Y);`
- Raw string literals for multi-line text and prompts; interpolated raw strings (`$$"""..."""`) when the text itself contains braces.
- `Task.WhenAll` for independent parallel I/O, built from a `Select` of tasks.

# Namespaces, usings and layout

- File-scoped namespace matching the folder structure: `namespace MyApp.Services;`
- `using` directives outside the namespace, one group, no blank lines inside it. Sort them; if the project sorts `System.*` first, do the same.
- Four-space indent, braces on the same line (K&R) including for `else`, `catch` and `finally`, one blank line after the using block and between members.
- Inside a method, use blank lines to separate logical steps, and put a blank line before a bare `return;` that follows a block.
- Do not use `this.` unless it disambiguates a parameter. Do not add `#region`.
- Long lines are acceptable; when a call stops being readable, break its arguments one per line with the closing `)` on its own line. Fluent chains break before the dot and indent one level:

```csharp
builder.AddOptions()
    .WithRetries(3)
    .WithTimeout(TimeSpan.FromSeconds(30));
```

# Declarations: prefer `var`

Default to `var`. The type is almost always evident from the initializer, and naming a long generic type costs the reader more than the inference does. `var` is correct even when the value alone is not the giveaway — never restate a long type just to be explicit.

```csharp
var retryCount = 0;                                  // int is obvious
var request = new OrderRequest { Id = id };          // obvious from the initializer
var orders = await repository.GetOrdersAsync(id);    // obvious from the call
var map = new Dictionary<string, List<Order>>();     // a long type, said once rather than twice
var total = items.Sum(x => x.Price);

for (var i = 0; i < length; i++) { ... }
foreach (var item in items) { ... }
```

Write the type out only when the initializer hides it and the stated type is short or carries the meaning — an interface, base type, or upcast where *that* type is the point:

```csharp
HttpResponseMessage response;                        // declared first, assigned later
foreach (KeyValuePair<string, Item> kvp in map)      // the pair type is the intent
IEnumerable<Order> pending = orders.Where(IsPending); // interface is deliberate
ReadOnlySpan<char> span = text.AsSpan();             // span/buffer locals
```

Never write the type twice in one statement; when you do keep the type, use target-typed `new()` (`Dictionary<string, int> counts = new();`).

# Naming

| Symbol | Convention | Example |
|---|---|---|
| Types, members, namespaces, enum values | PascalCase | `OrderService`, `GetOrdersAsync`, `Status.Pending` |
| Interfaces | `I` + PascalCase | `IOrderRepository` |
| Instance fields | `_camelCase` | `_logger`, `_retryCount` |
| Primary-constructor parameters | camelCase, no underscore | `repository`, `logger` |
| Private/internal constants | UPPER_SNAKE_CASE | `MAX_RETRIES`, `CACHE_KEY` |
| Private/internal static readonly fields | `_camelCase` | `_defaultTtl`, `_jsonOptions` |
| Public constants and static readonly fields | PascalCase | `MaxRetries`, `DefaultTimeout` |
| Locals and parameters | camelCase; terse is fine when the scope is small | `db`, `res`, `kvp`, `ex`, `i` |
| Cancellation tokens | `cancellationToken` in signatures, `ct` inside a method body | `CancellationToken ct` |
| Async methods | no mandatory `Async` suffix; follow the interface and neighbours | `Run`, `HandleAsync` |
| Extension members | verb or query phrase | `ToCsvRow`, `SimilarityWith`, `IsWeekend` |

Do not prefix private helpers or static methods with `_` — that marks fields. Nested DTOs and input types live inside the class that owns them unless the project keeps all DTOs in a shared location.

# Braces and control flow

- Braces are always required, even for a single statement. Never `if (items.Length == 0) return;`.

<good_example>
```csharp
if (items.Length == 0)
{
    return;
}
```
</good_example>

- Guard clauses and early returns at the top of a method instead of nesting.
- Null and negative guards first: `if (reply is null) { return; }`.
- `switch` expressions to select a value, `switch` statements for side-effecting branches.
- A single-statement `case` may stay on one line (`case '\\': sb.Append('\\'); break;`); anything longer gets a block.
- Expression-bodied members are fine for one-liners, including interface members: `public decimal Total => Items.Sum(x => x.Price);` Use a block body for anything longer than one clear expression.

# Collapse null checks into patterns

The `is` operator performs the null check for you, so "not null, and its member is something" is a single expression. Never write two conditions where one pattern says the same thing.

<bad_example>
```csharp
if (request is not null && request.RetryCount > 2)
if (user is not null && user.Name is not null && user.Name.Length > 0)
if (result is not null && result.Flag == true)
if (order is not null && order.Total > 0 && order.Items.Count > 0)
```
</bad_example>

<good_example>
```csharp
if (request?.RetryCount is > 2)
if (user?.Name is { Length: > 0 })
if (result?.Flag is true)
if (order is { Total: > 0, Items.Count: > 0 })
```
</good_example>

A `?.` chain followed by `is <pattern>` is false when anything along the chain is null — identical to `x is not null && x.A...`, so the collapse does not change behaviour. Rules:

- Several member checks on the same object become one property pattern: `x is { A: ..., B: ... }`, and it evaluates `x` once.
- Chains compose freely: `if (a?.B?.C is { Length: > 0 })` means `a`, `B` and `C` are all non-null and `C` is not empty. Works for strings, arrays and any collection with `Length`/`Count`.
- Prefer `is true` / `is false` / `is not true` over `== true` / `!= false`, including on `bool?`.
- Prefer `is null` / `is not null` over `== null` / `!= null`.
- `{ Length: > 0 }` is exactly "non-null and non-empty"; it does not trim, so use `string.IsNullOrWhiteSpace` when whitespace-only input must also be rejected.
- Bind the value when you need it: `if (order?.Customer is { } customer)` — `{ }` is the "not null" pattern.
- `?.` needs a reference type or a nullable; for a non-nullable struct member use the pattern on the member directly (`if (point.X is > 2)`).
- Keep the explicit two-condition form when null and "present but out of range" need different handling (a different log message or reply), or when the second condition does not come from the first receiver: `if (a is not null && b.IsReady)` cannot be folded.

# Async, cancellation and lifetime

- `CancellationToken` is the last parameter of a method. Name it `cancellationToken` in the signature and `ct` inside the body.
- Thread the token through every call it can reach. Only pass `CancellationToken.None` when the incoming token is already cancelled (cleanup paths, error replies) or in a `finally`.
- Prefer `using var` for disposables, including `using var cts = new CancellationTokenSource(timeout);`.
- Treat `OperationCanceledException` as a distinct, expected path rather than a failure; an empty `catch (OperationCanceledException) { }` is acceptable when cancellation is the normal exit.
- Never `async void`. Fire-and-forget is `_ = SomeAsync();` — the discard makes the intent explicit, and the method should handle its own exceptions.
- Do not add locks or `Interlocked` around state unless it is genuinely shared. When it is, use a `readonly object _gate = new();` with `lock (_gate)` for synchronous state, or a `SemaphoreSlim` when the critical section awaits.

# Errors, results and logging

- Throw for programmer and precondition errors (`ArgumentException`, `ArgumentOutOfRangeException`, `NotSupportedException`); use `FormatException` for input the user typed. Catch at boundaries (entry points, handlers, command or tool methods) and log there rather than deep inside.
- When callers are expected to handle failure as data, return a result envelope instead of throwing: a small record with static `Success` / `Failure` factories and an `IsSuccess` check.
- Never swallow an exception without logging it. Never surface raw exception or provider text to an end user — log the detail, reply generically.
- Structured logging only. Never interpolate into the message template:

<bad_example>
```csharp
logger.LogInformation($"Loaded {count} items");
```
</bad_example>

<good_example>
```csharp
logger.LogInformation("Loaded {ItemCount} items", count);
```
</good_example>

- Placeholders are PascalCase. Use `@` when the object should be destructured: `logger.LogWarning(ex, "Retry failed for {@Request}", request);`
- Pass the exception as the first argument, then the message, then the values.
- Level choice: `Trace`/`Verbose` for hot-path detail behind an `IsEnabled` guard; `Debug` for cache hits and misses, external calls, and other development-time detail; `Information` for state changes (startup, shutdown, deletions, dispatch); `Warning` for recoverable or degraded paths; `Error` for caught exceptions and failed operations; `Critical`/`Fatal` for unrecoverable failures.
- Inject the logger (`ILogger<T>` or the project's equivalent) through the constructor. Do not use a static logger or create one inside a method, unless the project already has an established static logging facade.

# Services and dependencies

- Register dependencies in the composition root, grouped into blocks under `// Section` comments (logging, data access, external clients, application services), with the lifetime chosen deliberately: singleton for stateful or shared services, scoped per request where the framework has scopes, transient for stateless helpers.
- Hide non-owned infrastructure behind a small interface (`IFooClient`) and depend on the interface so implementations stay swappable and testable.
- Metrics and telemetry come from an injected factory or meter bag rather than ad-hoc static counters; name them consistently with the project's existing prefix and attach values as tags rather than encoding them in the metric name.
- For cache or remote-store access, use the cache-aside shape: read, return on hit, otherwise fetch, write back with an explicit TTL, return. Build keys from stable lowercase delimited segments (a namespace prefix plus the identifying parts), declare fixed keys as a `const string`, and keep TTLs in `static readonly TimeSpan` fields. Use a transaction or a single batched call when several writes must land together.
- For independent I/O operations, start them together and await with `Task.WhenAll` rather than awaiting serially.

# Comments and member layout

- Comments explain *why*, not *what*. Keep them terse and delete them once the code stops needing them.
- Do not add XML doc comments (`///`) by default; use them when the project's existing public API documents itself that way.
- Order inside a type: constants, then static readonly fields, then instance fields, then public members, then private helpers placed near their callers. Keep the file's existing order when editing.

# Verification

Before claiming a change works:

1. Build with the project's own build command and leave no new warnings behind.
2. For anything you are guessing at — serialization behaviour, LINQ semantics, async and threading, regex, framework or language-version quirks — run a throwaway experiment instead of asserting it. Prefer a scratch/experiment skill such as `csharp-lab` when it is installed; otherwise use a temporary file-based or throwaway project outside the repo. Never present untested behaviour as fact, and never fabricate output you did not produce.
3. Re-read your own diff against this file: no reintroduced `private` on new members, no long type restated where `var` would read better, no interpolation inside log templates, no braceless single-statement `if`, no single-use helper methods, no `this.` that disambiguates nothing, and no `x is not null && x.Y …` that `x?.Y is …` states in one expression.
