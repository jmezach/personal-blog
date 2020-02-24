---
title: "Building a Pull Request bot with Azure Functions - Part 3 - Operating it"
date: 2020-02-24T20:06:15+01:00
draft: false
tags: [ ".NET", "Azure Functions", "DevOps" ]
categories: [ "Azure" ]
---

This is the third post in my series on building a pull request bot using Azure Functions. If you haven't read my previous posts I strongly recommend doing so before diving into this post. Remember that this series is a part of the [Applied Cloud Stories](http://aka.ms/applied-cloud-stories) initiative.

- [Part 1: Introduction]({{< relref "building-a-pull-request-bot-part1" >}})
- [Part 2: How it works]({{< relref "building-a-pull-request-bot-part2" >}})
- [Part 3: Operating it]({{< relref "building-a-pull-request-bot-part3" >}})

In this post I want to give you a feel for how we operate our pull request on a daily basis. How do we deploy it, how do we make sure it is running, etc.

## CI/CD
Of course we are applying continuous integration and continuous deployment practices to the bot, just like we do for our product. And of course we use [Azure Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/) to do it. For our build pipeline we don't have anything fancy really. Azure Functions v2 runs on .NET Core, so we have a pipeline that runs `dotnet restore`, `dotnet build`, `dotnet test` and `dotnet publish` in that order. For the publish step we target the actual Azure Functions project so that we get a nice ZIP file containing everything we need to run. Finally we have two steps that upload artifacts to Azure DevOps, one that uploads the ZIP and one that upload the Azure Resource Manager (ARM) template (more on that later).

At the moment the pipeline is defined using the visual pipeline editor. I guess that's fine for such a simple pipeline, but we have been experimenting with YAML pipelines for parts of our product so we might switch to that at some point. But that's a topic in and of itself so I'll save that for a future post.

For our deployment we're using Release definitions in Azure Pipelines. We have created two stages here. The first stage deploys the infrastructure we need to run on using the ARM template I mentioned earlier. It's really just one [Azure Resource Group Deployment](https://github.com/microsoft/azure-pipelines-tasks/blob/master/Tasks/AzureResourceGroupDeploymentV2/README.md) step that takes the ARM template and applies it to a resource group in Azure. The nice thing about this is that its idempotent. If you deploy the same template over and over the infrastructure will just stay the same. If you do make a change only that change is applied and the rest stays the same.

As for the contents of the ARM template, I've made a graphical representation of it using the excellent [ARM Template Viewer extension for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=bencoleman.armview):

{{< figure src="/img/building-a-pullrequest-bot/arm-template.png" title="ARM Template" >}}

It's not very complicated really. We have an Azure Function App (top right) that uses a Storage Account (bottom right) to store the state and an App Service Plan (bottom left) that provides the necessary infrastructure to run our Function app. At the moment we're using the Consumption Plan so that we only really pay for what we use. Finally there's the Application Insights resource (top left) that gives us the power to monitor the bot and look at logs etc. More on that later.

> As a side note, I currently do not have visibility into the costs we're making for this bot. This is because we have CSP (Cloud Solution Provider) subscription for which Azure Cost Management has only recently became available. Unfortunately that requires some migration work that's currently ongoing. So far it barely registers on our overall Azure bill though.

After we've deployed our infrastructure we can deploy the application itself. This is done using a [Azure App Service deploy](https://github.com/microsoft/azure-pipelines-tasks/blob/master/Tasks/AzureRmWebAppDeploymentV4/README.md) step that simply uploads the ZIP we've built in our build pipeline to Azure. With that deployment we also pass in some configuration variables, such as the URL to our Azure DevOps organization, a personal access token and the subscription identifiers of the service hooks we've created in Azure DevOps for authentication purposes (see my [second post]({{< relref "building-a-pull-request-bot-part2" >}}) in this series). Most of the variables are defined as secret variables on the release pipeline.

To make sure that we leave the bot in a working state after deployment we've also added a [deployment gate](https://docs.microsoft.com/en-us/azure/devops/pipelines/release/approvals/gates?view=azure-devops) on the last stage of the pipeline. This calls into a Health Check API we've implemented in the bot itself, which is just an HTTP triggered function that uses ASP.NET Core's [Health Checking](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-3.1) capabilities. In this health check we're checking to make sure we can access the storage account and that the subscription identifiers that have been configured are valid and are enabled by calling into Azure DevOps. So far we haven't set up an automatic periodic check of the health, but at least we know that after a deployment things are still working.

## Monitoring
