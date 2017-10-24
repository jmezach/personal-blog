---
title: "Reactive Extensions"
date: 2010-11-04T00:00:00+02:00
draft: false
tags: [ "async", "C#", "RX" ]
categories: [ ".NET" ]
---

For the past couple of weeks I’ve been looking into the Reactive Extensions framework. This resulted into a presentation for some of my colleagues about what Rx is, how it can be used and some of the cool stuff you can do with it. I delivered that presentation yesterday evening. Today I wanted to share the sources of the demo’s I’ve used during this presentation. You can download these from here: [RxDemos.zip](/wp-content/uploads/archief/4ea03a7745a116.96068325.zip). Please note that I’ve also included a couple of demo’s on the new async programming feature in C# vNext, so you’ll need the Async CTP to get this to compile. You can download that from: [Visual Studio Async CTP](http://msdn.microsoft.com/en-gb/vstudio/async.aspx). You will also need the Windows Phone 7 SDK which can be download from: [Windows Phone 7 SDK](http://create.msdn.com/en-us/home/getting_started).

The zip file contains the following projects:

*   **ConsoleDemos**

> Contains quite a few simple demo’s that demonstrate how some of the core concepts of Reactive Extensions work. I’ve split these up into several separate files to explain the different concepts:
> 
> *   **AsyncFileReader.cs**: Implements IAsyncEnumerable<T> and IAsyncEnumerator<T> from Rx which reads lines from a file on disk asynchronously.
> *   **CSharpAsync.cs**: Demonstrates how the new asynchronous programming feature in C# vNext integrated with Rx.
> *   **ObservableFactoryMethods.cs**: Demonstrates a couple of factory methods that are available on Observable that allow you to quickly create Observable collections.
> *   **ObservableProgress.cs**: Simple implementation of the new IProgress<T> interface in the Async CTP that combines progress reporting with Rx to enable an application to respond to progress changes in the asynchronous operation.
> *   **Program.cs**: Contains the start up code.
> *   **Sample.txt**: Sample file used by the AsyncFileReader class.
> *   **Scheduling.cs**: Demonstrates how scheduling works in Rx using ObserveOn and SubscribeOn.
> *   **Subjects.cs**: Demonstrates the different Subject<T> types that are available.
> *   **Utilities.cs**: Contains some utility methods for writing observable collections to the console.

*   **DemoService**

> A simple WCF service that is used for demoing the FromAsyncPattern feature.

*   **TwitterPhoneDemo**

> This is the Twitter demo I’ve used during the presentation. It is based on the Windows Phone 7 Panorama Application template so most of the code you get out-of-the-box. It used the TweetSharp library to communicate with Twitter. TweetSharp uses Hammock to do all the REST calls to Twitter and the OAuth stuff. Unfortunately there was a bug with streaming in the current Hammock version that I’ve had to patch. That patch has now been integrated into the main Hammock source so a next version should fix that.
> 
> The interesting bits are in **ExtensionMethods.cs,** **TwitterContext.cs** and **MainViewModel.cs**. That first code file contains an extension method called LoadAsync. This can be called on an ITwitterLeafNode (it’s an extension method) which basically represents a full request to Twitter in the TweetSharp API. LoadAsync returns a Subject on which OnNext is called asynchronously as the results come in. There is another function called LoadAsyncWithReplay that has pretty much the same logic but it uses a ReplaySubject instead, which allows a Subscriber to get any results that were returned before it subscribed.
> 
> **TwitterContext.cs** uses said extension method and exposes a couple of requests as Observable collections. There are requests to get the currently logged in user, get the public timeline, the friends timeline and so on. All these properties work in pretty much the same way. They construct the request using the TweetSharp API and then use LoadAsync to turn them into an IObservable. Normally we would have used FromAsyncPattern here, but the problem was that TweetSharp didn’t exactly follow the pattern (for instance, it does have a Begin method, but no corresponding End).
> 
> **MainViewModel.cs** then wires up the requests created by **TwitterContext.cs** to the ObservableCollections (not to be mistaken with IObservable collections, these are used for data binding specifically) on the view model. What happens is that whenever an item is posted on an IObservable (ie. OnNext is called) it gets added to the ObservableCollection. This will then automatically update the UI because it is databound to the ObservableCollection.

*   **TwitterPhoneDemo.Foundation**

> This is a Windows Phone 7 Class Library. It contains two reusable components, the TextBoxImmediateUpdateBehavior and the BusyIndicator. The first is a Behavior which causes a TextBox control to immediately update its underlying binding whenever someone types into it. By default a TextBox only updates its binding when it loses focus. I wanted the aforementioned behavior because this allowed me to react to the typing of the query immediately.
> 
> BusyIndicator is code that I’ve copied from the Silverlight Toolkit. It’s a simple control that renders a busy indicator whenever data is loading and then displays its content when it’s done loading.

*   **WindowsFormsDemos**

> Contains a couple of demo’s about FromEvent and FromAsyncPattern. The FromEvent demo simply converts the MouseMove event into an IObservable. The FromAsyncPattern demo calls the DemoService (which waits 2 seconds before returning a result) and then asynchronously waits for the result to come in.
> 
> Finally there is a demo in here that demonstrates how the new asynchronous programming features in C# vNext can help you to convert an existing synchronous way of doing things into an asynchronous version without having to restructure the code. By simple adding a couple of keywords at the right places this suddenly becomes asynchronous.

Well, that’s quite a lot to take in. I’ve been preparing for this session for more than a month now and I’m glad the wait is finally over. I think Rx and the new async programming feature will greatly simplify UI development, but before you can really leverage it you need to understand the new way of doing things which isn’t always easy. Once you get the hang of it though, you will see the sheer power of these concepts.

If you have any question regarding the sources don’t hesitate to leave a comment or send me an email.