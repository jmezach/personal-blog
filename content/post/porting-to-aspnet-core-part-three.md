---
title: "Porting to Aspnet Core Part Three"
date: 2018-11-10T14:59:07+01:00
draft: true
tags: []
categories: []
images: []
banner: ""
description: ""
---

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
