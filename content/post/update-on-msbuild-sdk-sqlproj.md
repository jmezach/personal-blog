---
title: "An update on MSBuild.Sdk.SqlProj"
date: 2020-09-09T18:42:10Z
draft: false
tags: [ ".NET", "SQL Server", "MSBuild", "DevOps" ]
categories: [ ".NET" ]
---

A couple of months ago I first wrote about an open source project that I started called [MSBuild.Sdk.SqlProj]({{< relref "introducing-msbuild-sdk-sqlproj" >}}). Since then it has gained some traction and I'm happy to say it is probably the most succesful open source project I've launched ever. So I though this would be a good time to give an update on the project and share some of the things that are new. If you're unfamiliar with the project I recommend reading the [announcement blog post]({{< relref "introducing-msbuild-sdk-sqlproj" >}}) first.

## Community update
Popularity of open source projects is always a though to measure, but I wanted to share some numbers just to give an idea:

- Downloads on [NuGet.org](https://www.nuget.org/packages/MSBuild.Sdk.SqlProj): > 10.000
- [GitHub](https://github.com/rr-wfm/MSBuild.Sdk.SqlProj) stars: 55
- GitHub contributors: 7
- GitHub issues filed: 35
- GitHub issues closed: 28
- GitHub pull requests merged: 21
- Views of the original blogpost: 1650

Now I realise that for some open source projects these numbers are peanuts, but I feel that for a project such as this with its limited scope these are not too shabby. I'm also quite happy to say that we've taken on 2 external contributors who help out with triaging issues and reviewing pull requests: [ErikEJ](https://github.com/erikej) and [jeffrosenberg](https://github.com/jeffrosenberg).

Initially the project was hosted on my own personal GitHub account, but we've sasince then moved it to my employers [GitHub organization](https://github.com/rr-wfm/). We felt this was in the best interest of the project and a natural evolution of it. You can read more about that transition in the [announcement issue](https://github.com/rr-wfm/MSBuild.Sdk.SqlProj/issues/35).

Some blog posts have also been written about this project, including [this one](https://erikej.github.io/efcore/2020/05/11/ssdt-dacpac-netcore.html), [this one](https://erikej.github.io/efcore/2020/08/31/ssdt-project-azure-datastudio.html) and [this one](https://shawtyds.wordpress.com/2020/08/26/using-a-full-framework-sql-server-project-in-a-net-core-project-build/). I've also recorded a [dotnetFlix episode](https://dotnetflix.com/player/104) showcasing the project and how to use it. Finally I've submitted a session for the upcoming [.NET Conf 2020](https://www.dotnetconf.net/) that I'm hoping will be selected.

## New features
Of course we haven't stopped working on the project itself. Since its inception we've added quite a number of features with every release. You can have a look at the [release notes](https://github.com/rr-wfm/MSBuild.Sdk.SqlProj/releases) for every release of course, but here are some of the highlights.

### SQLCMD variables and pre-/post-deployment scripts
This was probably the most requested feature just after I launched the project. SSDT projects have had the possibility to include a pre- and post-deployment script, which can also leverage SQLCMD variables. These scripts are run at deployment time just before or after the model itself is being deployed. Unfortunately only one pre-deployment and one post-deployment script is supported. SSDT projects solve this limitation by allowing included scripts using the `:r <some-other-script>.sql` syntax. This is not yet supported by MSBuild.Sdk.SqlProj but has been requested in [this issue](https://github.com/rr-wfm/MSBuild.Sdk.SqlProj/issues/23). If this is important to you, feel free to give that issue a thumbsup.

### Publishing support
Having the ability to build the project on Mac and Linux is great, but it would be even better to have support for deploying your project to a local SQL Server as well (perhaps one running in a Docker container). We added that in version 1.2.0. Now you can run `dotnet publish` on your project and it will be deployed to a local SQL Server. Of course you can customize things like the server to deploy to, the name of the database, the username/password used to connect and pretty much every other aspect of the deployment by setting some properties in your project file. Refer to the [README](https://github.com/rr-wfm/MSBuild.Sdk.SqlProj#publishing-support) for more details on this feature.

### Floating versions and incremental build support
One of the key things I wanted to achieve with MSBuild.Sdk.SqlProj was having the ability to install NuGet packages into your project. While that already worked quite well with the initial version, we made some changes in version 1.3.0 to allow for having floating versions. For example, you can now add `<PackageReference Include="SomeOtherDatabasePackage" Version="1.1.*" />` to your project file, and you will end up with a reference to the latest patch version of the package available.

We also improved build times by avoiding rebuilding everything if none of the input files had changed. This was especially useful in combination with the publish support added in the previous version since you might need to redeploy the database multiple times during development without changing the definition. Do note that there is currently [an issue](https://github.com/rr-wfm/MSBuild.Sdk.SqlProj/issues/48) with respect to the pre- and post-deployment scripts. Changes to those do not trigger the rebuild currently.

### Template pack
The next thing we added was a set of templates that allows you to quickly create new projects that use MSBuild.Sdk.SqlProj using the .NET CLI. This leverages the `dotnet new` experience in the .NET SDK, so after you've installed the template pack you can run `dotnet new sqlproj` and you will get a new project that's ready to go. We even included a bunch of item templates for common SQL objects such as tables, views and stored procedure so you can quickly create those objects in your project as well.

For now these templates only work from the command line, which is good for Mac and Linux users, but not ideal for Windows users that work with Visual Studio. Fortunately the Visual Studio team is working on a feature that will allow you to use these templates from Visual Studio directly, rather than having to drop down to the command line. The feature is currently in preview and we've already submitted some feedback to make sure it works great with these templates so stay tuned for that.

### External databases
As mentioned previously, having the ability to pull in definitions of database objects using NuGet packages is really powerful and one of the core design principles behind the project. But the SSDT projects also allow you to reference objects that are defined in an external database, thus not the one you're currently building. We added support for that in MSBuild.Sdk.SqlProj in version 1.6.0. For example, now you can add `<PackageReference Include="SomeOtherDatabasePackage" Version="1.1.0" DatabaseVariableLiteralValue="MyOtherDatabase" />` to your project file and then you can use objects from that database using `[MyOtherDatabase].[dbo].[SomeTable]` for example.

## Conclusion
I feel that the future is bright for this project. I'm really happy with the traction it is gaining and the support I've been getting from the community. I'm also pleased with the pace of innovation and how quickly we've been able to add new features. If you haven't given it a try yet, please do so and report any issues you run into. We are happy to help out.