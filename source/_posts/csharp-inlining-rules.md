---
title: Reasons for C# inlining are (a bit) more complex than you think.
date: 2020-03-07T17:37:36+02:00
tags:
  - C#
  - Optimization
categories:
  - Programming
author: Michael Yarichuk
top_img: benchmarkdotnet.png
cover: /2020/03/07/csharp-inlining-rules/csharp.png
---
The Twitter sometimes can serve as a place of unexpected insights and very interesting technical questions! For example, the question asked in a tweet here: https://twitter.com/EgorBo/status/1236324907723771904

Apparently, in the following code, C# compiler will inline ``Test1`` and not ``Test2``
```cs
using System;
public static class C
{
    public static void Test1()
    {
        Validate(42, 42);
    }
    
    public static void Test2(int a, int b)
    {
        Validate(a, b);
    }      
    
    public static void Validate(int a, int b)
    {
        if (a == 0 || b == 0)
            throw new Exception("a or b == 0");
    }
}
```

I knew that C# can inline some methods by using various heuristics, but never had the opportunity to know exactly *why*. That is why I was excited to take a look at the [link the .Net Core source](https://github.com/dotnet/runtime/blob/master/src/coreclr/src/jit/inlinepolicy.cpp#L658-L659) the original tweet poster provided. Apparently, passing constant arguments that are then used in a conditional statement (brfalse and brtrue IL commands), increases the *profitability* of the method beeing inlined. 
Browsing the code in ``*inlinepolicy.cpp*`` was extremely enlightening. The reason above might not be enough for .Net to inline the method; it merely increases the chance it would be inlined. There is much more to this: the source file has many descriptions of various code use-cases that would increase "profitability" for the Jit to inline. For example, the [frequency of calls to the method](https://github.com/dotnet/runtime/blob/master/src/coreclr/src/jit/inlinepolicy.cpp#L684) would increase the chances. 
Overall, all sorts of intersting things can be learned from browsing the source code of .Net Core - for example, if IL size of the method is 16 bytes or less, it will be always inlined. (the constant can be seen [here](https://github.com/dotnet/runtime/blob/6dd6b6cc9a60768cf96235a5990bec4894e9106c/src/coreclr/src/jit/inline.h#L902) and its usage in code [here](https://github.com/dotnet/runtime/blob/6dd6b6cc9a60768cf96235a5990bec4894e9106c/src/coreclr/src/jit/inlinepolicy.cpp#L513))

In his book "Functional Thinking: Paradigm Over Syntax", Neal Ford said *Always understand one level below your normal abstraction layer.* 
This is one of those cases, in my opinion. Understanding when and why CLR Jit inlines methods is not really required to write functioning code, but knowing at least a thing on when inlining happens *will* give you an edge when you are writing high perf code. There are good reasons why C# JIT doesn't inline everything; writing inline-able code can give very nice performance boost (as method calls are expensive).
