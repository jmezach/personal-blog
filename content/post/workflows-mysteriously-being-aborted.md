---
title: "Workflows mysteriously being aborted"
date: 2008-06-19T00:00:00+02:00
draft: false
tags: [ ".NET", "WF", "WCF" ]
categories: [ ".NET" ]
---

Part of the project I'm currently working on involves a workflow system we have build ourselves on top of Windows Workflow Foundation. It consist of a WCF service that contains task, a front end that displays these to users and of course several workflow activities for creating tasks and waiting for them to be completed by the users. Last Friday it was finally time to perform an installation of the system on our acceptance environment so the users could test it. The installation went fine so we were assuming everything was working fine, but yesterday a user reported a problem where she had completed a task, but the next task wasn't appearing in the front end.

Luckily for us we had tracing on on the system, so we investigated the trace log only to find that our workflows were being aborted. We were quite puzzled as to why this was happening, as the system was running fine in our own test environment. So it had to do something with the configuration of the servers. Then I remembered that our colleague [Marcel](http://blogs.infosupport.com/marcelv/archive/2006/03/06/4148.aspx) also encountered a problem where workflows were being aborted when he added a **SqlWorkflowPersistenceService** to the workflow runtime. In his blog he mentioned something about DTC (Distributed Transaction Coordinator) not being set up properly.

As we had problems with MSDTC and Workflow Foundation before we suspected that this was the problem. But we still weren't sure because when it happened before there we're logs all over the place about it. But we couldn't find any in this case. Neither the tracing, the Windows Event Log nor our own log files contained anything even remotely resembling an exception. It was only after I installed WinDbg on the machine in question and attached it to our Windows service that was hosting the **WorkflowRuntime** that I was able to confirm that it was indeed MSDTC that was causing the workflows from being aborted:

```
System.Workflow.Runtime Error: 0 : SqlWorkflowPersistenceService(00000000-0000-0000-0000-000000000000): Exception thrown while persisting instance: Network access for Distributed Transaction Manager (MSDTC) has been disabled. Please enable DTC for network access in the security configuration for MSDTC using the Component Services Administrative tool.  
System.Workflow.Runtime Error: 0 : stacktrace :&nbsp;&nbsp;&nbsp; at System.Transactions.Oletx.OletxTransactionManager.ProxyException(COMException comException)  
  at System.Transactions.TransactionInterop.GetOletxTransactionFromTransmitterPropigationToken(Byte[] propagationToken)  
  at System.Transactions.TransactionStatePSPEOperation.PSPEPromote(InternalTransaction tx)  
  at System.Transactions.TransactionStateDelegatedBase.EnterState(InternalTransaction tx)  
  at System.Transactions.EnlistableStates.Promote(InternalTransaction tx)  
  at System.Transactions.Transaction.Promote()  
  at System.Transactions.TransactionInterop.ConvertToOletxTransaction(Transaction transaction)  
  at System.Transactions.TransactionInterop.GetExportCookie(Transaction transaction, Byte[] whereabouts)  
  at System.Data.SqlClient.SqlInternalConnection.EnlistNonNull(Transaction tx)  
  at System.Data.SqlClient.SqlInternalConnection.Enlist(Transaction tx)  
  at System.Data.SqlClient.SqlInternalConnection.EnlistTransaction(Transaction transaction)  
  at System.Data.SqlClient.SqlConnection.EnlistTransaction(Transaction transaction)  
  at System.Workflow.Runtime.Hosting.DbResourceAllocator.GetEnlistedConnection(WorkflowCommitWorkBatchService txSvc, Transaction transaction, Boolean& isNewConnection)  
  at System.Workflow.Runtime.Hosting.PersistenceDBAccessor..ctor(DbResourceAllocator dbResourceAllocator, Transaction transaction, WorkflowCommitWorkBatchService transactionService)  
  at System.Workflow.Runtime.Hosting.SqlWorkflowPersistenceService.System.Workflow.Runtime.IPendingWork.Commit(Transaction transaction, ICollection items)
```

Normally though, when a **WorkflowRuntimeService** encounters an exception that it cannot handle, the **ServicesExceptionNotHandled** event on the **WorkflowRuntime** is raised. We had an event handler on this event in our implementation which would publish the exception to the log files and also write a trace message containing the exception. We couldn't find any trace of the above mentioned exceptions though, so apparently the **SqlWorkflowPersistenceService** was handling the exception itself and was silently ignoring it.

So, if you are encountering problems where workflows are being aborted and you can't find out what is wrong, it's possible that MSDTC is to blame. You could then try setting up DTC properly or avoiding DTC altogether by using the **[SharedConnectionWorkflowCommitWorkBatchService](http://forums.microsoft.com/MSDN/ShowPost.aspx?PostID=216639&SiteID=1)**. If this doesn't solve your problems you can try turning up the tracing to be more verbose by following the instructions [here](http://wiki.windowsworkflowfoundation.eu/default.aspx/WF/TracingWorkflowFoundation.html).