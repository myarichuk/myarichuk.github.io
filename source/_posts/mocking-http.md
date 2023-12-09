---
title: 'Mocking HttpClient the simple way'
date: 2023-12-09T16:06:32+02:00
author: Michael Yarichuk
tags:
  - C#
categories:
  - Programming
top_img: mock-http.jpg
cover: mock-http.jpg
---

Ever found yourself deep in unit tests, only to realize you need to mock an HttpClient? Yep, we've all been there. There are many ways of doing it, but which way is the best? Let me show you a straightforward yet flexible way to mock HttpClient. It's simple, effective, and it's going to make your life a whole lot easier.

So, let's begin with implementing two helpers, which will be useful in setting up our mocks.

First, we create a custom ``HttpMessageHandler`` to intercept and handle HTTP requests in our tests.

```cs
/// <summary>
/// A custom HttpMessageHandler for mocking HttpClient requests.
/// </summary>
public class MockHttpMessageHandler : DelegatingHandler
{
   private readonly Func<HttpRequestMessage, CancellationToken, Task<HttpResponseMessage>> _handlerFunc;

   /// <summary>
   /// Initializes a new instance of the MockHttpMessageHandler class with a specified handler function.
   /// </summary>
   /// <param name="handlerFunc">The function to process HTTP requests.</param>
   public MockHttpMessageHandler(Func<HttpRequestMessage, CancellationToken, Task<HttpResponseMessage>> handlerFunc) => 
      _handlerFunc = handlerFunc ?? throw new ArgumentNullException(nameof(handlerFunc));

   protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken) =>
      _handlerFunc(request, cancellationToken);
}
```

Next up, we have the ``StreamingContent`` class. This nice piece of code helps us simulate real-world network conditions right in our tests. Think of it as a 'troublemaker' for your HttpClient – it streams data in chunks and can intentionally slow things down, mimicking a sluggish network. Plus,you can use it to inject various network issues like timeouts or intermittent connectivity problems. This is especially handy when you want to test how your application handles those not-so-perfect scenarios.

```cs
/// <summary>
/// Custom HttpContent for streaming data in chunks, simulating a slow network.
/// </summary>
public class StreamingContent : HttpContent
{
    private readonly byte[] _data;
    private readonly int _chunkSize;

    /// <summary>
    /// Initializes a new instance of the StreamingContent class with specified data and chunk size.
    /// </summary>
    /// <param name="data">The data to stream.</param>
    /// <param name="chunkSize">The size of each data chunk.</param>
    public StreamingContent(byte[] data, int chunkSize)
    {
        _data = data ?? throw new ArgumentNullException(nameof(data));
        _chunkSize = chunkSize;
    }

    protected override async Task SerializeToStreamAsync(Stream stream, TransportContext? context)
    {
        for (int i = 0; i < _data.Length; i += _chunkSize)
        {
            int remaining = _data.Length - i;
            int count = Math.Min(remaining, _chunkSize);

            await stream.WriteAsync(_data, i, count);
            await Task.Delay(50); // Simulate slow network or "inject" issues
        }
    }

    protected override bool TryComputeLength(out long length)
    {
        length = _data.Length;
        return true;
    }
}
```

Ok, now that we have the helpers, we can simply do the following. This is actually a snippet of a unit test I wrote for one of my pet projects. As you can see, ``HttpClient`` is a real object, but the ``DelegatingHandler`` inside will be doing the heavy lifting of mocking.

```cs
byte[] responseData = new byte[/* size of the mock response */];
// fill the responseData with dummy data we would expect to get as response

var responseMessage = new HttpResponseMessage(HttpStatusCode.OK)
{
   Content = new StreamingContent(responseData, bufferSize)
};

var mockHandler = new MockHttpMessageHandler(
   (request, cancellationToken) => Task.FromResult(responseMessage));
var httpClient = new HttpClient(mockHandler) { BaseAddress = new Uri(/* some base address */) };
```

Well, there you have it! With a few lines of code and a couple of simple helpers, we have done it. No more spending too much time on complex setups or head-scratching scenarios. So, go ahead, give it a whirl and see how it simplifies your test cases. Got questions, thoughts, or a cool test scenario to share? I'd love to hear all about it :)