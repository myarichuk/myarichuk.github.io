---
title: Middleware implementation in ASP.Net Core is weird  
date: 2019-11-27 08:21:44  
categories:
  - Programming
  - ASP.Net
tags: 
 - C# 
 - ASP.Net Core
 - Middleware
author: Michael Yarichuk
top_img: nancy_middleware_code.jpg
cover: /2019/11/27/weird-aspnetcore-middleware/nancy_middleware_code.jpg
---
This might seem like a clickbait-y article, but... it really looks this way! Allow me to explain.  
For one of my pet projects, I considered implementing REST endpoints using [Nancy](http://nancyfx.org/), a nice and low ceremony web framework that I like.  
To my surprise, in order to host it in .Net Core, as evident from the [example here](https://github.com/NancyFx/Nancy/tree/master/samples/Nancy.Demo.Hosting.Kestrel), I would need to use **Microsoft.AspNetCore.Owin** as a "mediator" between Kestrel and Nancy. Seeing this as an excuse to write something in the area I haven't looked into yet, I looked into implementing a middleware component to run Nancy engine directly.  (You can find what I wrote in NuGet or in [Github](https://github.com/myarichuk/Nancy.Hosting.Kestrel))
To my surprise, a minimal middleware implementation looks like this:  
```cs
public class FoobarMiddleware
{
  private readonly RequestDelegate _next;
  public FoobarMiddleware(RequestDelegate next)
  {
     _next = next;
  }
  
  public async Task InvokeAsync(HttpContext context)
  {
     //execute middleware code     
     await _next(context); //continue executing next middleware components
     //execute some more code
  }
}
```
And then, in the [**Startup** class of ASP.Net Core initialization](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup), the middleware is "registered" like this:  
```cs
internal class Startup
{
  public void Configure(IApplicationBuilder app) => app.UseMiddleware<FoobarMiddleware>();
}
```
This seemed like too much "magic" happening - I would expect the middleware class to be required to implement abstract class or interface in an API like this. I don't like too much magic happening in my code so I decided to take a look at ASP.Net Core implementation and look at the  ``UseMiddleware<T>()`` source.  

Well, the relevant part looks like this:
```cs
public static class UseMiddlewareExtensions
{
   internal const string InvokeMethodName = "Invoke";
   internal const string InvokeAsyncMethodName = "InvokeAsync";
   
   // some code...
        
   public static IApplicationBuilder UseMiddleware(this IApplicationBuilder app, Type middleware, params object[] args)
   {
        
      //some code...
            
      return app.Use(next =>
      {
         //fetch middleware methods with reflection!
         var methods = middleware.GetMethods(BindingFlags.Instance | BindingFlags.Public);
         var invokeMethods = methods.Where(m =>
                    string.Equals(m.Name, InvokeMethodName, StringComparison.Ordinal)
                    || string.Equals(m.Name, InvokeAsyncMethodName, StringComparison.Ordinal))
                    .ToArray();

         //some code...           
         
         var methodInfo = invokeMethods[0];
         if (!typeof(Task).IsAssignableFrom(methodInfo.ReturnType))
         {
            throw new InvalidOperationException(Resources.FormatException_UseMiddlewareNonTaskReturnType(InvokeMethodName, InvokeAsyncMethodName, nameof(Task)));
         }
         
         //verify the amount and the type of parameters on "Invoke" or "InvokeAsync"
         var parameters = methodInfo.GetParameters();
         if (parameters.Length == 0 || parameters[0].ParameterType != typeof(HttpContext))
         {
            throw new InvalidOperationException(Resources.FormatException_UseMiddlewareNoParameters(InvokeMethodName, InvokeAsyncMethodName, nameof(HttpContext)));
          }
          
          // some error handling and other code
          
          //create instance of middleware object with reflection!
          var instance = ActivatorUtilities.CreateInstance(app.ApplicationServices, middleware, ctorArgs);
         //the rest of the method...
```
It turns out ASP.Net Core uses reflection to check if middleware class implements either **Invoke** or **InvokeAsync** methods AND uses reflection to instantiate it.  
This is a weird choice, I think. Why use reflection when the same can be achieved at compile time using interface inheritance? Is it a good practice? A good question, to which (at least for now) I don't have an answer.