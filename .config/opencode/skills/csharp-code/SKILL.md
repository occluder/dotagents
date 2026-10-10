---
name: csharp-code
description: Defines preferences and guidelines for writing C# code.
---

This skill defines personal code-writing preferences and guidelines for C# code.

This skill should dictate the *default* way of writing code; Use this skill unless the user explicitly asks otherwise, or if there is another set of guidelines defined elsewhere. 

# Omit the `private` keyword

Omit the `private` keyword where possible, it is the default accessibility level and can be easily inferred by the user.

You should generally always try to do this as the keyword is simply boilerplate.

# Prefer primary constructors

Assign-only constructors should be turned into primary constructors

<bad_example>
```csharp
public class User
{
    string _name;
    int _age;

    public User(string name, int age)
    {
        _name = name;
        _age = age;
    }
}
```
</bad_example>

<good_example>
```csharp
public class User(string name, int age)
{
    // ...
}
```
</good_example>

This is especially noteworthy for dependency injection:

<service_example>
```csharp
public class MyService(ILogger<MyService> logger, IOtherService otherService) : IMyService
{
}
```
</service_example>

# Avoid creating unecessary methods

Do not create methods that are called from only one location containing simple logic (1-10 lines of code)

<bad_example>
```csharp
static string GetLocalIconPath(ItemRecord item)
{
    return Path.Combine(IconCacheDirectory, Path.GetFileName(item.Icon));
}
```
</bad_example>

The example above is a snippet from a real project. The method defined consists of only 1 line of code, and is only called from one location within the same class.

Defining methods such as this clutters classes and adds an unnecessary navigation cost to users. Instead, write the code directly in the caller's location. It is very much easier to deduce what is going on in one slightly bloated method rather than multiple scattered ones.

# Prefer `ValueTask` when async is conditional

Use `ValueTask` over `Task` when the method can return without running async code.

<good_example>
```csharp
async ValueTask<Item[]> GetItems()
{
    if (_items is not null)
    {
        return _items;
    }

    _items = await FetchItems();
    return _items;
}
```
</good_example>

# Instance Method vs. Extension Method

**Core heuristic:** A method belongs *on* the type if it is central to the type's responsibility or requires non-public state. It should be an extension method if it is a convenience operation that could be written entirely against the type's public surface.

## Decision checklist

1. **Does it need private/protected members?** -> Instance method (no choice).
2. **Is it core to the type's identity?** Would a consumer be surprised *not* to find it on the type? -> Instance method.
3. **Does it need to be virtual / overridable?** -> Instance method.
4. **Is it a transformation, query, or convenience built on public API?** -> Extension method.
5. **Would putting it on the class bloat it with tangential concerns?** -> Extension method.
6. **Do you not own the type?** → Extension method (no choice).

When in doubt, ask: *"If someone reads the class name, would they expect this method to be there?"*

<bad_example>
```csharp
public static class OrderExtensions
{
    // BAD: Calculates discount using internal pricing rules the class should own.
    // Consumers expect Order to know how to price itself.
    public static decimal CalculateTotal(this Order order, PricingEngine engine)
    {
        return order.Items.Sum(i => engine.GetPrice(i) * i.Quantity) - order._internalDiscount;
        // Won't even compile — _internalDiscount is private.
        // The fact that you *want* private access is the signal.
    }
}
```
</bad_example>

<good_example>
```csharp
// Instance method — core responsibility, needs internal state
public class Order
{
    private decimal _loyaltyDiscount;

    public decimal CalculateTotal(PricingEngine engine)
    {
        return Items.Sum(i => engine.GetPrice(i) * i.Quantity) - _loyaltyDiscount;
    }
}
```
</good_example>

<bad_example>
```csharp
// Instance method for a tangential convenience that bloats the class
public class Customer
{
    public string Name { get; set; }
    public string Email { get; set; }

    // BAD: Formatting for a CSV export is not core to Customer.
    // Every new output format adds another method here.
    public string ToCsvRow() => $"\"{Name}\",\"{Email}\"";
    public string ToJsonSnippet() => $"{{\"name\":\"{Name}\"}}";
    public string ToVCard() => $"BEGIN:VCARD\nFN:{Name}\nEND:VCARD";
}
```
</bad_example>

<good_example>
```csharp
// Extension methods — convenience formatting built on public API
public static class CustomerFormattingExtensions
{
    public static string ToCsvRow(this Customer c) => $"\"{c.Name}\",\"{c.Email}\"";
    public static string ToJsonSnippet(this Customer c) => $"{{\"name\":\"{c.Name}\"}}";
    public static string ToVCard(this Customer c) => $"BEGIN:VCARD\nFN:{c.Name}\nEND:VCARD";
}
```
</good_example>

<good_example>
```csharp
// Extension on a type you don't own — classic use case
public static class DateTimeExtensions
{
    public static bool IsWeekend(this DateTime dt) =>
        dt.DayOfWeek is DayOfWeek.Saturday or DayOfWeek.Sunday;
}
```
</good_example>

Define extension classes in `Extensions/`. Use broad terms (e.g., `NumericExtensions`) over specialized (e.g., `Int32Extensions`) to minimize file count.

# Code formatting

This section highlights preferred code style and formatting practices. Do NOT conform to the styles in the files you edit unless explicitly stated by a user or different skill/guidelines.

## Naming Conventions

- Fields should be camelCase and starting with underscore
    - Example: _myField
- Primary constrcutor field should be camelCase, no underscore
    - Example: myField
- Constants should follow upper snake-case
    - Example: MAX_WAIT_MINS

## No single-line if statements

Avoid:
<bad_example>
```csharp
if (users.Length == 0) return;
```
</bad_example>

Prefer:
<good_example>
if (users.Length == 0)
{
    return;
}
</good_example>
