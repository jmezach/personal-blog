---
title: "Data binding in code using Reactive Extensions"
date: 2011-03-10T00:00:00+02:00
draft: false
tags: [ "C#", ".NET Framework" ]
categories: [ ".NET" ]
---

First of all, it’s been a while since my last post here. I’ve busy working with SharePoint in the past few months and although there is plenty to write about SharePoint, a lot of it has already been written by other people, so I didn’t feel the need to write about it here. But lately I’ve found the time to experiment with Reactive Extensions again, which I’m quite fond of as you might remember from me previous [post](http://blogs.infosupport.com/blogs/jonathan/archive/2010/11/04/reactive-extensions.aspx) about the subject. So I thought it would be a good

As most of my readers will know, both Silverlight and WPF have a very powerful data binding mechanism. This mechanism enables some interesting scenario’s, one of which is of course the Model-View-ViewModel (MVVM) pattern which relies heavily on data binding. But these mechanisms can only be used from XAML, or in MVVM terms, the view. But sometimes you might want to use data binding between different View Model instances in code. This can be hard to accomplish, especially if you want to do it in a loosely coupled way.

But when you think of it, data binding in Silverlight and WPF is based on change notifications coming from an object. And these change notifications are based on the INotifyPropertyChanged interface which defines an event called PropertyChanged. If there is something Reactive Extensions is good at its events. So what if we can somehow turn this PropertyChanged event into an IObservable event stream that will contain the new values. If we have that, we can easily set the new value to a property on some other object. So, let’s get started.

## Identifying the property

Before we can do anything, we need some way of identifying the property we want to turn into an IObservable. We could of course use the name of the property as a string, but that would mean we wouldn’t have a compile time check. The typical way to solve this problem is by using Expressions. I’m not going to explain to much about this as it is a well documented elsewhere, but I decided to write a simple extension method to convert an Expression into a PropertyInfo (from System.Reflection):

```csharp
public static PropertyInfo ToPropertyInfo(this Expression> expression)
{
    // Get the body of the expression
    Expression body = expression.Body;
    if (body.NodeType != ExpressionType.MemberAccess)
    {
        throw new ArgumentException("Property expression must be of the form 'x => x.SomeProperty'", "expression");
    }
 
     // Cast the expression to the appropriate type
     MemberExpression memberExpression = (MemberExpression)body;
     return memberExpression.Member as PropertyInfo;
}
```

Basically this turns our expression (which should be pointing to a property on TTarget) into a PropertyInfo instance that we can use to manipulate the property. I choose to write this as an extension method because I’ll probably be using it elsewhere.

## Turning a property into an IObservable

Now that we can find out which property we want to use, we can start turning it into an IObservable. Unfortunately the INotifyPropertyChanged interface only sends us a notification when a property changes, but it doesn’t tell us the new value of the property. So we’ll need to first be notified of the events and then when one comes in, get the current value of the property. For the first bit we’ll use Observable.FromEvent which turns an event into an IObservable:

```csharp
// Convert the PropertyChanged event to an Observable
var eventObservable = Observable.FromEvent(
    h => h.Invoke,
    h => source.PropertyChanged += h, h => source.PropertyChanged -= h);
```

This is a bit of a round-about way of doing things, but it does make sure that things don’t break at run-time if the PropertyChanged event is somehow renamed. As this is not very likely, we can rewrite the above code to this much simpler version:

```csharp
// Convert the PropertyChanged event to an Observable
var eventObservable = Observable.FromEvent(source, "PropertyChanged");
```

Now that we have an IObservable that is triggered each time the PropertyChanged event on our source is raised, we need to do some filtering and selecting. As we’re only interested in PropertyChanged events concerning the property we’ve identified, we can filter all the rest out. We’ll use the Where operator to do this (remember that we can use all the LINQ query operators on IObservables to). Then once we have those, we can turn our property changed notification into the new value of the property by using the Select operator and getting the value from the source object using the PropertyInfo object we’ve retrieved earlier:

```csharp
// Filter the event and return it
return eventObservable.Where(e => e.EventArgs.PropertyName == property.Name)
                      .Select(e => (TValue)property.GetValue(source, null));
```

We now have an IObservable that will contain a new value when the PropertyChanged event is triggered. When we combine this we can put it into a nice extension method that we can use on any object that implements INotifyPropertyChanged. Here is what that extension method looks like:

```csharp
public static IObservable ObservableForProperty(this TSource source, Expression> propertyExpression)
    where TSource : INotifyPropertyChanged
{
    // Get the information for the property
    PropertyInfo property = propertyExpression.ToPropertyInfo();
    if (property == null)
    {
        // Expression does not indicate a property
        throw new ArgumentException("Property expression must point to a valid property", "propertyExpression");
    }

    // Convert the PropertyChanged event to an Observable
    var eventObservable = Observable.FromEvent(source, "PropertyChanged");

    // Filter the event and return it
    return eventObservable.Where(e => e.EventArgs.PropertyName == property.Name)
        .Select(e => (TValue)property.GetValue(source, null));
}
```

## Pushing the new value to another object

Now that we have an IObservable that is triggered on each new value, we can subscribe to it and do something useful with the new value. In our case, we want to push the new value to a property on another object. To do this, we first need to identify the property we want to push the new value to. Luckily we can use the same method as we’ve done before, using the ToPropertyInfo extension method. Once we have that we can simply subscribe to the source observable (created using the ObservableProperty extension method as seen above) and in the OnNext we use the PropertyInfo object to set the value to the target property:

```csharp
public static IDisposable BindTo(this IObservable observable, TTarget target, Expression> propertyExpression)
{
    // Get the information for the property
    PropertyInfo property = propertyExpression.ToPropertyInfo();
    if (property == null)
    {
        // Expression does not indicate a property
        throw new ArgumentException("Property expression must point to a valid property", "propertyExpression");
    }
  
    // Subscribe to the observable
    return observable.Subscribe(value => property.SetValue(target, value, null));
}
```

## Putting it all together

With these extension methods in place we can now write code such as:

```csharp
class DocumentViewModel : INotifyPropertyChanged
{
    private string _name;
 
    public string Name
    {
        get { return _name; }
        set
        {
            if (_name != value)
            {
                _name = value;
                RaisePropertyChanged("Name");
            }
        }
    }
}
  
class DocumentTabViewModel : INotifyPropertyChanged
{
    private string _title;
  
    public string Title
    {
        get { return _title; }
        set
        {
            if (_title != value)
            {
                _title = value;
                RaisePropertyChanged("Title");
            }
        }
    }
}
  
DocumentTabViewModel documentTab = new DocumentTabViewModel();
DocumenViewModel document = new DocumentViewModel();
  
document.ObservableProperty(document => document.Name)
        .Select(name => "Document: " + name)
        .BindTo(documentTab, target => target.Title);
```

This will synchronize the name of the DocumentTab with the filename of the Document and prepend “Document: ” to it. Obviously this is just a simple example of what you can do, but once you realize that Reactive Extensions aren’t only about events generated by a user but about any .NET event you can really start to see the power of it.