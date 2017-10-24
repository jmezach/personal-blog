---
title: ".NET Platform Standard and the magic of \"imports\""
date: 2016-05-24T00:00:00+02:00
draft: false
tags: [ ".NET", ".NET Core", ".NET Standard" ]
categories: [ ".NET" ]
---

In case you missed it, [.NET Core RC2](https://blogs.msdn.microsoft.com/dotnet/2016/05/16/announcing-net-core-rc2/) has been released by Microsoft last week along with [ASP.NET Core RC2](https://blogs.msdn.microsoft.com/webdev/2016/05/16/announcing-asp-net-core-rc2/) and a [preview of the tooling](https://blogs.msdn.microsoft.com/visualstudio/2016/05/16/announcing-updated-web-development-tools-for-asp-net-core-rc2/). Lots of exciting new stuff that I highly encourage checking out.

Now that RC2 is out there I guess a lot more people will be checking it out and playing with it and there’s one topic that seems to confuse a lot of people. So I thought I’d write a blog about it to clear things up a bit. It all has to do with a new concept called…

## .NET Platform Standard

So what is it? Well, according to the [documentation](https://github.com/dotnet/corefx/blob/master/Documentation/architecture/net-platform-standard.md) it is:

> a specific versioned set of reference assemblies that all .NET Platforms must support as defined in the [CoreFX repo](https://github.com/dotnet/corefx)

But what does that actually mean? I guess the easiest way to think of it is as an interface that sits between your code and a particular place where .NET can run such as the full .NET Framework, .NET Core, Mono, UWP, Silverlight, etc. Basically it guarantees that your code can run on top of all the platforms that implement that interface. And that interface is versioned, meaning that if your code targets a lower version of the interface it will run on more platforms, but the interface will also be smaller. If instead you target a higher version of the interface, you’ll restrict your code to run on less platforms, but it will have a larger interface available to it.

### Why was it introduced?

Basically the .NET Platform Standard is meant as a better version of the Portable Class Library (or PCL’s) concept that we already know. As you might recall, Portable Class Libraries were introduced to make it easier to write .NET code that could run on a variety of platforms without having to cross-compile your code making it easier to share code between backend and frontend for example.

However, there is a big problem with how PCL’s work today. When you create a new PCL project you have to pick which platforms you want to support and as a result you get the API surface that is common between all those platforms. This works fine, but what happens when an entirely new platform comes along? I have to go back to my PCL project, add the new platform, recompile my library and produce a new NuGet package (since the platforms I’m targeting are hardcoded into the package). In fact a new version of NuGet must also be made since it knows about all the possible combinations of platforms.

This is exactly why the .NET Platform Standard was introduced. Whenever a new platform comes along, it will need to implement a particular version of the .NET Platform Standard and every library that targets that version (or a lower version) will just work on that platform without requiring a rebuild or a different package.

### How does it work in practice?

Typically if you’re writing a library that you intend to share publicly (ie. on NuGet.org) you’ll want to target the lowest possible .NET Platform Standard version since that means your code can be used in the most places. At the moment, those versions are defined as _netstandard1.0_ through to _netstandard1.5_ so you’ll want to target _netstandard1.0_. But of course it is not that easy, since you might need to use a particular API in your library that’s not available in _netstandard1.0_ (for instance System.Collections.Concurrent). So you’ll have to move to a higher version and while you do that you’re going to exclude some of the older platforms so there is a tradeoff here.

Be careful though, since at the moment at least there’s nothing blocking you from including the missing pieces as a dependency. For example having the following in your project.json will compile just fine:

```json
{
  "version": "1.0.0-*",

  "dependencies": {
    "NETStandard.Library": "1.5.0-rc2-24027",
    "System.Collections.Concurrent": "4.0.12-rc2-24027"
  },

  "frameworks": {
    "netstandard1.0": {
      "imports": "dnxcore50"
    }
  }
}
```

When I first tried this it didn’t make much sense to me because why then wouldn’t _netstandard1.0_ include _System.Collections.Concurrent_ in the first place? The point is that _System.Collections.Concurrent_ wasn’t available in some of the earlier versions of .NET hence it is not included in _netstandard1.0_. If I add it as a dependency it might work as long as it doesn’t require anything from the underlying platform. In other words, as long as the dependency I’m bringing in is purely managed code it should work, but as soon as it requires platform support things will be a bit more tricky. If you want to be sure that your library will work stick with the dependencies you pull in by depending on NETStandard.Library.

Talking about that, you might have noticed in the project.json that I’m depending on _NETStandard.Library_ 1.5.0-rc2-24027 and not on 1.0\. How does that work? Let’s have a look at that NETStandard.Library package:

[![Dependencies](https://blogs.infosupport.com/wp-content/uploads/2016/05/image_thumb.png "image")](https://blogs.infosupport.com/wp-content/uploads/2016/05/image.png)

As can be seen in the screenshot above _NETStandard.Library_ simply lists different dependencies depending on the target framework of the project it is being installed in. You can also see _System.Collection.Concurrent_ being included with _netstandard1.1_, but not under _netstandard1.0_. The same for _System.Net.Http_. You can also nicely see the difference in Visual Studio when you change the “frameworks” part of your project.json from netstandard1.0 to netstandard1.1 for example. Once the automatic package restore runs you will see different packages show up beneath _NETStandard.Library_ in your References folder.

[![References for .NET Standard 1.0](https://blogs.infosupport.com/wp-content/uploads/2016/05/image_thumb-1.png "image")](https://blogs.infosupport.com/wp-content/uploads/2016/05/image-1.png)

[![References for .NET Standard 1.1](https://blogs.infosupport.com/wp-content/uploads/2016/05/image_thumb-2.png "image")](https://blogs.infosupport.com/wp-content/uploads/2016/05/image-2.png)

## The magic of “imports”

Now all this .NET Platform Standard library stuff is nice and definitely an improvement over the current PCL model, but there is a problem. As it is right now, there aren’t many libraries on NuGet.org that actually target one of the _netstandard_ versions. So if I want to build an application that runs on .NET Core and I want to use a library from NuGet.org I need to wait until there is a version of the library available that targets netstandard.

Obviously that’s not a really good situation to be in since it will slow down adoption of .NET Core, which in turn causes slow downs in library authors to start targeting netstandard. It is a bit of chicken and egg problem. So to remedy this situation “imports” was added to the project.json. But what does it do?

If I create a new .NET Core application, I get the following in my project.json:

```json
"frameworks": {
  "netcoreapp1.0": {
    "imports": "dnxcore50"
  }
}
```

What it says right here is that I’m targeting the _netcoreapp1.0_ framework (which is the default framework for a .NET Core application). With the “imports” bit I’m also stating that any dependency I want to include that targets _dnxcore50_ should be treated as compatible with _netcoreapp1.0_. Basically I’m instructing the tooling to ignore the fact that _dnxcore50_ isn’t the same thing as _netcoreapp1.0_ because I know it is the same thing (or more precisely Microsoft knows that they are the same).

Interestingly I can put any other framework in there as well. For example, lets add Reactive Extensions to our application:

```json
{
  "version": "1.0.0-*",
  "buildOptions": {
    "emitEntryPoint": true
  },

  "dependencies": {
    "Microsoft.NETCore.App": {
      "type": "platform",
      "version": "1.0.0-rc2-3002702"
    },
    "Rx-Main": "2.2.5"
  },

  "frameworks": {
    "netcoreapp1.0": {
      "imports": "dnxcore50"
    }
  }
}
```

Unfortunately Rx-Main (or more precisely its dependencies) do not currently support _netcoreapp1.0_ so we get an error when restoring packages:

[![Unsupported target framework](https://blogs.infosupport.com/wp-content/uploads/2016/05/image_thumb-3.png "image")](https://blogs.infosupport.com/wp-content/uploads/2016/05/image-3.png)

Let’s have a look at what the package does support:

[![Supported target frameworks](https://blogs.infosupport.com/wp-content/uploads/2016/05/image_thumb-4.png "image")](https://blogs.infosupport.com/wp-content/uploads/2016/05/image-4.png)

Quite a lot of them, but not what we need. Good thing is that we can “import” one of these frameworks to get things working again. For example, adding _portable-net45+winrt45+wp8+wpa81_ to our “imports” causes the package restore to complete successfully and we can use those assemblies.

A word of warning is in order here though. Theoretically I could “import” any framework that’s currently out there I can install pretty much any package into my project. However, that doesn’t mean that my application will actually run and work. If a package I installed into my project is using an API that’s not available on the platform that my application will be running on it will simply fail at runtime.

Therefore you should be careful when using “imports” and make sure that you test your application properly on the actual framework it will be running on. Think of it as when you’re using “imports” you’ve turned off the tooling and you are on your own and its your job to make sure that your application works correctly.

## Conclusion

Introducing something new into an existing ecosystem is always a tricky thing, especially if you try to do it retroactively which is what Microsoft is trying to do here. Using “imports” seems to do something magical while in reality it just turns off some of the guardrails that are in place to protect you from doing something that won’t or might not work. But still, I think that the direction that this is going is a good one.

I hope that sometime in the not too distant future all libraries available on NuGet.org (or at least the most commonly used ones) will be targeting one of the _netstandard_ versions removing some of the weird framework names that we have right now (all the portable ones and dnxcore and dotnet5.4) in the process. I’m sure that Microsoft will be helping out with that effort by providing library authors with some [tools](https://twitter.com/davidfowl/status/733926411224776705) to make this transition easier. Once that is in place it will become a lot easier to figure out which libraries I can use in my application depending on where I want my application to run.

Unfortunately we are not there yet which means we’ll have to endure some pain as the community makes this transition. I’m pretty confident though that the .NET Platform Standard is here to stay and it will be an improvement over the current model. Be sure to check out the [documentation](https://github.com/dotnet/corefx/blob/master/Documentation/architecture/net-platform-standard.md) on this subject and if you want to know a little bit more about the reasons behind all this check out this [video](https://channel9.msdn.com/events/ASPNET-Events/ASPNET-Fall-Sessions/Class-Libraries) by David Fowler and Damian Edwards from the ASP.NET team who do quite a good job at explaining it (although with different names).