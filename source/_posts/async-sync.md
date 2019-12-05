---
title: Synchronization primitives + async/await = trouble
date: 2019-12-05 08:38:06
tags:
  - C#
  - Async
  - Programming
  - Multithreading
categories:
  - Programming
  - Multithreading
author: Michael Yarichuk
top_img: top.jpg
cover: /2019/12/05/async-sync/cover.jpg
---

Recently, my colleague was investigating an interesting issue. A ``ReaderWriterLockSlim`` was sometimes throwing the following exception when releasing a read lock.
```exception
System.Threading.SynchronizationLockException: 'The read lock is being released without being held.'
```

The code looked fairly straightforward and it *should* have worked properly, but after some meditation on the mysteries of C# and the universe, he noticed something interesting. An *await* call between taking the lock and releasing it. 

Roughly, the problematic code had the following pattern:
```cs
var rwl = new ReaderWriterLockSlim();
rwl.EnterReadLock();

//doing stuff

await Task.Delay(1000); //an async call with 'await'

//doing some more stuff

rwl.ExitReadLock();
```

A suspicion was confirmed by looking at the [reference implementation](https://github.com/microsoft/referencesource/blob/master/System.Core/System/threading/ReaderWriterLockSlim/ReaderWriterLockSlim) of ``ReaderWriterLockSlim`` - a field that tracks the amount of locks taken is defined like this:
```cs
 [ThreadStatic]
 private static ReaderWriterCount t_rwc;
```

The lock taking code looks like this (omitting some code for clarity):
```cs
private bool TryEnterReadLockCore(TimeoutTracker timeout) 
{
  //some code
  lrwc = GetThreadRWCount(false); //get read/write lock count PER THREAD
  //some more code
  lrwc.readercount++;
  //even more code
}

```

That was enough to understand what was the issue. Using *async/await* yields current thread until the *async* method finishes execution and then fetches another thread from the thread pool to continue execution - and since ``ReaderWriterLockSlim`` uses ``ThreadStatic`` to store the counts of read/write locks, no wonder it thinks the lock was never taken after *async/await* call!
  
The solution in such cases is either removing usage of *async/await* or utilizing a different set of synchronization primitives that are compatible with thread yields resulting from *await* calls.
There is excellent series of blog posts on the topic by Stephen Toub, where he shows how async-compatible synchronization primitives can be implemented. The first blog post you can find [here](https://devblogs.microsoft.com/pfxteam/building-async-coordination-primitives-part-1-asyncmanualresetevent/).
Also, check out [AsyncEx](https://github.com/StephenCleary/AsyncEx) - a very handy library with full set of async-compatible synchronization primitives, so you don't *have to* implement your own.