---
title: "Throttling workflow services"
date: 2010-01-25T00:00:00+02:00
draft: false
tags: [ "wcf", "wf", ".NET" ]
categories: [ ".NET" ]
---

For the project I’m currently working on I did some research into the throttling of workflow services using WCF and WF with .NET 3.5\. We didn’t have much experience with throttling of services, let alone workflow services so we decided to some simple tests to determine the possibilities and effects of throttling. Before diving into the technical details of how to do throttling I want to look at the scenario we had.

### Scenario

In our current implementation we have three different workflows which are all hosted inside the same IIS application. The “Main Workflow” is started through a service operation. It goes on and invokes a couple of different services and at some point it calls a service operation to start “Sub Workflow 1”. This operation is actually called several times and “Sub Workflow 1” performs most of its work asynchronously, so the “Main Workflow” will start up these “Sub Workflows” and then stops.

The implementation of “Sub Workflow 1” eventually makes a call to an external service (thus a different IIS application) and waits for this call to complete. The external service then makes a call back to our workflow service starting yet another workflow (dubbed “Sub Workflow 2”). Schematically this looks like this:

![Image no longer available](/banners/placeholder.png "image")

The problem with this approach is that if we were to start up 30 of these sub workflow 1’s we would have all threads blocked waiting for the call to the external service to complete. Because the external service was making a call back to the same application there wouldn’t be enough threads available to process these requests, causing timeouts. Obviously this was not what we wanted.

The first solution we came up with was to split up the workflows into different applications. Logically the workflows should be in one application because they had a strong relation to each other, but we could separate them for technical reasons. So we tested this by creating a new IIS application and rerouting the traffic there. This solved our immediate problem, but we quickly found out that some of the other services couldn’t handle the load of possibly hundreds of workflows making calls to them at the same time. So we decided to look into throttling the workflows so there wouldn’t be more than a fixed number of workflows executing in parallel.

### Testing throttling

Out-of-the box WCF already supports the throttling of services. There is a ServiceThrottlingBehavior that you can use to throttle the maximum number of concurrent instances, concurrent calls and concurrent sessions. Unfortunately we couldn’t find any documentation that showed that this would also work for workflow services and the effects that these settings would have on them. So we decide to do a simple proof of concept to see how the throttling settings would affect workflow services.

To test this we made a simple solution which had a Contract project containing the WCF service contracts for both the workflow service and the external service. We also made service agents for both contracts and a UnitTest project where we implemented our scenarios. Finally we made a sequential workflow which looks like this:

![Image no longer available](/banners/placeholder.png "image")

We’ve used some custom activities here, so let me explain what this workflow does:

1.  Start a new instance of the workflow by calling _IFirstService.FirstOperation()_.
2.  Synchronously call _ISecondService.SecondOperation()_.
3.  Return the result for _IFirstService.FirstOperation()_.
4.  Asynchronously call _ISecondService.ThirdOperation()_ and wait for it to complete.

To make this test as real as possible we implemented a mock for the _ISecondService_ service contract. The mock increases a counter in the _SecondOperation_ call (so we can determine how many workflows were running in parallel). In the _ThirdOperation_ call we used a **ManualResetEvent** so we could control when they were released. We then determined three scenarios of interest:

1.  What happens if the **ServiceThrottlingBehavior** is configured to allow a maximum of 3 concurrent instances and we would call the _FirstOperation_ 4 times. We expected that the fourth call would block until we’ve released one of the other three workflows  by setting the **ManualResetEvent**.
2.  What happens in the above scenario if one of the workflows would get suspended? We expected that the fourth call would then be unblocked.
3.  What happens if the suspended workflow from the above scenario was resumed while the other three were still waiting for the result of the _ThirdOperation_ call?

### Results

After fiddling around with the tests to create the scenarios described above we found out that the **ServiceThrottlingBehavior** also works for workflow service, so that was good. In scenario 1 everything worked as we expected. The fourth call was being blocked until we released one of the other three workflows. Once we did that our counter would be increased to 4 so we knew that the fourth workflow actually executed.

Scenario two however didn’t work out-of-the-box. The reason for this was that when a workflow suspends it is not automatically unloaded. This means that the workflow would still be running, thus our fourth workflow didn’t start as there were only three workflows allowed to execute at the same time. But if we unloaded the workflow when it got suspended (meaning it would also get persisted), the fourth workflow would start and everything worked exactly as in scenario one.

The last scenario doesn’t work either. When a request to resume a workflow instance came in while the other three were still executing this request would still be executed, thus four workflows would be running in parallel. This might seem odd, but we reasoned that resuming the workflow instance isn’t done through the WF/WCF integration, but through the underlying **WorkflowRuntime** directly. We could possibly implement this behavior ourselves, but it might not be important enough to do so.

### Conclusion

We concluded that throttling with workflow services basically works how you would expect it to work. I can understand the reason for not immediately unloading a workflow instance when it becomes suspended although it would have been nice if these could be configured somehow. Fortunately we already had a **WorkflowRuntimeService** in place where we could implement this functionality ourselves.