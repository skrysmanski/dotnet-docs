# Generic Host (a.k.a. `IHostBuilder`) - demystified

.NET (Core) provides a feature called [**Generic Host**](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host) - which is also known by its interfaces [`IHostBuilder`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostbuilder?view=dotnet-plat-ext-6.0) and [`IHost`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihost?view=dotnet-plat-ext-6.0).

This feature runs all modern ASP.NET Core applications and provides features that may also be pretty useful to non-ASP.NET Core applications:

* Dependency Injection
* Configurations/Settings/Options
* Logging

`IHostBuilder` uses a lot of "magic" that's hard to understand (or at least it's hard to understand how it works):

```c#
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

So, this article's goal is to explain the inner workings of `IHostBuilder`.

## How to Use the Generic Host

First, let's quickly see how you can use the Generic Host:

```c#
IHostBuilder hostBuilder = Host.CreateDefaultBuilder(args);

// ... configure host (builder) here ...

IHost host = hostBuilder.Build();

host.Run();
```

### NuGet Packages

If you're running an ASP.NET Core project, you get access to the Generic Host out-of-the-box.

If not, the interfaces `IHostBuilder` and `IHost` are defined in NuGet package [Microsoft.Extensions.Hosting.Abstractions](https://www.nuget.org/packages/Microsoft.Extensions.Hosting.Abstractions).

However the method [`Host.CreateDefaultBuilder()`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.host.createdefaultbuilder?view=dotnet-plat-ext-6.0) is defined in NuGet package [Microsoft.Extensions.Hosting](https://www.nuget.org/packages/Microsoft.Extensions.Hosting) which gives you the full Generic Host functionality.

## The Primary Use Case of Generic Host

The primary use case of the Generic Host is **run services** (HTTP or non-HTTP).

Such a service is started when the application starts, and is shut down when the application shuts down.

Because of this, a Generic Host application does *not* shut down on its own but **runs indefinitely** until shut down by some external means (e.g. by hitting `Ctrl+C`).

**The services** run by the Generic Host must implement [`IHostedService`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostedservice?view=dotnet-plat-ext-6.0) and are registered with `AddHostedService()` (here: `MyService`):

```c#
hostBuilder.ConfigureServices(services => services.AddHostedService<MyService>());
```

These services are automatically created, started and stopped by the Generic Host.

*Side note:* Conversely, the Generic Host was *not* designed to be used with **interactive console applications** (but it can be used with them with some "tricks").

## Deferred Actions - Why it looks so "complicated"

Almost all methods in `IHostBuilder` don't take "concrete" things - they only take **delegates**.

For example, to register a service, you have to use something like this:

```c#
hostBuilder.ConfigureServices(services => services.AddHostedService<MyService>());
```

The idea behind this design is (most likely) that all the *expensive* operations (like building a service or reading all configurations) is *deferred* until `IHostBuilder.Build()` is called.

## The Fundamentals

`IHost`/`IHostBuilder` require two systems to work: **dependency injection** and **configurations**.

Additional to these, there are a few "adjacent" systems:

* The content root
* The host environment (`IHostEnvironment`)
* The host builder context (`HostBuilderContext`)

**Logging** is the last building block. It's completely optional (as far as `IHost`/`IHostBuilder` is concerned) but automatically enabled when `Host.CreateDefaultBuilder()` is used.

These building blocks are visible in the implementation of `HostBuilder.Build()`:

```c#
BuildHostConfiguration();
CreateHostingEnvironment();
CreateHostBuilderContext();
BuildAppConfiguration();
CreateServiceProvider();
```

We'll be looking at each of these in the following sections (however in a different order than in the code).

### The Host Environment

The first thing we need to look at is the **host environment** ([`IHostEnvironment`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostenvironment?view=dotnet-plat-ext-6.0)). It provides access to some very fundamental pieces of information:

```c#
public interface IHostEnvironment
{
    string EnvironmentName { get; set; }

    string ApplicationName { get; set; }

    string ContentRootPath { get; set; }

    IFileProvider ContentRootFileProvider { get; set; }
}
```

With:

* `EnvironmentName` is a string that identifies the "environment" the host runs in. Usually this is either `"Production"`, `"Staging"`, or `"Development"` (as defined in `Microsoft.Extensions.Hosting.Environments`). If not explicitly specified, it defaults to `"Production"`. Among other things, this property is used to load the appropriate `appsettings.<Environment>.json`.
* `ApplicationName` is only informational - at least it's not used (as in: read) by `HostBuilder` itself.
* `ContentRootPath` and `ContentRootFileProvider` define a directory where the application's content files are loaded from. In ASP.NET Core applications, this defines from where to load static web site files (like `.html`, `.js`, `.css`). It's also used to find the `appsettings.json` files.

The values for all these properties are obtained from the **host configuration** (see below) via a set of predefined keys ([`HostDefaults`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.hostdefaults?view=dotnet-plat-ext-6.0)):

```c#
var hostingEnvironment = new HostingEnvironment()
{
    ApplicationName = hostConfiguration[HostDefaults.ApplicationKey] ?? Assembly.GetEntryAssembly()?.GetName().Name,
    EnvironmentName = hostConfiguration[HostDefaults.EnvironmentKey] ?? Environments.Production,
    ContentRootPath = hostConfiguration[HostDefaults.ContentRootKey] ?? AppContext.BaseDirectory,
};

hostingEnvironment.ContentRootFileProvider = new PhysicalFileProvider(hostingEnvironment.ContentRootPath);
```

You get access to the host environment through the **host builder context** (see next section).

### The Host Builder Context

The host builder context ([`HostBuilderContext`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.hostbuildercontext?view=dotnet-plat-ext-6.0)) can be optionally passed to most delegates registered in the `IHostBuilder`.

It's defined like this:

```c#
public class HostBuilderContext
{
    public IHostEnvironment HostingEnvironment { get; set; }

    public IConfiguration Configuration { get; set; }

    public IDictionary<object, object> Properties { get; }
}
```

With:

* `HostingEnvironment`: the **host environment** (see previous section)
* `Configuration`: first the **host configuration**, then later the **application configuration** after it has been built (see below)
* `Properties`: same instance as the `IHostBuilder.Properties` property (can be used for anything not covered by the other two)

### Configuration

The [configuration system](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration) is the first thing created in `HostBuilder.Build()`.

There are actually two different configurations:

* The **host** configuration
* The **application** configuration

Each configuration is represented by a [`IConfigurationRoot`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.iconfigurationroot?view=dotnet-plat-ext-6.0) object. (They're actually stored as [`IConfiguration`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.iconfiguration?view=dotnet-plat-ext-6.0) objects which is possible because `IConfigurationRoot` inherits `IConfiguration`.)

#### Host Configuration

The **host configuration** is created first and can be customized through [`IHostBuilder.ConfigureHostConfiguration()`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostbuilder.configurehostconfiguration?view=dotnet-plat-ext-6.0). It's accessible via `HostBuilderContext.Configuration`.

The primary goal of the host configuration is to populate the **host environment** (see above).

When the host builder is created by just using `new HostBuilder()`, the host configuration is empty by default. However, if you use `Host.CreateDefaultBuilder()`, two host configurations are added:

```c#
// Set the ContentRoot to the current directory
// Via: builder.UseContentRoot(Directory.GetCurrentDirectory())
hostBuilder.ConfigureHostConfiguration(configBuilder =>
{
    configBuilder.AddInMemoryCollection(new[]
    {
        new KeyValuePair<string, string>(HostDefaults.ContentRootKey, Directory.GetCurrentDirectory())
    });
});

// Enable configuration providers: environment variables and command line
hostBuilder.ConfigureHostConfiguration(configBuilder =>
{
    configBuilder.AddEnvironmentVariables(prefix: "DOTNET_");
    if (args is { Length: > 0 })
    {
        configBuilder.AddCommandLine(args);
    }
});
```

This means the host configuration has access to:

* the content root
* and any configuration value specified either
  * through an environment variable (whose name starts with `DOTNET_`)
  * or specified on the command line

This especially means that the host configuration has **no access to the `appsettings.json`** or any other configuration provider (by default; unless it's registered via `IHostBuilder.ConfigureHostConfiguration()`).

*Note:* The *host* configuration is only used while creating the *application* configuration (i.e. within delegates registered via `IHostBuilder.ConfigureAppConfiguration()`; see next section). After that, the application configuration replaces the host configuration (in `HostBuilderContext.Configuration`).

#### Application Configuration

The **application configuration** is created second - directly after the **host builder context** (see above) has been created - and can be customized through [`IHostBuilder.ConfigureAppConfiguration()`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostbuilder.configureappconfiguration?view=dotnet-plat-ext-6.0).

It inherits the host configuration (i.e. its values/configuration providers). Once it has been built, it replaces `HostBuilderContext.Configuration` (which before contained the *host* configuration).

If `new HostBuilder()` was used to create the host builder, no additional configuration providers are added.

If `Host.CreateDefaultBuilder(args)` was used to create the host builder, the default application configuration is defined like this:

```c#
hostBuilder.ConfigureAppConfiguration((hostingContext, configBuilder) =>
{
    IHostEnvironment env = hostingContext.HostingEnvironment;
    bool reloadOnChange = hostingContext.Configuration.GetValue("hostBuilder:reloadConfigOnChange", defaultValue: true);

    configBuilder.AddJsonFile("appsettings.json", optional: true, reloadOnChange: reloadOnChange)
                    .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: reloadOnChange);

    if (env.IsDevelopment() && env.ApplicationName is { Length: > 0 })
    {
        var appAssembly = Assembly.Load(new AssemblyName(env.ApplicationName));
        configBuilder.AddUserSecrets(appAssembly, optional: true, reloadOnChange: reloadOnChange);
    }

    configBuilder.AddEnvironmentVariables();

    if (args is { Length: > 0 })
    {
        configBuilder.AddCommandLine(args);
    }
});
```

I.e. the following configuration providers will be used:

* `appsettings.json` and `appsettings.<EnvironmentName>.json`
* User secrets when the environment is `"Development"`
* Environment variables (this time *without* prefix - unlike with the host configuration)
* The command line arguments

Note that the command line arguments are "re-registered" here so that they can overwrite configurations from the `appsettings.json` files.

### Dependency Injection

The last component created in `IHostBuilder.Build()` is the [dependency injection system](README.DependencyInjection.md).

It's registered via [`IHostBuilder.UseServiceProviderFactory()`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostbuilder.useserviceproviderfactory?view=dotnet-plat-ext-6.0) and represented by the [`IServiceProviderFactory<TContainerBuilder>`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.iserviceproviderfactory-1?view=dotnet-plat-ext-6.0) interface:

```c#
namespace Microsoft.Extensions.DependencyInjection;

public interface IServiceProviderFactory<TContainerBuilder> where TContainerBuilder : notnull
{
    TContainerBuilder CreateBuilder(IServiceCollection services);

    IServiceProvider CreateServiceProvider(TContainerBuilder containerBuilder);
}
```

The `CreateBuilder()` method wraps/converts the [`IServiceCollection`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.iservicecollection?view=dotnet-plat-ext-6.0) into an instance of `TContainerBuilder`. This type is dependent on the dependency injection framework being used.

The `CreateServiceProvider()` takes the return value of `CreateBuilder()` and converts it into an implementation of [`IServiceProvider`](https://learn.microsoft.com/en-us/dotnet/api/system.iserviceprovider?view=net-6.0).

*Note:* By default, .NET's built-in dependency injection framework is used - see below.

The `IHostBuilder.Build()` method uses this interface basically like this:

```c#
public IHost Build()
{
    IServiceCollection services = new ServiceCollection();

    // Adds some default services here.
    ...

    // These actions are registered via "IHostBuilder.ConfigureServices()".
    foreach (var configureServicesAction in _configureServicesActions)
    {
        configureServicesAction(services);
    }

    TContainerBuilder containerBuilder = _serviceProviderFactory.CreateBuilder(services);

    // The container actions are registered via "IHostBuilder.ConfigureContainer<TContainerBuilder>()".
    foreach (var containerAction in _configureContainerActions)
    {
        containerAction.ConfigureContainer(containerBuilder);
    }

    // "_appServices" is of type "IServiceProvider"
    _appServices = _serviceProviderFactory.CreateServiceProvider(containerBuilder);

    return _appServices.GetRequiredService<IHost>();
}
```

*Note:* The actual implementation is a little bit more complicated because it can't use `TContainerBuilder` directly.

*Note 2:* The implementation always creates the `services` variable as [`ServiceCollection`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicecollection?view=dotnet-plat-ext-6.0). This means that `IServiceCollection` just exists to have a clean interface for consumption. But it's not intended to be replaced with a different implementation (at least not when using [`HostBuilder`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.hostbuilder?view=dotnet-plat-ext-6.0)).

#### The Default Dependency Injection Framework

.NET comes with a built-in dependency injection framework that's used by default by any `HostBuilder` instance (unless replaced with a call to `IHostBuilder.UseServiceProviderFactory()`).

The factory for the built-in dependency injection framework ([`DefaultServiceProviderFactory`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.defaultserviceproviderfactory?view=dotnet-plat-ext-6.0)) is basically just this (note that `TContainerBuilder` is simply `IServiceCollection`):

```c#
public class DefaultServiceProviderFactory : IServiceProviderFactory<IServiceCollection>
{
    public IServiceCollection CreateBuilder(IServiceCollection services)
    {
        return services; // "No op"
    }

    public IServiceProvider CreateServiceProvider(IServiceCollection containerBuilder)
    {
        return containerBuilder.BuildServiceProvider(); // BuildServiceProvider() = Use built-in DI framework
    }
}
```
