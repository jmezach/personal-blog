---
title: "Porting from ASP.NET to ASP.NET Core: Part 3 - Mechanics of porting"
date: 2018-11-23T16:28:11+01:00
draft: false
tags: [ ".NET", ".NET Core", ".NET Development", "porting", "open source" ]
categories: [ ".NET", "Series" ]
---
This post is part of a series:

- [Part 1: Introduction]({{< relref "porting-to-aspnet-core-part-one" >}})
- [Part 2: Third party dependencies]({{< relref "porting-to-aspnet-core-part-two" >}})
- [Part 3: Mechanics of porting]({{< relref "porting-to-aspnet-core-part-three" >}})
- Part 4: Deployment and packaging options 

If you haven't read any of the previous posts I recommend you do this now to get some context that I'll be referring to in this post. If you have, continue on with the nitty gritty details of porting an app to ASP.NET Core.

## Getting our house in order
So with our third party dependencies available on .NET Core it's time to start actually porting the code over. Before you do that though, make sure you're using a version control system (preferably Git) and create a branch before even starting porting. For many this may seem obvious, but I've seen this go wrong too many times. In the case of *Augurk* I'm working on the [feature/net-core-port](https://github.com/Augurk/Augurk/tree/feature/net-core-port) branch.

Before I started porting the *Augurk* solution consisted of 5 projects:

* Augurk, the ASP.NET host application containing the HTML/CSS/JavaScript for the frontend
* Augurk.Api, a class library containing the WebAPI controllers, etc.
* Augurk.Entities, a class library defining the shape of the objects being communicated through the API
* Augurk.Specifications, an MSTest project containing the *Gherkin* feature files describing the functionality of Augurk itself (but no automation :worried:)
* Augurk.Test, an MSTest project containing a couple of unit tests

My first step was to [migrate from packages.config to PackageReference](https://docs.microsoft.com/en-us/nuget/reference/migrate-packages-config-to-package-reference) for all of those projects except the ASP.NET project. I did this because .NET Core project files use PackageReference exclusively anyway and it would make it easier to replace the entire project file later on. It's also a painless experience that requires just a few clicks. Unfortunately it does not currently work for ASP.NET projects because of [various limitations](https://github.com/NuGet/Home/issues/5877) of that project system. Since that project would undergo quite a bit more changes I decided to just leave it as it was. After this change I made sure that everything still compiled and tests would still run successfully (which wasn't the case at first).

Next step was to merge the Api project back into the main ASP.NET project. I decided to do this because I knew that I would eventually move all of the client side stuff (the AngularJS app) into the **wwwroot** folder so I would have a clean separation of the frontend and backend logic anyway. And it would mean I would have one less project file to deal with. One interesting thing I noticed while doing this is that if you move the files inside Visual Studio it will just make a copy and delete the original file. That means I would also lose the version control history of those files which I didn't want. So I used the good old git command line to move files from their original location to the new location (git mv). Once that was done I could just include the files into the Augurk project and remove the Augurk.Api project. Again I made sure that everything was still compiling and tests were green.

## Porting the code

Now it's time to actually move the code to .NET Core. To do this I basically removed all the project files and replaced them with new ones using dotnet new on the commandline:

* Augurk -> dotnet new webapi
* Augurk.Entities -> dotnet new classlib
* Augurk.Specifications -> dotnet new mstest
* Augurk.Test -> dotnet new mstest

This step actually resulted in the removal of a lot of code as can be seen in [this commit](https://github.com/Augurk/Augurk/commit/909a1f6e5fa8860369dd499fc184630e8b455f19). The nice thing about the new project types is that they are a lot smaller by removing all kinds of boilerplate. We no longer have to list all the C# files that we want to include in the project, instead every C# file is included by default (although you can exclude files). Personally I think this combined with PackageReference can be a big reason for moving to .NET Core. Just think of the implications this has on developer productivity, especially with large teams working on the same codebase.

Of course after doing this it didn't compile immediately (that would have been nice though). But after adding back the project references that got lost, as well as the NuGet packages things were looking a lot better already. There's a couple of things to point out though:

#### AssemblyInfo.cs

.NET Framework projects have an **AssemblyInfo.cs** file that's included in the project. The information that's there can now be set in the project file (.csproj). However, if you've kept the file around as would be the case in such a migration, you get a compile error stating that there are duplicate attributes. Simply removing the **Properties** folder (that contains the AssemblyInfo.cs file) should fix the build errors, but you might want to copy the information from there into your project file. Have a look at this [GitHub issue](https://github.com/dotnet/sdk/issues/2) that contains a mapping between the two.

#### Just controllers
ASP.NET Core no longer has a distinction between an MVC and API controller class. Instead the base class for either one of them is simply called Controller so you'll need to change that in your code everywhere. Additionally you'll probably have a using of **System.Web.Http** which will need to be replaced with **Microsoft.AspNetCore.Mvc**.

#### Error handling
We had used **Request.CreateResponse()** and **Request.CreateErrorResponse()** in our code, but those are no longer available. There is a [compatibility shim](https://docs.microsoft.com/en-us/aspnet/core/migration/webapi?view=aspnetcore-2.1#compatibility-shim) available that should allow you to continue using these, but since they are [going away in ASP.NET Core 3.0](https://github.com/aspnet/Announcements/issues/319) we might as well fix these now.

It is recommended to use the methods on **Controller** such as **BadRequest()** or **Created()**. But these methods do not return an **HttpResponseMessage** but an **ActionResult** instead, so you'll also need to change the return type of those operations. The good thing is that these changes should make the API better documented through things such as Swagger.

#### Routing
The **RoutePrefix** attribute is gone. Instead, just put a **Route** attribute on the controller and it basically acts as a **RoutePrefix**.

#### RavenDB
I also had to make a couple of changes in how we used RavenDB since we went from version 3.0 to 4.1 which included a couple of breaking changes. Luckily there was [documentation available](https://ravendb.net/docs/article-page/4.1/csharp/migration/client-api/introduction) about how to make these changes, but there are a couple of loose ends that we'll need to look into to see what the appropriate replacement is.

## Booting up the frontend
Once I managed to compile the application and make it run I had to make a couple of adjustments in order for our AngularJS frontend to work. First of all I needed a way to let our ASP.NET Core app serve up the static files. Its perfectly capable of doing this, but there are a couple of different ways to do this. For example, you can add this to the **Configure** function inside your **Startup** class:

```csharp
app.UseStaticFiles()
```

It seemed like that was all I needed, but when I did this I still got a 404 Not Found error when navigating to the application. That's weird, because I did have an index.html inside my wwwroot folder. After searching for it, it turned out I had to use something else:

```csharp
app.UseFileServer()
```

Internally this also calls **UseStaticFiles()** but it also sets up defaults so that going to the root will return the content of index.html (if there is such a file). There's a lot of middleware available that can do all kinds of things, but it isn't always immediately obvious which one you should use in which situation.

## Making it run
With these changes in place I was able to compile and run *Augurk* on Windows and everything seemed to work. I didn't have any data in it yet though, so I used our existing command line tool to upload the documentation (written in *Gherkin* feature files obviously) into the new version to see if it really worked. Interestingly that didn't quite as I expected. I got **NullReferenceException**'s while doing the upload. Initially that didn't make much sense to me, so I started debugging. The controller operation that handles the upload of the feature files looks like this:

```csharp
[Route("api/v2/products/{productName}/groups/{groupName}/features")]
public class FeatureV2Controller
{
    ...
    [Route("{title}/versions/{version}")]
    [HttpPost]
    public async Task<ActionResult<Feature>> PostAsync(Feature feature, string productName, string groupName, string title, string version)
    {
        if (!feature.Title.Equals(title, StringComparison.OrdinalIgnoreCase))
        ...
    }
    ...
}
```

While debugging I noticed that the *feature* parameter was *null* here. How could that be, since it worked fine before and I was sure that I didn't change anything in the commandline tool. Turns out that there was a change in how ASP.NET Core WebAPI handles these situations. As you can see the controller operation has quite a few parameters, one complex type and a couple of string parameters. In previous version of WebAPI it just assumed that since all the other parameters are coming from the route template (the combination of the **RoutePrefix** and **Route** attributes, although that distinction is gone in ASP.NET Core) it must be the *feature* parameter that is in the body. With **ASP.NET Core** this is no longer assumed, so we must be explicit about the fact that we want the feature to come from the body, which you do by adding the *FromBody* attribute to the parameter. There is an entire [blogpost](https://andrewlock.net/model-binding-json-posts-in-asp-net-core/) by Andrew Lock on this subject, so refer to that if you want to have the details.

After adding in the missing attribute, everything worked again. We had a couple of other controller operations that relied on the same assumption, so I reviewed them and made the necessary changes accordingly. Definitely something to watch out for, since it wasn't immediately obvious what was happening.

## Cross-platform
Of course, one of the reasons for doing this porting effort in the first place was to make *Augurk* available cross-platform. So once I had everything working on *Windows* I committed the changes (actually I used several commits in the process) and cloned the repository on my Mac (I was running in Parallels before). I ran **dotnet run** and just like that everything was working just fine, natively on my Mac. That's pretty awesome.

I also noticed that now that *Augurk* is running on .NET Core it's also significantly faster than it was on .NET Framework. Obviously this also stems from the fact that I'm now running it natively on my Mac rather than virtualized on Windows, but even on Windows it's faster now. Changes made in RavenDB also help out with this as well.

## Looking forward
Now that the hard part is done and it's running on .NET Core we can look at some further enhancements that we can make. First of all I've already looking into using dependency injection. We were using what I'd like to call poor-mans dependency injection where we'd have a public constructor for each class that calls into an internal constructor with the concrete dependencies that should be used at runtime. During tests we could use the internal constructor to replace dependencies thanks to the **InternalsVisileTo** attribute. While that worked, it makes sense to use a proper dependency injection mechanism now that **ASP.NET Core** has one built-in. I've already started the work on that in a separate branch.

We've also looked into further changes we want to make as part of the *Augurk 3.0* release which will be the first version that will run cross-platform. One of the important things there is to have a look at the deployment options which I'll cover in my next post.