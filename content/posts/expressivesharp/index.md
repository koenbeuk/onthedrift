---
date: "2026-10-02"
tags:
- .NET
- EFCore
- EFCore.Projectables
- ExpressiveSharp
title: "ExpressiveSharp: Projectables, reimagined"
---

Back in 2021 I wrote about a new library called [EntityFrameworkCore.Projectables](https://github.com/EFNext/EntityFrameworkCore.Projectables). The idea was simple: mark a property or method with `[Projectable]`, let a source generator produce a companion expression tree for it, and EF Core can suddenly translate your computed properties to SQL. No more duplicating logic, no more over-fetching.

A lot has happened since then. Projectables grew support for block-bodied methods, switch expressions, pattern matching, constructors and more. A big shout-out goes to [Fabien Ménager](https://github.com/PhenX) here, who is behind most of those features, along with a docs website, code fixes and a whole lot of performance work. But every one of those features had to fight the same wall, and at some point I had to admit that the wall was not going anywhere. So I started over. Today I want to introduce [ExpressiveSharp](https://github.com/EFNext/ExpressiveSharp), a reimagined Projectables that is, well, just done better. Let's dive in and see what changed and, more importantly, why.

### The wall we kept hitting

Let's start with a query that looks perfectly reasonable:

```csharp
dbContext.Orders
    .Where(o => o.User?.EmailAddress != null)
    .Select(o => new { o.Id, Email = o.User?.EmailAddress });
```

This does not compile:

```text
error CS8072: An expression tree lambda may not contain a null propagating operator
```

Expression trees (`Expression<Func<...>>`) were introduced together with LINQ in C# 3.0, back in 2007. Since then we got null-conditional operators, pattern matching, switch expressions, tuples, index and range, collection expressions... and most of them don't work inside an expression tree lambda. The compiler simply refuses. There are [good reasons](https://github.com/dotnet/csharplang/discussions/158) for this. Every LINQ provider out there walks expression trees with its own visitor, and the moment the compiler emits a new node type, every one of those providers breaks at the same time. `System.Linq.Expressions` itself is effectively archived these days. It's not going to change.

Projectables worked around this for `[Projectable]` members. Since we generate the expression ourselves, we can take a `?.` and rewrite it into something an expression tree does understand. The problem is how we did that. Projectables rewrites C# *syntax*. Every language feature needed its own hand-written rewriter: one for null-conditionals, one for switch expressions, one for patterns, one for interpolation. Every rewriter came with its own edge cases, and anything we didn't explicitly handle ended up as a diagnostic. Positional patterns? Not supported. A `?.` in your projectable? That's an error until you pick a `NullConditionalRewriteSupport` strategy for that member.

And even when all of that works, it only works *inside* a `[Projectable]` member. The query from above, the one you're actually writing, is still stuck in 2007.

There was a second thing that bothered me. Three days after publishing that first post, I opened [an issue](https://github.com/EFNext/EntityFrameworkCore.Projectables/issues/4) on the Projectables repo:

> The current library is tied to EFCore (by name convention only). It would be relatively trivial to spawn of a new project that encapsulates the Projectable attribute, code generation and the ExpandProjectables() extension methods so that non EF projects can also take advantage of projectables.

I closed it a while later as out of scope. It never really left my mind though.

### Why not just Projectables v7?

Because the things I wanted to change are exactly the things Projectables is built on. Syntax rewriting, being tied to EF Core, the compatibility modes and per-member configuration knobs. Changing any of those is a rewrite, and a rewrite that breaks everyone who depends on Projectables today. A new name gives us a clean slate without pulling the rug out from under existing users. And as we'll see later, migrating is mostly automated anyway.

### Meet ExpressiveSharp

Let's revisit our old friend, the User:

```csharp
public class User
{
    public int Id { get; set; }
    public string? EmailAddress { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public ICollection<Order> Orders { get; set; }

    [Expressive]
    public string FullName => $"{FirstName} {LastName}";
}
```

Looks familiar? `[Projectable]` became `[Expressive]`, and that's about it. Enabling it in EF Core is equally familiar:

```csharp
services.AddDbContext<ApplicationDbContext>(options =>
    options
        .UseSqlite(...)
        .UseExpressives());
```

And querying it:

```csharp
dbContext.Users.Select(x => new { x.Id, x.FullName });
```

```sql
SELECT "u"."Id", "u"."FirstName" || ' ' || "u"."LastName" AS "FullName"
FROM "Users" AS "u"
```

So far, nothing new: just like with Projectables, `FullName` gets inlined into the query and translated to SQL. The interesting part is what gets generated. Here is the generated code for `FullName` (trimmed a bit for readability):

```csharp
// [Expressive]
// public string FullName => $"{FirstName} {LastName}";
static Expression<Func<User, string>> FullName_Expression()
{
    var p__this = Expression.Parameter(typeof(User), "@this");
    var expr_0 = Expression.Property(p__this, typeof(User).GetProperty("FirstName", ...));
    var expr_1 = Expression.Constant(" ", typeof(string));
    var expr_2 = Expression.Property(p__this, typeof(User).GetProperty("LastName", ...));
    var expr_3 = Expression.Call(typeof(string).GetMethod("Concat", ..., new[] { typeof(string), typeof(string), typeof(string) }, null), expr_0, expr_1, expr_2);
    return Expression.Lambda<Func<User, string>>(expr_3, p__this);
}
```

Projectables used to generate a *lambda* and let the compiler turn that into an expression tree, which is exactly why it had to rewrite modern syntax into old syntax first. ExpressiveSharp skips that step entirely. It doesn't look at your syntax at all. It looks at Roslyn's `IOperation` tree, which is the compiler's fully resolved view of your code. By the time we get to see `$"{FirstName} {LastName}"`, the compiler has already decided that this is a call to `string.Concat(string, string, string)`. Implicit conversions, operator overloads, what a pattern actually tests for: all of that is already figured out. We just map each operation to the matching `Expression.*` factory call.

This is the big one. Instead of a growing pile of rewriters for individual syntax features, there is one mapping from compiler operations to expression nodes. Switch expressions, every kind of pattern (yes, including positional and list patterns), tuples, `with` expressions, index and range, collection expressions and checked arithmetic all fall out of that same approach. Most of the time the result only uses node types that have been around since 2007, so LINQ providers don't notice a thing. Where we do need something newer, like a block with local variables, `UseExpressives()` rewrites it back into a shape EF Core understands before it ever sees it.

### Modern C# in the query itself

Remember our query that didn't compile? Let's expose our sets slightly differently:

```csharp
public class ApplicationDbContext : DbContext
{
    public ExpressiveDbSet<User> Users => this.ExpressiveSet<User>();
    public ExpressiveDbSet<Order> Orders => this.ExpressiveSet<Order>();
}
```

And now:

```csharp
dbContext.Orders
    .Where(o => o.User?.EmailAddress != null)
    .Select(o => new
    {
        o.Id,
        Email = o.User?.EmailAddress,
        Label = o.Status switch
        {
            OrderStatus.Pending => "Waiting on payment",
            OrderStatus.Shipped or OrderStatus.Delivered => "On its way",
            _ => "Unknown"
        }
    });
```

This should blow up, right? A null-conditional operator *and* a switch expression with an `or` pattern, right there in our query. Yet it compiles, and EF Core produces:

```sql
SELECT "o"."Id", "u"."EmailAddress" AS "Email", CASE
    WHEN "o"."Status" = 0 THEN 'Waiting on payment'
    WHEN "o"."Status" IN (1, 2) THEN 'On its way'
    ELSE 'Unknown'
END AS "Label"
FROM "Orders" AS "o"
INNER JOIN "Users" AS "u" ON "o"."UserId" = "u"."Id"
WHERE "u"."EmailAddress" IS NOT NULL
```

So how does this work? On an `ExpressiveDbSet<T>`, `Where` and `Select` accept a plain `Func<...>` instead of an `Expression<Func<...>>`, which is why the compiler is happy to accept modern syntax. Then at compile time, a second source generator uses [interceptors](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12#interceptors) to replace each of those call sites with code that builds the expression tree (using the same `IOperation` approach as before) and forwards it to the regular `Queryable` method. The delegate is never actually called. There is no runtime conversion from delegate to expression either, it's all done at build time.

Also note that we didn't have to configure anything for the `?.` operator. ExpressiveSharp always generates the faithful version (`o.User != null ? o.User.EmailAddress : null`), and `UseExpressives()` strips that null check again before EF Core sees it, since SQL already propagates nulls for us. The `NullConditionalRewriteSupport` knob is gone.

### Real world scenarios

Let's bring back the scenario from the original post: users with orders, and orders with items. This time we also want to put our users into tiers based on how much they've spent:

```csharp
public class Order
{
    ...
    [Expressive]
    public double PriceSum => Items.Sum(i => i.Quantity * i.UnitPrice);
}

public class User
{
    ...
    [Expressive]
    public double TotalSpent => Orders.Sum(o => o.PriceSum);

    [Expressive]
    public string Tier => TotalSpent switch
    {
        >= 1000 => "Gold",
        >= 100 => "Silver",
        _ => "Bronze"
    };
}

dbContext.Users.Select(x => new { x.FullName, x.Tier });
```

`Tier` uses `TotalSpent`, which uses `PriceSum`. All of them get expanded:

```sql
SELECT "u"."FirstName" || ' ' || "u"."LastName" AS "FullName", CASE
    WHEN (
        SELECT COALESCE(SUM((
            SELECT COALESCE(SUM(CAST("o0"."Quantity" AS REAL) * "o0"."UnitPrice"), 0.0)
            FROM "OrderItems" AS "o0"
            WHERE "o"."Id" = "o0"."OrderId")), 0.0)
        FROM "Orders" AS "o"
        WHERE "u"."Id" = "o"."UserId") >= 1000.0 THEN 'Gold'
    WHEN (
        SELECT COALESCE(SUM((
            SELECT COALESCE(SUM(CAST("o2"."Quantity" AS REAL) * "o2"."UnitPrice"), 0.0)
            FROM "OrderItems" AS "o2"
            WHERE "o1"."Id" = "o2"."OrderId")), 0.0)
        FROM "Orders" AS "o1"
        WHERE "u"."Id" = "o1"."UserId") >= 100.0 THEN 'Silver'
    ELSE 'Bronze'
END AS "Tier"
FROM "Users" AS "u"
```

What about methods with arguments? Let's look for big recent orders that are no longer pending, using a property pattern:

```csharp
public class Order
{
    ...
    [Expressive]
    public bool IsBigRecentOrder(DateTime since)
        => this is { Status: not OrderStatus.Pending, PriceSum: >= 100 } && CreatedDate > since;
}

var since = DateTime.UtcNow.AddDays(-7);
dbContext.Orders.Where(x => x.IsBigRecentOrder(since));
```

```sql
SELECT "o"."Id", "o"."CreatedDate", "o"."Status", "o"."UserId"
FROM "Orders" AS "o"
WHERE "o"."Status" <> 0 AND (
    SELECT COALESCE(SUM(CAST("o0"."Quantity" AS REAL) * "o0"."UnitPrice"), 0.0)
    FROM "OrderItems" AS "o0"
    WHERE "o"."Id" = "o0"."OrderId") >= 100.0 AND "o"."CreatedDate" > @since
```

Our `since` argument was captured as a parameter, just like it was with Projectables, so EF Core can cache and reuse the query plan.

### Things Projectables never could

The new foundation also made room for a couple of features that were long overdue.

**Members you don't own.** Projectables had `UseMemberBody`, which let you point a member at an alternative implementation, but only within the same type. `[ExpressiveFor]` works for any type, including the BCL. Say we want to use `Math.Clamp` in a query, which EF Core can't translate:

```csharp
static class MathMappings
{
    [ExpressiveFor(typeof(Math), nameof(Math.Clamp))]
    static int Clamp(int value, int min, int max)
        => value < min ? min : value > max ? max : value;
}

dbContext.OrderItems.Select(i => new { i.Id, Quantity = Math.Clamp(i.Quantity, 1, 10) });
```

```sql
SELECT "o"."Id", CASE
    WHEN "o"."Quantity" < 1 THEN 1
    WHEN "o"."Quantity" > 10 THEN 10
    ELSE "o"."Quantity"
END AS "Quantity"
FROM "OrderItems" AS "o"
```

**Overrides.** One of the longest-standing Projectables issues is [#74](https://github.com/EFNext/EntityFrameworkCore.Projectables/issues/74): override a projectable property in a derived entity and the override gets ignored. ExpressiveSharp dispatches on the runtime type:

```csharp
public class Order
{
    ...
    [Expressive]
    public virtual string Describe() => $"Order #{Id}";
}

public class SubscriptionOrder : Order
{
    public int IntervalInDays { get; set; }

    [Expressive]
    public override string Describe() => $"Subscription #{Id}, every {IntervalInDays} days";
}

dbContext.Orders.Select(o => new { o.Id, Description = o.Describe() });
```

```sql
SELECT "o"."Id", CASE
    WHEN "o"."Discriminator" = 'SubscriptionOrder' THEN 'Subscription #' || CAST("o"."Id" AS TEXT) || ', every ' || COALESCE(CAST("o"."IntervalInDays" AS TEXT), '') || ' days'
    ELSE 'Order #' || CAST("o"."Id" AS TEXT)
END AS "Description"
FROM "Orders" AS "o"
```

**Not just EF Core.** And finally, issue #4. The core `ExpressiveSharp` package knows nothing about EF Core. Call `.AsExpressive()` on any `IQueryable<T>` and you get both the modern syntax and the `[Expressive]` expansion. There is a MongoDB integration as well, and if all you need is a plain expression tree, `ExpressionPolyfill.Create(...)` gives you one with modern syntax and no queryable involved at all.

### An experiment on the side: window functions

There is one more package I want to mention, with a big asterisk. `ExpressiveSharp.EntityFrameworkCore.RelationalExtensions` adds SQL window functions like `ROW_NUMBER`, `RANK`, `LAG` and `LEAD` to EF Core:

```csharp
dbContext.Orders.Select(o => new
{
    o.Id,
    o.UserId,
    Rank = WindowFunction.Rank(Window.PartitionBy(o.UserId).OrderByDescending(o.CreatedDate))
});
```

```sql
SELECT "o"."Id", "o"."UserId", RANK() OVER(PARTITION BY "o"."UserId" ORDER BY "o"."CreatedDate" DESC) AS "Rank"
FROM "Orders" AS "o"
```

Do treat this as very much experimental. It works on EF Core 8 through 10, but EF Core 11 no longer lets third-party SQL expressions render themselves, which is exactly what this package relied on. The EF team [considers that](https://github.com/dotnet/efcore/issues/38977) an unintended extensibility pattern, so EF Core 11 is not supported, and whether it ever will be is an open question. Native window function support in EF Core itself is [on the backlog](https://github.com/dotnet/efcore/issues/12747). The rest of ExpressiveSharp is not affected by this and works fine on EF Core 11.

### Migrating from Projectables

Moving from Projectables should be mostly mechanical. Swap the package:

```bash
dotnet remove package EntityFrameworkCore.Projectables
dotnet add package ExpressiveSharp.EntityFrameworkCore
```

Then build. The package ships with analyzers that flag `[Projectable]`, `UseProjectables()` and the old namespaces, and each of them comes with a code fix. A *Fix All in Solution* takes care of most of it. Things like `UseMemberBody`, `CompatibilityMode` and the `NullConditionalRewriteSupport` settings need a bit of manual attention, and the [migration guide](https://efnext.github.io/ExpressiveSharp/guide/migration-from-projectables) walks through each of them. Do note that ExpressiveSharp targets .NET 8 and up.

### Standing on the shoulders of contributors

ExpressiveSharp would not exist without everything we learned from Projectables, and Projectables would not be where it is without the people who contributed to it. Fabien I already mentioned, but I also want to call out [Zoe Roux](https://github.com/zoriya), who added query root rewriting so projectable properties could be loaded without an explicit `Select`, and fixed eager includes along the way. And of course everyone else who sent in a pull request, opened an issue or answered someone else's: thank you! A lot of the issues you reported directly shaped how ExpressiveSharp works today.

### Wrapping up

Projectables started as a way to stop duplicating logic between our entities and our queries, and I think it did a great job at that. But over the years, most of the work went into teaching it more and more C#, one syntax feature at a time. ExpressiveSharp flips that around: the compiler already understands C#, so we let it do the understanding and only translate the result. That one change is what made modern syntax in the query itself, external member mappings, polymorphism and provider independence possible.

There are of course still limits. An `[Expressive]` member can only contain things your LINQ provider knows how to translate, and the usual expression tree restrictions (no `ref`, no `dynamic`, no side effects) still apply. The [documentation](https://efnext.github.io/ExpressiveSharp/) covers all of that, along with recipes and an interactive playground.

ExpressiveSharp is available on [NuGet](https://www.nuget.org/packages/ExpressiveSharp) and on [GitHub](https://github.com/EFNext/ExpressiveSharp). Give it a try, and if you run into a query that doesn't translate the way you'd expect, I'd love to hear about it.
