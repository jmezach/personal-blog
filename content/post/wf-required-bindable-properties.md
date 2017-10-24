---
title: "WF: Required bindable properties"
date: 2007-09-19T00:00:00+02:00
draft: false
tags: [ ".NET", "WF", "WCF" ]
categories: [ ".NET" ]
---

Today I was working on a bit of workflow we have in our project. We've written a couple of activities to handle some of the processing done in our application (such as sending out emails at a specified point in time). These activities have been working quite well, but they lacked some kind of validation on required properties or how they were used in the workflow. So I was assigned the task to implement some of that stuff.

Windows Workflow Foundation has build in support for doing validations on workflow activities. This is as simple as building a class that extends the ActivityValidator class and then applying an ActivityValidatorAttribute to your custom activity that specifies the class that implements the validation logic. But that is just one way to do it, because there is also something that is called a ValidationOption enumeration, which has the values None, Optional and Required and an accompanying ValidationOptionAttribute. So I decided to take a look at that first.

I quickly found out that the ValidationOptionAttribute only works when used with instance dependency properties in Workflow Foundation. Workflow Foundation distinguishes between meta dependency properties, which are immutable at runtime and instance dependency properties which obviously can be changed at runtime. So this didn't work for us, because most of the properties we wanted to put some validation on were supposed to be bindable (for instance the To property on our SendMail activity).

So I had a look using .NET Reflector (an excellent tool by the way) to see what other activities we're doing. However I couldn't really find any activity that had both bindable properties as well as required properties. I quick search on Google didn't turn up much either. But I thought it must be possible so I had a go at implementing a validator using the pattern described earlier.

The problem with dependency properties that are bound at runtime is obviously that you can't know at compile time whether a value is available. So we decided to make it so that for a bindable property it must either have a specific value set explicitly, or a binding must be set (which implies that at runtime things might still go wrong, but that's another issue). Implementing the code for the validator wasn't especially difficult, but I still think it's a bit too much code for something as simple as this, but I'll let you decide. Here is the code for the validator:

```csharp
internal class BindablePropertyDemoActivityValidator : ActivityValidator  
{  
  public override ValidationErrorCollection ValidateProperties(ValidationManager manager, object obj)  
  {  
    ValidationErrorCollection err = new ValidationErrorCollection();  
    err.AddRange(base.ValidateProperties(manager, obj));  

    BindablePropertyDemoActivity activity = obj as BindablePropertyDemoActivity;  
    if (activity == null)  
    {  
      throw new ArgumentException("BindablePropertyDemoActivityValidator not applied to BindablePropertyDemoActivity", "obj");  
    }  

    if (activity.Parent != null)  
    {  
       if (String.IsNullOrEmpty(activity.Bindable) && !activity.IsBindingSet(BindablePropertyDemoActivity.BindableProperty))  
       {  
          err.Add(new ValidationError("BindableProperty is required", 7000, false, "BindableProperty"));  
       }  
    }  

    return err;  
  }  
}
```

This will make sure that the Bindable property of my sample BindablePropertyDemoActivity is either set to some static text or bound to some other property somewhere else in the workflow. If it's not, a red exclamation mark will be shown next to the activity and a red dot will also appear in the property editing window. In fact, compiling is a no go as well, because the validators are also called when the workflow is being compiled. This is nice of course, but it also poses a little problem I haven't been able to figure out.

As you can see in the code I'm checking to see whether the Parent of the activity isn't equal to null. If I remove that if statement I get an error compiling the activity itself. Although this makes sense, because when the activity itself is compiled the property will not be set to anything, I don't see such a check when reflecting the code of existing activities. Maybe because they mostly seem to check metadata properties only, I'm not sure. This approach works, but it means that the activity cannot be used as a root activity. I guess that isn't a big deal, because the classes provided by workflow should probably be at the root anyway.

What I still don't like about this approach is the amount of code required to do this stuff. I was thinking about making something a bit more generic (like a base class for validators that can check these kind of required properties itself), but I need to have access to the actual DependencyProperty object, which makes it a bit harder to make it generic. I guess I'll have to think about it some more :).