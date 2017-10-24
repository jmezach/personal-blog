---
title: ".NET Core at Build 2016"
date: 2016-04-02T00:00:00+02:00
draft: false
tags: [ ".NET", ".NET Core" ]
categories: [ ".NET" ]
---

As some of you might have seen on Twitter or Facebook I have attended Microsoft’s Build 2016 conference in San Francisco over the last week. It was an interesting experience with some good announcements, although to be honest I was expecting a little bit more. Then again Microsoft has become a lot more open about what they are doing which kind of takes the thunder away from these conferences. Free Xamarin, Azure Service Fabric generally available, Microsoft Cognitive Services and the Bot Framework are still great announcements though. Since I’m the .NET Core Chapter Lead at Info Support I’ve focused my attention during the conference on .NET Core and ASP.NET Core. I’ve attended a couple of sessions on that topic, but the most interesting one was today on the last day of the conference. It was a session by Daniel Roth on deploying ASP.NET Core applications.

## Deployment Story for ASP.NET Core

With ASP.NET Core a lot is going to change to the deployment story when compared with previous versions of ASP.Net. Things are going to be a lot simpler, but quite a bit different from what most of us are used to. Basically the deployment story boils down to three simple steps and I’ll go through each of them:

1.  Restore packages
2.  Prepare your app for publishing
3.  Deploy app to target environment

### Restore packages

As you might know .NET Core is no longer a big framework all bundled into one single thing. Instead, to use .NET Core you restore a whole bunch of NuGet packages on your machine that together make up the API surface your app is targeting. What’s great about this is that you only include the parts that your application actually needs which makes the footprint of your application a lot smaller. To perform this step all you have to do is run the following command on the command line:

```bash
dotnet restore
```

This will look at the project.json in your projects folder and restore the packages listed there into your user profile. Having those packages in your user profile means you no longer need to have multiple copies of those packages on your machine on all kinds of different places. That will save some precious disk space I’m sure.

### Prepare app for publishing

The next step is to prepare your app for publishing. What this basically does is layout the necessary binaries into a folder structure so that you can copy that onto any machine and run the application there. Again, performing this step is very easy:

```bash
dotnet publish
```

What’s interesting here though is that there are a couple of ways in which you can publish your app. One way is to publish your app so that it can run on a machine that has .NET Core installed. This model is essentially the same as the model we have been using up to now where you deploy your app to a machine that has the .NET Framework installed and you simply re-use the system wide .NET Framework in your app. It also has the same drawbacks of that model in that all applications running on the target machine use the same version of the framework. Updates to the framework can thus impact all the applications running in that machine. Fortunately there is another option where the framework and the runtime are published together with the application. The benefit of this is that different applications running on the same machine can use their own versions of the framework and the runtime depending on their needs. What’s even more interesting is that publishing an ASP.NET Core application this way produces an executable which acts like a console application. Once you’ve run the publish command you can simply double click the resulting executable and it will simply work. In the future we’ll probably also see native compilation here, meaning that we can compile our ASP.NET application natively for the platform your targeting which will undoubtedly increase performance. The final option is to publish the app so that it runs on the full .NET Framework we already know and love. Not much changes with this option compared to the first option, it will just run on the full .NET Framework instead of .NET Core.

### Deploy app to target environment

Once we’ve published the app we can simply copy the resulting folder onto a target machine and run it there and it will just work. So what you might be wondering right now is how the requests that are coming into an application are served. Interestingly a typical ASP.NET Core application will include Kestrel, a lightweight HTTP server that runs cross-platform. Obviously Kestrel isn’t as battle tested as something like IIS or Nginx so Microsoft doesn’t recommend exposing it to Internet-like traffic. Instead they recommend running Kestrel behind a proxy such as IIS or Nginx. To help with that the ASP.NET team has built a new IIS module specifically for ASP.NET core. What happens there is that when a request comes into IIS it will be forwarded to Kestrel and handled by your application. Obviously there are a lot of other options for hosting your application. You can run it on Azure, Docker or pretty much any other platform you can think of. There are so many options here that I won’t go into all of them for this blog post.

## Conclusion

I think ASP.NET Core will fundamentally change how we develop .NET applications. In a couple of years I’m sure most new applications will be build on top of this new platform. It will be interesting to see how these applications are going to be hosted with all the new options that are available right now. I can’t wait for more news on the RC2 that will hopefully be available soon.