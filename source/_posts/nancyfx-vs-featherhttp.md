---
title: NancyFx vs. FeatherHttp
date: 2020-04-25T12:28:54+03:00
tags:
  - C#
  - Programming
  - REST
  - Framework
  - Performance
categories:
  - Programming
author: Michael Yarichuk
top_img: top.jpg
cover: https://httpwg.org/asset/http.svg
---

As a long time fan of the lightweight syntax of the awesome [NancyFx web framework](http://nancyfx.org/), I was really curious when I stumbled upon [FeatherHttp](https://github.com/featherhttp/framework), a new low-ceremony framework for building web services in .Net Core applications.  
  
Naturally, I was curious not only about the syntax but about the performance as well. So I decided to compare their raw performance, using the awesome [wrk2 tool](https://github.com/giltene/wrk2).
As a first step, I created two HTTP servers with one endpoint that does the same work in both cases.

1. NancyFx server code looks like this:
```cs
public class HelloWorldModule : NancyModule
{
    public HelloWorldModule() => 
        Get("/foo/bar", @params => Response.AsJson(new { Message = "Hello World!" }));
}

public class Program
{
    static async Task Main(string[] args) => 
        await CreateWebHostBuilder().Build().RunAsync();

    public static IWebHostBuilder CreateWebHostBuilder() =>
        new WebHostBuilder()
            .UseKestrel()
            .UseStartup<Startup>();
}

internal class Startup
{
    public void Configure(IApplicationBuilder app) => app.UseNancy();
}
```

2. And FeatherHttp server code looks like this:
```cs
class Program
{
    static async Task Main(string[] args)
    {
        var app = WebApplication.Create(args);

        app.MapGet("/foo/bar", 
            async http => 
                await http.Response.WriteJsonAsync(new { Message = "Hello World!" }));
        await app.RunAsync();
    }
}
```
> I really LOVED the syntax of FeatherHttp - really concise and low-ceremonly. It is even shorter than NancyFx!

The next step was to actually run the benchmark tool - *wrk*. I ran the following command for both of the services, which were compiled using .Net Core 3.1
```bash
wrk -c500 -t4 -d30s -R2000 http://localhost:5000/
```

I got the following results.  
1. For NancyFx I got
```command
Running 30s test @ http://localhost:5000/
  4 threads and 500 connections
  Thread calibration: mean lat.: 1080.288ms, rate sampling interval: 4083ms
  Thread calibration: mean lat.: 803.334ms, rate sampling interval: 3108ms
  Thread calibration: mean lat.: 934.588ms, rate sampling interval: 3475ms
  Thread calibration: mean lat.: 919.774ms, rate sampling interval: 3850ms
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.19s     1.60s    8.94s    64.88%
    Req/Sec   459.00     21.23   499.00     68.42%
  51947 requests in 30.09s, 1.37GB read
  Non-2xx or 3xx responses: 51947
Requests/sec:   1726.33
Transfer/sec:     46.60MB
```

2. For FeatherHttp I got
```command
Running 30s test @ http://localhost:5000/
  4 threads and 500 connections
  Thread calibration: mean lat.: 30.436ms, rate sampling interval: 92ms
  Thread calibration: mean lat.: 31.322ms, rate sampling interval: 95ms
  Thread calibration: mean lat.: 2.804ms, rate sampling interval: 10ms
  Thread calibration: mean lat.: 33.685ms, rate sampling interval: 97ms
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    22.65ms   15.65ms  64.58ms   55.21%
    Req/Sec   525.25    341.35     3.00k    73.96%
  58471 requests in 30.04s, 5.52MB read
  Non-2xx or 3xx responses: 58471
Requests/sec:   1946.75
Transfer/sec:    188.21KB
```

The numbers mean that FeatherHttp is a winner by a slight margin. Though, 10% more throughput is hardly a small number, if handling of high number of requests/sec is needed.  
Note that this is a very basic comparison of performance capabilities of both frameworks. First, despite it's relatively complete API, FeatherHttp is at an alpha stage, so its performance might change in the future, for better or for worse.  
Second, and more importantly, a true performance benchmark would involve comparing the speed of routing algorithm, memory allocations per request and per use of various features and more.
  
Overall, I am excited by FeatherHttp framework and intend to use it in one of my pet projects as soon as opportunity arises :)