---
title: "Building a Pull Request bot with Azure Functions - Part 2 - How it works"
date: 2020-02-10T09:14:51+01:00
draft: false
tags: [ ".NET", "Azure Functions", "DevOps" ]
categories: [ "Azure" ]
---

This is the second post in my series on building a pull request bot using Azure Functions. If you haven't read my [first post]({{< relref "building-a-pull-request-bot-part1" >}}) I strongly recommend doing so before diving into this post.

- [Part 1: Introduction]({{< relref "building-a-pull-request-bot-part1" >}})
- [Part 2: How it works]({{< relref "building-a-pull-request-bot-part2" >}})

In this post I'll dive into the details of how the bot actually works, from receiving notifications from Azure DevOps to managing state and posting comments back to the pull request.

## Receiving notifications
The first thing we'll need for a pull request bot to work is to receive notifications when pull requests are created. Since we were using Azure DevOps (Server) we could use its Service Hooks feature to receive these notifications. Of course this could work with other systems as well, such as GitHub which has a similar feature. All these features are generally referred to as *webhooks*. Both Azure DevOps and GitHub will essentially do an HTTP POST request to an endpoint and will send along a payload containing the details of an event that has occurred. HTTP triggered Azure Functions are ideal for this. For example, this is the definition of the Azure Function that receives the pull request created event from Azure DevOps:

```csharp
[FunctionName(nameof(PullRequestCreated))]
public async Task<IActionResult> PullRequestCreated(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "pullrequests/created")] HttpRequest req,
    [OrchestrationClient] DurableOrchestrationClientBase orchestrator,
    ILogger log)
{
}
```

There are a couple of things to note here. First of all, you might notice that this isn't a static method while most examples for Azure Functions use static methods. This is because we're using dependency injection which requires an instance of class to be created in order for the dependencies to be injected.

The other thing you might notice is that we're using `AuthorizationLevel.Anonymous`, which means that anybody can execute this function. That doesn't mean that we don't do any authentication, but we've implemented it as part of the function itself. The request body contains a subscription identifier which is the identifier of the service hook in Azure DevOps. We've configured the bot with a set of subscription identifiers that are allowed, and deny requests that either do not have a subscription identifier or if the identifier is not in the list of authorized identifiers. In hindsight we could have used `AuthorizationLevel.Function` instead, which uses a key that we could have configured on the Azure DevOps side, which made the overall configuration a little bit easier. Never too late to change it I guess.

Next step was to actually do something with the payload. As it turns out, Microsoft has a [webhooks library](https://www.nuget.org/packages/Microsoft.AspNet.WebHooks.Receivers.VSTS/) with the definitions of all the payloads that Azure DevOps posts to your endpoint. That makes it a lot easier to deserialize the request body into something we can use. It contains among other things the pull request that has been created, who created it, the subscription identifier that triggered the event being posted, etc.

Of course we also want to run our validations again when the pull request has been updated so we added an additional HTTP triggered function which is almost the same:

```csharp
[FunctionName(nameof(PullRequestUpdated))]
public async Task<IActionResult> PullRequestUpdated(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "pullrequests/updated")] HttpRequest req,
    [OrchestrationClient] DurableOrchestrationClientBase orchestrator,
    ILogger log)
{
}
```

Finally we figured that while we were at it it makes sense to also run our validations whenever a build completes that was triggered as part of a pull request so that we can do additional checks on those builds as well. So we added yet another HTTP triggered function to do just that:

```csharp
[FunctionName(nameof(BuildCompleted))]
public async Task<IActionResult> BuildCompleted(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "builds/completed")] HttpRequest req,
    [OrchestrationClient] DurableOrchestrationClientBase orchestrator,
    ILogger log)
{
}
```

As you can probably guess by now all these functions have a similar implementation which revolves around the `OrchestrationClient` that is injected into each of those functions, which is the entry point for Durable Functions. Let's dive in to see what happens there.

## Orchestration

The orchestration is what maintains the state of the validations that have run for a pull request. When the pull request created function from above runs, it starts a new instance of the orchestration:

```csharp
// Build the analysis context
var instanceId = payload.Resource.PullRequestId.ToString();
var pullRequest = new PullRequest
(
    currentState: payload.Resource.Status,
    pullRequestId: payload.Resource.PullRequestId.ToString(),
    projectId: payload.Resource.Repository.Project.Id,
    repositoryId: payload.Resource.Repository.Id,
    repositoryName: payload.Resource.Repository.Name,
    sourceRefName: payload.Resource.SourceRefName,
    targetRefName: payload.Resource.TargetRefName,
    created: payload.Resource.CreationDate
);

var context = new AnalysisContext(pullRequest);

// Initialize a new orchestration
await orchestrator.StartNewAsync(nameof(PullRequestFlow), instanceId, context);
```

As you can see in the code sample above we first create an object to represent the pull request, with some basic information that we get from the payload such as its state, the repository it has been created in, the source and target branches, etc. We then wrap that into an `AnalysisContext` (more on that later) and start a new instance of the `PullRequestFlow` orchestration with the context and give it an identifier that matches the identifier of the pull request.

The pull request flow is a durable orchestration that will keep running until the associated pull request is completed. It doesn't do much in and of itself, but delegates actual work to other functions and coordinates those. 