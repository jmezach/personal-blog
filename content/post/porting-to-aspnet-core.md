---
title: "Porting from ASP.NET to ASP.NET Core"
date: 2018-11-05T21:45:50+01:00
draft: false
tags: [ ".NET", ".NET Core", ".NET Development", "porting", "open source" ]
categories: [ ".NET" ]
---

Last week I took to [Twitter](https://twitter.com/jmezach/status/1057354206271733760) and asked if anyone of my followers would be interested in a session about .NET Core for developers that are using .NET Framework today. I wasn't really expecting all that much by sending that tweet, but with a little help from [Immo Landwerth](https://twitter.com/terrajobst) who's on the .NET team, that tweet seemed to have sparked quite a bit of interest in such a session. Perhaps the announcements that [ASP.NET Core 3.0 will run on .NET Core 3.0 only](https://blogs.msdn.microsoft.com/webdev/2018/10/29/a-first-look-at-changes-coming-in-asp-net-core-3-0/), and will no longer be available on .NET Framework, as well as the news that [.NET Standard 2.1 will not be implemented by .NET Framework 4.8](https://blogs.msdn.microsoft.com/dotnet/2018/11/05/announcing-net-standard-2-1/) had something to do with that as well. With these announcements it is becoming increasingly clear that .NET Core is here to stay and it seems to be the future of .NET. And with that, the need for porting existing code to (ASP).NET Core is become increasingly important.

So when I recently spent some time porting one of the open source projects I work on from ASP.NET to ASP.NET Core I figured I might as well write a blog post about my experience in doing so. Oh, and before you ask, yes I did submit a couple of proposals to [NDC Porto](https://ndcporto.com/) as well as a local event [dotNedSaturday](https://dotnedsaturday.nl/) on this very subject. I just hope they will be selected.

## The app
Before I dive into the nitty gritty details, let's go over the app that I've ported from ASP.NET to ASP.NET Core. It is an open source project that a colleague of mine started way back in [2014](https://github.com/Augurk/Augurk/commit/95fc1d16902cb086c89d3c1dd00181049cdae6fc) called [*Augurk*](https://augurk.github.io). It's a [living documentation](https://searchsoftwarequality.techtarget.com/definition/living-documentation) platform that was initially born out of a need to have a single place for the company we were (and still are) working at so that business people could easily find the documentation of the entire system without having to go through numerous Git repositories. There were a couple of alternatives, even back then, but none of them was a good fit for what we needed.

It's not a very complex application, but still it has some interesting aspects. For example, it's entirely API based, meaning that everything you can do is based on an API with an AngularJS (yes, that is Angular 1.0) frontend on top of it. The API was build with ASP.NET WebAPI on .NET Framework 4.5. It uses an embedded version of [RavenDB](https://ravendb.net/) as its data store (although it initially used SQL Server) since we wanted to make it really easy to get started without having to go and install all kinds of infrastructure.

Fast forward to today and we're seeing an increasing interest in *Augurk*. Quite a few of the [Info Support](https://www.infosupport.com) teams are using it or are thinking about it. This year we've also quite a bit more team to spend on it and we've build a big new feature that we are really excited about. But the world has also changed quite a bit since 2014. For example *Augurk* is currently distributed as an WebDeploy package. While that made sense at the time, having to setup an IIS Web Server, copying the zip over, deploying the package into IIS and changing the Web.config seems like a lot of work, especially compared to the convenience of spinning up a Docker container that suddenly everybody seems to be doing. Additionally we recognize that teams that are doing [Specification By Example](https://searchsoftwarequality.techtarget.com/definition/Specification-by-example-SBE) and/or [behavior driven development](https://searchsoftwarequality.techtarget.com/definition/Behavior-driven-development-BDD) aren't limited to using the .NET platform, so it doesn't make much sense to require them to setup Windows infrastructure just to run *Augurk*.

In summary, a lot of reasons to investigate a port of *Augurk* from ASP.NET on .NET Framework to ASP.NET Core so that we can run cross-platform. But how do we go about such an undertaking?

## First things first
Most software these days consists of not just the code we write, but a lot of third party dependencies as well. And of course *Augurk* is no different. So the first thing you'll want to do is to check if your third party dependencies are even available on .NET Core. Luckily for us there is a really simple way to check this: [I Can Has .NET Core](https://icanhasdot.net/). It works by scanning your packages.config or .csproj to figure out the packages you depend on. It then checks to see if those packages are available on .NET Core. Since *Augurk* is an open source project we can let it scan a GitHub repository directly and you can have a look at the results yourself [here](https://icanhasdot.net/result?github=Augurk~2FAugurk) (or at least until our changes are in the master branch). If you don't have an open source project you can also just upload a packages.config or .csproj individually.

The nice thing about this tool is that you get a graph of your dependencies, which are then color coded according to what is available, what is not available, known alternatives, etc. For example, here is the graph for *Augurk*:

<TODO>Insert image here</TODO>

And the associated legend:

<TODO>Insert image here</TODO>

Obviously if you have a lot of dependencies that are unsupported you're going to have a challenge porting your app. Sometimes though there are viable alternatives. And if the dependency is open source you have the option to file an issue in that dependency's repository and maybe they'll fix it. Or you could help them out porting it of course.

In the case of *Augurk* the biggest issue we had for a long time was the availability of RavenDB and especially the embedded version of it. Luckily for us a version of RavenDB that supports .NET Core has been released recently in the form of version 4, and with version 4.1 the embedded option also came back. We are also using [SpecFlow](https://specflow.org/) in our project which has a preview version (3.0) available that supports .NET Core although there are some remaining issues with that support.

## Getting our house in order
So with our third party dependencies available on .NET Core it's time to start actually porting the code over. Before you do that though, make sure you're using a version control system (preferably Git) and create a branch before even starting porting. For many this may seem obvious, but I've seen this go wrong too many times. In the case of *Augurk* I'm working on the [feature/net-core-port](https://github.com/Augurk/Augurk/tree/feature/net-core-port) branch.

Before I started porting the *Augurk* solution consisted of 5 projects:

* Augurk, the ASP.NET host application containing the HTML/CSS/JavaScript for the frontend
* Augurk.Api, a class library containing the WebAPI controllers, etc.
* Augurk.Entities, a class library defining the shape of the objects being communicated through the API
* Augurk.Specifications, an MSTest project containing the feature files describing the functionality of Augurk itself (but no automation :worried:)
* Augurk.Test, an MSTest project containing a couple of unit tests

My first step was to [migrate from packages.config to PackageReference](https://docs.microsoft.com/en-us/nuget/reference/migrate-packages-config-to-package-reference) for all of those projects except the ASP.NET project. I did this because .NET Core project files use PackageReference exclusively anyway and it would make it easier to replace the entire project file later on. It's also a painless experience that requires just a few clicks. Unfortunately it does not currently work for ASP.NET projects because of [various limitations](https://github.com/NuGet/Home/issues/5877) of that project system. Since that project would undergo quite a bit more changes I decided to just leave it as it was. After that I made sure that everything still compiled, tests would still run successfully (which wasn't the case at first).

Next step was to merge the Api project back into the main ASP.NET project. I decided to do this because I knew that I would eventually move all of the client side stuff (the AngularJS app) into the wwwroot folder anyway so I would have a clean separation of the frontend and backend logic anyway. And it would mean I would have one less project file to deal with. One interesting thing I noticed while doing this is that if you move the files inside Visual Studio it will just make a copy and delete the original file. That means I would also lose the version control history of those files which I didn't want. So I used the good old git command line to move files from their original location to the new location (git mv). Once that was done I could just include the files into the Augurk project and remove the Augurk.Api project. Again I made sure that everything was still compiling and tests were green.

## Porting the code

Now it's time to actually move the code to .NET Core. To do this I basically removed all the project files and replaced them with new ones using dotnet new on the commandline:

* Augurk -> dotnet new webapi
* Augurk.Entities -> dotnet new classlib
* Augurk.Specifications -> dotnet new mstest
* Augurk.Test -> dotnet new mstest

This step actually resulted in the removal of a lot of code as can be seen in [this commit](https://github.com/Augurk/Augurk/commit/909a1f6e5fa8860369dd499fc184630e8b455f19). The nice thing about the new project types is that they are a lot smaller by removing all kinds of boilerplate. We no longer have to list all the C# files that we want to include in the project, instead every C# file is included by default (although you can exclude files). Personally I think this combined with PackageReference can be a big reason for moving to .NET Core. Just think of the implications this has on developer productivity, especially with large teams working on the same codebase.

Of course after doing this it didn't compile immediately (that would have been nice though). But after adding back the project references that got lost as well as the NuGet packages things were looking a lot better already. There's a couple of points to point out though:

* .NET Framework projects have an AssemblyInfo.cs file that's included in the project. This file is no longer required and can be deleted. The information that's there can now be set in the project file (.csproj). However, if you've kept the file around as would be the case in such a migration, you get a compile error about duplicate attributes.
* ASP.NET Core no longer has a distinction between an MVC and API controller class. Instead the base class for either one of them is simply called Controller so you'll need to change that (as well as the using directives).
* We had used **Request.CreateResponse()** and **Request.CreateErrorResponse()** in our code, but they are no longer available. Instead it is recommended to use the methods on **Controller** such as **BadRequest()** or **Created()**. But these methods do not return an **HttpResponseMessage** but an **ActionResult** instead, so you'll also need to change the return type of those operations. The good thing is that these changes should make the API better documented through things such as Swagger.
* The **RoutePrefix** attribute is gone. Instead, just put a **Route** attribute on the controller and it basically acts as a **RoutePrefix**.

I also had to make a couple of changes in how we used  RavenDB since we went from version 3.0 to 4.1 which included a couple of breaking changes. Luckily there was documentation available about how to make these changes, but there are a couple of loose ends that we'll need to look into to see what the appropriate replacement is.

## Booting up the frontend
Once I managed to compile the application and make it run I had to make a couple of adjustments in order for our AngularJS frontend to work. First of all I needed a way to let our ASP.NET Core app serve up the static files. Its perfectly capable of doing this, but there are a couple of different ways to do this. For example, you can add this to the **Configure** function inside your **Startup** class:

```csharp
app.UseStaticFiles()
```

It seemed like that was all I needed, but when I did this I still got a 404 Not Found error when navigating to the application. That's weird, because I did have an index.html inside my wwwroot folder. After searching for it, it turned out I had to use something else:

```csharp
app.UseFileServer()
```

Internally this also calls **UseStaticFiles** but it also sets up defaults so that going to the root will return the content of index.html (if there is such a file). There's a lot of middleware available that can do all kinds of things, but it isn't always immediately obvious which one you should use in which situation.

But with that in place I was able to start *Augurk*. Unfortunately I didn't have any data in it so it didn't do much just yet. So I 


## Conclusion

