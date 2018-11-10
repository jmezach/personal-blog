---
title: "Porting from ASP.NET to ASP.NET Core: Part 2 - Third party dependencies"
date: 2018-11-07T21:07:49+01:00
draft: false
tags: [ ".NET", ".NET Core", ".NET Development", "porting", "open source" ]
categories: [ ".NET", "Series" ]
---

This post is part of a series:

- [Part 1: Introduction]({{< relref "porting-to-aspnet-core-part-one" >}})
- [Part 2: Third party dependencies]({{< relref "porting-to-aspnet-core-part-two" >}})
- Part 3: Mechanics of porting
- Part 4: Deployment and packaging options 

If you haven't read the [first post]({{< relref "porting-to-aspnet-core-part-one" >}}) yet I suggest reading that first since it introduces the app that I'm porting and some of the reasons behind it.

## Are my dependencies available on .NET Core?
Most software these days consists of not just the code we write, but a lot of third party dependencies as well. And of course *Augurk* is no exception. So the first thing you'll want to do is to check if your third party dependencies are even available on .NET Core. Luckily for us there is a really simple way to check this: [I Can Has .NET Core](https://icanhasdot.net/). It works by scanning your packages.config or .csproj to figure out the packages you depend on. It then checks to see if those packages are available on .NET Core. Since *Augurk* is an open source project we can let it scan a GitHub repository directly and you can have a look at the results yourself [here](https://icanhasdot.net/result?github=Augurk~2FAugurk) (or at least until our changes are in the master branch). If you don't have an open source project you can also just upload a packages.config or .csproj individually.

The nice thing about this tool is that you get a graph of your dependencies, which are then color coded according to what is available, what is not available, known alternatives, etc. For example, here is the graph for *Augurk*:

![Dependency Graph](/img/porting-to-aspnet-core/dependency-graph.png)

And the associated legend:

![Legend](/img/porting-to-aspnet-core/legend.png)

As we can see in the graph there are only 4 packages that are unsupported on .NET Core: *Swagger-Net* (and the associated *FromHeaderAttribute* package), *MSTest.TestAdapter* and *RavenDB.Database*. As for Swagger-Net there are some good alternatives available as [documented here](https://docs.microsoft.com/en-us/aspnet/core/tutorials/web-api-help-pages-using-swagger?view=aspnetcore-2.1). The *RavenDB.Database* package got deprecated as part of the 4.0 release, so that shouldn't be a problem for us. And why *MSTest.TestAdapter* is listed as unsupported I honstely don't know, because it works just fine on .NET Core.

There's also a bunch of packages that have known replacements available, but all of them are packages that are part of ASP.NET. Since we're porting to ASP.NET Core we already know that these are going to be replaced, so no worries there. Additionally we are also using [SpecFlow](https://specflow.org/), which doesn't yet have an official release that's supported on .NET Core, but a pre-release version is available. At the moment we don't really use it though, so we're fine with using the pre-release version for now.

## What to do if something is unsupported?
Obviously depending on your project you might have a lot more packages that are unsupported on .NET Core. If that's the case than the first thing to check is to see if there is viable alternative available. Sometimes they can be hard to find, but maybe you get lucky. Replacing it with an alternative is of course going to add to the time required to port the application, but it might be worth it. You can even consider swapping out the dependency first, before porting to .NET Core, and make sure that it works correctly for your application.

But if there is no alternative available then I suggest checking if the package is being developed as open source. Many packages on NuGet are in fact open source, so chances are there is a GitHub repository for it somewhere. Usually there's a link to the repository in the package metadata as well. If it is open source, it's always a good idea to file an issue in the repository asking for .NET Core support (but make sure to check if there isn't such an issue already). This lets the owners of the package know that there is a demand for that package to be available on .NET Core.

And if you want to take things a step further you can also try to help out the owners of the library to port it to .NET Core. I'm sure that a pull request would be much appreciated (although you should discuss it in an issue first). If you are using it there are probably others who could benefit from a port as well.

## Conclusion
For any port to .NET Core is important to have your dependencies available there as well. Luckily for us this is the case with *Augurk*, although it took some time before *RavenDB* was ready to work on .NET Core. Your mileage may vary, but it might be a good idea to get start with a dependency scan as I've shown here to get an idea of what and what isn't available on .NET Core.