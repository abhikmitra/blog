---
title: Durable Task Framework Internals - Part 5 (Interesting usages of TPL in DTF)
date: "2020-04-27T18:00:00.284Z"
description: "In this part, we look at some of the interesting usages of the Task Parallel Library in DTF "
---
### Durable Task Framework Series
This post is **part 5** of a series of posts on DTF.
1. [Durable Task Framework Internals - Part 1 (Dataflow and Reliability)](https://abhikmitra.github.io/blog/durable-task/)
2. [Durable Task Framework Internals - Part 2 (The curious case of Orchestrations)](https://abhikmitra.github.io/blog/durable-task-2/)
3. [Durable Task Framework Internals - Part 3 (Tracker Queue, Instance History, and JumpStart)](https://abhikmitra.github.io/blog/durable-task-3/)
4. [Durable Task Framework Internals - Part 4 (Terminated Orchestrations & Middlewares)](https://abhikmitra.github.io/blog/durable-task-4/)
5. [Durable Task Framework Internals - Part 5 (Interesting usages of TPL in DTF)](https://abhikmitra.github.io/blog/durable-task-5/)
6. [Durable Task Framework Internals - Part 6 (Orchestration Execution Flow)](https://abhikmitra.github.io/blog/durable-task-5/)

Do you think there is more that I should cover or something I should fix? Please raise an [issue](https://github.com/abhikmitra/blog/issues) and let me know.

---
In this post, we are going to cover all the interesting Dot Net's Task Parallel Library concepts that I found here. Some of them might not be advanced, but I have put them as I have not used these often in my regular code.

###  Thread Static
ThreadStatic is an attribute that marks a variable as static. You can read more about it [here](https://www.c-sharpcorner.com/article/overview-of-threadstatic-attribute-in-c-sharp/). ThreadStatic is pretty simple to grasp. The `OrchestrationContext` has a *ThreadStatic* variable to check any time during your orchestration to see if you are on the correct thread. Now, this usually is not required, as Orchestrations should not have async operations. But if Orchestration is executing a piece of code that you have no control over and you want to check that when control comes back to your code for illegal task continuations, this would be an excellent way to check. Based on the PR [here](https://github.com/Azure/durabletask/pull/171#discussion_r183195689), this is precisely how it is used by the Azure Durable Functions team.

### Synchronization Context
Synchronization Context is a little difficult to grasp. Here are the articles you should read with to understand SynchronizationContext.
- [What is SynchronizationContext ?](https://hamidmosalla.com/2018/06/24/what-is-synchronizationcontext/)
- [SynchronizationContext and Console Apps](https://devblogs.microsoft.com/pfxteam/await-synchronizationcontext-and-console-apps/)

```csharp
Console.WriteLine("Thread {0}", Thread.CurrentThread.ManagedThreadId)
await task1
Console.WriteLine("Thread {0}", Thread.CurrentThread.ManagedThreadId)
```
In the above code, there is no guarantee that pre and post-task, both threads are going to be the same. What await is going to try and do is to bring back the back to control to the same SynchronizationContext. If it can find a SynchronizationContext, it will use the thread, if the `SynchronizationContext.Current` is null, it will choose any thread of the thread pool and continue on it.
Now there are cases where SynchronizationContext is not null, like in a WPF app or other UI facing apps. In case of console apps, the starting SynchronizationContext is  null. Durable Task Framework runs the orchestration on the same thread, irrespective of whether you are running the hub in a console app or in some other way. So to achieve this, in the `TaskOrchestrationExecutor` just before Orchestration is executed, DTF sets the SynchronizationContext and unsets it after the Orchestraton is complete.
Now when you do an `await` inside the orchestration, it might still span a thread to do an async operation, but when the execution of the async operation is completed, it will always come back to the starting thread using the SynchronizationContext.

### TaskCompletionSource

These 2 articles do a pretty good job of explaining `TaskCompletionSource`.
- [Task vs. TaskCompletionSource](https://www.pluralsight.com/guides/task-taskcompletion-source-csharp)
- [When should TaskCompletionSource be used ?](https://stackoverflow.com/questions/15316613/when-should-taskcompletionsourcet-be-used)

So now, you know that TaskCompletion sources is used to convert an event-driven code to a more promise based/awaitable pattern. 
Now in DTF, we have discussed that Orchestrations get executed multiple times. Let's assume that an orchestration as one below

```csharp
public async Task<bool> RunTask()
{
    var task1 = context.ScheduleWithRetry<bool>(typeof(TestActivity1), options, "Test Input1");
    var str = new string('x', 500 * 1024 * 1024);
    await Task.Delay(90000);
    Console.WriteLine(str.Substring(str.Length - 10));
    str = null;
    return true;
}
```

And from somewhere else in the code.
```csharp
public Task ExecuteOrchestrationAsync(string runTimeState)
{
    RunTask();
    return Task.FromResult(0);
}
```
So, we are invoking RunTask and not waiting for it to complete. As a result, now we are stuck with 500 MB of data, which will be in memory for 90 seconds. Considering that orchestrations can run days. What would you expect for the below piece of code?

```
public async Task<bool> RunTask()
{
    var task1 = context.ScheduleWithRetry<bool>(typeof(TestActivity1), options, "Test Input1");
    var str = new string('x', 500 * 1024 * 1024);
    await task1;
    Console.WriteLine(str.Substring(str.Length - 10));
    str = null;
    return true;
}
```

Tasks typically take much more than 90 seconds. And considering the orchestration would be invoked at least 2 times, we are looking at 1 GB of data per task. And if you remember Part 2, Orchestration gets executed multiple times, and it is perfectly normal for orchestration to Execute `await task` and never come back. So what happens is that `ScheduleWithRetry` gives back a TaskCompletionSource. 

And whenever the framework detects that 
- There is no reference to TaskCompletionSource, so it cannot be set as completed.
- The async call itself is not awaited

It garbage collects the memory.

So hence, you will not see the memory leak that you would typically expect with async operations as the framework is smart enough to distinguish between the tasks that may complete and tasks that will never complete.
This is not generally a recommended pattern, as per the [experts](https://devblogs.microsoft.com/pfxteam/dont-forget-to-complete-your-tasks/).
