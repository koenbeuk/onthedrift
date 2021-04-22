---
date: "2021-04-22"
tags:
- .NET
- Dependency Injection
title: "Activator utilities: activate anything!"
---

Dependency injection (DI) is a well-known technique that helps in writing more maintainable code. dotNET has excellent support for Dependency injection and is heavily used in platforms such as ASP.NET Core. However did you know there is a way to automatically instantiate a type with constructor arguments provided from an IServiceProvider without having to register that type with the DI Container first? That's where [ActivatorUtilities](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.activatorutilities?view=dotnet-plat-ext-5.0) come in.

<!--more-->
## .NET and Dependency injection
In .NET, Dependencies are typically resolved through an interface `IServiceProvider`. This is backed by a DI container (ServiceProvider) that holds all registrations of possible instances, including their lifetimes. This means that the .NET DI Container is responsible for both managing registrations, lifetimes, and hooking them up together. For this post, I assume that you are familiar with DI and its implementation within .NET. To learn more about using DI in .NET, check out the official [Microsoft documentation](//docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection). Examples given in this post are available on [Github](https://github.com/koenbeuk/ActivatorUtilityTests)

## Resolving dependencies
To resolve dependencies. You'll typically get your hands on an `IServiceProvider` instance. This same instance can then be used to dynamically resolve dependencies as shown below:

```csharp
var serviceProvider = new ServiceCollection()
      .AddTransient<Services.TransientService>()
      .BuildServiceProvider();

var service = serviceProvider.GetService<TransientService>();
``` 
This basic use-case of .NET's IServiceProvider works great when the dependency was registered beforehand with the ServiceProvider. But what if we want to get an instance of something that is not registered with the ServiceProvider:

```csharp
class MyController { 
    public MyControll(Services.TransientService service) { ... }
}

var serviceProvider = new ServiceCollection()
    .AddTransient<Services.TransientService>()
    .BuildServiceProvider();

var service = serviceProvider.GetService<TransientService>();

// Attempt 1: Resolve it from the ServiceProvider
serviceProvider.GetService<MyController>(); // FAIL

// Attempt 2; Create an instance manually
new MyController(new TransientService()); // SUCCESS!

// Attempt 3: Create an instance and resolve arguments through DI
new MyController(serviceProvider.GetService<TransientService>()); // SUCCESS!
```
In this case, Attempt 1 fails. The reason for this is that MyController was never registered with the DI container and therefore, the DI container would not be able to know anything about the lifetime of MyController. In this example, MyController is not even a dependency but rather a consumer.

Attempt 2 succeeds as it just works around DI altogether! Not great as we're now back to square 1 and responsible for constructing and managing the lifetime of the Service we just created.

So what about attempt 3? Well, it's a legitimate use case as it resolves dependencies from the DI container while we still manage construction and the lifetime of our consumer class. However, it can get a bit tedious. What if we required more than 1 dependency? We would have to make a call to ServiceProvider.GetService for each dependency. Perhaps worse: What if we did not know about the type of MyController at compile time? This is exactly the problem that ASP.NET MVC is facing where it has to construct Controllers of which it did know nothing of at compile-time and yet has to fulfill all dependencies. 

So now we're dealing with 2 subsequent issues:
How can we create an instance of a type when we know at compile-time, the constructor but not the type?
How can we create an instance of a type when we know neither the type nor the constructor at compile-time?

Luckily there is an easy answer and it's more powerful than you think!

## The manual approach
When taking a dependency on the package: Microsoft.Extensions.DependencyInjection you'll get a type called `ActivatorUtilities` for free! But before we head into that, let's see if we can at least tackle some of these issues ourselves. First off: 

### How can instantiate MyController at runtime when we know the constructor but not the type?

There is an easy way to this within just .NET, there are several.

```csharp
var controllerType = typeof(MyController);

// Attempt 1
var controller = controllerType
              .GetConstructor(new[] { typeof(TransientService) })
              .Invoke(new[] { ServiceProvider.GetService<TransientService>()  }); // WORKS

// Attempt 2
var controller = Activator.CreateInstance(controllerType, ServiceProvider.GetService<TransientService>()); // WORKS

```
Attempt 2 shows the use of the `Activator` class which has been around since the early days of .NET. It's a powerful tool in your arsenal that can be used to creates an instance of the specified type using the constructor that best matches the specified parameters.

Great, so now we know how to create an instance of a type without having to know the type at compile-time. So what about our next issue?

### How can we create an instance of a type when we know neither the type nor the constructor at compile-time?

Now it gets a bit trickier. Both attempts shown above to create a new instance of a type at runtime require us to pass in an array of arguments that match in number, order, and type the parameters of the constructor to invoke. 

```csharp
var controllerType = typeof(MyController); // Imagine we don't know this at compile time

var constructor = controllerType.GetConstructors().Single();
var controller = constructor.Invoke(
        constructor.GetParameters()
            .Select(parameter => ServiceProvider.GetService(parameter.ParameterType))
            .ToArray()
        );
```
There is no obvious advantage of using the `Activator` class here since we had to rely on Reflection to get a constructor to invoke. We also introduced the assumption that our type always has just one controller. Not great but it kinda... works.

## Introducing ActivatorUtilities
Since instantiating a type at runtime with constructor arguments resolved from an `IServiceProvider` is not an easy process. Microsoft included a powerful but not well-known class called `ActivatorUtilities`. This can be found in the `Microsoft.Extensions.DependencyInjection.Abstractions` NuGet package which you'll likely already have an (in)direct dependency on. Our last example can be rewritten with ease as:

```csharp
var controller = ActivatorUtilities.CreateInstance<MyController>(ServiceProvider);
```
`ActivatorUtilities` will now create an instance of MyController by calling its constructor with any dependencies resolved through the `IServiceProvider`. This is excellent, so we're done right? Well, no. First, remember that assumption where we assumed that there was only 1 constructor. Let's take that assumption away and see what happens.

### Activation patterns
Let's first assume that `MyController` has 2 constructors. One taking no arguments and the other taking a `TransientService` argument (which is registered with our DI container). How will ActivatorUtilities behave in this case?

```csharp 
class MyController { 
    public MyController() { 
        Console.WriteLine("Feeling empty!");
    } 
    public MyController(TransientService service) {
        Console.WriteLine("I've been served!");
    }
}

var subject = ActivatorUtilities.CreateInstance<MyController>(ServiceProvider);
```
When we run this code we'll find that 'Feeling empty!' gets written to the console. Does this mean that ActivatorUtitilies looks for the most fitting constructor (and in this case, the one with the least amount of work)? Well, the answer is no, `ActivatorUtilities` simply picks the first constructor that can do the job. So if the constructor that takes in the service was declared above the parameterless constructor then MyController would have been instantiated using that constructor. Only public constructors are considered.

We can however tell ActivatorUtilities which constructor to use, for this we have the `ActivatorUtilitiesConstructorAttribute` available. When applied to a constructor, this constructor will always be used by ActivatorUtilities.

```csharp 
class MyController { 
    public MyController() { 
        Console.WriteLine("Feeling empty!");
    } 
    [ActivatorUtilitiesConstructor]
    public MyController(TransientService service) {
        Console.WriteLine("I've been served!");
    }
}

var subject = ActivatorUtilities.CreateInstance<MyController>(ServiceProvider);
```
As expected, this will write: 'I've been served!' to the console. You can only apply ActivatorUtilitiesConstructor to one constructor otherwise, you'll get an InvalidOperationException during runtime.

There is a third way of selecting what constructor should be used, this can be done through explicit arguments.

### Explicit arguments
ActivatorUtilties allows you to specify constructor arguments manually, lets take our previous controller and create an instance with an explicit transient service and see what happens:

```csharp
class MyController { 
    public MyController() { 
        Console.WriteLine("Feeling empty!");
    } 
    public MyController(TransientService service) {
        Console.WriteLine("I've been served!");
    }
}

var subject = ActivatorUtilities.CreateInstance<MyController>(ServiceProvider, new TransientService());
``` 
In this case again, 'I've been served!' is written to the console. Remember that ActivatorUtitilties picks the first constructor that can do the job. In this case, our first empty does not accept any arguments while one explicit argument is given so it's not considered as a valid candidate.
If we were to provide additional explicit arguments then we would get an `InvalidOperationException` informing us that no suitable constructor for activation could be located.

This is not only useful for selecting what constructor should be used. It also allows us to pass in explicit arguments that could or should otherwise not be resolved through dependency injection. Let's take a look at an example.

```csharp
class MyController { 
    public MyController(ITransientService service1, CancellationToken cancellationToken, ISingletonService service2) { ... }
}

var cancellationTokenSource = new CancellationTokenSource();
var subject = ActivatorUtilities.CreateInstance<MyController>(ServiceProvider, cancellationTokenSource.Token);
```
Not only were we able to provide an argument that would otherwise not be resolved through our DI container, but we were also still able to accept that argument in our constructor in whatever order we like.  ActivatorUtilities only cares for the order of explicit arguments when multiple arguments are provided that are of the same type.

### Activation methods
So far you've seen an overload that requires an explicit type argument. However, just like our old Activator class, we also have an overload available that accepts a Type argument as shown below:

```csharp
var controller = ActivatorUtilities.CreateInstance(ServiceProvider, typeof(Subject));
```
This is ideal for when you don't know the actual type that you're constructing at compile time.

There is also an overload that returns a delegate that can be invoked to create an instance, optionally accepting explicit arguments. Let see an example.

```csharp
var factory = ActivatorUtilities.CreateFactory(typeof(Subject), new Type[] { });
var instance = factory.Invoke(ServiceProvider, null) as Subject;
```

Or when using explicit arguments

```csharp
var factory = ActivatorUtilities.CreateFactory(typeof(Subject), new Type[] { typeof(TransientService) });

var service = new TransientService();
var instance = factory.Invoke(ServiceProvider, new object[] { service }) as Subject;
```

This is particularly useful when performance is an issue as a suitable constructor is only located and analyzed once.

A third and less prominent feature is `GetServiceOrCreateInstance`. This little-known feature enables default or fallback implementations in case our DI container was not directly able to resolve an instance. Consider that we were in the need of an instance of a controller. It would be great if we could 1. Check with our DI container if it can resolve an instance of our controller and if not, create an instance ourselves. 

```csharp
var controller = ActivatorUtilities.GetServiceOrCreateInstance(ServiceProvider, typeof(MyController));
```

As a consumer of this code, I would only need to care that my controller can be resolved from the DI container, and optionally I could register it so that I can also control its lifetime. 


## Conclusion 
ActivatorUtilities is a powerful tool in your belt when dealing with DI  in general. You may not need it often but when you do, it's there and it just works. So how what does the future have in place for ActivatorUtilities? Well, probably not that much. It's rich in features already as code generation is on the horizon as a potential alternative.



