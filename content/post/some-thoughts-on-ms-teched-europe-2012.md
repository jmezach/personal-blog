---
title: "Some thoughts on MS TechEd Europe 2012"
date: 2012-06-30T00:00:00+02:00
draft: false
tags: [ "Azure", "JavaScript", "Microsoft", "TechEd", "Windows Server 2012" ]
categories: [ ".NET", "Cloud", "Software Architecture" ]
---

Last week I attended Microsoft’s TechEd Europe conference in the RAI in Amsterdam. I had a great week. I’ve learned a lot of new stuff and I met with some interesting people. And since it has been a while since I’ve written anything on this blog I’d thought I’d do a little wrap-up of the conference and share with you some of my personal thoughts. I would like to stress that these are just my thoughts. They don’t necessarily represent how Info Support is thinking about these subjects. So let’s go.

## Cloud

Obviously one of the things Microsoft talked about extensively is this whole cloud computing thing. Microsoft is trying to bridge the gaps between their public cloud offering, that is Windows Azure, and their private cloud solutions using Windows Server 2012 and the whole suite of System Center products. I think that is great because it allows companies to scale out from their on-premise systems to the cloud in a matter of moments which is particularly great when you have a business that has peaks in sales at certain times.

Fortunately (or at least I think its fortunate because otherwise I would probably be out of a job) that doesn’t make writing and designing applications that run in the cloud (whether that is a public or private cloud) any easier. We as software architects and developers really need to start thinking about how to design our applications in order to allow it to scale both up and down. I think there are a lot of developers out there that really need to come to terms with this shift in how we design and write our applications.

The good thing though is that Windows Azure kind of forces you to think about these things. By separating web and worker roles you already get that scaling up because you can just run more workers when the queue containing the work to be done is starting to fill up. But I think that is only of the things we can do to allow our applications to scale up and down.

## Continuous deployment

Microsoft also talked extensively about continuous deployment. Only a couple of weeks ago they announced the new Windows Azure portal and some of the new capabilities you have there. You can now set up an Azure Website and link it to a Team Project in TFS (although only the TFS Service that is in the cloud for now). Once that is done you get a special build definition in TFS that you can run which will automatically build and deploy your web site.

This is a great feature, but again it requires you to think more about how you design your application. How can I make it super simple to deploy? How do I handle the database? Thinking about some of the projects I’ve been working on I think we’re a long way from being able to continuous deploy our applications. You really need to think about it right from the start of your project. I talked about this with one of my colleagues and he said that in the project he was working on they took quite some time in the beginning to set this up, but it was really paying off now. Doing it in retrospect at the end of a project without thinking about it upfront will probably get you into a world of trouble.

## JavaScript

There were quite a few sessions about JavaScript during this conference, and I think that the language is becoming more and more important for developers to understand and use. I attended to sessions on JavaScript. The first one was about the language itself. It gave me a good understanding of the language and it was frankly quite different from what I expected. As the speaker explained, a lot of people still see JavaScript as a silly scripting language. But this session really gave me the insight to know that that is not the case. In fact, it has some really cool ideas that other languages don’t have.

The second session I attended was about the new features in Visual Studio 2012 related to JavaScript. I must say, Microsoft went to great lengths to make JavaScript a first class language within Visual Studio. One of the really cool things they did an awesome job on is the IntelliSense support. As you may or may not know, JavaScript is a dynamically typed language which is quite different from a statically typed language such as C#. To provide you with accurate IntelliSense is quite difficult in such a language, because you won’t know what the type of certain variable might be until you actually run the code.

Turns out that that is exactly what Microsoft is doing with Visual Studio 2012. They run portions of your code in order to provide you with the most accurate IntelliSense. Under the hoods this is accomplished using the Chakra engine that is also being used by IE9 and IE10. Of course there are some caveats here, some things that won’t work, but from the demo’s I’ve seen it looks to be a pretty complete solution. It’s also extensible, so you can for example parse JSDOC style comments and provide that to the IntelliSense engine (in fact, they’ve already done it, but it will not be in the RTM of Visual Studio, but will probably be released as an extension later).

Overall it is pretty cool stuff and it will make web development using JavaScript a lot easier. But it doesn’t stop there, because we can use JavaScript to write Windows 8 applications as well, and the tools support that as well.

## Conclusion

I guess we can say that there are some interesting things going on in IT at the moment and a lot of us will need some time to adjust to that. These were just a couple of the things that really stood out for me, but there’s more of course. In fact, I attended a session on F# which was really interesting as well, but I’ll need some time to dive into that a bit more.

So what do you think? Will designing and writing applications become easier with all these new tools and developments? Or will it become harder?