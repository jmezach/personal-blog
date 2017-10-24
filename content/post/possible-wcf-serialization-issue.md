---
title: "Possible WCF Serialization Issue?"
date: 2007-08-15T00:00:00+02:00
draft: false
tags: [ ".NET", "WCF" ]
categories: [ ".NET" ]
---

As I was writing in my previous post there is something interesting going on in WCF. But before I go into the details, let me explain the situation.

There are 3 services in our project, one business service, one process service and one front end. We use WCF to communicate between these services. We are using basicHttpBinding and we have a central set of XSD schema's from which we generate code using the svcutil tool provided with .NET Framework 3.0\. This has been working pretty good for us, but for a while now we are experiencing random exceptions when WCF calls are being made.  

These exceptions are of type System.ExecutionEngineException. According to the documentation these exceptions are thrown whenever the .NET Runtime suffers from some internal error. However, the exception itself doesn't provide any meaningful information about what caused the exception to occur. In fact, we haven't been able to get any useful information from either the EventLog or our own logging framework. There are some messages in the EventLog but they only indicate that there was a fatal execution engine error and that the faulting module is mscorwks.dll.

Because the .NET Runtime itself is crashing, the application pool that is serving our service (we are hosting inside IIS) gets shutdown. If the exception occurs in the application pool serving our front end the end user is presented with a login dialog. Obviously this is unacceptable.

Unfortunately the exception occurs on pretty much all the machines we are using, from development to production. We haven't been able to reproduce the error consistently and enabling all types of logging and tracing that are available (including WCF's extensive logging and tracing options) haven't been able to provide us with a cause for this exception to occur. So after trying to establish the cause of this exception by using logging and tracing for a couple of weeks, we tried using the debugging tools provides by Microsoft to debug the ASP.Net worker processes.

What we have been able to find out so far that an exception occurs inside the service. We have written an extension for WCF which allows us to have exceptions that occur in the service to be automatically translated to either a FunctionalException or a TechnicalException, so these should be handled gracefully. Both FunctionalException and TechnicalExceptions are serialized and returned to the caller, but according to the stack trace we have been able to retrieve something goes wrong while serializing the exception.  

We are still not sure what is causing this, but something goes very wrong. We are currently trying to create a full dump, instead of a minidump so that we hopefully can see what data is being passed around. If we found out anything new, I will definitely blog about it again.