---
title: "Using Invoke-SqlCmd in TFS Build 2015"
date: 2015-07-03T00:00:00+02:00
draft: false
tags: [ "Team Build", "TFS", "TFS 2015" ]
categories: [ "Continuous Delivery" ]
---

Lately I've been diving into the new build engine in TFS 2015 RC (RTM release is planned for 20th of July). While running a build, I got the following error message:

> Mixed mode assembly is built against version 'v2.0.50727' of the runtime and cannot be loaded in the 4.0 runtime without additional configuration information.

This got me puzzled for a while, especially since the build continued to run and it was doing what it was supposed to do, except that at the end the build turned red because of this error. The code I was building targeted .NET Framework 4.5, so that couldn't be the problem. So what was raising this error?

Since I was working on a script that would eventually deploy databases during the build I assumed this had something to do with it. In the script I used Invoke-SqlCmd to run a query. So I logged into the build server, ran the Invoke-SqlCmd command and it worked just fine. No error message whatsoever. I then turned to Google to see if there was any relation between the error message I got and the Invoke-SqlCmd command I was using. Turns out there is, and it's fairly well documented [here](http://www.sqlmusings.com/2011/11/19/resolving-powershell-v3-ise-error-with-invoke-sqlcmd/) and [here](https://github.com/adamdriscoll/poshtools/issues/192).

However, I didn't get the error when I ran the command directly on the machine. So why does it fail during my build? As described at the links above, the solution to this problem is to set the **_useLegacyV2RuntimeActivationPolicy_** attribute to **_true_** in the app.config of the process hosting PowerShell. So I had a look at the powershell.exe.config that was installed on the build server machine (which is running PowerShell 4.0) and it turns out that this particular attribute is already set to true (at least it was on my machine). </span>So why then does the build fail?

All this got me thinking: How does the new build engine actually run tasks, which are essentially PowerShell scripts? As it turns out, the new build engine is using its own in-process PowerShell host. So rather than invoking powershell.exe to run the scripts, it is running them right inside its own process. This makes sense of course since it allows all the tasks that are running in the build to share some state.

Unfortunately this also means that in order to get the Invoke-SqlCmd command to work without errors, one has to edit the configuration file of the agent by hand and add the attribute as described above. Be careful however, since the build agents have become self-updating in TFS 2015 so the configuration file might be overwritten when a new version of TFS is installed. So for now, I would recommend not to use Invoke-SqlCmd in your build scripts.