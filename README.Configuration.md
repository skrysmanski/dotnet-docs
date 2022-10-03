# The .NET Configuration System

The [.NET configuration system](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration) provides a standardized way of expressing and reading configuration values.

## Hierarchical Configuration Values

**Hierarchical configuration values** are supported out-of-the-box. For example here the logging configuration:

```json
{
    "Logging": {
        "Console": {
            "FormatterOptions": {
                "TimestampFormat": "[HH:mm:ss] "
            }
        }
    }
}
```

If a configuration provider doesn't support a hierarchical representation but just flat strings (e.g. the command line, environment variables), **hierarchy levels are separated with a `:`**

    Logging:Console:FormatterOptions:TimestampFormat

**Environment variables** don't support `:` as hierarchy separator - they use `__` (double underscore) instead:

    Logging__Console__FormatterOptions__TimestampFormat

## Getting a Value

To get a configuration value, you need an instance of [`IConfiguration`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.iconfiguration?view=dotnet-plat-ext-6.0) (for how to get this, see below).

With this, you can call:

```c#
IConfiguration config = ...
int value = config.GetValue<int>("Parent:FavoriteNumber");
```

Alternatively you can bind whole **configuration sections** (see below) to objects.

For example, with this configuration:

```json
{
    "Settings": {
        "KeyOne": 1,
        "KeyTwo": true,
        "KeyThree": {
            "Message": "Oh, that's nice..."
        }
    }
}
```

And these classes:

```c#
public class Settings
{
    public int KeyOne { get; set; }
    public bool KeyTwo { get; set; }
    public NestedSettings KeyThree { get; set; }
}

public class NestedSettings
{
    public string Message { get; set; }
}
```

You can call:

```c#
IConfiguration config = ...
Settings settings = config.GetRequiredSection("Settings").Get<Settings>();
```

## Central Interfaces

There are three central interface for the configuration system:

* [`IConfiguration`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.iconfiguration?view=dotnet-plat-ext-6.0): Represents a set of key/value configuration properties.
* [`IConfigurationRoot`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.iconfigurationroot?view=dotnet-plat-ext-6.0): Represents the root of an `IConfiguration` hierarchy.
* [`IConfigurationSection`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.iconfigurationsection?view=dotnet-plat-ext-6.0): Represents a section of configuration values.

Both `IConfigurationRoot` and `IConfigurationSection` inherit from `IConfiguration`.

Conceptually both `IConfigurationRoot` and `IConfigurationSection` represent a configuration hierarchy.

## Configuration Sections

**Configuration sections** allow you to get a whole **subtree (sub hierarchy)** of the configuration hierarchy.

To obtain a configuration section use this extension method:

```c#
IConfigurationSection GetRequiredSection(this IConfiguration configuration, string key)
```

For example, given this configuration:

```json
{
    "Logging": {
        "Console": {
            "FormatterOptions": {
                "TimestampFormat": "[HH:mm:ss] "
            }
        }
    }
}
```

Calling `configurationRoot.GetRequiredSection("Logging")` gives you (conceptually):

```json
{
    "Console": {
        "FormatterOptions": {
            "TimestampFormat": "[HH:mm:ss] "
        }
    }
}
```

## Accessing the Configurations via Generic Host

If you're using a [Generic Host](README.GenericHost.md) application, you have several possibilities to get access to the configuration system:

* In `IHostBuilder` via `HostBuilderContext.Configuration` (`HostBuilderContext` can be injected as parameter in the delegates registered in `IHostBuilder`)
* Via dependency injection (as `IConfiguration`)
* Via [Options](https://learn.microsoft.com/en-us/dotnet/core/extensions/options). They can be registered in the [dependency injection system](README.DependencyInjection.md) like so:

  ```c#
  hostBuilder.ConfigureServices((context, services) =>
  {
      services.Configure<MyOptions>(context.Configuration.GetSection("MyOptions"));
  });
  ```

  And can then be injected in a constructor like so:

  ```c#
  public ExampleService(IOptions<MyOptions> options) { }
  ```

## Manual Creation

If you don't use a [Generic Host](README.GenericHost.md) application, you can manually create the configuration system like so:

```c#
IConfiguration config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .AddEnvironmentVariables()
    .Build();
```

## Configuration Providers

**Configuration Providers** define where configuration values are read from.

They're registered in the [`IConfigurationBuilder`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.iconfigurationbuilder?view=dotnet-plat-ext-6.0) like so:

```c#
configurationBuilder.AddJsonFile("appsettings.json");
configurationBuilder.AddEnvironmentVariables();
```

With `IHostBuilder`, providers are registered with these methods:

```c#
// host configuration
ConfigureHostConfiguration(Action<IConfigurationBuilder> configureDelegate);

// application configuration
ConfigureAppConfiguration(Action<HostBuilderContext, IConfigurationBuilder> configureDelegate);
```

**Note:** The **order** in which the providers are registers **matters**! If the same configuration key is provided by multiple providers, the provider that was **added last** is used.

.NET comes with several [configuration providers built-in](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration-providers). Here a few select ones.

### In-Memory Provider

```c#
configurationBuilder.AddInMemoryCollection(
    new Dictionary<string, string>
    {
        ["SecretKey"] = "Dictionary MyKey Value",
        ["TransientFaultHandlingOptions:Enabled"] = bool.TrueString,
        ["TransientFaultHandlingOptions:AutoRetryDelay"] = "00:00:07",
        ["Logging:LogLevel:Default"] = "Warning"
    }
);
```

### JSON File Provider

```c#
hostBuilder.ConfigureAppConfiguration((hostBuilderContext, configurationBuilder) =>
    {
        IHostEnvironment env = hostBuilderContext.HostingEnvironment;

        configurationBuilder
            .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
            .AddJsonFile($"appsettings.{env.EnvironmentName}.json", true, true);
    });
```

### Command Line Provider

```c#
configurationBuilder.AddCommandLine(args);
```

Configuration values can be set with [various patterns](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration-providers#command-line-configuration-provider) - for example like this:

```
--Parent:FavoriteNumber 42
```

### Environment Variables Provider

Provides configuration values via environment variables.

By default, the variables **do not require a prefix**. However, you can specify that only variables with a certain prefix are to be used:

```c#
configurationBuilder.AddEnvironmentVariables(prefix: "CustomPrefix_");
```

For example, to set the `TimestampFormat` for the console logger via environment variable, use this:

```
set Logging__Console__FormatterOptions__TimestampFormat="[HH:mm:ss] "
```
