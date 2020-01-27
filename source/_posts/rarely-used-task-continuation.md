---
title: Even simple stuff, like C# TPL can surprise you!
date: 2019-12-17T20:27:10.288Z
tags:
  - C#
  - TPL
  - WTF
  - Debugging
categories:
  - Programming
  - WTF
author: Michael Yarichuk
top_img: https://upload.wikimedia.org/wikipedia/commons/0/0d/C_Sharp_wordmark.svg
cover: /2019/12/17/rarely-used-task-continuation/cover.jpg
---
What do you think would happen if you run this code? Can you guess *without* actually running it?  
Now try running it and see what happens. It might actually surprise you...  
<iframe width="100%" height="600" src="https://dotnetfiddle.net/Widget/7k28o5" frameborder="0"></iframe>

I know I was surprised by this, it took me a minute or two and a glimpse into [relevant documentation page](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcontinuationoptions?view=netcore-3.0) to understand what happened here.  
The ``backgroundTask`` in the above code is actually a *continuation task*, and not the *background worker*. Since the *background worker* runs successfully, the *continuation task* is canceled, as it should. Actually, the code above should be rewritten like this:

```c#
var backgroundTask = Task.Run(() =>
{
   Thread.Sleep(2000); //simulate some work
});

backgroundTask.ContinueWith(_ =>
{
    Console.WriteLine("Running the continuation!");
}, TaskContinuationOptions.NotOnRanToCompletion);

await backgroundTask; //we registered the continuation, now we can wait for the worker...
```

All in all, I think it was a nice little issue. Also, this is yet again a proof of the timeless axiom: *assumption is the mother of all fuckups*