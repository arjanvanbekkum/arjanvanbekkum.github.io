---
layout: post
title: "Unexpected process end and database lock with .NET Core WebApi and Entity Framework Core."
date: 2020-06-08
summary: "Developing a WebApi with Async database communication within containers can show unexpected process ending and lock when not with Entity Framework Core 3.1 and .NET Core 3.1"
minute: 5
tags: [dotnet, azure, docker, kubernetes]
---

Last week we saw that the process of the new `WebApi` we created suddenly stopped at random times. Strangely there was no exception; no logging and the Docker container was still reported as healthy. 

Eventually, we fixed the problem, but it took us several days, and we could not find much on the internet, so let me share what we found so you can fix it without spending days searching. 

We created a service running in a Docker container on Azure Kubernetes Services (AKS). The service has a timer that triggers every several minutes. For storing the data, we use Azure SQL database. The service uses Entity Framework core version 3.1.4 and runs on net core 3.1. Performance is critical for running this process; we use async queries to get the data from the database. 

The first days the service runs more or less as expected, we didn’t notice any strange behavior, which was incorrect because it already showed problems, but we did not recognize them. 

We added lots of unit testing to make sure all the components are running well. We even implement SpecFlow tests to tets the complete process. Running the process in de debugger shows no exceptions of any kind. But when we deploy the container to AKS, we see the process triggered by the timer crashes after several runs. There is no exception and no logging in `Application Insights`. At first sight this looks like memory problems where the thread just disappeared. 

We again run the `WebApi` within our visual studio debugger to see if we can reproduce the problem. The process runs fine for several hours. It looks like the memory increases during the process, and the Garbage Collector does not clean the memory enough. So we force the memory to clean calling

```csharp
    GC.Collect();
    GC.WaitForPendingFinalizers();
```

This has a positive influence on the memory use, so we expect our model has a memory leak. To make sure we deploy this version to the acceptance slot on the AKS cluster. First, the service runs fine, but after a few hours, the process stops again without logging or exceptions. 

Next, we connected a memory profiler to see memory usage, and we increased the number of times the process is triggered. It now runs every 30 seconds; we see the garbage collector clean the memory as you should expect. 

But no matter how long we leave the process running, it does not seem to crash. To see if we can find a pattern, we added as much logging as possible to the process and redeploy it to AKS. Within several minutes the process is gone again, and the logging shows where it stopped. After three runs, we jump the conclusion; it got stuck in the final step of the process. Here it calls a library to update or insert data in the database. 

We eliminate the last step and redeploy the service again. Again the process runs fine for several hours but then stops again, but now in the step before the step, we just eliminated. 

So this got us thinking what is the big difference between our debugger, which seems to work just fine, and the version running on AKS. We concluded it seems far fetched, but it is the Docker container. So we added orchestration support to the solution and used visual studio to debug the container. After a few minutes, the process was gone, so it seems to stop only in a Docker container, the question remains why. There was no exception, and pausing the code showed us nothing, again it was it the last step calling the library. 

So we started the process again, this time it took a little longer to fail. So when we thought it was gone, we paused our code. This time it shows our EF query is waiting for a lock on the database. This seems strange because this is the only process running on the database. 

After searching for a solution to solve this problem, we ran into this thread 

<a href ="https://github.com/dotnet/SqlClient/issues/262#issuecomment-587518102">https://github.com/dotnet/SqlClient/issues/262#issuecomment-587518102</a>

At first it seems to have nothing to do with our problem but at the bottom is a comment about locks in EF core. This looks like the problem we have, the solution seems to update the `Microsoft.Data.SqlClient` NuGet package. 

This seems like a strange thing because we already checked if we needed to update the packages. According to the NuGet package manager there were no updates available. So we check our solution again, but no updates needed. 

<img src="/images/noupdatesnugetmanager.jpg" alt="no updates available" height="200"/>

EF core runs on the latest version available, but when you collapse the packages to see the sub-packages you see it uses an old version 1.0.19 of the `Microsoft.Data.SqlClient`. 

<img src="/images/hiddendatasqlclient.jpg" alt="hidden package" height="200"/>

We used the NuGet package manager to find the latest version (1.1.3) of the `Microsoft.Data.SqlClient` and installed the update. We deployed the new version and now it is running just fine.

Looking at the release notes for the `Microsoft.Data.SqlClient` we notice that version 1.1.1 has a fix for async queries that get locked. 

<a href="https://github.com/dotnet/SqlClient/blob/master/release-notes/1.1/1.1.1.md">https://github.com/dotnet/SqlClient/blob/master/release-notes/1.1/1.1.1.md</a>

So we learned that, while we thought we do not need to run the service in a container locally because it should not make a difference, we were wrong. We also learned you need to check the dependencies of packages manually to prevent these issues from happening. 

