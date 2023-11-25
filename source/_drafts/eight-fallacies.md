---
title: The Eight Fallacies Of Distibuted Systems
date: 2023-11-24 22:00:00
tags:
  - C#
  - Distributed Systems
  - Architecture
categories:
  - Programming
  - Best Practices
top_img: 8-fallacies-top.jpg
cover: 8-fallacies-top.jpg
---

Imagine yourself in a bustling Spanish restaurant, gathered around a table with friends, eagerly awaiting a variety of mouth-watering tapas. Each small plate offers a unique, delicious experience, and the anticipation of sharing these colorful and unique dishes is almost as enjoyable as the food itself. But have you ever thought about how a tapas experience can help you better understand distributed systems?

Picture this: you're at a lively Spanish eatery, surrounded by friends, all of you buzzing with excitement for an array of tapas. Each dish is a burst of flavor, promising a unique experience. Ever thought this scene could mirror the world of distributed systems?

Like a spread of tapas, distributed systems consist of various elements that must harmonize to deliver as per the business logic. But here's the catch: just as diners navigate the maze of choosing, sharing, and relishing diverse tapas, developers grapple with multiple challenges in distributed system design.

Enter the Eight Fallacies of Distributed Computing. Coined by Peter Deutsch and his team at Sun Microsystems, these fallacies are like hidden traps – easy assumptions that can derail system design. To build distributed systems that are scalable, robust, and efficient, recognizing and countering these fallacies is crucial.

## Network Is Reliable

{% asset_img 1st-fallacy.jpg Alt Illustration of the 1st network fallacy %}

Imagine you're at a bustling restaurant, eager to sample a variety of tapas with your friends. After the waiter passionately describes the menu, you make your selections, which he jots down in a small notebook. As you engage in conversation, waiting with growing anticipation, you notice that the dishes are taking unusually long to arrive. Eventually, the waiter returns, sheepishly explaining that due to the kitchen's busyness, your order was accidentally overlooked. While this is rare, it does happen, even in the best restaurants.

This situation parallels the fallacy in distributed systems of assuming the network is always reliable. Just like your order got lost in the kitchen's bustle, messages or requests in distributed systems, especially those using RPC calls, can be lost or delayed, resulting in unpredictable outcomes or outright failures.

To counter this, we need strategies akin to a restaurant double-checking orders during peak times. In the realm of distributed systems, this translates to implementing robust error handling and retry mechanisms. These are crucial for differentiating between transient network glitches, timeouts, and errors on the destination side. Without such mechanisms, as illustrated in the following code, we risk our digital orders getting lost in a busy 'kitchen'.

Let's consider a concrete example.

**Scenario Description:**  
We have a service named *Foobar* and an *orchestrator* tasked with communicating with *Foobar* via a REST call.

**Problem:**  
The orchestrator's monitoring systems detect a significant percentage of failed requests to the *Foobar* service, indicating unreliable network connections.

**The code (before):**
The initial, naive approach is straightforward. We simply create an abstraction for the *Foobar* service client and that's about it.

```cs
using System.Net.Http;
using System.Threading.Tasks;

public class FoobarServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUrl;

    public FoobarServiceClient(HttpClient httpClient, string serviceUrl)
    {
        _httpClient = httpClient;
        _serviceUrl = serviceUrl;
    }

    public async Task<string> GetDataAsync()
    {
        var response = await _httpClient.GetAsync(_serviceUrl);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
```

**Improved Approach:**

A more resilient approach involves using the [Polly](https://github.com/App-vNext/Polly) library. Polly is a .NET resilience and transient-fault-handling library that allows developers to express policies such as Retry, Circuit Breaker, Timeout, Bulkhead Isolation, and Fallback in a fluent and thread-safe manner. It's particularly useful for enhancing the robustness of network communication.

Incorporating Polly and the [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html) is crucial to prevent cascading failures in your system.

```csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;
using Polly;
using Polly.CircuitBreaker;
using Polly.Retry;

public class FoobarServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUrl;
    private readonly AsyncPolicy<HttpResponseMessage> _resiliencePolicy;

    public FoobarServiceClient(HttpClient httpClient, string serviceUrl)
    {
        _httpClient = httpClient;
        _serviceUrl = serviceUrl;
        
        // Circuit Breaker policy definition
        var circuitBreakerPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .AdvancedCircuitBreakerAsync(
                failureThreshold: 0.5, // Break if >50% actions fail
                samplingDuration: TimeSpan.FromSeconds(10), // Over a 10-second window
                minimumThroughput: 8, // At least 8 actions in the window
                durationOfBreak: TimeSpan.FromSeconds(30) // Break for 30 seconds
            );

        // Define a retry policy: retry up to 3 times with an exponential back-off
        var retryPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

        // Wrap the retry and circuit breaker policies - first execute retry and then circuit breaker
        _resiliencePolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy);
    }

    public async Task<string> GetDataAsync()
    {
        // Execute the HTTP call within the context of the combined resilience policies
        using var response = await _resiliencePolicy.ExecuteAsync(() => _httpClient.GetAsync(_serviceUrl));
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
```

{% note info %}
Note: In the examples above, we are omitting logging and error handling for clarity.
{% endnote %}

The concept of a *circuit-breaker pattern* might seem abstract from the code alone. Here's a simplified depiction of its operation with the policies as they are defined in code.

{% mermaid %}
sequenceDiagram
    participant Client as Client Service
    participant Server as Server Service

    loop Retry Loop
        rect rgb(255, 230, 230)
            Note over Client,Server: Attempt 1..3 (Retries)
            Client->>Server: Request
            Server-->>Client: Failure
        end
    end

    rect rgb(255, 255, 204)
        Note over Client,Server: Circuit Breaker Check
        Note over Client: If >50% fail in 10s with ≥8 requests, open circuit
    end

    alt Circuit Closed
        rect rgb(204, 255, 204)
            Client->>Server: Request
            Server-->>Client: Success/Failure
            Note over Client: Normal Operation
        end
    else Circuit Open
        rect rgb(255, 204, 204)
            Client->>Server: Request
            Server--XClient: No Call (Circuit Open)
            Note over Client: Wait for Break Duration (30s)
        end
    end

    rect rgb(255, 255, 204)
        Note over Client,Server: Circuit Half-Open (Retry after wait)
        Client->>Server: Request
        alt Success
            Server-->>Client: Success
            Note over Client: Circuit Closed
        else Failure
            Server-->>Client: Failure
            Note over Client: Circuit Opens Again
        end
    end
{% endmermaid %}

### Explanation

In this sequence, the Client Service communicates with the Server Service. The behavior is as follows:

1. **Retry Loop**: The Client attempts to communicate with the Server. If a request fails, it retries up to 3 times. Each set of retries for a request counts as one failure event for the Circuit Breaker.

2. **Circuit Breaker Check**: The Circuit Breaker evaluates failures: if over 50% of requests fail within 10 seconds with at least 8 requests, the circuit opens.

3. **Circuit State**:
   - **Circuit Closed**: Normal operation with the Client sending requests to the Server.
   - **Circuit Open**: Further requests are blocked for 30 seconds.

4. **Circuit Half-Open**: After the break, the circuit is half-open. A new request tests if the Server is operational:
   - **Success**: The circuit closes, resuming normal operation.
   - **Failure**: The circuit reopens, and the waiting cycle repeats.

Overall, taking into account the possiblity of network failures is crucial in distributed systems. Rather than assuming reliability or just not thinking about failure, it's important to actively plan for failure. Implementing robust error handling mechanisms, like retries and circuit breakers through tools like Polly, is not just a backup plan—it's an essential part of creating resilient systems.

## Latency Is Zero

{% asset_img 2nd-fallacy.jpg Alt Illustration of the 2nd network fallacy %}

Imagine you're at a popular tapas restaurant renowned for its team of specialized cooks, each an expert in preparing a specific type of dish. You and your friends, excited by the variety, order an array of tapas. You might expect that since all dishes are being prepared in the same kitchen, they should arrive at your table simultaneously. However, as the meal progresses, you notice significant delays in serving some dishes.

For instance, a dish requiring a rare ingredient takes longer as the chef searches through the storeroom for it. Another, more complex dish takes time because of the intricate preparation required by the specialist cook. Meanwhile, simpler dishes prepared by faster cooks arrive much earlier. This disparity in preparation and serving times leads to an uneven dining experience, with some tapas being enjoyed fresh and others arriving later than anticipated, diminishing their appeal.

This scenario in the tapas restaurant mirrors the challenges of network latency in distributed systems. Just as you experienced delays due to various preparation factors, despite all dishes coming from the same kitchen, in distributed systems, different services, although part of the same network, can have varying response times. These delays, akin to the time taken for finding rare ingredients or intricate cooking processes, can lead to asynchronous and uneven system performance. Recognizing and planning for these varying 'preparation times' is crucial to ensuring a more consistent and efficient experience, both at a tapas table and in a distributed computing environment.

To make this concept more concrete, let's consider a realistic scenario.

**Scenario Description:**

- **Service A (Orchestrator):** Manages and monitors business logic sessions. It employs a timeout mechanism to ensure sessions don't hang indefinitely.
- **Service B (Worker Service):** Executes steps of the business logic flow, regularly updating the Orchestrator on its progress.

**Problem:** Service A, the Orchestrator, has a timeout mechanism that might not adequately account for the network latency in communication with Service B. If Service B takes longer to respond due to network latency or unforeseen delays in executing the business logic, Service A may prematurely terminate the session, interrupting the workflow, even though the flow is proceeding normally.

{% note info %}
In this example, we assume an [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call) communication between the services.
{% endnote %}
**The code (before):**

```csharp
// In Service A (Orchestrator)
public async Task MonitorSessionAsync(Guid sessionId)
{
    var timeout = TimeSpan.FromSeconds(10);
    var endTime = DateTime.UtcNow + timeout;

    while (DateTime.UtcNow < endTime)
    {
        if (CheckIfSessionCompleted(sessionId)) // This dequeues updates received from Service B
        {
            // Process completion
            return;
        }
        await Task.Delay(1000); // Check every second
    }
    TerminateSession(sessionId); // Timeout and terminate session
}
```

**Improved Approach:**

1. **Adjust Timeouts:** Increase timeout duration in the Orchestrator to account for expected network delays.
2. **Periodic Heartbeats:** Implement a heartbeat or periodic update mechanism from Service B to keep Service A informed, even if the main task is still processing.

{% note info %}
Note that I intentionally left out logging, error handling and other niceties for the sake of clarity.
{% endnote %}

**The code (after):**

```csharp
// In Service A (Orchestrator)
public async Task MonitorSessionAsync(Guid sessionId)
{
    var extendedTimeout = TimeSpan.FromSeconds(30); // Adjusted timeout
    var endTime = DateTime.UtcNow + extendedTimeout;

    while (DateTime.UtcNow < endTime)
    {
        if (CheckIfSessionCompleted(sessionId) ||
            CheckIfHeartbeatReceived(sessionId)) // we also check if service B is functioning
        {
            // Process completion or heartbeat received
            endTime = DateTime.UtcNow + extendedTimeout; // Extend timeout on heartbeat
        }
        await Task.Delay(1000); // Check every second
    }
    TerminateSession(sessionId); // Timeout and terminate session
}

// In Service B (Worker Service)
public async Task ExecuteTaskAndUpdateAsync(Guid sessionId)
{
    // Execute task
    // Periodically send heartbeat to Orchestrator
}
```

Essentially, we integrate a heartbeat call into the monitoring *service A* will be doing and in doing so, ensure that as long as *service B* is **alive** but delayed, we won't time out the work session *service B* is doing.

{% note info %}
Heartbeats in this case can be anything, from a NOP message
{% endnote %}

{% mermaid %}
sequenceDiagram
    participant A as Service A (Orchestrator)
    participant B as Service B (Worker Service)

    Note over A,B: Business Logic Session Starts
    A->>B: Initiate Task
    loop Health Check
        B->>A: Heartbeat Signal
        A->>A: Check Health Status
        Note over A: Extend Session Timeout on Receipt
    end
    B->>A: Task Completion Signal
    A->>A: Process Task Completion
    Note over A,B: Session Ends
{% endmermaid %}

In a distributed environment, always account for the inevitable latency. Design your systems, especially orchestrators and workers, to expect and gracefully handle these delays. Adjust timeouts and use heartbeats or periodic updates to maintain synchronization and avoid premature session termination.

And overall, always plan for network latency by adjusting timeouts and implementing mechanisms like heartbeats to keep systems in sync. This approach reduces the risk of interrupted workflows and ensures more stable and reliable inter-service communication.

Certainly! Here's a draft for the third fallacy, "Bandwidth is Infinite," using the tapas metaphor:

## Bandwidth is Infinite

{% asset_img 3rd-fallacy.jpg Alt Illustration of the 3rd network fallacy %}

Imagine returning to the beloved tapas restaurant with your friends. This time, your group, excited by the menu's variety, places a large and intricate order, making adjustments and additions. Each change increased the complexity of the order, putting strain on the kitchen's capacity to manage and execute your and other restaurant client requests efficiently. As a result, the kitchen staff become overwhelmed, leading to delayed and mixed-up orders. This chaos in the kitchen and subsequent delays in serving reflect the chaos and slowdowns caused by overloading a network beyond its capacity.

This scenario mirrors the "Bandwidth is Infinite" fallacy in distributed systems. Similar to how the kitchen struggles with the complex, constantly changing order, a network can get overwhelmed when too much data is sent without considering its capacity limits. Overestimating bandwidth can lead to network congestion, delayed data transmission, and in severe cases, lost or incomplete data packets. Recognizing the limits of network bandwidth is crucial for efficient data flow and overall system performance.

Now, let's take a look at a more practical example (which is based on a real-life case) that illustrates this.

**Scenario Description:**
There is a distributed system with web-based frontend, allowing interaction with the system. The data in the system only grows, there are the soft deletes in the system.

**Problem:**  
A database query for a certain endpoint is returning a full dataset, resulting in high latency (around 10 seconds) due to the large volume of data being transferred over the network.

**The code (before):**
```csharp
public async Task<List<LargeDataSet>> FetchSomeImportantData(string userId)
{
    // Fetching full dataset without projection
    return await _dbContext.LargeDataSets.Where(row => row.AssignedUserId == userId).ToListAsync();
}
```

**Improved Approach:**  
By applying a projection to the database query, we can make the code transfer only the relevant data, reducing the data volume. This is a simplified scenario, in the real-life use case such change improved endpoint latency from roughly 10 seconds to roughly 1 second.

**The code (after):**
```csharp
public async Task<List<OptimizedDataSet>> FetchSomeImportantData(string userId)
{
    // Applying projection to fetch only necessary fields
    return await _dbContext.LargeDataSets
                           .Where(row => row.AssignedUserId == userId)
                           .Select(data => new OptimizedDataSet
                           {
                               EssentialField1 = data.EssentialField1,
                               EssentialField2 = data.EssentialField2,
                           })
                           .ToListAsync();
}
```
  
In a distributed environment, and client/server communication of a database and a service is indeed a distributed environment, always account for the size of the data. All networks, regardless of their type, have finite bandwidth. Overload the network, things get slow. Or timeout. Or both.
