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

Luckily the underlying technology DacFx has had a [public API](https://devblogs.microsoft.com/ssdt/dacfx-public-model-tutorial/) for quite some time which allows creating a .dacpac package, which is the primary output of a .sqlproj project. And after finding this [GitHub Issue](https://github.com/microsoft/DACExtensions/issues/20) I saw that it had been ported to .NET Core as well. Combined with the knowledge of [MSBuild SDK's](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-use-project-sdk?view=vs-2019) I figured maybe we can solve this issue and bring existing projects forward with some minor sacrifices.

## First steps

My first step was to actually make an MSBuild SDK. There are some [examples](https://github.com/microsoft/MSBuildSdks) available and as we're not producing an assembly I used the `Microsoft.Build.NoTargets` SDK as a starting point for my project. Fairly quickly I was able to produce a NuGet package locally (using `dotnet pack`) that I could then reference as an SDK from another project using a project file like this:

```xml
<Project Sdk="MSBuild.Sdk.SqlProj/1.0.0">
    <PropertyGroup>
        <TargetFramework>netstandard2.0</TargetFramework>
    </PropertyGroup>
</Project>
```

Note here that the version number is actually important. By adding the version number a NuGet based SDK resolver built into MSBuild kicks in which will download the SDK package from whatever NuGet feeds you have configured, puts it into your local cache. After that your project file is essentially rewritten to something like this:

```xml
<Project>
    <Import Project="C:\Users\<your-username>\.nuget\packages\MSBuild.Sdk.SqlProj\1.0.0\Sdk\Sdk.props" />

    <PropertyGroup>
        <TargetFramework>netstandard2.0</TargetFramework>
    </PropertyGroup>

    <Import Project="C:\Users\<your-username>\.nuget\packages\MSBuild.Sdk.SqlProj\1.0.0\Sdk\Sdk.targets" />
</Project>
```

With that in place we can define some properties, items and targets for MSBuild to pick up inside the `Sdk.props` and `Sdk.targets` files. If your new to MSBuild be sure to check out their [documentation](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild?view=vs-2019). I'm no expert on MSBuild to be honest, but I did learn a lot from this project.

My next challenge was how I was going to invoke the DacFx public API from MSBuild. For my first attempt I really just wanted to build a `TSqlModel` using the files that were part of the project.  My initial idea was to build a custom MSBuild task that would accept the input files, read them one by one and add them to the model:

```csharp
public class DacpacBuildTask : Task
{
    public ITaskItem[] InputFiles

    public override bool Execute()
    {
        using var model = new TSqlModel(SqlServerVersion.Sql150, new TSqlModelOptions());
        foreach (var inputFile in InputFiles)
        {
            model.AddObjects(File.ReadAllText(inputFile));
        }

        DacPackageExtensions.BuildPackage("output.dacpac", model, new PackageMetadata { }, new PackageOptions { });
        return true;
    }
}
```

Unfortunately I quickly found out that MSBuild has quite a few issues with [loading dependencies at runtime](https://natemcmaster.com/blog/2017/11/11/msbuild-task-with-dependencies/) (thanks to Nate). So I quickly abandoned this path and decided to focus on writing a command line tool that would build the `.dacpac` package and then execute that tool from the `Sdk.targets` file.

So I wrote a very rough first implementation of the command line tool as a separate .NET Core console application that looked pretty much like this:

```csharp
class Program
{
    static void Main(string[] args)
    {
        using var model = new TSqlModel(SqlServerVersion.Sql150, new TSqlModelOptions());
        foreach (var inputFile in args)
        {
            model.AddObjects(File.ReadAllText(inputFile));
        }

        DacPackageExtensions.BuildPackage("output.dacpac", model, new PackageMetadata { }, new PackageOptions { });
    }
}
```

I then started working on calling the above tool from the SDK package. I first added this to the `Sdk.props` file to make sure that all the `.sql` files within the project folder are included as `Content` items.

```xml
<ItemGroup>
    <Content Include="**\*.sql" />
</ItemGroup>
```

We can then use that item to build up the command line for the tool I talked about earlier from the `Sdk.targets`:

```xml
<Target Name="CoreCompile">
    <PropertyGroup>
      <InputFileArguments>@(Content->'&quot;%(FullPath)&quot;', ' ')</InputFileArguments>
    </PropertyGroup>
    <Exec Command="dotnet $(MSBuildThisFileDirectory)../tools/BuildDacpac.dll $(InputFileArguments)" />
</Target>
```

Now when I run a `dotnet build` on the project the command line tool is invoked with all `.sql` files inside the project's folder (and subfolders) passed as arguments. It reads those files in to build a `TSqlModel` and writes it to a `.dacpac` package. Sweet, now I don't have to put each individual `.sql` file into my project file anymore, which already saves a ton of merge conflicts. An additional benefit is that this also allows building a `.dacpac` on non-Windows machines, which wasn't possible before.

## Supporting package references
But my primary motivation for this was to allow these projects to support `PackageReference`. Fortunately since we now have a `.csproj` instead of a `.sqlproj` all the tools you're already familiar with basically just work. You can install a NuGet package into the project using the NuGet Package Manager in Visual Studio, using the `Install-Package` command from the Package Manager Console or by using `dotnet add package` from the command line.

Of course all those gestures only really add a `PackageReference` element to our project file and don't do much more. Now it's up to our MSBuild SDK package to do something useful with that information. I needed some way of resolving the `PackageReference` into a physical location on disk so I could figure out if there's a `.dacpac` in their somewhere that we can reference. MSBuild has what's called a path property for this which can be enabled like this:

```xml
<Project Sdk="MSBuild.Sdk.SqlProj/1.0.0">
    <PropertyGroup>
        <TargetFramework>netstandard2.0</TargetFramework>
    </PropertyGroup>

    <ItemGroup>
        <PackageReference Include="MyDatabasePackage" Version="1.0.0" GeneratePathProperty="true" />
    </ItemGroup>
</Project>
```

What this is supposed to do is create an MSBuild property named `PkgMyDatabasePackage` which contains the physical location of that package on disk (ie. `C:\Users\<your-user-name>\.nuget\packages\MyDatabasePackage\1.0.0`). However this behavior isn't very predictable. Sometimes MSBuild generates this property automatically, even if you do not set the `GeneratePathProperty` attribute. And sometimes even if you set it, it still doesn't generate the property. It also doesn't seem to work with older versions of MSBuild.

So instead of relying on this flaky behavior I decided, let's make this simple and use the `NuGetPackageRoot` property which points to the root of the users package cache (ie. `C:\Users\<your-user-name>\.nuget\packages`). From there we can just append the package ID and version and we get the same path. I wrote a small MSBuild target to capture this:

```xml
<Target Name="ResolveDatabasePackageReferences">
    <ItemGroup> 
        <_ResolvedPackageReference Include="%(PackageReference.Identity)">
            <PhysicalLocation>$(NuGetPackageRoot)%(PackageReference.Identity)\%(PackageReference.Version)</PhysicalLocation>
        </_ResolvedPackageReference>
        <DacpacReference Include="@(_ResolvedPackageReference)" Condition="Exists('%(PhysicalLocation)\tools\%(Identity).dacpac')" />
    </ItemGroup>

    <Message Importance="normal" Text="Resolved package reference %(_ResolvedPackageReference.Identity) to %(_ResolvedPackageReference.PhysicalLocation)" />
    <Message Importance="normal" Text="Resolved database package references: @(DacpacReference)" />
</Target>
```

First we resolve all package references to their physical location by create a new item that includes the physical location as metadata. Then we use that item to filter out all the packages that do not have a `.dacpac` file inside the `tools` folder whose name matches the package ID. I added the `Message` tasks for diagnostic purposes. Now we are able to pass in the `.dacpac` files to the command line tool by slightly modifying our `CoreCompile` target:

```xml
<Target Name="CoreCompile" DependsOnTargets="ResolveDatabasePackageReferences">
    <PropertyGroup>
      <InputFileArguments>@(Content->'&quot;%(FullPath)&quot;', ' ')</InputFileArguments>
      <ReferenceArguments>@(DacpacReference->'-r &quot;%(PhysicalLocation)\tools\%(Identity).dacpac&quot;', ' ')
    </PropertyGroup>
    <Exec Command="dotnet $(MSBuildThisFileDirectory)../tools/BuildDacpac.dll $(ReferenceArguments) $(InputFileArguments)" />
</Target>
```

The next step was adding those references to the `TSqlModel` so that any objects defined in them could be resolved. Unfortunately this is where things started to get ugly. The DacFx public API doesn't support adding references to other `.dacpac` files. Luckily after some [ILSpy](https://github.com/icsharpcode/ILSpy)'ing I found out that all the logic is there, it's just that the API hasn't been made available publicly. That meant I had to do some reflection, so I wrote a little extension method:

```csharp
public static void AddReference(this TSqlModel model, string referencePath)
{
    var service = model.GetType().GetField("_service", BindingFlags.NonPublic | BindingFlags.Instance).GetValue(model);
    var dataSchemaModel = service.GetType().GetProperty("DataSchemaModel", BindingFlags.NonPublic | BindingFlags.Instance).GetValue(service);

    var customData = Activator.CreateInstance(Type.GetType("Microsoft.Data.Tools.Schema.SchemaModel.CustomSchemaData, Microsoft.Data.Tools.Schema.Sql"), "Reference", "SqlSchema");
    var setMetadataMethod = customData.GetType().GetMethod("SetMetadata", BindingFlags.Public | BindingFlags.Instance);
    setMetadataMethod.Invoke(customData, new object[] { "FileName", referencePath });
    setMetadataMethod.Invoke(customData, new object[] { "LogicalName", Path.GetFileName(referencePath) });
    setMetadataMethod.Invoke(customData, new object[] { "SuppressMissingDependenciesErrors", "False" });

    var addCustomDataMethod = dataSchemaModel.GetType().GetMethod("AddCustomData", BindingFlags.Public | BindingFlags.Instance);
    addCustomDataMethod.Invoke(dataSchemaModel, new object[] { customData });
}
```

> Yes, that's not very pretty and nothing I generally recommend doing. But in this case I figured that if I could limit it to this single extension method I could probably live with it, given the potential benefits.

Now all I needed to do is wire up the arguments to this extension method. Order is important though, so I made sure that we add the references first before we try to add any of the objects from the input files. Otherwise you'll get validation errors when the package is being saved.