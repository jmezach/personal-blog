---
title: "Developing for .NET Core"
date: 2016-04-22T00:00:00+02:00
draft: false
tags: [ ".NET Core", "Linux", "OSX", "Windows" ]
categories: [ ".NET" ]
---

If you’ve read my [previous post](https://blogs.infosupport.com/net-core-at-build-2016-2/) you know that I have a keen interest in .NET Core. For the last two weeks I’ve been installing new RC2 builds on my machine almost on a daily basis to see if I could get something working and I’m glad to see that as of a couple of days ago things seem to be coming together. Obviously it still is a bit of a moving target but I feel like a lot of things are starting to settle.

In this post I wanted to share a bit about my experience working with all the new stuff and to give a bit of a look at what the developer experience is going to be when .NET Core goes RTM.

## Command line tools

As you might have noticed a lot of the work being done on .NET Core is centered around the new command line interface (or [dotnet-cli](https://github.com/dotnet/cli)). Being a Windows developer this might be a bit of surprise since we’re all used to just doing File –> New Project in Visual Studio and we take it from there. But consider for a moment that .NET Core is all about being cross-platform (on Linux and Mac OS) where the Visual Studio that we all know and love doesn’t run and it makes a lot more sense. That’s not to say that we can’t do File –> New Project anymore with .NET Core, it just means that there is more than one way to create a new .NET project which I think is a good thing.

So let’s suppose for a moment that we are working on a Mac and we want to create a new .NET project. How do we do that? Well, it’s simple, just type the following on a command line and press enter:

```bash
dotnet new
```

This command creates two files in our current folder, a _project.json_ and a _Program.cs_. _Program.cs_ contains a bit of code which you would also see when you create a new console application in Visual Studio so nothing new there. In _project.json_ you will find something like this:

```json
{
  "version": "1.0.0-*",
  "compilationOptions": {
    "emitEntryPoint": true
  },
  "dependencies": {
    "Microsoft.NETCore.App": {
      "type": "platform",
      "version": "1.0.0-rc2-3002439"
    }
  },
  "frameworks": {
    "netcoreapp1.0": {
      "imports": "dnxcore50"
    }
  }
}
```

In a sense _project.json_ contains much of the same information that you would find in the project file that is created by Visual Studio. It is also the place where you define what external dependencies (or NuGet packages) you want to pull in. At the moment we’re only pulling in Microsoft.NETCore.App which is in fact a meta-package that includes a whole bunch of NuGet packages that all together make up the Base Class Library (BCL) as we know it (albeit slightly different).

Since we are pulling in all those packages we need to be able to restore those to our machine so that we can use them. To do that we can run the following command:

```bash
dotnet restore
```

There are quite a few packages that you need to get a simple app running and if you look closely at the output of the restore command you’ll see some familiar namespaces come by such as _System.Collections.Generic_, _System.IO_ and _System.Linq_. They all used to be in one assembly called mscorlib.dll, but now they are all their own NuGet package meaning that they can (more or less) version independently from each other which I think is a great feature and will surely help stimulate innovation on the .NET platform. Have a look at the _.nuget_ folder in your user profile to get a sense of all the packages you are pulling in.

With the packages restored we can now build our application. To do that we simply run the following command:

```bash
dotnet build
```

Behind the scenes this will eventually run csc to compile our C# code into a .NET assembly. If we have a look at the _bindebugnetcoreapp1.0_ folder we will find a DLL here with the name of the folder where we created our application (in my case _dotnetcore.dll_). Interestingly there is no executable (.exe) here even though I have a _Main()_. This is because if we run _dotnet build_ we get what’s called a portable app. The idea here is that I can take the contents of my bin folder to another machine which already has .NET Core installed and run it there (using _dotnet run)._ This also works cross-platform. I can take a binary build on a Windows machine, copy it to a Mac or Linux machine and run it there or do it the other way around.

To show you that this actually works, here is a screenshot of compiling and running a brand new project (created using _dotnet new_) on OSX:

[![dotnet run on OSX](https://blogs.infosupport.com/wp-content/uploads/2016/04/Screenshot-2016-04-22-15.35.45_thumb.png "Screenshot 2016-04-22 15.35.45")](https://blogs.infosupport.com/wp-content/uploads/2016/04/Screenshot-2016-04-22-15.35.45.png)

I then copied the _dotnetcore-osx.dll_ (produced by _dotnet build_) from my OSX machine to my Windows machine and ran it there:

[![Run the same assembly on Windows](https://blogs.infosupport.com/wp-content/uploads/2016/04/image_thumb-1.png "image")](https://blogs.infosupport.com/wp-content/uploads/2016/04/image-1.png)

Of course building a project on Windows and running it on OSX works just as well and although I haven’t tried it myself yet it should also work with Linux. I did notice that I need to have the exact same build of the dotnet-cli on both machines for this to work, but that will obviously change once .NET Core is released.

To make this work I need to have .NET Core installed on the target machine I want to run on which isn’t always convenient. On top of that, I’m sharing the runtime and framework libraries with all the apps on the machine which means that if I update .NET Core on the machine all of the apps start using the new version. In fact, that is the situation with the .NET Framework that we know right now but it isn’t always what you want.

Luckily we can transform our portable app into a standalone app by publishing our application with the following command:

```bash
dotnet publish
```

Publish will gather everything we need to run the application on a particular runtime and framework. Before we can use it we need to specify what the runtimes and frameworks are that we want to run on. We’ve already specified the framework that we want to run on in our project.json, which is _netcoreapp1.0,_ but we haven’t specified the runtimes. So we need to add a little bit more to our _project.json_:

```json
"runtimes": {
  "win7-x64": { }
}
```

In this case I’m telling the tools that I want my application to run on 64-bit Windows 7 and up. There is a whole bunch of those runtimes available and you can use _dotnet –info_ to figure out what the runtime identifier (or RID) is for your current environment. Once I’ve added that to my _project.json_ and ran _dotnet restore_ again (which will bring down native assemblies for many of the NuGet packages we’ve been using so far) I can use the publish command to get a complete set of files that I need for my application to run on a particular platform. Here is a screenshot of what that looks like:

[![Publish output](https://blogs.infosupport.com/wp-content/uploads/2016/04/image_thumb-2.png "image")](https://blogs.infosupport.com/wp-content/uploads/2016/04/image-2.png)

As you can see in the screenshot there are **115** files there which is quite a lot. You’ll also notice that I now have an executable that I can use to run my application.

What’s important here though is that with this approach I loose the ability to take these files and run them on a Mac for instance. However, I can put these files onto a USB-drive or a file share and run them on some other Windows machine without having to install anything. I don’t have to install the .NET Framework and more importantly I get to use a version of .NET Core that I want to use without having to worry about other .NET applications running on the same machine getting broken.

## Editing and debugging code

So, enough about the command line tools. Let’s have a look at what the editing experience looks like. As noted above, there will be support in Visual Studio for building .NET Core applications. In fact, the team has shown an early version of that in the <font color="#ff0000">[ASP.NET Community Standup on July 19th](https://www.youtube.com/watch?v=hfJgGHqVcLI)</font>. However we won’t be seeing the full blown Visual Studio running on Linux or OSX anytime soon, so what do we use if we want to develop .NET applications on those platforms (if you’re not into vim, Emacs, nano or any other console editor out there)?

My guess is that Visual Studio Code, a more lightweight version of Visual Studio that runs cross-platform (including Windows) will be the tool of choice here. It was launched last year as a preview, but has already seen its [1.0 release](http://code.visualstudio.com/blogs/2016/04/14/vscode-1.0) and it is advancing rapidly. It also has great support for C# and .NET, although that’s no longer included out-of-the-box. Instead you’ll have to install the C# extension. For that you’ll get syntax highlighting, IntelliSense and a lot of other features you might be familiar with from Visual Studio.

What is really interesting is the debugging support that is now available for .NET Core applications within Visual Studio Code. To get that, you’ll need to install a prerelease version at the moment, which you can get from <font color="#ff0000">[here](https://github.com/OmniSharp/omnisharp-vscode/releases)</font>. Once you’ve got it all installed you can debug a .NET Core application almost as if you were using the full-blown Visual Studio. Just hit F5 and watch the magic happen:

[![Debugging on OSX with Visual Studio Code](https://blogs.infosupport.com/wp-content/uploads/2016/04/image_thumb-3.png "image")](https://blogs.infosupport.com/wp-content/uploads/2016/04/image-3.png)

Once again, this works across all three platforms (although I couldn’t get Visual Studio Code to run on Linux due to <font color="#ff0000">[this bug](https://github.com/Microsoft/vscode/issues/3451)</font>). In fact, the debugger itself is an application that is build on .NET Core and you’ll see it being build when you install the C# extension which is pretty neat.

## Conclusion

It looks like things are really starting to come together with .NET Core. I now have a working development environment set up on Windows, OSX and Linux (except for Visual Studio Code) and the power of running the following commands to get going shouldn’t be underestimated:

```bash
dotnet new
code .
```

Of course there are still quite a few rough edges. Sometimes when I install a new build things stop working that used to work before, but I guess that is to be expected with not yet released stuff. I’m looking forward to the RC2 and RTM releases and of course the tooling in Visual Studio. If you want to try it out for yourself, go to the [dotnet-cli](https://github.com/dotnet/cli) repository on GitHub, download the latest version of the .NET Core SDK Installer for your platform and get going.