---
title: ".NET Core Hackathon 2018"
date: 2018-06-14T20:00:00+02:00
draft: false
tags: [ ".NET", ".NET Core", ".NET Development", "Hackathon", "open source" ]
categories: [ ".NET" ]
---

It has been a while since I've posted something here on my blog. That's not because I haven't done anything, just that I've been really busy lately with all kinds of things. For example, I've been delving into the container world and delivered a [hands-on workshop](https://github.com/jmezach/globalazurebootcamp2018kube) with [Kubernetes](https://kubernetes.io/) on [Azure](https://azure.microsoft.com/en-us/) as part of the [Global Azure Bootcamp 2018](https://global.azurebootcamp.net/) back in April. That was a lot of fun, since I didn't know much about Kubernetes before I started working on that, so I learned a lot in a relatively short amount of time, but I digress.

Of course, that doesn't mean that I've turned away from .NET Core. In fact, I had the honor to organize the second .NET Core Hackathon, which was held on the 2nd of June at Startup Village in Amsterdam. We held a similar event last year, which was organized by my colleague [Edwin van Wijk](https://twitter.com/evanwijk). Back then, we had about 10 of our colleagues joining us in Amsterdam to fix some bugs and submit some pull requests to the [corefx](https://github.com/dotnet/corefx) repo. You can check out a video of that event over on [YouTube](https://www.youtube.com/watch?v=t1JWX8D0ptE&t=16s).

# Fast-forward to 2018

So, this year it was my turn to organize a similar event. But we wanted to do things a little bit different. First of all, we wanted to include more people, so it wouldn't be just our colleagues. Second, we wanted to do it together with the .NET Core team at Microsoft. After last year Edwin and I stayed in contact with [Karel Zikmund](https://twitter.com/ziki_cz) from the team. They thought our hackathon was a great idea, so we worked together this time around.

That meant that on the day of the hackathon (2nd of June) we had a couple of people from the team standing by to answer our questions and to help us out if we would run into issues. Especially [Viktor Hofer](https://twitter.com/viktorhofer) was very helpful during the day.

# An impression

So what did we do on that day? First of all, we gathered at [Startup Village](https://startupvillage.nl/) in Amsterdam. It was the same location as last year, but it's such a great place to get into mood to do some hacking, that we decided to stick with it for this years edition. Of course, we started with some coffee, in our limited edition .NET Core Contributor mugs:

![.NET Core Contributor Mug](/img/net-core-hackathon-2018/contributor-mug.jpg)

Next I started with a small presentation about what new contributors should know about the corefx repo. It's a big repository and there are some intricacies that can make it difficult for first time contributors to get started. There is some pretty good documentation on the [wiki](https://github.com/dotnet/corefx/wiki) for the repo, but I think it always helps to summarize some of that information into a small crash course like presentation.

![Kick-off](/img/net-core-hackathon-2018/kick-off.jpg)

After that, everybody just went straight in and tried to clone the repo, build it, run the tests, etc. Of course people ran into a couple of issues while doing that, so I tried to help them out as much as I could. And if I couldn't help them out myself we would turn to the Gitter channel that was set up by the team, where Viktor was more than willing to help people out with their issues. Around lunch time almost everybody had a working setup and was busy working on a bugfix or (small) new feature.

For lunch we ordered pizza, with the "something with meat" flavour being the most favorite choice. When it arrived, we went outside to decompress a bit and just sat back and relaxed for a bit whilst chatting with each other.

![Lunch](/img/net-core-hackathon-2018/lunch.jpg)

**Note:** There were some more people there, but I didn't get a shot of all of us ;).

# Getting our pull requests on

After lunch it was time to submit some pull requests. Personally I still think it's kind of magic when you submit a pull request that immediately almost 18 builds are kicked-off. I've been doing professional software development for almost 12 years now, but I don't think I've ever seen that many builds running for a single pull request.

In the end we ended up with a total of 6 pull requests on that day. It turned out that we even made a pull request to [coreclr](https://github.com/dotnet/coreclr), which is the .NET Runtime. Trust me, that's not the easiest piece of code to read, let alone write and tweak, so kudos to [Mikhail](https://twitter.com/MikhailShilkov) for making that happen.

Here's the list of pull request we have submitted:

- [Add All enum value to DecompressionMethods](https://github.com/dotnet/corefx/pull/30075)
- [Remove USE_ETW Compilation Constant from System.Diagnostics.Tracing.Tests](https://github.com/dotnet/corefx/pull/30076)
- [Implementation of IReadOnlyDictionary on GroupCollection](https://github.com/dotnet/corefx/pull/30077)
- [Improved code coverage for System.Net.Security](https://github.com/dotnet/corefx/pull/30082)
- [Implement corefx/#16619: Add FormattableString.CurrentCulture](https://github.com/dotnet/coreclr/pull/18253)
- [Implement #16619: Add FormattableString.CurrentCulture](https://github.com/dotnet/corefx/pull/30083)

# Conclusion

So, that's 6 pull requests, that have since been merged as well. That's a couple less than last year, but I think the things we worked on this year were also a bit more involved than last year. For example, last year we had a couple of items that merely removed code that was already dead. But this time, we had some tougher things to work on, which made it a bit more of challenge.

That being said, I think we all had a fun day and I hope that some of the people that were there will continue to contribute to .NET Core in the future as well. I was a bit sad that I didn't get to submit a pull request myself this time around, but I think that helping other people to get started contributing was worth it in the end.

I'm not sure if we'll do another event next year. I guess we'll have to wait and see. Make sure you follow me on Twitter and you'll be the first to know. Also, if you're interested in organizing such an event in your part of the world, do let me know. Together with the team, we can do some awesome things.

![Impression](/img/net-core-hackathon-2018/impression.jpg)