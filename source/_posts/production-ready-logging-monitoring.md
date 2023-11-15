---
title: "'Production Ready' Non-Negotiable: Structured Logging and Monitoring"
date: 2023-11-15 23:00:00
tags:
  - Software Engineering
  - Theory
  - DevOps
  - Production-Ready
categories:
  - Programming
  - Best Practices
top_img: /2023/11/15/production-ready-2/production-ready-2.jpg
cover: /2023/11/15/production-ready-2/production-ready-2.jpg
---

Hey there! In the [first post](https://www.graymatterdeveloper.com/2023/11/11/production-ready-1/) of the series, we briefly introduced the concept of *production-ready* software and mentioned that there are several non-negotiable aspects that must be present in a *production-system* ready system.

In this post, we will take a look at the first non-negotiable - the unsung heroes of any software system. Structured Logging and Monitoring.

*Oh, but logging is pretty much a given*, I might hear you say. And monitoring? Why would we need that? Our system is up and passes automatic and QA tests, so what's the problem?

Sound familiar? I've been there too. If you've heard this from a fellow developer or, even worse, caught yourself saying it, I would say it's time for a serious perspective shift. [Structured logging](https://www.atatus.com/glossary/structured-logging/) and [monitoring](https://www.digitalocean.com/community/tutorials/an-introduction-to-metrics-monitoring-and-alerting) are more than just a tedious task or an item on a review checklist. Let's take a look why.

## Structured Logging: Not Just Another Log File

Imagine you’re a detective in a crime show. The logs in your application are your clues. Now, what would you prefer? A messy pile of notes or a *well-organized* file with clear, concise, and relevant information? That’s where structured logging comes in.

{% note info %}
What do I mean by *well-organized*?

Imagine a log message in a traditional format: "2023-11-15 10:00:00 ERROR: Login failed for user 12345". Now, see it transformed into a structured format: `{"timestamp": "2023-11-15T10:00:00", "level": "ERROR", "message": "Login failed", "userId": 12345}`. The difference? The latter is far easier to parse and analyze programmatically.

{% endnote %}

Structured logging is about creating log messages that are consistent, searchable, and meaningful. Instead of the traditional text-based logs, structured logs are formatted in a way (like JSON) that machines can easily read and humans can understand without pulling their hair out.

{% note info %}

Here it would be useful to mention [Kibana](https://www.elastic.co/kibana) and [Splunk](https://www.splunk.com/), powerful tools in managing structured logs. They are not the only ones that can handle structured logs, of course, but they are pretty ubiqutous.

* Kibana, part of the Elastic Stack, offers robust log visualization and advanced search (query) capabilities.
* Splunk, while more comprehensive, provides powerful search, analysis, and visualization features. These tools can transform structured logs from mere data points into insightful, actionable information, aiding in efficient system monitoring and troubleshooting.

Those two tools do similar jobs, but in the end, they are not the same: Kibana excels in visualizations and is ideal when you need to analyze log patterns over time. Splunk, on the other hand, is a powerhouse for digging deep into data, making it suitable for complex, data-intensive environments.

{% endnote %}

### Why Structured Logging is Awesome

1. **Searchability**: Ever tried finding a needle in a haystack? That’s what sifting through traditional log files feels like. With structured logging, you can easily filter and search for specific information. Tools like Kibana provide advanced query capabilities, allowing to make advanced filters for log data.
2. **Consistency**: Consistency is key, especially when you’re dealing with large, distributed systems. Structured logs ensure that your log format stays uniform across different parts of your application.
3. **Context**: If properly configured and implemented, structured logs provide context. They tell you not just what happened, but also where and why it happened, making debugging more convenient.
4. **Correlation**: If [correlation Id](https://filipnikolovski.com/posts/correlating-logs/) is used, structured logs can be used to correlate events from different components or services.

### Best Practices

* **Choose the Right Level**: Debug, info, warning, error – each level serves a purpose. Use them wisely.
* **Include Essential Information**: Timestamps, error codes, user IDs – the more context, the better. Don't just announce 'there was an error' - write exactly what happened.
* **Keep It Clean**: Log what’s necessary. Too much information can be as bad as too little. What's worse, too much logs may even cause various [performance issues](https://betterprogramming.pub/logs-can-have-a-strong-impact-on-stability-performance-and-garbage-collection-fc47c600a1e0)

## Monitoring: Your Software’s Health Check

Now, let’s talk about monitoring. If logging is the ‘what’ of your application’s story, monitoring is the ‘how’ and ‘when’. It’s like a health check that tells you how your application is doing at any given moment.

Monitoring is your software's constant health check. It's about keeping an eye on the essentials — things like disk and memory usage, cache sizes, and queue lengths. Think of it as the dashboard in your car, not just warning you when you're running on fumes (or about to crash), but also providing a regular update on the overall health of your system. It ensures your software doesn't derail unexpectedly.

### Key Aspects of Effective Monitoring

1. **System Metrics**: Keep an eye on the basics – CPU usage, memory usage, disk I/O, network stats. These are like the vital signs of your application.
2. **Application Metrics**: These are specific to your application. Number of active users, transaction throughput, response times, queue and cache sizes – metrics that tell you about the performance and health of your application.
3. **Alerts**: What happens when something goes wrong? You should have alerts set up to notify you of potential issues before they escalate.

{% note info %}
**Note that:**

* Battle-proven monitoring toolkits like [Prometheus](https://prometheus.io/docs/introduction/overview/) often have out-of-the-box [alerting features](https://prometheus.io/docs/alerting/latest/overview/)
* More often than not, toolkits like Prometheus have [exporters](https://prometheus.io/docs/instrumenting/writing_exporters/) for monitoring data. For example, [this exporter](https://github.com/prometheus/node_exporter) provides metrics for a certain machine (like disk, memory or network usage)

{% endnote %}

### Monitoring Best Practices

* **Define Meaningful Thresholds**: Not every spike in CPU usage is a crisis. Set thresholds that make sense for your application.
* **Avoid Alert Saturation**: Too many unnecessary alerts and you start ignoring them, just like that fairy tale about the boy who cried 'wolf'. Ensure that your alerts are meaningful and actionable.
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

What *is* the moral? Simple. Monitoring showed us a symptom – the Disk Queue Length. That told us **what** was wrong. But it's the logs that told us the **why** – the simultaneous backup processes. One without the other? Probably it would have taken me much longer to diagnose and fix the problem.

## Conclusion

In the realm of production-ready software, structured logging and monitoring are not just nice-to-haves; they’re non-negotiable. They are essential because they give you the insights and foresight needed to ensure your application is not just surviving but thriving in production.

{% note info %}

Another thing, and it's something that I've seen happened in the past. Structured logging and monitoring should not be forgotten. They should be used to inform your development teams, contributing to a feedback loop that continuously improves your application, based on real operational data. I'd say that part of data-driven decisions about the future of your system should take data from the structured logging and monitoring.

{% endnote %}

## What's Next?

In the next post, we will take a look at the second non-negotiable, performance metrics and discuss why is it important.
