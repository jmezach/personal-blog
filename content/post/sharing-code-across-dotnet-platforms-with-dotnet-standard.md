---
title: "Sharing code across .NET platforms with .NET Standard"
date: 2016-12-05T00:00:00+02:00
draft: false
tags: [ ".NET", ".NET Core", ".NET Development", "Visual Studio" ]
categories: [ ".NET" ]
---

One of the key things that makes Xamarin such a great platform for developing mobile applications is that you can leverage your existing skills with C# and .NET and use them to create awesome Android and iOS apps. This also meant that you could take existing code written for .NET and use it in your Xamarin apps.  

Of course, in the early days of MonoDroid and MonoTouch (before Xamarin was even a company) this wasn’t as easy as it is today. Both MonoDroid and MonoTouch ran on the Mono framework after all, which isn’t the same as the .NET Framework we used on our desktops. Some API’s simply didn’t exist so your code that ran fine on the desktop could break when running in your Xamarin app.  

Therefore Portable Class Libraries (PCL’s) were created to make that experience better. By explicitly specifying on which .NET platforms you wanted your code to run (desktop, Xamarin, Windows Phone, etc.) you got the intersection of API’s that were available on those platforms so your code wouldn’t just blow up. A big improvement (trust me, I know, I’ve been there) but it did come with a couple of drawbacks.  

One of the biggest drawbacks perhaps was that when a new platform came along you needed to check a box to indicate you want to run on that new platform, recompile your code and package it up into a shiny new package. You even had to do this if the new platform implemented all the same API’s as the other platforms you were already targeting.  

Now, with the introduction of .NET Core, Microsoft realised this wasn’t a very sustainable model. So they introduced a new concept, .NET Standard. A lot has already been said about .NET Standard, including one of my [blogposts](https://blogs.infosupport.com/net-platform-standard-and-the-magic-of-imports/), but if you want a quick primer on what .NET Standard is I highly recommend checking out [this video](https://www.youtube.com/watch?v=YI4MurjfMn8&list=PLRAdsfhKI4OWx321A_pr-7HhRNk7wOLLY&index=1) by Immo Landwerth from Microsoft. He does a great job explaining what it is and what it means.  

For this blogpost however, I wanted to go a little deeper into how all this is going to work out for you as a developer.  

## Disclaimer

Before I do that however I want to be clear that everything I'm showing here is still very much alpha quality. Some things work, some things work sometimes and some things are simply broken. Im hoping all of this will just work once Visual Studio 2017 RTMs, but there might be delays between now and then so don't take my word for it.  

## Creating a .NET Standard library

So, let's start with the basics. Let's say I want to create a library in C# and use that with my .NET Framework or .NET Core backends as well as my Xamarin app. First thing we need to do is create a new project. Lets fire up Visual Studio 2017 RC and create a new project:

![Create new project](https://blogs.infosupport.com/wp-content/uploads/2016/11/1480453225.gif)

That was easy right? Nothing really new so far. But let's have a look at our dependencies:

![Project dependencies](https://blogs.infosupport.com/wp-content/uploads/2016/12/1480453417-1-1.png)

As you can see from the screenshot above we have a dependency on this NuGet package called NETStandard.Library and we're using version 1.6.0\. That package in turn depends on a whole set of other packages with varying versions.  

Now, based on this information one might think that our newly created library targets .NET Standard 1.6\. Perhaps surprisingly, thats not the case. In fact, out-of-the-box our library targets .NET Standard 1.4\. But how do we know this? Let's have a look at our project file:

![Project file](https://blogs.infosupport.com/wp-content/uploads/2016/12/1480453719-1-1.png)

Some things to note here. First of all, this project is using the new MSBuild format that is going to replace the project.json format we've been using for .NET Core for quite a while now. I know some might be upset about that, but the good news is that as you can see in the screenshot the MSBuild format has become quite a bit more terse compared to how they used to be (and it might become even more terse before the tools RTM).  

Also, note that I'm editing the project file while the project is still open in Visual Studio. This is a new feature and it means that I can edit my project file and Visual Studio will pick up the changes automatically. Although I must admit that right now, that doesn't always work properly, depending on what you change. For example, adding a package reference works fine, but changing the target framework doesn't sometimes. I'm sure that will be fixed before RTM though.  

Speaking about the target framework, you can see right there that we're targeting .NET Standard 1.4\. So, why then are we referencing the NETStandard.Library NuGet package version 1.6? Well, first of all NETStandard.Library 1.6(.0) is the first stable version of that package, so there's no lower version we can target. But also, there's no need to reference an older version because NETStandard.Library is merely what's called a meta-package. It doesn't contain any code itself, rather it points to other packages that then get downloaded and included in my project so I can compile against them. And the "beauty" of NuGet packages is that I can define a different set of dependencies for a particular target framework and when I install such a package NuGet will figure out what dependencies to install based on the target framework of the project I'm installing the package in. Let's see that in action by changing the target framework of my library to .NET Standard 1.0:

![Change target framework](https://blogs.infosupport.com/wp-content/uploads/2016/11/1480455162.gif)

As you can see, the list of dependencies becomes a bit smaller when targeting .NET Standard 1.0 because the API surface of that version is considerably smaller than 1.4\. We can see what changed in terms of API surface by having a look at the [documentation](https://github.com/dotnet/standard/blob/master/docs/versions.md) on GitHub and clicking on the various versions of .NET Standard. We can even get a diff between versions, such as [this one](https://github.com/dotnet/standard/blob/master/docs/versions/netstandard1.2_diff.md) comparing between 1.1 and 1.2.  

It is important to note that the API surface between different versions of .NET Standard should only become bigger with every new version. This means that by definition version 1.6 for example has all the same API's as all the versions before it. That also means that if I write a library which targets the oldest version (1.0) it will work on any .NET platform out there (provided it implements .NET Standard). However, I also have the thinnest API surface available to me to program against. So, let's start writing some code to see how this works in practice.  

## Writing .NET Standard code

Our .NET Standard library now targets 1.0, the oldest version of .NET Standard, so what code can we write? Turns out, quite a bit already. Unfortunately theres no description of 1.0 API's available, but you can find many of the things youre used to if youre .NET developer in there. Things like System.Collections.Generic, System.Linq and System.IO should all be there for you to use. If youre doing trivial things you can probably go with .NET Standard 1.0 but for more complicated code you might want to go to higher versions.  

For example, concurrent collections (such as ConcurrentDictionary) are available from .NET Standard 1.2\. And everyone's favourite Console.WriteLine is only available starting from version 1.3\. You probably shouldn't write to the console from library code directly anyway, but still. So what happens if we try to use some of those APIs in our library right now?

![Using unsupported API's](https://blogs.infosupport.com/wp-content/uploads/2016/12/1480624312-1.png)

We get red squiggles of course! That was probably what you expected, although I didnt get this the first time I tried it (probably because of a bug in the tooling). So how do we fix this?  

Well, one way to do it is to simply move to a version of .NET Standard that does support the APIs we want to use. For example, we can change our target framework in the project file and see what happens:

![Changing target framework](https://blogs.infosupport.com/wp-content/uploads/2016/12/1480625544.gif)

Red squiggles gone! Awesome! (Note that I had to unload the project and reload it again due to some bugs in the tools. Hopefully those will be fixed by the time they RTM).  

But, wait a minute. What does this mean for consumers of our library? If we refer back to the [versions table](https://github.com/dotnet/standard/blob/master/docs/versions.md) I mentioned earlier we can see that our .NET Standard 1.3 library can no longer be used on .NET 4.5.1 and lower! Now, depending on the audience of your library this may or may not be a problem. But it is important to note that as you go up in .NET Standard versions you are excluding older .NET platforms from using your library.  

Back when .NET Standard (or .NET Platform Standard as it was called then) was first announced there was talk about the tools helping you make these decisions. For example, using an API thats not supported in the version of .NET Standard you are currently targeting would be marked in grey and a Quick Action would pop up indicating you can upgrade to a higher version where that API is supported. It could then also show you which .NET platforms you would "leave behind" if you upgrade to that newer version. At this time though, this doesnt seem to be available and my guess is that it wont be available when the tools RTM. Hopefully that will come soon after.  

So what if we still want to support older versions of a .NET platform, but also use new APIs that are only available on the newer platforms? This is where cross-targeting comes in.  

## Cross-targeting

Cross-targeting allows us to target different versions of .NET Standard (or indeed, different target frameworks such as .NET Standard and .NET Framework 4.6 for example). Of course, this is something that has always been possible with .NET. What Ive seen most teams do is just create two (or more) projects that target the various frameworks you wanted to support and then include the same source code files in both (or all) those projects.  

This has worked well for us, but it was a pain to do. First of all because we had to make sure that we included every file in both projects. There wasnt some magic thing we could do to make sure that all the files were always included in both projects. Second of all, it was difficult to package this up into a NuGet package. If it was just one project we could just add a .nuspec file and run _nuget pack .csproj_ and it would create a nice package for us, including all the dependencies defined in packages.config. But with multiple projects this approach no longer works, so we had to manually pull together the binaries of the different projects and combine them into a single package.  

Fortunately, with the new project types, all of this should go away. We can just add target frameworks to our project file and well get the output we need. So let’s see how that works. First thing we need to do is modify our project file, like this:

![Cross targetting](https://blogs.infosupport.com/wp-content/uploads/2016/12/1480627126.gif)

As you can see I’m changing the element into the plural TargetFrameworks and adding netstandard1.0 to the list. Normally this should be enough, but again due to bugs I have to reload the project. But when I do and then open our code file, we’ll notice something new in the dropdown where we can select a project:

![Multiple target frameworks](https://blogs.infosupport.com/wp-content/uploads/2016/12/1480627492-1.png)

It now shows the .NET Standard version we’re currently targeting. Also, when hovering over a method call such as the Console.WriteLine() here it also shows us that this API is available in .NET Standard 1.3, but not in .NET Standard 1.0\. If we choose .NET Standard 1.0 in the dropdown above we’re getting red squiggles. If we build our project it will also give us a compile error since we’re using an API that doesn’t exist in .NET Standard 1.0\. So how do we fix this?  

We can use conditional compilation symbols to fix our situation. Let’s change our code so that it compiles again:

![Conditional compilation symbols](https://blogs.infosupport.com/wp-content/uploads/2016/12/1480627791-1.png)

By using the NETSTANDARD1_3 compilation symbol we can make small tweaks in our code for different versions of .NET Standard. Again, all this was possible before but the tools make it a much nicer experience now.  

But what about the NuGet package? Well, if we run dotnet pack ClassLibrary1.csproj from the command line (I couldn’t find an option in Visual Studio yet to do this) we get a single ClassLibrary1.0.0.0.nupkg file. If we look inside it with NuGet Package Explorer, this is what we’ll find:

![NuGet Package](https://blogs.infosupport.com/wp-content/uploads/2016/12/1480628146-1.png)

A nice clean NuGet package with two different assemblies targeting .NET Standard 1.0 and .NET Standard 1.3.  

Note that currently it doesn’t seem to be possible to create a NuGet package this way that can target .NET Standard and .NET Framework. I’ve tried adding net462 as a target framework and although the project will still load it doesn’t compile because it can’t find some basic types such as System.Object. I guess this is to be expected since it doesn’t know where to find mscorlib which defines these types. We’ll have to see how this plays out when the tools RTM.  

## Conclusion

So we’ve seen how we can write code that compiles against a particular .NET Standard version, which should then run on any .NET platform that implements at least that version of .NET Standard. We’ve even seen how we can use conditional compilation to cross-target our code to different versions of .NET Standard so that we can keep supporting older versions of the standard (and thus, older platforms) while still being able to use new API’s available in higher versions of the standard.  

What I haven’t shown yet is how we can reference our library from an application. There’s a reason for that, because as it stands right now, referencing our .NET Standard class library in a .NET Framework console application within the same solution doesn’t work. There’s some native code in Visual Studio that prevents you from doing this, which will hopefully be fixed when these tools RTM. Installing a NuGet package that targets .NET Standard should work however, provided that the target framework of your project supports that version of .NET Standard.  

Speaking about the tools, we’ve also seen that they aren’t quite finished. This might come as a surprise to some since Visual Studio 2017 is currently an RC, but Microsoft has been very open about the .NET Core tooling (.NET Standard included) still being at "alpha" quality. The plan is still to RTM these tools when Visual Studio 2017 RTM’s, but it still remains to be seen if that is the case. Things can still change.  

That being said, I do think that the new tools show great promise for the future. The new csproj format is a lot more terse, making it easier to edit by hand. Also, merge conflicts on these files are less likely to occur, which is a life saver if you’re working in teams (which most of us do). Finally, being able to cross-target within the same project can also come in handy if you have lots of different platforms to target. Then again, with .NET Standard we might not have to do much cross-targeting anymore on .NET.  

With all these improvements on the horizon I do hope that we can all benefit from this in the near future. At the moment all this goodness only seems to work for .NET Standard and .NET Core projects, but not for your regular .NET Framework or Xamarin projects. But it is all based on MSBuild now so it shouldn’t be too long before we can take advantage of this in those projects as well.