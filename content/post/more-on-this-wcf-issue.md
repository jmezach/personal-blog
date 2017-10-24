---
title: "More on this WCF issue"
date: 2007-08-31T00:00:00+02:00
draft: false
tags: [ ".NET", "WCF" ]
categories: [ ".NET" ]
---

This week I've been spending some more time on this WCF issue I've blogged about before. Unfortunately I still don't have a solution for the problem, and I doubt that I will ever find one. However, I did find some interesting things I thought I should blog about.

I found out that the bug isn't in the actual serialization process, but rather in the generation of the code that is supposed to do the actual serialization. WCF's DataContractSerializer generates code on the fly to serialize and deserialize objects according to a specified data contract. The generated code is then called as a sort of anonymous delegate to perform the actual serialization or deserialization.

In the code that gets generated, serializing a nullable field is done by calling a function on the internal **XmlObjectSerializerWriteContext** class in the **System.Runtime.Serialization** namespace. This function is called **GetHasValue** which takes a generic type parameter **T** and a parameter of type **T?**, or a **Nullable<T>**. The function is static and it simply calls the **HasValue** property on the provided argument and returns the result.

A reference to this method is acquired using reflection, simply by calling the **GetMethod** function on a **typeof(XmlObjectSerializerWriterContext)**. Sometimes however, this doesn't work and it returns **null**. This null reference is then passed to the **MakeGenericMethod** function which goes all the way down to the CLR which then fails because it is trying to dereference a null pointer.

What I haven't been able to figure out so far is why getting a reference to this static **GetHasValue** function sometimes fails and sometimes doesn't. There doesn't seem to be any pattern in when it fails and when it doesn't. I thought it might have something to do with the worker process unloading DLL's when the application pool is being recycled, but it's difficult to reproduce this behavior.

Here is a stack trace of when the problem occurs:

```
mscorwks!MethodDesc::GetMethodTable+0x12  
mscorwks!MethodDesc::StripMethodInstantiation+0x8  
mscorwks!RuntimeMethodHandle::StripMethodInstantiation+0xbd  
mscorlib_ni!System.RuntimeMethodHandle.StripMethodInstantiation()+0x6  
mscorlib_ni!System.Reflection.RuntimeMethodInfo.Equals(System.Object)+0x64  
mscorlib_ni!System.Reflection.CerHashtable`2[[System.__Canon, mscorlib],[System.__Canon, mscorlib]].Insert(System.__Canon[], System.__Canon[], Int32 ByRef, System.__Canon, System.__Canon)+0x9e  
mscorlib_ni!System.Reflection.CerHashtable`2[[System.__Canon, mscorlib],[System.__Canon, mscorlib]].Preallocate(Int32)+0x10e  
mscorlib_ni!System.RuntimeType+RuntimeTypeCache.GetGenericMethodInfo(System.RuntimeMethodHandle)+0xef  
mscorlib_ni!System.RuntimeType.GetMethodBase(System.RuntimeTypeHandle, System.RuntimeMethodHandle)+0x2a3490  
mscorlib_ni!System.Reflection.RuntimeMethodInfo.MakeGenericMethod(System.Type[])+0x156  
System_Runtime_Serialization_ni!System.Runtime.Serialization.XmlFormatWriterGenerator.UnwrapNullableObject(System.Reflection.Emit.LocalBuilder)+0x1cb  
System_Runtime_Serialization_ni!System.Runtime.Serialization.XmlFormatWriterGenerator.WriteValue(System.Reflection.Emit.LocalBuilder, Boolean)+0x6f287  
System_Runtime_Serialization_ni!System.Runtime.Serialization.XmlFormatWriterGenerator.WriteMembers(System.Runtime.Serialization.ClassDataContract, System.Reflection.Emit.LocalBuilder, System.Runtime.Serialization.ClassDataContract)+0x28d  
System_Runtime_Serialization_ni!System.Runtime.Serialization.XmlFormatWriterGenerator.WriteClass(System.Runtime.Serialization.ClassDataContract)+0x6f27b  
System_Runtime_Serialization_ni!System.Runtime.Serialization.XmlFormatWriterGenerator.GenerateClassWriter(System.Runtime.Serialization.ClassDataContract)+0x67  
System_Runtime_Serialization_ni!System.Runtime.Serialization.ClassDataContract.get_XmlFormatWriterDelegate()+0x50  
System_Runtime_Serialization_ni!System.Runtime.Serialization.ClassDataContract.WriteXmlValue(System.Runtime.Serialization.XmlWriterDelegator, System.Object, System.Runtime.Serialization.XmlObjectSerializerWriteContext)+0x17  
System_Runtime_Serialization_ni!System.Runtime.Serialization.XmlObjectSerializerWriteContext.SerializeWithoutXsiType(System.Runtime.Serialization.DataContract, System.Runtime.Serialization.XmlWriterDelegator, System.Object)+0x2a  
System_Runtime_Serialization_ni!System.Runtime.Serialization.DataContractSerializer.InternalWriteObjectContent(System.Runtime.Serialization.XmlWriterDelegator, System.Object)+0x8a  
System_Runtime_Serialization_ni!System.Runtime.Serialization.DataContractSerializer.InternalWriteObject(System.Runtime.Serialization.XmlWriterDelegator, System.Object)+0x1f  
```