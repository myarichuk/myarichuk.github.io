---
title: "'Production Ready' Non-Negotiable: Structured Logging and Monitoring"
date: 2023-11-14 23:00:00
tags:
  - Software Engineering
  - Theory
  - DevOps
  - Production-Ready
categories:
  - Programming
  - Best Practices
top_img: production-ready-2.jpg
cover: production-ready-2.jpg
---

Hey there! In the [previous post](https://www.graymatterdeveloper.com/2023/11/11/production-ready-intro/) of the series, we introduced the concept of *production-ready* software and mentioned that there are several non-negotiable elements that must be present in such a system.

So, let's take a look at the first non-negotiable - the unsung heroes of any software system. [Structured logging](https://www.atatus.com/glossary/structured-logging/) and [Monitoring](https://www.digitalocean.com/community/tutorials/an-introduction-to-metrics-monitoring-and-alerting).

*Oh, but we have logging; it's just some text that we print when stuff happens. Why complicate things?*, you might ask. *And monitoring? Why would we need that? We have already a testing suite, so there is no problem...*

Sound familiar? Me? I've been there. If you've heard this from a fellow developer, your team leader or even worse, caught yourself saying it, I would say it's time for a serious perspective shift. [Structured logging](https://www.atatus.com/glossary/structured-logging/) and [monitoring](https://www.digitalocean.com/community/tutorials/an-introduction-to-metrics-monitoring-and-alerting) are more than just a tedious task or an item on a review checklist. Let's take a look why. And before we continue, I think it's important to mention that what I write here is not a 'silver bullet'. There may be cases where the cost of implementing and maintaining such solutions are bigger than the value they would provide. Always be mindful of development costs :)

## Structured Logging: Not Just Another Log File

Imagine you’re a detective in a crime show. The logs in your application are your clues. Now, what would you prefer? A messy pile of notes or a *well-organized* file with clear, concise, and relevant information? That’s where structured logging comes in.

{% note info %}
What do I mean by *well-organized*?

Imagine a log message in a traditional format: "2023-11-15 10:00:00 ERROR: Login failed for user 12345". Now, see it transformed into a structured format: `{"timestamp": "2023-11-15T10:00:00", "level": "ERROR", "message": "Login failed", "userId": 12345}`. The difference? The latter is far easier to parse and analyze programmatically.

{% endnote %}

Structured logging is about creating log messages that are consistent, searchable, and meaningful. Instead of the traditional text-based logs, structured logs are formatted in a way (like JSON) that machines can easily read and humans can understand without pulling their hair out.

{% note info %}

Without proper tooling, structured logging would remain a fancy concept.  
[Kibana](https://www.elastic.co/kibana) and [Splunk](https://www.splunk.com/), are powerful and battl-proven tools in managing structured logs. They are not the only ones that can handle structured logs, of course, but they are pretty ubiqutous.

* Kibana, part of the Elastic Stack, offers robust log visualization and advanced search (query) capabilities.
* Splunk is a bit more holistic, it also provides powerful search features, it has analysis, and visualization features too.

Those two tools do similar jobs, but in the end, they are not exactly the same: Kibana excels in visualizations and data discovery. It is ideal when you need to analyze log patterns over time. Splunk, on the other hand, is a powerhouse for digging deep into data, making it more suitable for complex, data-intensive environments.

{% endnote %}

### Why Structured Logging is Awesome

1. **Searchability**: Ever tried finding a needle in a haystack? That’s what sifting through traditional log files feels like. With structured logging, you can easily filter and search for specific information. Tools like Kibana provide advanced query capabilities, allowing you to make advanced filters for log data.
2. **Consistency**: Consistency is key, especially when you’re dealing with large, distributed systems. Structured logs ensure uniform log formatting across different parts of your application.
3. **Context**: If properly configured and implemented, structured logs provide context. They tell you not just what happened, but also where and why it happened, making debugging more convenient.
4. **Correlation**: If [correlation Id](https://filipnikolovski.com/posts/correlating-logs/) is used, structured logs can be used to correlate events from different components or services.

### Best Practices

* **Choose the Right Level**: Debug, info, warning, error – each level serves a purpose. Use them wisely.
* **Include Essential Information**: Timestamps, error codes, user IDs – the more context, the better. Don't just announce 'there was an error' - write exactly what happened.
* **Keep It Clean**: Log what’s necessary. Too much information can be as bad as too little. What's worse, too much logs may even cause various [performance issues](https://betterprogramming.pub/logs-can-have-a-strong-impact-on-stability-performance-and-garbage-collection-fc47c600a1e0)

## Monitoring: Your Software’s Health Check

Now, let’s talk about monitoring. If logging is the ‘what’ of your application’s story, monitoring is the ‘how’ and ‘when’. It’s like a health check that tells you how your application is doing at any given moment.

Monitoring serves as a constant health check for your software. It's about keeping an eye on the essentials — things like disk and memory usage, cache sizes, and queue lengths. Think of it as the dashboard in your car, not just warning you when you're running on fumes (or about to crash), but also providing a regular update on the overall health of your system. It ensures your software doesn't derail unexpectedly.

### Key Aspects of Effective Monitoring

1. **System Metrics**: Keep an eye on the basics – CPU usage, memory usage, disk I/O, network stats. These are like the vital signs of your application.
2. **Application Metrics**: These are specific to your application. Number of active users, transaction throughput, response times, queue and cache sizes – metrics that tell you about the performance and health of your application.
3. **Alerts**: What happens when something goes wrong? You should have alerts set up to notify you of potential issues before they escalate.

{% note info %}
**Note that:**

* Battle-proven monitoring toolkits like [Prometheus](https://prometheus.io/docs/introduction/overview/) often have out-of-the-box [alerting features](https://prometheus.io/docs/alerting/latest/overview/)
* More often than not, toolkits like Prometheus have out-of-the-box [exporters](https://prometheus.io/docs/instrumenting/writing_exporters/) that provide data. For example, [this exporter](https://github.com/prometheus/node_exporter) is a really good provider of *system metrics* (like disk, memory or network usage)

{% endnote %}

### Monitoring Best Practices

* **Define Meaningful Thresholds**: Not every spike in CPU usage is a crisis. Set thresholds that make sense for your application.
* **Avoid Alert Saturation**: Excessive, unnecessary alerts can lead to indifference, much like the boy who cried 'wolf' in the famous fairy tale. Ensure that your alerts are meaningful and actionable.
* **Historical Data is Gold**: Make sure to keep historical data, be it a database or tools like Kibana. Without historical data, how will you debug a critical system failure at 2am?

## Structured Logging and Monitoring: Better Together

When structured logging and effective monitoring work together, just like Power Rangers zords, they merge into something even more powerful. Logs provide the detailed narrative of your application’s life, while monitoring keeps an eye on the overall health and performance. Together, they ensure that you’re not just prepared for when things go wrong, but also equipped to prevent issues in the first place.

### Better Together?

Why? Let me give you an example I have encountered. While working for a NoSQL database vendor, a seemingly weird issue landed on my desk. A server-wide backup failed.
The customer had a server hosting numerous databases on the same instance. The server was far from being overloaded and had plenty of hardware power to spare. One day, a server-wide backup of the database service decides to fail, for some reason. Now, while rare, but such a failure isn’t new. By the way, all these databases were scheduled to start their backup journey to a NAS drive, roughly at the same cosmic moment.

### Monitoring Meets Logging?

First, I checked the monitoring and a peculiar oddity caught my eye: disk usage stats were weird. The Disk Queue Length was really, really high. That was the first piece of the puzzle.

Checking the logs was next. It didn't take too long to find the next piece of the puzzle: about a thousand databases decided to start their backup at the same time. The feature worked flawlessly during development, but the developers simply didn't take into account the possiblity that so many databases would try to backup its data on a NAS drive.

### The Aha Moment: Too Much, Too Soon

Putting these pieces together – the crying-for-help disk queue length and a large amount of backups from our logs allowed me to reach a rather obvious conclusion. NAS simply timed out under the heavy load. So the solution was straight forward - I simply made the backups serial and not parallel. But that's not the moral of the story.

What *is* the moral? Simple. Monitoring showed us a symptom – the Disk Queue Length. That told us **what** was wrong. But it's the logs that told us the **why** – the simultaneous backup processes. Without one complementing the other, it would likely have taken me much longer to diagnose and resolve the problem.

## Conclusion

In the world of production-ready software, structured logging and monitoring are non-negotiable. They are essential because they give you the insights and foresight needed to ensure your application is not just surviving but thriving in production.

{% note info %}

Another thing, and it's something that I've seen happened in the past. Structured logging and monitoring should not be forgotten. They should be used to inform your development teams, contributing to a feedback loop that continuously improves your application, based on real operational data. I'd say that part of data-driven decisions about the future of your system should take data from the structured logging and monitoring.

{% endnote %}

## Series Roadmap

- 🔖 [**Introduction: Understanding Production-Ready Software**](https://www.graymatterdeveloper.com/2023/11/11/production-ready-intro/)
- 📈 [**Part 1: Structured Logging and Monitoring**](https://www.graymatterdeveloper.com/2023/11/15/production-ready-logging-monitoring/) - *(You Are Here)*
- ⏳ [**Part 2: Performance Metrics: Ensuring Reliability Under Pressure**](https://www.graymatterdeveloper.com/2023/11/15/production-ready-performance-metrics/)
- 🧰 [**Part 3: Memory Dumps: Deciphering the Black Box**](https://www.graymatterdeveloper.com/2023/11/15/production-ready-memory-dumps/)
- 🧪 [**Part 4: Comprehensive Testing: From Unit to Integration**](#) *(Coming Soon)*
- 🛡️ [**Part 5: Beyond the Basics: Security, Scalability, and Documentation**](#) *(Coming Soon)*

## What's Next?

In the [next post](https://www.graymatterdeveloper.com/2023/11/15/production-ready-performance-metrics/), we will explore the second non-negotiable: performance metrics and discuss why they are important.