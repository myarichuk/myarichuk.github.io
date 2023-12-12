---
title: Works On My Machine(tm)
date: 2023-12-12T22:56:29+02:00
categories:
  - Programming
  - Debugging
tags:
 - Debugging
 - .Net
 - C#
 - Gotcha
top_img: top.jpg
cover: cover.jpg
---

Oh, but it works on my machine! What gives?! Today, I found myself asking the uncaring monitor over and over again, while adding logs and going over existing ones, meditating over code while making sure I didn't do any silly mistakes and overall being frustrated. It's these kinds of head-scratchers that remind us why debugging is both an art and a science.

### The Unseen Culprit

Picture this: a seemingly innocuous orchestrator service, working perfectly in the local environment, suddenly throws a tantrum when deployed. After many hours of puzzling over this, I discovered that the devil is in the details — or more accurately, in an unhandled exception lurking in an ``IHostedService`` constructor. I paid too much attention to code and possible environmental issues that I missed a very simple issue: a file I wanted to load into cache in the ``IHostedService`` constructor simple didn't exist in the deployed instance, because CI/CD wasn't deploying it.

### The Smoking Gun

I remembered that a background service shouldn't bring down a service like this, at least not without a stack trace of the exception. But apparently, any unhandled exception in the hosted service thread would bring the process down, a la ``StackOverflowException``.
The interesting part, there was a reason why I remembered this: Before .Net 6, an unhandled exception thrown in ``BackgroundService`` (which implements ``IHostedService``) wouldn't bring down the process, but apparently, Microsoft [changed that](https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/6.0/hosting-exception-handling) in .Net 6. Well, I learned something new today!

### Solutions... Solutions...

So, what's the solution? Well, if you use ``IHostedService`` as your background workers, the only way to make your code resilient is to wrap your code in *try-catch* blocks to ensure logging before rethrowing exception, if you do decide to crash your service.
If you use ``BackgroundService``, however, Microsoft offers a global solution to control the behavior of exception handling. Simply set a ``BackgroundServiceExceptionBehavior`` flag in the **Startup.cs** code of the host and voilà! The *old* behavior is back.
To be more exact, setting ``BackgroundServiceExceptionBehavior`` to ``Ignore`` will instruct the runtime to log the unhandled exception and continue as if nothing happened. Sometimes, you want to *fail fast*, sure, and that's why the new unhandled exception handling of the ``BackgroundService`` is useful. But in some cases, you want your service to be as resilient as possible and provide at least partial functionality to your users.

```cs
// credit (from MS article): https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/6.0/hosting-exception-handling
services.Configure<HostOptions>(hostOptions =>
{
   hostOptions.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.Ignore;
});
```

{% note info %}

Just in case you are not aware of .Net's background services, it's really easy.
A ``BackgroundService`` is a way to create a long running task, which it's lifetime tied to host's lifetime. (In this case, .Net host can be a Windows service, Linux daemon, a web service). See more in [this article](https://learn.microsoft.com/en-us/dotnet/core/extensions/windows-service?pivots=dotnet-8-0).

A class implementing ``IHostedService`` has pretty much the same role as ``BackgroundService`` but it's *bare metal* kind of framework feature. [This article](https://www.site24x7.com/learn/ihostedservice-and-backgroundservice.html) has a nice overview of the difference between the two.

{% endnote %}

### To Ignore or Not to Ignore: That's the `BackgroundServiceExceptionBehavior` Question

Okay, before we wrap it up, this `BackgroundServiceExceptionBehavior.Ignore` business is not as straightforward as you might think, and here's why.

#### The 'Fail Fast' Philosophy: A Blessing and a Curse

First up, the idea of 'failing fast.' Sounds pretty good, doesn't it? Stop the disaster before it wreaks havoc. This approach is gold when you’re dealing with systems where even a hint of data going sideways or invariants being broken, like in distributed consensus protocols, could spell disaster. In these scenarios, failing fast is like an emergency stop button – it’s there to save the day.

#### The Domino Effect in the RPC World

So *fail-fast* it is, right? Not so fast! In the intricate web of RPC-based systems, one service throwing a fit can start a nasty ripple effect. Think of it like a poorly planned domino setup. And it really doesn't matter why the service get stuck and stops responding! I recall this one time a service ground pretty much to a halt by ThreadPool starvation. What started as a small glitch snowballed into an hour-long outage of a mission-critical system. Nobody wants that, of course.

#### Making the Smart Call

Here's the deal: your game plan depends on your playing field. If you a system with RPC-based services, especially a dense mesh of such services, letting them take a hit and keep going. Log the error, keep the lights on, and avoid a full-blown blackout.

But if you're in the event or message queue world, where services are more like lone wolves than inter-dependent team players, then maybe let them fail fast. They crash, they burn, but they don't take down the whole neighborhood with them.

#### Crafting Your Safety Net: The Art of Controlled Chaos

But hey, it's not either-or choice: there is another option, an approach I call a *Service Denier*. This approach would involve actively monitoring for specific error patterns or broken invariants. Upon detection, these deniers can shut down the affected service or component, effectively acting as a circuit breaker to prevent broader system impact.

#### Service Deniers: A Silver Bullet?

While the 'Service Denier', or perhaps a Circuit-Breaker approach sounds like an elegant solution to prevent cascading failures, it's not without its own set of quirks and nightmarish pitfalls. For starters, there's a fine line between being cautious and being overzealous. Imagine a scenario where your service denier is a bit too trigger-happy, shutting down services at the slightest hiccup. This could lead to a situation where you're inadvertently causing more downtime than the original errors would have. It's like using a sledgehammer to swat an annoying mosquito - effective, perhaps, but overkill. Additionally, implementing a smart monitoring system that can accurately distinguish between a fatal error and a transient issue not necessarily easy, and in some cases can introduce a non-trivial overhead. For example, there was one time I instrumented my code for gathering [Prometheus metrics](https://prometheus.io/), I made Prometheus related code so complex that it increased endpoint latency by almost 40%!
Any safety net code requires upholding a delicate balance, a keen understanding of your system's behavior, and, frankly, a bit of trial and error. Over-engineering safety nets might not only add unnecessary complexity but also obscure the root causes of issues, leading to a wild goose chase. In essence, while crafting your Service Denier, it’s crucial to carefully tread the line between vigilance and paranoia.

#### My 2 Cents

- **Match the Strategy to the System**: No two systems are the same, so don’t go looking for a one-size-fits-all solution here. Typically, hybrid approaches with decisions on case-by-case basis work best.
- **Eyes Wide Open**: Set up solid monitoring and ideally, do stress and load testing - lots of issues may pop up when your system under stress or close to it's maximum throughput. Know how your system behaves when it hits a snag.
- **Embrace the Safety Nets**: Get familiar with patterns like rate limiting and circuit breakers. They’re your secret weapon against unexpected disasters.

### Diving (A Little Bit) Deeper

Given that my bug occured as an unhandled exception in the constructor, I asked myself: would .Net runtime somehow catch and log the exception in the constructor of the ``BackgroundService``? It seemed unlikely, but I still took a quick peek. And indeed, the policy relates only to ``BackgroundService::ExecuteAsync()``, as can be seen in the [.Net Runtime repo](https://github.com/dotnet/runtime/blob/f9529fd9c3fe8a970fb7a68b7fddd15fb686b5dd/src/libraries/Microsoft.Extensions.Hosting/src/Internal/Host.cs#L176). Yes, it *is* an excuse I just made to have a look at what happens under the hood.

To sum it up, the moral of the story is to never discard the possiblity of silly exceptions; in fact, it's better to be paranoid about exceptions! *Just because you are paranoid, it doesn't mean nobody is after you!* And now, go write some code, possibly with lots of *try-catch* statements :)