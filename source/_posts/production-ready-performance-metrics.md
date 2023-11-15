---
title: "'Production Ready' Non-Negotiable: Performance Metrics"
date: 2023-11-15T19:43:55+02:00
tags:
  - Software Engineering
  - Theory
  - DevOps
  - Production-Ready
categories:
  - Programming
  - Best Practices
top_img: production-ready-performance.jpg
cover: production-ready-performance.jpg
---

Here's a curious challenge I faced at Hibernating Rhinos, as part of the core team for the [NoSQL database - RavenDB](https://ravendb.net/). We received a support ticket that was puzzling: why was a query, pulling roughly 100 results, taking several seconds to complete? This was odd, especially given the database's modest size and the use of a local network.

### Investigation Kicks Off

This conundrum landed on my desk. Fortunately, I was able to replicate the issue on my system. My first step? Checking the query's performance metrics. Surprisingly, the query itself was fast, efficiently processing and delivering results.

{% asset_img time_on_server.png Alt query time spent in-server %}

### Uncovering More Clues

Next, I turned to Fiddler to inspect the query (RavenDB uses REST, so Fiddler's a handy proxy for debugging). And there it was: a crucial hint.

{% asset_img latency.jpg Alt latency in fiddler output %}

The query result contained a hundred documents, totaling a hefty three megabytes. That was the root of the latency issue. Being able to replicate the problem locally simplified my investigation. But, let's be honest, not all production issues are this accommodating. Without historical performance data, pinpointing issues like data size, network glitches, workflow inefficiencies, or clunky database queries would be akin to a needle-in-a-haystack scenario.

The good news? Once we identified the bottleneck, the solution was straightforward. We advised the customer to add a projection to their query. They really needed only 2 of the 60 fields per document. This small change reduced the data payload significantly, slashing query latency by over 90%.

### Why Bother With Performance Metrics?

This anecdote, I hope, highlights the significance of performance metrics, particularly those that archive historical data. But there's more to it. Performance metrics are invaluable not only for troubleshooting but also for enhancing cost-efficiency, especially in cloud environments. Reducing data transfers can significantly cut operational expenses, a crucial factor considering the hefty costs often associated with cloud services.

### A Word on User Experience

And don't forget about user experience. Faster, more responsive software means a smoother, more satisfying experience for users. Ultimately, that's what we're all aiming for, right?

### Performance Metrics vs. Monitoring

In the [previous post](https://www.graymatterdeveloper.com/2023/11/14/production-ready-logging-monitoring/), we talked about monitoring. But performance metrics are a different ball game.

Here's a common confusion (I've been there too): mixing up monitoring and performance metrics. Yes, both are crucial, but they're distinct. Monitoring is like your software's regular health checkup, keeping tabs on critical stuff like disk and memory usage. It's the dashboard in your car, warning you of potential issues and giving you a general health update.

Performance metrics, on the other hand, are about measuring your software's efficiency. How quickly does it handle requests? Can it cope with heavy traffic? These metrics are your deep-dive into your software's performance, showing you exactly where to tweak and improve.

In short, monitoring watches over your software's daily health, while performance metrics are your go-to for optimization and solving performance bottlenecks. Both are essential for your software to be truly production-ready.

## Series Roadmap

- 🔖 [**Introduction: Understanding Production-Ready Software**](https://www.graymatterdeveloper.com/2023/11/11/production-ready-intro/)
- 📈 [**Part 1: Structured Logging and Monitoring**](https://www.graymatterdeveloper.com/2023/11/15/production-ready-logging-monitoring/)
- ⏳ [**Part 2: Performance Metrics: Ensuring Reliability Under Pressure**](https://www.graymatterdeveloper.com/2023/11/15/production-ready-performance-metrics/) - *(You Are Here)*
- 🧰 [**Part 3: Memory Dumps: Deciphering the Black Box**](https://www.graymatterdeveloper.com/2023/11/15/production-ready-memory-dumps/)
- 🧪 [**Part 4: Comprehensive Testing: From Unit to Integration**](#) *(Coming Soon)*
- 🛡️ [**Part 5: Beyond the Basics: Security, Scalability, and Documentation**](#) *(Coming Soon)*

## What's Next?

Next up, we'll delve into the third non-negotiable: memory dumps and related tools, discussing their importance and impact.
