---
date: "2021-06-01"
tags:
- .NET
- EFCore
- EFCore.Projectables
title: EF Projections on computed properties and methods without a hassle!
draft: true
---

One of EF's main selling points is that it allows you to write queries without having to deal with the underlying database technology being used. This however has it's limitations as you as a developer will have likely encountered. EFCore is only able to handle expressions that are typed as such. As a result, if you try to select anything from a locally computed property or method then EFCore will have to fall back to client side evaluation to compute the result of that expression which may be inefficient and is certainly limiting!

Consider the following example:
```csharp
public class User
{
    public int Id { get; set; }
    public string EmailAddress { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }

    public string FullName => FirstName + " " + LastName;
}

// Our query
dbContext.Users.Select(x => x.FullName).ToList();
```

This query is getting compiled (targeting SQLLite) as follows:
```sql
SELECT "u"."Id", "u"."EmailAddress", "u"."FirstName", "u"."LastName"
FROM "Users" AS "u"
```

This actually works! However, there are some issues with the former generated query. First, we've overfetched. We only really needed the FirstName and LastName. Yet our projection included the Id and EmailAddress. This may seem insignificant but once we add more properties to our user, this will build up. Secondly, this is very limiting. Imagine we want to filter on the FullName, as such:

```csharp
dbContext.Users.Where(x => x.FullName.Contains("Jon"));
```

This will blow up in EF since EF is unable to translate this valid CSharp expression to SQL as it does not know how to translate FullName into a proper SQL call. We could have called `Users.AsEnumerable().Where(x => x.FullName.Contains("Jon"));` and this would have worked but again, we would be overfetching as EF would first pull in all our users into our DbContext and then perform the Where clause in memory.

We could of course re-implement the FullName property within our Query and things just work, e.g.
```csharp
dbContext.Users.Where(x => (x.FirstName + " " + x.LastName).Contains("Jon"));
```

This works but it requires us to duplicate our logic. Luckily for us there are well established open source libraries such as [LINQKit](https://github.com/scottksmith95/LINQKit) that help us achieve these projections without the need to pull things in memory or write manual SQL. These libraries work by having you write Expressions that can then be used by EF and your code to both have an efficient translation to SQL (or whatever your database provider requires)- as well as be able to be computed on the client. Our hypothetical example would now look something like this:

```csharp
public class User
{
    public int Id { get; set; }
    public string EmailAddress { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }

    public static Expression<Func<Product, bool>> FullName()
    {
        return u =>  FirstName + " " + LastName;
    }
}

// Our query
dbContext.Users.AsExpandable().Select(x => x.FullName()).ToList();
```

This will now effectively compile our LINQ Query into optimal SQL that only includes what is needed. This is what SQL Lite produces:
```sql
SELECT "u"."Id", "u"."FirstName", "u"."LastName"
FROM "Users" AS "u"
WHERE ('Jon' = '') OR (instr((COALESCE("u"."FirstName", '') || ' ') || COALESCE("u"."LastName", ''), 'Jon') > 0)
```

We were able to filter within the produced query and only fetch the fields that we needed. Great! untill we need to  know FullName on the client. We would then have to call something like: 
```csharp
user.FullName().Compile().Invoke(user);
```

Again, not such a great experience and certainly not good for performance. We could leave them side-by-side, meaning one implementing using Expressions and one implemented as a normal computed property but that would either require us to implement FullName twice or take a perforamnce hit. 

If you recall from our LINQ Query, we also had to make a call to `AsExpandable()` to ensure that the LINQ Query would be rewritten to translate our `FullName()` call into the implementation that EF would actually understand. This again adds more complexity as well to the query and thereby also incurring a performance impact on the execution of that query.

These libraries have helpers us over time but its 2021 and we have new tools in our toolbelt! introducing: [EntityFrameworkCore.Projectables](https://github.com/koenbeuk/EntityFrameworkCore.Projectables)!

### EntityFrameworkCore.Projectables
This library intends to tackle the above problems and much more! For a while now we have access to SourceGenerators. A Generator allows us to produce additional source code based on your source code. What this means is that we can now automatically produce Expression methods for your properties and methods. All you need to do is mark those properties and methods for which you'd like an Expression method to be generated with an attribute. Perhaps that is hard to understand so lets examine what we can do with our example:

```csharp
public class User
{
    public int Id { get; set; }
    public string EmailAddress { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }

    [Projectable]
    public string FullName => FirstName + " " + LastName;
}
```

All we had to do was add an Attribute on the FullName property and this will generate for us:

```csharp
public static class BasicSample_User_FullName
{
    public static System.Linq.Expressions.Expression<System.Func<global::BasicSample.User, string>> Expression => 
        (global::BasicSample.User @this) => @this.FirstName + " " + @this.LastName;
}
```

We've received a new class (tugged away in a namespace: `EntityFrameworkCore.Projectables.Generated`) that contains a companion Expression of our FullName implementation. This is great, but how can we use it? Well we just call it in our query!

```csharp
dbContext.Users.Where(x => x.FullName.Contains("Jon"));
```
This query should blow up right? We did not make a call to `AsExapandable` as we did before. How does EF know to use our generated expression instead of our normal property?

This query will translate to SQL perfectly fine without overfetching, for as long as we've enabled Projectables with our DbContext. How do we do that? We call `UseProjectables()` on our OptionsBuilder! e.g. in case of using DI:
```csharp
serviceProvider.AddDbContext<ApplicationDbContext>(options => {
        options
            .UseSqlite(...)
            .UseProjections();
    })

```

Or when not using DI:
```csharp
override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(...);
    optionsBuilder.UseProjections();
}
```
At this stage, EF is fully equiped to translate any query that queries on a property or method that is marked with `[Projectable]` to use the source generated version internally. Since this happens early in the processing pipeline, no additional performance overhead is essentially added as EFCore does a great job in caching compiled queries.

What this then allows us to do is something as the following:
```csharp
var  users = dbContext.Users.Select(x => new { x.Id, x.FullName });
foreach (var user in users) {
    Console.WriteLine(user.FullName);
}
```

Our generated SQL would hence be as simple as:
```sql
SELECT "u"."Id", (COALESCE("u"."FirstName", '') || ' ') || COALESCE("u"."LastName", '') AS "FullName"
FROM "Users" AS "u"
```

We have everything that we wanted. No overfetching, no performance overhead and essentially no additional code except for our added `Projectable` attribute.

### Take it to the limit!
So far so good, We've been able to query a very simple projectable property. But what about a more complicated example. Let say our User has Orders and we want to have a computed property exposed on the User that can tell us how much this user has spent so far on Orders. Perhaps something like this:

```csharp
...

public class Order {
    ...
    [Projectable]
    public double PriceSum => Items.Sum(x => x.TotalPrice);
}

public class User {
    ...
    [Projectable]
    public double TotalSpent => this.Orders.Sum(x => x.PriceSum);
}

var mostValuableUser = dbContext.Users
    .OrderByDescending(x => x.TotalSpent)
    .Select(x => x.Id)
    .FirstOrDefault();
```

What this query would generate should be of no suprise: 
```sql
SELECT "u"."Id"
FROM "Users" AS "u"
ORDER BY (
    SELECT COALESCE(SUM((
        SELECT COALESCE(SUM(CAST("o"."Quantity" AS REAL) * "o"."UnitPrice"), 0.0)
        FROM "OrderItem" AS "o"
        WHERE "o0"."OrderId" = "o"."OrderId")), 0.0)
    FROM "Order" AS "o0"
    WHERE "u"."Id" = "o0"."UserId") DESC
LIMIT 1
```

We really have an optimized query here. Note that we we're able to call `PriceSum` on OrderItems which in itself is another `Projectable` property and EF was able to translate to its Expression implementation.

So what about calling into Methods and espceially if these methods take in additional arguments? We can serve those too!

```csharp
public class User {
    ...
    [Projectable]
    public IEnumerable<Order> GetRecentOrders(DateTime createdAfterDate) 
        => this.Orders.Where(x => x.CreatedDate > createdAfterDate);
}

// Our query again
var createdAfterDate = DateTime.UtcNow.AddDays(-7); // Orders created within the last 7 days
dbContext.Users.Select(x => new {
    UserId = x.Id,
    Orders = x.GetRecentOrders(createdAfterDate)
})
```

This again provides us with the following SQL:
```sql
SELECT "u"."Id", "t"."OrderId", "t"."CreatedDate", "t"."ProductId", "t"."UserId"
FROM "Users" AS "u"
LEFT JOIN (
    SELECT "o"."OrderId", "o"."CreatedDate", "o"."ProductId", "o"."UserId"
    FROM "Order" AS "o"
    WHERE "o"."CreatedDate" > @__createdAfterdate_0
) AS "t" ON "u"."Id" = "t"."UserId"
ORDER BY "u"."Id", "t"."OrderId"
```

Our CreatedAfterDate argument was captured as a parameter and passed in such that our query plan can effectively be compiled and reused.

Lets take this on step further. We want to have an extension method that helps us find the most valuable recent order that has a value over whatever we give it. Let's look at our scenario first:
```csharp
public static class UserExtensions {
    public static Order GetMostValuableRecentOrder(this User user, DateTime createdAfterDate, double minimumValueToBeConsidered) 
        => user.Orders
            .Where(x => x.CreatedDate > createdAfterDate)
            .Where(x => x.PriceSum >= minimumValueToBeConsidered)
            .OrderByDescending(x => x.PriceSum)
            .FirstOrDefault();
}

// And our query again
var query = dbContext.Users
    .Select(x => new {
        Name = x.FullName,
        TotalSpent = 0//x.TotalSpent
    })
    .OrderByDescending(x => x.TotalSpent);
```

And when we execute our SQL we get something that is slightly unexpected:
```sql
SELECT (COALESCE("u"."FirstName", '') || ' ') || COALESCE("u"."LastName", ''), "t0"."OrderId", "t0"."CreatedDate", "t0"."ProductId", "t0"."UserId"
FROM "Users" AS "u"
LEFT JOIN (
    SELECT "t"."OrderId", "t"."CreatedDate", "t"."ProductId", "t"."UserId"
    FROM (
        SELECT "o0"."OrderId", "o0"."CreatedDate", "o0"."ProductId", "o0"."UserId", ROW_NUMBER() OVER(PARTITION BY "o0"."UserId" ORDER BY (
            SELECT COALESCE(SUM(CAST("o"."Quantity" AS REAL) * "o"."UnitPrice"), 0.0)
            FROM "OrderItem" AS "o"
            WHERE "o0"."OrderId" = "o"."OrderId") DESC) AS "row"
        FROM "Order" AS "o0"
        WHERE ("o0"."CreatedDate" > @__createdAfterdate_0) AND ((
            SELECT COALESCE(SUM(CAST("o1"."Quantity" AS REAL) * "o1"."UnitPrice"), 0.0)
            FROM "OrderItem" AS "o1"
            WHERE "o0"."OrderId" = "o1"."OrderId") >= @__minimumValueToBeConsidered_1)
    ) AS "t"
    WHERE "t"."row" <= 1
) AS "t0" ON "u"."Id" = "t0"."UserId"
ORDER BY (
    SELECT COALESCE(SUM(CAST("o2"."Quantity" AS REAL) * "o2"."UnitPrice"), 0.0)
    FROM "OrderItem" AS "o2"
    WHERE (
        SELECT "o3"."OrderId"
        FROM "Order" AS "o3"
        WHERE (("u"."Id" = "o3"."UserId") AND ("o3"."CreatedDate" > @__createdAfterdate_0)) AND ((
            SELECT COALESCE(SUM(CAST("o4"."Quantity" AS REAL) * "o4"."UnitPrice"), 0.0)
            FROM "OrderItem" AS "o4"
            WHERE "o3"."OrderId" = "o4"."OrderId") >= @__minimumValueToBeConsidered_1)
        ORDER BY (
            SELECT COALESCE(SUM(CAST("o5"."Quantity" AS REAL) * "o5"."UnitPrice"), 0.0)
            FROM "OrderItem" AS "o5"
            WHERE "o3"."OrderId" = "o5"."OrderId") DESC
        LIMIT 1) IS NOT NULL AND ((
        SELECT "o6"."OrderId"
        FROM "Order" AS "o6"
        WHERE (("u"."Id" = "o6"."UserId") AND ("o6"."CreatedDate" > @__createdAfterdate_0)) AND ((
            SELECT COALESCE(SUM(CAST("o7"."Quantity" AS REAL) * "o7"."UnitPrice"), 0.0)
            FROM "OrderItem" AS "o7"
            WHERE "o6"."OrderId" = "o7"."OrderId") >= @__minimumValueToBeConsidered_1)
        ORDER BY (
            SELECT COALESCE(SUM(CAST("o8"."Quantity" AS REAL) * "o8"."UnitPrice"), 0.0)
            FROM "OrderItem" AS "o8"
            WHERE "o6"."OrderId" = "o8"."OrderId") DESC
        LIMIT 1) = "o2"."OrderId")) DESC
```

We can see multiple projectable properties and methods at work here. First we've shown that we can query on extension methods as long as they are marked as Projectables. We can subsequently call additional projectable properties and methods and all can make use of parameters that we passed in. The generated SQL however is less than optimal and we could do better by manually writing SQL. This however is unrelated to this project and more of an issue with what EF and its database provider can do at the moment.

### Conclusion
[EntityFrameworkCore.Projectables](https://github.com/koenbeuk/EntityFrameworkCore.Projectables) is a great tool to have when working within the confines of EFCore. It allows us to write our logic once and reuse it both within our queries as well as on the client side. With great power comes great responsibilities though as we may be better off just embracing our database and use stored procedures for optimal performance. Regardless. Using Projectables enables a whole new set of paradigmes that were difficult to achieve without it. I hence strongly suggest you checkout this project and see how it works for you.

