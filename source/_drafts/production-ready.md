---
title: Production-Ready is not just a buzzword
date: 2023-12-11 19:15:48
tags:
  - Software Engineering
  - Theory
  - DevOps
categories:
  - Programming
  - Theory
top_img: /2023/12/11/production-ready/production_ready_background.jpg
cover: /2023/12/11/production-ready/production_ready_background.jpg
---

## The What and the Why

In the tech world, 'production-ready' is a phrase that gets tossed around a lot—sometimes more like a hot potato than a clear standard. It seems like everyone has their own take, often shrouded in personal biases or cloaked in industry jargon. In this post, we're cutting through the fluff and the corporate mumbo-jumbo. We're diving deep into what 'production-ready' really means for software. From the crucial importance of structured logging to the non-negotiable need for comprehensive testing, we'll explore the essentials that separate a robust, reliable product from a ticking time bomb waiting to implode in the production environment.

## Non-negotiables

Let's start with things that absolutely should be in a *production-ready* software. Of course, I am aiming here at a general use-case; context matters - implementing a comprehensive monitoring system or a comperhensive test suite for [non-mission critical](https://www.techtarget.com/searchitoperations/definition/mission-critical-computing) internal tool may well be a waste of time, for example.

### Structured Logging and Monitoring

But logging is pretty much a given, I might hear you say. "Ugh, okay. I'll just add some logging here, here and here. Let's finish this code review..."
Sound familiar? I've been there too. If you've heard this from a fellow developer or, even worse, caught yourself saying it, it's time for a serious perspective shift. [Structured logging](https://www.atatus.com/glossary/structured-logging/) and [monitoring](https://www.digitalocean.com/community/tutorials/an-introduction-to-metrics-monitoring-and-alerting) are more than just a tedious task or an item on a review checklist. They are the unsung heroes of the software world, often overlooked in favor of flashy new AI features or sleek UI elements.
But it's these behind-the-scenes players that keep your software sailing smoothly through the stormy seas of production. They're not just about spitting out lines of text for an eventual, maybe-never glance. They're about creating a coherent story of your application’s life in the wild, bringing order to chaos. And you know, when things hit the fan at 2 AM, you'll be craving logs that read like a detective novel, not a cryptic riddle. Heck, I have seen plenty of logs that announce there was an error without actually saying what it was!

It's not just about logging, though. Monitoring is your eagle-eyed lookout on the crow's nest, alerting you before disaster strikes. It's the art of distinguishing a one-off glitch from the symptoms of a deeper issue. Without effective monitoring, you're essentially flying blind, hoping for the best but ill-prepared for the worst. At a minimum, monitoring should include historical data on key system metrics, like disk and memory usage, perhaps GC related information if you use a managed language, and crucial application internals, such as cache sizes and internal queue lengths. When your production instance goes foobar at 2 AM, this historical data can be a lifesaver in pinpointing the cause of your system's crash.

Now, let's hit the brakes for a moment and address a common mix-up (yes, I've been guilty of this confusion too!): monitoring versus performance metrics. Both are critical, but they're as different as apples and oranges.

Monitoring is your software's constant health check. It's about keeping an eye on the essentials — things like disk and memory usage, cache sizes, and queue lengths. Think of it as the dashboard in your car, not just warning you when you're running on fumes (or about to crash), but also providing a regular update on the overall health of your system. It ensures your software doesn't derail unexpectedly.

On the other side of the coin, we have performance metrics. This is where you measure your software's horsepower. How fast does it handle requests? Does it yield under the stress of heavy user traffic, or does it hold its ground? Performance metrics offer a deep dive into your software's efficiency under various conditions. It's like a detailed report card, pinpointing exactly where improvements are needed for your software to perform at its peak.

In essence, while monitoring, well, monitors your software's day-to-day well-being, performance metrics are your strategic tool for performance optimization and troubleshooting bottlenecks. Both are vital, but they serve different roles in the realm of production-ready software.

Next, let's discuss *performance metrics*, because why not?

### Performance Metrics

Here's an intriguing puzzle I ran into at Hibernating Rhinos, where I was part of the core team for an awesome [NoSQL database - RavenDB](https://ravendb.net/). We got a support ticket that was quite a head-scratcher: why was a query fetching around 100 results consistently taking several seconds, even on a not-so-large database and over a local network?

This mystery landed on my desk for investigation. Luckily, I managed to replicate the issue on my end. The first thing I did was check the performance metrics of the query. Interestingly, it showed the query itself was quick on its feet – taking very little time to process and spit out the results.

{% asset_img time_on_server.png Alt query time spend in-server %}

Next, I used Fiddler to take a look at the query (RavenDB communicates via REST, using Fiddler as a debugging proxy is quite convenient), and there it was – something quite telling.

{% asset_img latency.jpg Alt latency in fiddler output %}

A hundred documents in the query result, weighing a hefty three megabytes? That spelled out the latency issue. In this case, being able to replicate the problem locally made my investigation easier. But let's be real, not all production issues are this cooperative. Without historical performance metrics, pinpointing the root cause – be it data size, network issues, internal workflows, or inefficient database queries – would be like finding a needle in a haystack.

And here's the fun part: after identifying the bottleneck, the fix was straightforward. By adding a projection to the query, the customer drastically cut down the payload (apparently, they required only 2 out of roughly 60 fields per document), and voilà, query latency dropped by over 90%.

But wait, there’s more to performance metrics than just troubleshooting. They're also about cost-efficiency, especially in cloud-based setups. Less data transmission equals lower operational costs – a crucial factor given the associated costs in cloud environments.

Lastly, an often-overlooked aspect: user experience. It's simple – better performance equals happier users. Faster responses lead to a smoother experience, and who doesn't want that?

### Facilities For Memory Dumps

Local debugging is usually straightforward. Depending on your tooling and language, of course, but with the right tools, almost any language offers convenient features like breakpoints, the ability to inspect variable values, and even modify the state of application objects on the fly.

But what happens when we face a memory leak or a process crash a week after deploying to production? Worse still, in a complex system, reproducing the exact conditions for an issue to manifest can be incredibly challenging. So, what can we do? That's where memory dumps come in.

#### What Are Memory Dumps?

Think of a memory dump like a crime scene photo. It's a snapshot of your application's memory at a specific moment in time, usually right when things went south. Just like detectives analyze a crime scene to figure out what happened, developers analyze memory dumps to understand what was going on in the application when it crashed or behaved unexpectedly. Sometimes, this process is called *post-mortem debugging* for rather obvious reasons.

Memory dumps capture everything – from the values of variables to the state of each thread running in your application. It's like freezing time and dissecting your application's brain, allowing us to take a peek into its inner workings at the moment of failure.

#### Why should we care?

In the realm of production, memory dumps are invaluable. They allow us to diagnose issues that are elusive and hard to replicate in a controlled development environment. Think of them as your black box in an airplane, offering crucial insights when everything else fails. With a memory dump, typically, you can:

* Trace the root cause of a crash or memory leak.
* Understand the state of your application's memory before it crashed.
* Identify patterns or anomalies that might not be evident during normal debugging.
* For managed (garbage collected) languages, identify causes for frequent GC cycles.

{% note info %}
Note, I'm pretty deep into the .NET world, so if you are a .Net dev you can start here: [a guide on setting up dump debugging for .NET](https://www.graymatterdeveloper.com/2020/02/12/setting-up-windbg/). But dealing with memory dumps is not something exclusive to .Net. Here are some starting points for getting your tooling ready for handling memory dumps in various languages:

* For the Python wizards: [Python Memory Dump Guide](https://gist.github.com/toolness/d56c1aab317377d5d17a)
* Java gurus, check this out: [Speeding up Java Heap Dumps](https://medium.com/platform-engineer/speeding-up-java-heap-dumps-with-gnu-debugger-c01562e2b8f0)
* JavaScript/Node.js enthusiasts can start here: [Using Heap Snapshot in Node.js](https://nodejs.org/en/docs/guides/diagnostics/memory/using-heap-snapshot)
* Ruby rockstars, perhaps you will find it useful: [Ruby ObjectSpace Documentation](https://docs.ruby-lang.org/en/master/ObjectSpace.html)
{% endnote %}

#### So, what do I mean by 'facilities'?

Different languages need different tools to capture memory dumps, or *core dumps*, as they're known in the Linux world. For instance, .NET developers typically use the [dotnet-dump](https://github.com/dotnet/diagnostics/blob/main/documentation/dotnet-dump-instructions.md) tool to gather memory dumps with .NET-specific information. Node.js developers, would use [Chrome Dev Tools](https://nodejs.org/en/docs/guides/diagnostics/memory/using-heap-snapshot#get-the-heap-snapshot) or perhaps the [heapdump](https://www.npmjs.com/package/heapdump) npm module.

Regardless of the environment, capturing a memory dump necessitates the installation of specific tools. To ensure on-demand memory dump capabilities in production, it's crucial to include the installation of these tools as part of your deployment process. So, for example, for a production deployment in .Net docker, we would install the *dotnet-dump* tool and it's prerequisites (.Net SDK).

### Comprehensive Testing

Now, let's talk about testing. I like to think about testing as a safety net that attempts to catch bugs before they become make production system go foobar.
I know, it sounds cliche, but it is just that. In this realm, there are three musketeers: unit testing, functional testing, and integration testing. Each plays a distinct role and we will talk a bit about them.

#### Unit Testing: The Building Blocks

Unit testing is all about the 'small' stuff. It focuses on individual components or functions of your application, testing them in isolation. Think of it like checking each brick for cracks before you start building a wall. It’s about making sure that each piece of your code does exactly what it’s supposed to do, no more, no less. On a practical side, unit tests typically use [mocks](https://www.geeksforgeeks.org/software-engineering-mock-introduction/) to *isolate* the 'surrounding' functionality of a certain piece of code.

#### Functional Testing: The User’s Perspective

In theory, functional testing is about testing your application from the user's perspective, ensuring that it behaves as expected. This type of testing doesn’t care much about the internals; instead, it looks at the application as a black box and checks if it meets the specified requirements. It's like testing if the wall you built with those bricks actually holds up and serves its purpose.
In practice, you would often mock 'external' servies, be they actual services in microservice architecture or software modules that are not under test in a monolith architecture.

#### Integration Testing: The Symphony

Then comes integration testing, where the magic of collaboration happens. This type of testing checks how different modules or services work together. It’s about ensuring that when your code’s components interact, they do so as expected. Integration testing is like making sure that all the walls, floors, and ceilings in a building come together to create a structure that stands firm and functions as intended.
In practice, for an integration test, you would spin up a *real* system, even if is comprised of multiple services and an in-memory database. For that, it is possible to use console functionality to spin up the services, or if a language has special facilities like [.Net has](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-7.0), we could use that.

#### Other Testing Types: The Situational Lineup

Beyond the big three - unit, functional, and integration testing, there's a lot of other testing types. These types of testing are super important, but they’re like spices in a dish - you use them based on the flavor you're going for. They’re situational, often tailored to the specific needs of your project.

* **Performance Testing:**  It's about making sure your system only bends but doesn't break in a hurricane. It's also very useful for checking the breaking points and throughput of your application.
* **Security Testing:** This is like your app’s digital immune system check-up. In a digital jungle filled with everything from casual script kiddies to advanced cybercriminals, security testing isn't just important; it's vital. But if you are creating an internal tool that will never be published, this kind of testing suddenly becomes less important. In any case, I think it deserves a whole separate blog post. Or three.
* **Automated Testing (automated UI testing):** It's a variant of black-box testing, where the implementers automate user activities and 'simulate' clicks on buttons in the UI. A lifesaver for repetitive tasks and a staple in modern CI/CD pipelines.
* **Usability Testing:** This is all about making sure your app doesn't leave your users scratching their heads. It's the difference between a smooth ride and a maze.

#### Meaningful tests?

I once joined a company that proudly boasted about their 3,000 tests and impressively high test coverage. But when I dug into the actual tests, I stumbled upon a very interesting discovery: they completely lacked [assertions](https://en.wikipedia.org/wiki/Test_assertion)! As a result, every single test they had was effectively meaningless. This might be an extreme example, but it drives my point home. Not all tests are created equal.

{% note info %}
If you're new to testing, here's why assertions are so important:

* An assertion in testing is like a checkpoint in a race - it verifies if you're on the right track. It's a statement that checks whether a certain condition is true. For example, after executing a function, an assertion might check if the output matches what you expect.
* Without assertions, a test might run your code but won’t tell you if the code is actually doing what it's supposed to. The test only checks if the code runs without crashing, not whether it's producing the correct results.
* Writing a test without an assertion is like cooking without tasting the food. You might follow the recipe (your code), but without tasting it (asserting), you won't know if the result is actually good.
{% endnote %}

Ok, so what makes a test both good and meaningful, beyond just having proper assertions? First off, let's talk about the AAA pattern: *Arrange, Act, Assert*. This foundational structure in testing also has a variant known as *given-when-then*, part of Behavior-Driven Development (BDD) practice, which focuses on the end-user experience and translates user stories into actual tests.

A key principle for both approaches is focusing on minimal scope. What does that mean? Essentially, tests are like a microscope. Just as a microscope zooms in on a tiny area to reveal the smallest details, a good test narrows its focus to examine one specific segment of code or a single functionality aspect.
This focused approach ensures each test is clear and concise in what it's checking. If it's a unit or functional test, it should isolate the part being tested, making it easier to pinpoint what's wrong when a test fails and simplifying test maintenance over time. Integration tests won't contain mocks, of course, but they should still test only a single functionality aspect.
This is particularly crucial in complex systems, where testing too many aspects at once can muddy the waters and make it hard to determine exactly what went wrong if a test fails.

Now, let's dive into another crucial aspect of our testing saga: regression testing. Ever squashed a hard-to-find bug and felt like a rockstar, only to have another one pop up, mocking you? That's where regression testing steps in, like a superhero, to save the day.
Think of regression testing as your software's guardian. It remembers the bugs and issues you've fought in the past and keeps an eye out to ensure they don't creep back in. Whenever you patch a bug or tweak a feature, regression testing is that meticulous friend who double-checks everything to make sure nothing else got messed up in the process.

Then, there's the 'happy path' testing. I've seen tests that perfectly replicate a code path executed when everything is fine: no invalid user input, no transient errors, no timeouts, or unhandled exceptions. This approach, however, often overlooks potential issues.

* Ideally, we should create tests for every code branch and every possible exception that can be thrown.
* In practice, given time constraints, it's common to test the happy path first. But then, it's crucial to identify code flows where severe issues might occur and create tests for them. Aiming for about 80% code coverage is a good metric, but don't over-rely on it. Review the tests to ensure they actually test what they're supposed to.

Finally, let's touch on the balance between unit and integration tests. With enough unit tests, even a small code change can ripple through your test base, increasing development costs. So, as your number of tests grows, consider balancing unit tests with integration tests. The latter tends to do more 'black box' checks, which often lowers maintenance costs. Remember, developer time is expensive!