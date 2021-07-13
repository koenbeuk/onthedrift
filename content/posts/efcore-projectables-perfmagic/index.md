---
date: "2021-07-01"
tags:
- .NET
- EFCore
- EFCore.Projectables
title: Figuring out a performance mistery with EF Core and Projectables
draft: true
---

Last time, I wrote about a new library called [EntityFrameworkCore.Projectables](https://github.com/koenbeuk/EntityFrameworkCore.Projectables). In short this library allows us to query over properties and methods completely written in CSharp. While I was implementing benchmarks to test the overhead of this Library, I was suprised to find out that the performance was actually Increased when using Projectables over not using Projectables. Surely I must have made a mistake somewhere right? Well, lets dive in and see if we can find that out!


### Our scenario

First, let us create a sample scenario:

```csharp
public class User {
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

And now, lets create a query that does something simple, namely selecting the concatinated forms of FirstName and LastName for all users, lets implement our default (non projectable) benchmark first:

```csharp
[Benchmark(Baseline = true)]
public void WithoutProjectables() {
    var query = dbContext.Users
        .Select(x => x.FirstName + " " + x.LastName);

    query.ToQueryString();
}
```

Ok, that was easy. What about our Projectable benchmark? We could add a FullName property on our User class or simply use an extension method, lets go for the latter approach as this keeps the User class clean.

```csharp
public static class UserExtensions {
    [Projectable]
    public static string FullName(this User user)
        => user.FirstName + " " + user.LastName;
}

[Benchmark]
public void WithProjectables() {
    var query = dbContext.Users
        .Select(x => x.FullName());

    query.ToQueryString();
}
```

Both queries result in the same generated QueryString namely (using the SQL Server database provider this time),
```sql
SELECT (COALESCE([u].[FirstName], N'') + N' ') + COALESCE([u].[LastName], N'')
FROM [Users] AS [u]
```

Given that the generated SQL is the same for both queries as they essentially do the same thing, What we would also expect performance to be equal. Perhaps our `WithoutProjectables` implementation would be slightly faster as it seems to have less to do, lets run these benchmarks and see the results:


![FullName benchmark results](fullname-results.png "FullName benchmark results")

This is unexpected. The `WithProjectables` benchmark almost shows a 40% performance improvement over the `WithoutProjectables` benchmark while seemingly doing more. Lets dive in and see if we can figure out what is going on here.

### Diving in

Let's see what happens under the hood. First any method or property that is marked with the [Projectable] attribute receives a generated companion expression tree that can be used by EF Core at runtime to swap out the Property/Method call for the actual expression tree. Lets see what this looks like:

```csharp
public static System.Linq.Expressions.Expression<System.Func<User, string>> Expression => 
    (User user) => user.FirstName + " " + user.LastName;
```

This generated Expression Tree is exactly as we we're expecting. Following the runtime logic, we can now simulate what is going on behind the scenes, namely A call like this:

```csharp
dbContext.Users.Select(x => x.FullName());
```

Will get translated by the runtime into a call like this:
```csharp
dbContext.Users.Select(user => user.FirstName + " " + user.LastName);
```

And because of this translation, there is essentially no difference in the expression tree that gets compiled by EF when comparing between the `WithoutProjectables` and `WithProjectables` variants.

Now we know that we dont generate some special code that is perhaps better optimized compared to our manual attempt, let's take a step back and shift gears. What does it actually mean to call `.ToQueryString()` on either of these queries:

```csharp
var withoutProjectablesQuery = dbContext.Users
    .Select(x => x.FirstName + " " + x.LastName);

var withProjectablesQuery = dbContext.Users
    .Select(x => x.FullName());
```

We can deduce that the `withoutProjectablesQuery` is slightly more complex compared to the `withProjectablesQuery`. The former has 2 MemberAccessCalls (FirstName and LastName) as well as calls to string concatination. The latter simply calls has a call to a static extension method (FullName), which implicitly performs the MemberAccess as well as the string Concatination. The latter's expression tree is thereby more compact compared to the former. 

However, A simple method call to FullName is not translatable as EF has no insights into the actual implementation details. That's why the [EntityFrameworkCore.Projectables](https://github.com/koenbeuk/EntityFrameworkCore.Projectables) library swaps out- at runtime, The MethodCall to FullName with the actual implementation details that we're captured in the generated companion expression. We however, can still make use of the fact that an Expression of a call to FullName has a unique identity and since the implementation is determined at compile time, we can use this fact to delay the translation from the method call  to FullName into it's appropriate implementation!

EF Core is already very well optimized and comes with a number of Extension options that are available to us. This library is using by default: [IQueryTranslationPreprocessor](https://docs.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.query.querytranslationpreprocessor).

> A class that preprocesses the query before translation.

This translation hook is a perfect place to swap out the method call to FullName with its implementation details. We we're able to keep our call to FullName within the expression tree up to this point which means that EF Core is able to [cache](https://docs.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.query.querytranslationpreprocessor?view=efcore-5.0) the generated SQL (which implicitly will also ensure that our preprocessor has had a chance to Expand our FullName call) while mapping it to our reduced expression tree shape. This means that even if we run our query 10 times, we only had to perform the expansion once.


I mentioned the expression tree shape. EF Core will not cache an Expression Tree with generated SQL but rather the shape of an expression tree with generated SQL. This has the advantage of a better cache reuse since its likely that your queries are generated dynamically. To caclulate the shape of an Expression tree though, EF Core will have to Visit all expressions in a tree and build up some form of hash code that represents the shape of that tree.

By reducing the size of the Tree due to encapsulated portions that are represented by projectable method and property calls, we are able to leviate much of the work here! And this is why our Projectables `WithProjectables` benchmark performs better than our `WithoutProjectables` benchmark. After a first warmup run, our expansion has happened once and the generated QueryString is already cached. In subsequent runs, EF Core can simply compute the shape of an Expression Tree and based on that, return the generated QueryString from cache. Since our `WithProjectables` benchmark has a compacted Expression Tree, the time it takes for EF Core to calculate the shape of it's Expression Tree is thereby also reduced.

Let's see if we can exploit the above and generate even better performing queries:

```csharp
const string sample = "Jon Doe";

[Projectable]
public static int GetFriendsOfFriendsByFirstNameCount(this User user, string fullName)
    => user.Friends.SelectMany(x => x.Friends).Where(x => x.FullName() == fullName).Count();

[Benchmark(Baseline = true)]
public void WithoutProjectables() {
    var query = dbContext.Users
        .Select(x =>
            x.Friends.SelectMany(x => x.Friends).Where(x => x.FirstName + " " + x.LastName == sample).Count());

    query.ToQueryString();
}

[Benchmark]
public void WithProjectables() {
    var query = dbContext.Users
        .Select(x => x.GetFriendsOfFriendsByFirstNameCount(sample));

    query.ToQueryString();
}
```

We've essentially just complicated our benchmark by putting more work in the Expression tree. I our `WithoutProjectables` method, EF Core has more work to do to generate the shape of the expression tree. In our `WithProjectables` method, this work is again encapsulated behind the projectable method call. If we take the results of these tests then we can see the increased performance benefits:

|              Method |     Mean |    Error |   StdDev | Ratio |
|-------------------- |---------:|---------:|---------:|------:|
| WithoutProjectables | 32.83 us | 0.296 us | 0.231 us |  1.00 |
|    WithProjectables | 13.77 us | 0.164 us | 0.137 us |  0.42 |

The `WithProjectables` benchmark almost shows a 60% performance improvement over the `WithoutProjectables` benchmark! If we were to encapsulate more work inside our Projectable methods and Properties then we should expect even greater performance improvements.

### Wrapping up

Though the shown improvements are great, they are still limited to EF Core internally. When you actually start to send queries to your database then the difference in performance will likely be a much smaller fragment. There are also [other](https://github.com/dotnet/efcore/issues/25009) discussions active, that focus on query pre-compilation rather than dynamic compilation. This will always outperform queries with Projectable properties and methods.

In summary: Using Projectables is a great way of simplyfiying your code while making it more reuse