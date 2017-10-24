---
title: "Issue fixed"
date: 2008-04-30T00:00:00+02:00
draft: false
tags: [ ".NET", "WF", "WCF" ]
categories: [ ".NET" ]
---

I thought I'd write a little follow up on my [previous post](/2008/04/28/state-machine-workflows). We found out yesterday that the bug we found in Windows Workflow Foundation which caused us to make a work around, has been fixed in the recently released .NET Framework 2.0 and 3.0 SP1\. We haven't been able to find a mention of this bug in the release notes, but we had a small application that showed the bug was there, so we tested it on our development machine which were using the service packs and then on a test server which didn't have the service pack installed and it showed that it was working find on our development machines, but it didn't work on our test servers.

So yesterday we've removed the work around from the custom activity we had and created a patch for the application which can be rolled out to the production environment. We still have to delete all the broken workflows though, because once the currently existing timers go off, the activity will Execute again, and once that happens the users won't be able to use those workflows anymore. But at least the bug is fixed and we don't have to use the workaround anymore.

This doesn't mean however that you can start writing activities that return **ActivityExecutionState.Executing** from their Execute method when being used in an **EventDrivenActivity** in a state machine workflow. This will still prevent other **EventDrivenActivities** in the same state from becoming active when a message is delivered to their queue.