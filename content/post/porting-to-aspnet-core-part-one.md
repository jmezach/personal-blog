---
title: "Porting from ASP.NET to ASP.NET Core: Part 1 - Introduction"
date: 2018-11-05T21:45:50+01:00
draft: false
tags: [ ".NET", ".NET Core", ".NET Development", "porting", "open source" ]
categories: [ ".NET", "Series" ]
---

Last week I took to [Twitter](https://twitter.com/jmezach/status/1057354206271733760) and asked if anyone of my followers would be interested in a session about .NET Core for developers that are using .NET Framework today. I wasn't really expecting all that much by sending that tweet, but with a little help from [Immo Landwerth](https://twitter.com/terrajobst) who's on the .NET team, that tweet seemed to have sparked quite a bit of interest in such a session. Perhaps the announcements that [ASP.NET Core 3.0 will run on .NET Core 3.0 only](https://blogs.msdn.microsoft.com/webdev/2018/10/29/a-first-look-at-changes-coming-in-asp-net-core-3-0/), and will no longer be available on .NET Framework, as well as the news that [.NET Standard 2.1 will not be implemented by .NET Framework 4.8](https://blogs.msdn.microsoft.com/dotnet/2018/11/05/announcing-net-standard-2-1/) had something to do with that as well. With these announcements it is becoming increasingly clear that .NET Core is here to stay and it seems to be the future of .NET. And with that, the need for porting existing code to (ASP).NET Core is become increasingly important.

So when I recently spent some time porting one of the open source projects I work on from ASP.NET to ASP.NET Core I figured I might as well write a blog post about my experience in doing so. But as I was writing this post I figured it might be a better plan to write a series of posts that are shorted and more focused. So this is the first post in a series of posts in which I'll introduce the app I've ported and some of the reasons behind porting it. Then in the next post I'll talk about how to handle third party dependencies and some of the tools available to help you determine the feasibility of porting. After that I'll go through the actual mechanics of porting and the steps I took. Then finally I'll talk about some of deployment and packaging options that are now available:

- [Part 1: Introduction]({{< relref "porting-to-aspnet-core-part-one" >}})
- Part 2: Third party dependencies
- Part 3: Mechanics of porting
- Part 4: Deployment and packaging options 

Oh, and before you ask, yes I did submit a couple of proposals to [NDC Porto](https://ndcporto.com/) as well as a local event [dotNedSaturday](https://dotnedsaturday.nl/) on this very subject. Now I can only hope they will be selected.

## The app
So let's go over the app that I've ported from ASP.NET to ASP.NET Core. It is an open source project that a colleague of mine started way back in [2014](https://github.com/Augurk/Augurk/commit/95fc1d16902cb086c89d3c1dd00181049cdae6fc) called [*Augurk*](https://augurk.github.io). It's a [living documentation](https://searchsoftwarequality.techtarget.com/definition/living-documentation) platform that was initially born out of a need to have a single place for the company we were (and still are) working at so that business people could easily find the documentation of the entire system without having to go through numerous Git repositories. There were a couple of products in this space such as [Relish](https://relishapp.com/) and [Pickles](http://www.picklesdoc.com/) but none of them were a good fit for what we needed.

![Augurk](https://augurk.github.io/img/icon256.png)

It's not a very complex application, but it does have some interesting aspects that make it an ideal candidate for a porting exercise. For example, it's entirely API based, meaning that everything you can do is based on an API with an AngularJS (yes, that is Angular 1.0) frontend on top of it. The API was build with ASP.NET WebAPI on .NET Framework 4.5. Not a very exciting architecture these days, but back then it was still fairly new. It uses an embedded version of [RavenDB](https://ravendb.net/) as its data store (although it initially used SQL Server) since we wanted to make it really easy to get started without having to go and install all kinds of infrastructure.

## Reasons for porting
Fast forward to today and we're seeing an increasing interest in *Augurk*. Quite a few of the [Info Support](https://www.infosupport.com) teams are using it now. This year we've also had quite a bit more time to spend on it thanks to being sponsored by [Info Support Open Source](https://opensource.infosupport.com) and we've build a big new feature that we are really excited about. But the world has also changed quite a bit since 2014. For example *Augurk* is currently distributed as a WebDeploy package. While that made sense at the time, having to setup an IIS Web Server, copying the zip over, deploying the package into IIS and changing the Web.config seems like a lot of work, especially compared to the convenience of spinning up a Docker container that suddenly everybody seems to be doing.

Additionally we recognize that teams that are doing [Specification By Example](https://searchsoftwarequality.techtarget.com/definition/Specification-by-example-SBE) and/or [behavior driven development](https://searchsoftwarequality.techtarget.com/definition/Behavior-driven-development-BDD) aren't limited to using the .NET platform, so it doesn't make much sense to require them to setup Windows infrastructure just to run *Augurk*. 

So in order to increase the target audience of *Augurk* it makes sense to investigate a port of *Augurk* from ASP.NET on .NET Framework to ASP.NET Core so that we can run cross-platform. In fact, we've been thinking about porting it to .NET Core for quite a while now. Unfortunately we had to play the waiting game for some of our third-party dependencies which I'll talk about more in the next post.