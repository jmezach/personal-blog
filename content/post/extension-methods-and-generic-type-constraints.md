---
title: "Extension methods and generic type constraints"
date: 2011-05-16T00:00:00+02:00
draft: false
tags: [ "C#", ".NET Framework" ]
categories: [ ".NET" ]
---

I ran into an interesting problem the other day regarding extension methods and generic type constraints, so I though I’d write about it here. First, let me explain what I was trying to do. I wanted to define an extension method on any type that implements both INotifyCollectionChanged and ICollection<T>. Now as far as I know there is only one type that actually implements both interfaces, ObservableCollection<T>, but that’s not the point. I wanted it to be useable on any type that implements both interfaces.

Normally when one defines an extension method on a type you would specify the signature like this:

```csharp
public static void ExtensionMethod(this INotifyCollectionChanged target)
{
}
```

Now, obviously this poses a problem, because in this case we cannot specify that target must also implement ICollection<T> (or any other interface for that matter). So what do we do? We can introduce a generic type argument to our extension method and use generic type constraints to define where we want to allow our extension method to be used. Since we want to use ICollection<T> to constrain the use of our extension method, we will also need to introduce a generic type argument for T:

```csharp
public static void ExtensionMethod<TTarget, TItem>(this TTarget target)
    where TTarget : INotifyCollectionChanged, ICollection<TItem>
{
}
```

Unfortunately this doesn’t work. The problem is in the generic type argument TItem. Since there is no argument to this extension method of type TItem, type inference cannot determine what the type for TItem should be. If we add another parameter to this extension method of type TItem it does work:

```csharp
public static void ExtensionMethod<TTarget, TItem>(this TTarget target, TItem item)
    where TTarget : INotifyCollectionChanged, ICollection<TItem>
{
}
```

However, for my particular case I only wanted to return something of type TItem, rather than requiring something of type TItem into my method. What I wanted to achieve is something like the following:

```csharp
public static IEnumerable<TItem> ExtensionMethod<TTarget, TItem>(this TTarget target)
     where TTarget : INotifyCollectionChanged, ICollection<TItem>
{
}
```

This doesn’t seem to be possible however. Type inference only works for types that are passed in as arguments to the method. It doesn’t look at return types. In fact, there is a Code Analysis rule for this: [CA1004: Generic methods should provide type parameter](http://msdn.microsoft.com/en-us/library/ms182150.aspx), which specifies that there should be an argument for each generic type argument.

I haven’t been able to come up with a solution yet. I’ve tried to add an optional parameter and give it a default value of default(T), but obviously that isn’t going to work because the default value is stored with the actual implementation of the method, rather than at the call site. So if anyone has a solution to this problem, let me know in the comments.