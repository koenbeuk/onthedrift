---
date: "2021-04-23"
tags:
- .NET
- EFCore
- EFCore.Triggered
title: Triggers for Entity Framework Core
---

When our codebase grows, so does its complexity. One way of managing this growing complexity is by leveraging triggers. Essentially this means that we’re able to run arbitrary code whenever a database commit occurs. Luckily for us, EF Core already provides the necessary infrastructure to embrace triggers. All we need to add is a bit of plumbing. 
[EntityFrameworkCore.Triggered](https://www.nuget.org/packages/EntityFrameworkCore.Triggered) is a NuGet package that does just that. The source can be found over on [Github](https://github.com/koenbeuk/EntityFrameworkCore.Triggered). 
This is part 1 in a series on EntityFrameworkCore.Triggered

## Motivation

Having worked on different projects involving Entity Framework, I noticed a pattern where we often want to do ‘something’ in response to an action. Some examples of that are:

-	Automatically assign a CreatedOn / UpdatedOn timestamp when adding/updating a record.
-	Validate the password strength before committing a user account to the database.
-	Sending an email whenever an order gets placed.
-	Update an aggregate whenever we update a record.
-   Implementing a soft-delete strategy

Some of these could be done using database triggers. Database triggers are a technique where we can have arbitrary code invoked in response to an action in your database. For example, we could enforce the constraint to always assign a CreatedOn timestamp to a new record by creating an AFTER INSERT database trigger that then updates the CreatedOn column for added records. Here is an example of an AFTER INSERT trigger for SQL Server:

```sql
CREATE TRIGGER User_Enforce_CreatedOn ON dbo.Users
    FOR INSERT
AS
    UPDATE Users SET CreatedOn = GETDATE() WHERE Id = Inserted.Id
```

This sounds great but it comes with a few gotchas:

-	Database triggers, as the name suggests, are database-centric.  Some databases may not support them and others may support only specific features. EFCore has no native support for database triggers. Your luck may vary from database to database.
-	In the given example, we update a record after its inserted, Depending on your database, this can be bad for performance and can cause unexpected side effects (in this example, a hypothetical trigger that would update the UpdatedOn column of an updated record would get invoked because our record got updated in order to set the CreatedOn value).
-	Database triggers are restricted to the database. Sending out an email in response to something happening in your database is, though technically with most databases possible, not easy to implement- nor a recommended use of database triggers. 

Database triggers are for these reasons great for specific purposes: They can absolutely enforce certain data integrity constraints. No matter how a record got inserted, updated, or removed, we can pretty much ensure that all database triggers will have had a chance to run in accordance. They are however invisible to the rest of the application and unable to interact with it. We need something that gives us more control.

In comes the good old service class. Here is an example of a typical service class using EF Core:
```csharp
public class UserService : IUserService
{
    public int Add(UserTemplate template)
    {
        var user = new User(template);

        // 1. Enforce that the CreatedOn property is always set 
        user.CreatedOn = DateTime.Now;

        // 2. Add a 'welcoming' email to the database
        var welcomeEmail = new Email {
            User = user,
            Message: "Welcome to the club!..."
        };
        _applicationDbContext.Emails.Add(welcomeEmail);

        _applicationDbContext.Users.Add(user);
        _applicationDbContext.SaveChanges();

        // 3. Ensure that we send the email out (but only after we've ensured that everything got committed to the database)
        _emailService.Send(welcomeEmail);

        return user.Id;
    }
}
```

This is a technique that overcomes all of the above-mentioned limitations with database triggers. It leans on EF Core and therefore is database agnostic. We can set certain properties and do verification before the record gets committed to the database, thereby only causing one INSERT statement. Finally, we can interact with the rest of the application and therefore interact with e.g. our Email service. However, it does come with a few limitations on its own:

- This can get messy, fast! the more our application grows, the more business rules we get to enforce, the more code we get to slap onto this service. Single-responsibility is not with this one which makes it much harder to maintain and test.
- Similar to the previous point, code reuse becomes more of a pain. If the User entity inherits from a BaseEntity which in turn enforces the CreatedOn property to be set, we will either have to do it ourselves or use inheritance or encapsulation to comply with that requirement. 
- We're unable to enforce the use of our service. If we want to guarantee that the welcome email address is created and sent out, we will still have to either enforce strict discipline to ever only add new users by using the UserService- or find an alternative way of enforcing that rule.

There must be a better way of going about this...

> I am aware of [MediatR](https://github.com/jbogard/MediatR) and [EfCore.GenericEventRunner](https://github.com/JonPSmith/EfCore.GenericEventRunner). These libraries offer an alternative approach to dealing with this added complexity. I have opinions on these approaches and I think that they can complement EntityFrameworkCore.Triggered in many ways. I will not discuss them further in this article but I will probably come back on these projects in a later post.

## EntityFrameworkCore.Triggered

By bringing Triggers into EntityFrameworkCore we can create a hybrid between both approaches. With EntityFrameworkCore.Triggered, we're able to do this:

```csharp
public class SetCreatedOnDate : IBeforeSaveTrigger<User> {
    public Task BeforeSave(ITriggerContext<User> context, CancellationToken cancellationToken) {
        if (context.ChangeType == ChangeType.Added){ 
            context.Entity.CreatedOn = DateTime.Now;
        }

        return Task.CompletedTask;
    }
}
```

This ensures that for as long as we use our DbContext is used. the CreatedOn property will be set when a new user gets added to the database. Because this is a single class with a single responsibility, it's trivial to write appropriate unit tests to ensure its correct behavior.

We can then use the same technique to enforce our second requirement, which is to create a Welcome email for this new user. Since triggers are registered and resolved through dependency injection, we can easily enforce this rule:

```csharp
public class CreateWelcomeEmail : IBeforeSaveTrigger<User> {
    readonly ApplicationDbContext _applicationDbContext;

    public CreateWelcomeEmail(ApplicationDbContext applicationDbContext) {
        this._applicationDbContext = applicationDbContext;
    }

    public Task BeforeSave(ITriggerContext<User> context, CancellationToken cancellationToken) {
        if (context.ChangeType == ChangeType.Added) {  
            this._applicationDbContext.Emails.Add(new Email {
                User = user,
                Message: "Welcome to the club!..."
            });
        }

        return Task.CompletedTask;
    }
}
```

Note that we're not sending the email out here, we're only worried about creating the email. EntityFrameworkCore.Triggered by default supports recursion which means that changes to your DbContext from within your triggers will be recorded and subsequent triggers will be raised accordingly. 

Enforcing the rule to sending out the actual email is for that reason not much different:

```csharp
public class SendEmail : IAfterSaveTrigger<Email> {
    readonly EmailSendService _emailSendService;

    public CreateWelcomeEmail(IEmailSendService emailSendService) {
        this._emailSendService = emailSendService;
    }

    public Task AfterSave(ITriggerContext<Email> context, CancellationToken cancellationToken) {
        if (context.ChangeType == ChangeType.Added) { 
            _emailSendService.Send(email); // Initial email
        }
        else if (context.ChangeType == ChangeType.Modified && context.Entity.Message != context.UnmodifiedEntity.Message) {
            _emailSendService.Send(email); // In case the content was updated we want to resent this email
        }

        return Task.CompletedTask;
    }
}
```

A few things to note here:
- Instead of implementing IBeforeSaveTrigger, we've to implement IAfterSaveTrigger which as the name suggests runs after EFCores SaveChanges completes.
- Instead of having this tied to the UserService, we've created a generic implementation that instead triggers on email records. Now any email that gets created or modified will automatically be sent out by this trigger.
- We also have implemented a 'requirement' where whenever the message of an email record changes, we want to resend that email with the new message. This way we can show that triggers can invoke for all Insert/Modified/Deleted entities. We've also shown that we can access the current state of the entity using `Context.Entity` as well as the unmodified form using `Context.UnmodifiedEntity`. This as the example shows us, is great to test for property changes and can be used in all our trigger types including `IBeforeSaveTriggers`.

### Configuring triggers

In the above `SendEmail` trigger we made use of an `IEmailSendService` instance but where did it come from? As it turns out, Triggers are perfectly able to interect with your dependency injection container. Lets take a look at what a typical setup would look like:

```csharp
void ConfigureServices(IServiceCollection services) [
    services.AddDbContext<ApplicationDbContext>(options => {
        options.UseSqlServer(...);
        options.UseTriggers(triggerOptions => {
            triggerOptions.AddTrigger<SetCreatedOnDate>();
            triggerOptions.AddTrigger<CreateWelcomeEmail>();
            triggerOptions.AddTrigger<SendEmail>();
        });
    });

    services.AddSingleton<IEmailSendService, EmailSendService>(); 
}
```

As triggers are managed and invoked by your DbContext, its completely database agnostic. As long as your manipulate your database through your DbContext and as long as you do not use raw queries and bulk operations, Triggers will be guaranteed to run for your changes.

## Conclusion
At this point I hope you have a basic understanding of how EntityFrameworkCore.Triggered can be beneficial in implementing business logic. The product has evolved over the last couple of years in a generic library used by quite a few projects and it's battle tested.

Subsequent articles in this series will dive deeper into advanced features, internal workings and perhaps most important: case studies.