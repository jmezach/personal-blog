---
title: "Having Fun with the .NET Core Generic Host"
date: 2017-10-29T13:07:00+02:00
draft: false
tags: [ ".NET", ".NET Core" ]
categories: [ ".NET" ]
images: []
banner: ""
description: ""
---

As ASP.NET developers we’re fairly used to hosting our code inside Internet Information Services (IIS). However, since ASP.NET Core is cross-platform, hosting inside IIS isn't always an option. For that reason, the hosting model for ASP.NET Core applications looks quite a bit different. Of course, we can still host our code in IIS, but we also have the option to use Kestrel and run as a standalone application.

This new hosting model is visible in code through the **WebHostBuilder** API from **Microsoft.AspNetCore.Hosting**. For example, if we create a new ASP.NET Core application, we'll find the following code in the Program.cs file:

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        BuildWebHost(args).Run();
    }

    public static IWebHost BuildWebHost(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseStartup<Startup>()
            .Build();
}
```

Although this is a fairly simple piece of code, quite a lot is happening behind the scenes. Much of that "magic" is in the *CreateDefaultBuilder()* method. Here is what it does behind the scenes:

- Setup Kestrel (the webserver)
- Set the content root to the current directory
- Setup configuration (AppSettings.json, etc.)
- Setup logging
- Enable IIS integration
- Setup the Dependency Injection container
- Configure services

If we look at that list, there are a few things that are very specific to web applications, but there are also a number of things that are desirable for any type of application. In fact, the ASP.NET team has seen this as well and decided to further abstract the **WebHostBuilder** into a more generic **HostBuilder** API. You can read more about the reasoning behind it in [this GitHub issue](https://github.com/aspnet/Hosting/issues/1163). It is currently planned for the 2.1 release and although there is no preview available for that version, we can still try it out which is what I've done lately and I'll go over it in this post.

## Creating a console host
A typical example of where the hosting API would be useful is in a console application that just needs to keep running until someone stops it. For example a background processing service that reads messages from a queue and processes them. So let's create a new console application:

```bash
dotnet new console -n MessageProcessor
```

Next we'll need to get access to the HostBuilder API.

## Getting the HostBuilder API
Since the HostBuilder API isn't released yet, we'll need to use the ASP.NET CI Feed to get a preview version of it. We can access that feed by adding a **NuGet.config** to our project with the following content:

```xml
<configuration>
    <packageSources>
        <add key="ASP.NET CI Feed" value="https://dotnet.myget.org/F/aspnetcore-dev/api/v3/index.json" />
    </packageSources>
</configuration>
```

Next, we'll need to install the **Microsoft.Extensions.Hosting** package into our project. Note that this package isn't tied to ASP.NET Core specifically. It is a preview version though, so we'll need to pass the version number (2.1.0-preview1-27377 is the latest available as I'm writing this):

```bash
dotnet add package Microsoft.Extensions.Hosting -v 2.1.0-preview1-27377
```

## Using the HostBuilder API
Much like in ASP.NET Core applications we can use the HostBuilder API to start building our host and setting it up. In it's simplest form we can use the following code to get a console application that keeps running until it is stopped (for example using Control-C):

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;

public class Program
{
    public static async Task Main(string[] args)
    {
        var hostBuilder = new HostBuilder();
        await hostBuilder.RunConsoleAsync();
    }
}
```

Note that in order to run this, we'll need to set the language version of our project to latest because an async Main function is a feature of C# 7.1:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    ...
    <LangVersion>latest</LangVersion>
    ...
  </PropertyGroup>
</Project>
```

If we run our application now, we should see something like this:

```bash
dotnet run
Application started. Press Ctrl+C to shut down.
Hosting environment: Production
Content root path: /Users/jmezach/Projects/demos/MessageProcessor/bin/Debug/netcoreapp2.0/
```

Note that the output is very similar to what you get when you start an ASP.NET Core application. One might argue that the Content root path isn't very useful for a console application, but that is probably a result of the **WebHostBuilder** API being refactored into the **HostBuilder** API.

## Doing something useful
Of course this isn't very useful right now. All we got is a console application that starts and keeps running until we stop it, but we don't actually do anything. To fix that, we can write what's known as a hosted service. A hosted service is basically a piece of code that is run by the host when the host itself is started and the same for when it is stopped. This is represented in the **IHostedService** interface:

```csharp
/// <summary>
/// Defines methods for objects that are managed by the host.
/// </summary>
public interface IHostedService
{
    /// <summary>
    /// Triggered when the application host is ready to start the service.
    /// </summary>
    /// <param name="cancellationToken">Indicates that the start process has been aborted.</param>
    Task StartAsync(CancellationToken cancellationToken);
    /// <summary>
    /// Triggered when the application host is performing a graceful shutdown.
    /// </summary>
    /// <param name="cancellationToken">Indicates that the shutdown process should no longer be graceful.</param>
    Task StopAsync(CancellationToken cancellationToken);
}
```

Since we are building a message processor we'll probably need something that listens to a queue. I'll use [MassTransit](http://masstransit-project.com/) in this example, so let's write a **MassTransitHostedService**:

```csharp
public class MassTransitHostedService : Microsoft.Extensions.Hosting.IHostedService
{
    private readonly IBusControl busControl;

    public MassTransitBusControlService(IBusControl busControl)
    {
        this.busControl = busControl;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        await busControl.StartAsync(cancellationToken);
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await busControl.StopAsync(cancellationToken);
    }
}
```

It's simple really. We get an **IBusControl** instance injected through dependency injection and run **StartAsync** on it when host is started. When the host is stopped we run **StopAsync** so that we can gracefully shut everything down. Of course, we'll need to configure that **IBusControl** instance somewhere, as well as this hosted service so let's go back to our Program.cs and make some changes:

```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var hostBuilder = new HostBuilder()
            .ConfigureServices((hostContext, services) =>
            {
                services.AddSingleton<IBusControl>(serviceProvider =>
                {
                    return MassTransit.Bus.Factory.CreateUsingRabbitMq(cfg =>
                    {
                        var host = cfg.Host(new Uri("rabbitmq://localhost"), h =>
                        {
                            h.Username("guest");
                            h.Password("guest");
                        });
                    });
                });
                services.AddScoped<IHostedService, MassTransitHostedService>();
            });

        await hostBuilder.RunConsoleAsync();
    }
}
```

Note that we're using the **ConfigureServices()** method on the **HostBuilder** to add additional services to the dependency injection container. In this case we're adding **IBusControl** as a singleton and use MassTransit's RabbitMq transport as the implementation. Next, we add our **MassTransitHostedService** as a scoped service to the DI container as well. By doing this, the generic host will automatically run **StartAsync** on our hosted service, which in turn will call **StartAsync** on the IBusControl instance, essentially opening the connection to RabbitMq and starts listening for messages.

When we shut down the host with Control-C, the generic host will automatically call **StopAsync** on our hosted service, which again will call **StopAsync** on the IBusControl instance which will do some clean up.

## Host and app configuration
So far we've only used the dependency injection features of the generic host. But it can also setup configuration. To setup configuration we can use **ConfigureHostConfiguration** and **ConfigureAppConfiguration**. For example:

```csharp
var hostBuilder = new HostBuilder()
    .ConfigureHostConfiguration((config) =>
    {
        config.AddEnvironmentVariables();
    })
    .ConfigureAppConfiguration((hostContext, config) =>
    {
        config.SetBasePath(Environment.CurrentDirectory);
        config.AddJsonFile("appsettings.json", optional: false);
        config.AddJsonFile($"appsettings.{hostContext.HostingEnvironment.EnvironmentName}.json", optional: true);
        config.AddEnvironmentVariables();
    });
```

The difference between these two methods is that **ConfigureHostConfiguration** is used to influence the configuration of the host itself. For example, the host is aware of the environment it is running in. You can see this in the output when it starts. By default it uses a configuration setting called Environment and it recognizes a couple of standard values (Development, Staging and Production). We might want to change where that configuration setting is read from, for example from an environment variable or from a configuration file.

With **ConfigureAppConfiguration** we setup the configuration for our application. In the example above I change the base path to the directory where my app is being run from, rather than where the output is. Then I add **appsettings.json** as a required configuration file, and **appsettings.{environment}.json** as an optional override. Finally I add environment variables as an input for configuration as well.

After setting that up, we could use it to get the connection string to RabbitMq from the configuration file, rather than hardcoding it. To do that, we'll need to make a small change to our Program.cs:

```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var hostBuilder = new HostBuilder()
            .ConfigureHostConfiguration((config) =>
            {
                config.AddEnvironmentVariables();
            })
            .ConfigureAppConfiguration((hostContext, config) =>
            {
                config.SetBasePath(Environment.CurrentDirectory);
                config.AddJsonFile("appsettings.json", optional: false);
                config.AddJsonFile($"appsettings.{hostContext.HostingEnvironment.EnvironmentName}.json", optional: true);
                config.AddEnvironmentVariables();
            });
            .ConfigureServices((hostContext, services) =>
            {
                services.AddSingleton<IBusControl>(serviceProvider =>
                {
                    var configuration = serviceProvider.GetRequiredService<IConfiguration>();
                    return MassTransit.Bus.Factory.CreateUsingRabbitMq(cfg =>
                    {
                        var host = cfg.Host(new Uri(configuration.GetConnectionString("RabbitMq")), h =>
                        {
                            h.Username("guest");
                            h.Password("guest");
                        });
                    });
                });
                services.AddScoped<IHostedService, MassTransitHostedService>();
            });

        await hostBuilder.RunConsoleAsync();
    }
}
```

Now we can set the connection string to RabbitMq inside our appsettings.json, and possibly override its value for a particular environment by adding an appsettings.Development.json for example.

## Logging
Finally, there is one more cross-cutting concern for which we can use the generic host to do setup: logging. To setup logging there is another method we can use on the **HostBuilder**. Unsurprisingly it is called **ConfigureLogging**:

```csharp
var hostBuilder = new HostBuilder()
    .ConfigureLogging((hostContext, config) =>
    {
        config.AddSerilog();
    });
```

In this example I'm simply setting up [Serilog](https://serilog.net/) as our logging framework. We can do a lot of other things, like configure where to log to, etc. but I'll leave that as an exercise to the reader.

## Conclusion
Using the generic host makes building console applications that need to run until stopped an absolute breeze. What I like most about it though is the similarity with how things work in ASP.NET Core. If you're using ASP.NET Core for your frontend and need to do some background processing in a console app it makes sense to use the generic host for your console app.

One thing that sticks out for me though is that in ASP.NET Core we have a Startup class which does all the application configuration, while the Program.cs only deals with configuration of the host itself. That pattern isn't yet available here and I'm not sure it is coming. I did [ask on GitHub](https://github.com/aspnet/Hosting/issues/1249) though.

