---
title: Hello World with multiple microservices
date: 2019-11-28 10:21:23
tags:
  - C#
  - MassTransit
  - Microservices
  - Architecture
  - Programming
categories:
  - Programming
author: Michael Yarichuk
top_img: top.jpg
cover: /2019/11/28/hello-world-microservices/microservices.jpg
---
A friend of mine was having trouble finding simple but working example of microservices that he could tinker with, finding instead either buzzword-heavy theoretical articles or just samples in weird languages. I decided to prepare a simple project so it can be understood in short amount of time.  
  
The sample project is modeling a "Starbucks-style" coffee ordering process with a cashier, barista and order pick up counter with the following flow:
![](process.jpg)
  
  
### The sample project
For a tech stack, I will be using the awesome [MassTransit](https://masstransit-project.com/) with [RabbitMq](https://www.rabbitmq.com/) as transport and a client process that would send REST requests that "simulate" drink orders.

Here is a quick breakdown of what happens in the sample project
 1. **Client** process sends random drink orders via POST http requests
 2. **Cashier** receives client orders via POST request and puts it into a received orders queue (**BlockingCollection<T>**)
	```cs
	//Nancy module to serve as API for registering orders
	public class CashierAPI : NancyModule
    {
        public CashierAPI(BlockingCollection<Order> ordersQueue)
        {
            Post("/orders", args =>
            {
                ordersQueue.Add(this.Bind<Order>()); //put orders into "order queue"
                return Response.AsJson(new { Message = "OK" });
            });
        }
    }
    ```
    Then the service registers the order and forwards it as a [domain event](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation) to the the rest of the services by pushing it to the message bus.
    
    ```cs
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
            if (_orderQueue.TryTake(out var order, 500))
            {
                order.WhenReceived = DateTime.UtcNow;
                await _messageBus.Publish(order, stoppingToken);
            }
            else
                await Task.Delay(1000, stoppingToken);
    }    
    ```
 3. **Barista** is subscribed to "receive order" domain event, simulate random time working on it then push it to message bus as another domain event
    ```cs
    public async Task Consume(ConsumeContext<Order> context)
    {
        if (!context.Message.IsComplete)
        {
            await Task.Delay(_random.Next(500, 3000));
            context.Message.WhenCompleted = DateTime.UtcNow;
            await _messageBus.Publish(context.Message, CancellationToken.None);
        }

    }    
    ```
 4. And finally, ***PickUpCounter*** would receive the event of *completed* order and announce it to customers
  
{% note info %}
Note that the actual "business logic" code is interleaved with infrastructure code and there is (almost) no logging and error handling. This is something that *obviously* shouldn't happen in production-level code - this stuff is left out for better readability of code.
{% endnote %}
  
### Getting and running the project
* You can get the project from [its Github repository](https://github.com/myarichuk/Samples.MSA).
* Compiling and running it would require [.Net Core 3.0 SDK](https://dotnet.microsoft.com/download/dotnet-core/3.0).
* Running the project depends on local installation of **RabbitMq**. Take a look [here](https://www.rabbitmq.com/download.html) for RabbitMq install instructions.