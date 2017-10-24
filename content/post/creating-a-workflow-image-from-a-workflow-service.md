---
title: "Creating a workflow image from a workflow service"
date: 2009-11-18T00:00:00+02:00
draft: false
tags: [ ".NET", "WF", "WCF" ]
categories: [ ".NET" ]
---

#### Introduction

For the project I’m currently working on we wanted to be able to display an image of a running workflow instance for diagnostic purposes. There are numerous examples of this floating around the internet, but most of them focused on a client side application that would generate the image. This is fine if your client application has access to all the necessary assemblies, but if does not you’re going to run into some problems. In this post I’m going to explain how we solved this problem and some of the pitfalls you might run into.

#### Scenario

We currently have a fairly large Service Oriented Architecture. We’ve identified a couple of processes that we wanted to support using Windows Workflow Foundation. These processes are implemented as WCF services using Workflow as the actual implementation. There are a couple of these Process Services which each have one or more workflows.

Obviously these Process Services were running in IIS so in order to get an image of a running workflow service we needed some kind of front end application. We already had the infrastructure to resume or terminate suspended workflows from a ASP.Net web application so I decided to extend that with an image of the workflow.

#### Challenges

I had a look at the Workflow Monitor sample application included in WF and WCF samples for .NET 3.0/3.5\. This application basically re-hosts the Workflow designer and pulls data from the SQL tracking store to display which activities have been executed. A nice application, but it didn’t really suit my scenario because that would require my front end web application to have access to all assemblies that my workflow used. This included assemblies containing activities that I used in my workflow as well as the assembly that contained my workflow definition. Not very practical if you have multiple services because I would need to get the assemblies from each of them.

In addition to this, the Workflow Monitor application is a Windows Forms application with interactive controls and such. That didn’t work very well with my ASP.Net web application. And I only needed an image of the workflow anyway and didn’t need any of the interactive stuff. So I had a look around to see if I could find any implementation that would only create the image.

I quickly found [this](http://www.masteringbiztalk.com/blogs/jon/PermaLink,guid,79f45d4d-6e5a-437b-a230-d7df13ae18e7.aspx) sample. It pretty much did what I needed it to do (and even more using Ajax), but it would still need all of the assemblies of my service in order to work. So I decided to factor out the image creation and put this part on my service and then Stream back the image through WCF to my web application. Sounds simple, right?

#### Problems

Unfortunately it wasn’t so simple. I quickly ran into an UnauthorizedAccessException. Apparently my service was suddenly trying to access the registry, but it wasn’t allowed to. After some reflecting of het Workflow assemblies I found out that the WorkflowTheme class was the culprit. This class has a static constructor which tries to get the users theme settings from the registry. But my service was running under the Application Pool Identity so it didn’t have a user profile and thus wasn’t allowed to access the registry. Because this is a static constructor it will access the registry as soon as you use that class anywhere in your code, so there wasn’t much I could do about that. But IIS 7.0 (not sure about 6.0) has the option to load a user profile for an application pool and after setting that option, my problem was solved.

Then the next problem came up. In order to provide a visual indication that an activity has executed I had to read from the SQL tracking store. This can be accomplished by using the SqlTrackingQuery class to get a SqlTrackingWorkflowInstance and then reading the ActivityEvents property. This property will execute a stored procedure on the tracking store and then transform each returned row into an ActivityTrackingRecord. This class has a property ActivityType which returns the runtime type of the activity for which that record was created. The tracking store contains both the type name and the assembly name which is transformed to the runtime type while reading the ActivityTrackingRecords. But the default workflow tracking database only has room for a type name of 128 characters. The rest was truncated which resulted in an exception while reading the ActivityEvents property.

I decided to make this column quite a bit bigger because I had a couple of type names that were considerably longer. But then I ran into another problem. To increase performance an index was made on both the type name and the assembly name. Now my index was getting to big which SQL Server wouldn’t accept. In the end I changed the data type of the column from nvarchar to varchar. I don’t think a type name can contain unicode characters anyway and it saves half the space in the database.

#### Conclusion

It’s nice to be able to re-host the workflow designer and generate images off it. But it doesn’t work very well when the application is not a Windows Forms application. It would have been nice if there was a better way of doing this without depending on the registry or the Windows Forms infrastructure for that matter.

It also seems weird to say that type names would not be longer than 128 characters. With namespaces your type names can get quite long and it even gets worse when using generics.