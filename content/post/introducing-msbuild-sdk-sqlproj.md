---
title: "Introducing MSBuild.Sdk.SqlProj"
date: 2020-03-17T16:27:16+01:00
draft: false
tags: [ ".NET", "SQL Server", "MSBuild", "DevOps" ]
categories: [ ".NET" ]
---

At my current employer, R&R Workforce Management, we've done a pretty big investment on Microsoft's [SQL Server Data Tools](https://visualstudio.microsoft.com/vs/features/ssdt/) (SSDT) to manage our database changes. We made that decision when we migrated off of the Oracle database onto SQL Server which is quite a few years back now. At the time it was probably the most mature tool available that allowed us to version control our database changes much like we version controlled all our source code.

SSDT uses the so-called model based approach in that you define what you want your database to look like. Then at deployment time a diff is made against an actual database and the necessary steps to migrate from the current state of the database to the desired state is calculated and optionally ran on the target database. This means that our developers have to think less about how to come from the current state to the new state, they just define the new state and the tools take care of it.

Over time we've made various scripts to make allow developers to deploy their changes locally as well as scripts for automating the deployment to test and production environments from [Octopus Deploy](https://www.octopus.com). This has served us quite well over the years, but with some of the recent changes going on in Visual Studio and the related tooling it seems that the SSDT tools are kind of being left behind.

For example one thing that has always bothered me (and judging by this [GitHub Issue](https://github.com/NuGet/Home/issues/545) I'm not the only one) is the inability to install NuGet packages into a .sqlproj. Why would you want to do this? One reason is that SSDT projects do support referencing other SSDT projects, so it kinda makes sense to allow this using NuGet packages since that's been the way we share dependencies within the .NET ecosystem. Another reasons might be that you want to customize the MSBuild process which can be easily accomplished using NuGet packages, but alas is not possible currently with the .sqlproj project format.

Luckily the underlying technology DacFx has had a [public API](https://devblogs.microsoft.com/ssdt/dacfx-public-model-tutorial/) for quite some time which allows creating a .dacpac package, which is the primary output of a .sqlproj project. And after finding this [GitHub Issue](https://github.com/microsoft/DACExtensions/issues/20) I saw that it had been ported to .NET Core as well. Combined with the knowledge of [MSBuild SDK's](https://github.com/microsoft/MSBuildSdks) I figured maybe we can solve this issue and bring existing projects forward with some minor sacrifices.

## First steps
