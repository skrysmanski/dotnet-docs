# Dependency Injection in .NET

## Basics

NuGet Packages: `Microsoft.Extensions.DependencyInjection.Abstractions` or `Microsoft.Extensions.DependencyInjection`

"Manual" approach (i.e. without [`IHostBuilder`](README.GenericHost.md)):

```c#
// Create service collection for registering all services
IServiceCollection serviceCollection = new ServiceCollection();

serviceCollection.AddSingleton<ITest, Test1>();

// Create service provider to resolve/build objects.
IServiceProvider serviceProvider = serviceCollection.BuildServiceProvider();

var testService = serviceProvider.GetRequiredService<ITest>();
testService.DoSomething();
```

## Overwriting (Multiple Singletons for same Interface)

The last wins:

```c#
IServiceCollection serviceCollection = ...

serviceCollection.AddSingleton<ITest, Test1>();
serviceCollection.AddSingleton<ITest, Test2>();

IServiceProvider serviceProvider = ...

var testService = serviceProvider.GetRequiredService<ITest>(); // --> Test2
```
