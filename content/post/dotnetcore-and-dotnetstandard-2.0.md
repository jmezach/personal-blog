---
title: ".NET Core and .NET Standard 2.0"
date: 2017-05-13T20:33:00+02:00
draft: false
tags: [ ".NET", ".NET Core" ]
categories: [ ".NET" ]
---

This week I’ve attended Microsoft’s Build conference in beautiful Seattle. It has a been a very busy week, but we had a lot of fun attending the different sessions as well as talking to some of the members on the various product teams.  

Of course, during the week, my focus has been on .NET Core. I’ve attended most (if not all) of the sessions on .NET Core (and related technologies such as ASP.NET Core, Entity Framework Core and SignalR Core) and I even talked to people like Scott Hunter (program manager on the .NET team) and Immo Landwerth, one of the driving forces behind .NET Standard.  

During the conference the first preview version of .NET Core 2.0 has been released, and with that a first preview of .NET Standard 2.0 as well as ASP.NET Core 2.0\. In this post I want to talk a bit about .NET Standard 2.0 and what that means for developers going forward.  

## What is .NET Standard anyway?

I’ve blogged about this before, but just as a quick reminder, what is .NET Standard exactly? Basically it’s a contract. It’s a specification of all the API’s that a .NET platform has to implement in order to be called a .NET platform. The standard itself is even open source, and you can find out all the different versions and API’s that are in that version over on [GitHub](https://github.com/dotnet/standard/blob/master/docs/versions.md).  

.NET Standard 2.0 is a new version that significantly increases the number of APIs compared to the previous version (1.6.1). In fact, the API surface has more than doubled with .NET Standard 2.0 (go see for yourself [here](https://raw.githubusercontent.com/dotnet/standard/master/docs/versions/netstandard2.0_diff.md)).  

Now, whats important to consider is that most of those APIs aren't exactly new. Youll know most of them from .NET Framework already. What is new though is that they are now part of the standard, meaning that all .NET platforms will have to implement these APIs. This is pretty easy for .NET Framework and Xamarin (which is based on Mono) since they already have these API’s, but .NET Core needed to implement a lot of these API’s in order to be compatible with the new version of . NET Standard. Luckily, that work has also been done and it's what goes into .NET Core 2.0.  

## What’s in it for me?

By now you might be wondering, okay, this is nice and all, but all I’m getting is a bunch of API’s in .NET Core that I already had in .NET Framework so what’s the big deal? Well, it all has to do with compatibility. By having all these API’s included in .NET Standard we can guarantee that any library that targets .NET Standard will just work on any of the .NET platforms that support at least that version of .NET Standard.  

Furthermore, since the API surface has almost doubled, many of the libraries that are out there right now on NuGet should just work on any of the .NET platforms that support .NET Standard 2.0 as long as they use API’s that are part of the standard without requiring any changes to the library. This is great because we can now use a lot more libraries from NuGet in our .NET Core 2.0 applications than we were able to do with 1.0 or 1.1\. Microsoft estimates that about 70% of the libraries on NuGet should just work with .NET Core now.  

## How does it work in practice?

So, let’s have a look at an example of how this is going to work. First of all, in order to use any of the things I’m showing here, you’ll need to install the new Visual Studio 2017 15.3 preview. One of the great things about Visual Studio 2017 is that you can install different versions of it side by side, so don’t worry about having to install it unless you’re low on disk space ;). Just go to the Visual Studio Installer in your start menu and install the preview version from there.  

Second of all, we’ll need to install the .NET Core 2.0 preview 1 bits. This surprised me a little bit since I was expecting them to be installed together with Visual Studio but it seems that right now you’ll need to install the .NET Core bits separately. You can download those from [GitHub](https://github.com/dotnet/core/blob/master/release-notes/download-archives/2.0.0-preview1-download.md).  

Once that’s installed we can go ahead and create a new ASP.NET Core 2.0 project. To do so, go to File -> New Project and then choose ASP.NET Core Web Application. After that a new dialog pops up:

![New ASP.NET Project dialog](https://blogs.infosupport.com/wp-content/uploads/2017/05/1494698715.png)

From the dropdown in the top left corner of that dialog, make sure you select ASP.NET Core 2.0.  

Now, with a project created for ASP.NET Core 2.0 we can actually install NuGet packages that only support .NET Framework. This is something we couldn’t do with ASP.NET Core 1.0 or 1.1 because we would get an error when installing the package.  

**However, and this is important**, being able to install packages that target .NET Framework doesn’t mean that things will actually work at runtime. The reason for this is that .NET Framework implements more API’s than those that are part of the .NET Standard. Remember, .NET Standard is just a contract and .NET Framework 4.6.1 implements all the API’s of version 2.0 of .NET Standard, but it also has API’s that aren’t part of the standard. So as long as my .NET Framework library is only using API’s that are part of the contract things will just work, but as soon as a library uses API’s that are not of part of the contract things will break since .NET Core 2.0 doesn’t implement them.  

An example of a library that might cause problems is **NServiceBus**, a popular messaging library for .NET. I can install the NServiceBus packages from NuGet into my ASP.NET Core 2.0 just fine and it will compile. However, NServiceBus can use **MSMQ** as a transport which isn’t available on .NET Core 2.0 (since its a Windows only technology) so that will fail at runtime.  

As it is right now, this is something you’ll have to be aware of and the only thing you can do right now to protect you from issues is to test is properly (which you should always do of course). However I did hear some talk about a warning that might be added when .NET Core 2.0 RTM’s which will indicate that you’re using packages that do **not** target .NET Standard and thus you’re on your own.  

## Looking ahead

Given the issues described above, I think this is really the time to start thinking about targeting .NET Standard. With .NET Standard 2.0 a lot of the API’s are available again which should make porting your code to .NET Standard a lot easier. If you’re building your libraries, even if they are only for internal use within your organisation, I think it’s worth it to either port them to .NET Standard entirely, or cross-compile them to target both .NET Framework and .NET Standard (which is really easy to do with the new tooling in Visual Studio by the way). This will future proof those libraries and will allow you to use them in your existing .NET Framework code base, while being able to build new things using .NET Core.  

## Conclusion

With .NET Standard 2.0 support in .NET Core 2.0 I think we’ll see a lot more people getting interested in using .NET Core to build their applications. In fact I think that .NET Core 2.0 really is what 1.0 should have been. The tooling Is final now and the API surface is a lot closer to .NET Framework meaning it should be easier to move over to .NET Core.  

Obviously there are a lot of other improvements in .NET Core 2.0 and you’ll see see blog about these here as well, but I think .NET Standard 2.0 really is a game changer and now is the time for .NET developers to start thinking about it.
