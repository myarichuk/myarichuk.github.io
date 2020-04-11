---
title: More efficient string concatenation with string.Create()
date: 2020-04-10T01:17:31+03:00
tags:
  - C#
  - Performance
  - Optimization
categories:
  - Programming
author: Michael Yarichuk
top_img: http://www.graymatterdeveloper.com/2020/02/17/data-locality/benchmarkdotnet.png
cover: /2020/04/09/fun-string-create/string.svg
---
Strings are immutable in C#, this is a common knowledge. Unless you use pointers in unsafe blocks, that is.  Apparently, there is another way of making strings mutable.  
When I discovered ``string.Create()``, a not-so-new but for some reason overlooked (by me at least!) [method](https://docs.microsoft.com/en-us/dotnet/api/system.string.create?view=netcore-3.1) that was added since .Net Core 2.2 and Netstandard 2.1, I was really curious how it works, so naturally I looked at relevant .Net Core source code.

```cs
public static string Create<TState>(int length, TState state, SpanAction<char, TState> action)
{
    //omitted some input validation code
   
    //THIS is the interesting part!
    string result = FastAllocateString(length);
    action(new Span<char>(ref result.GetRawStringData(), length), state);
    
    //some more code
}
```
This means that the ``Span<char>`` that the ``action`` receives is a glorified pointer to raw memory of newly allocated string. So far, a similar effect could have been achieved by using pointers to strings in an ``unsafe`` block. 
Naturally, I decided to test how fast would be such approach. I decided to test the performance of string concatenation, if implemented using ``string.Create()``  

First, I implemented ``string.Join()`` using pointers in an ``unsafe`` method - using pointers allows to simply to modify the resulting string in place, without new allocations.
```cs
public unsafe static string JoinWithPtrs(IReadOnlyList<string> strings, string separator = null)
{
    int totalSize = 0;
    for(int i = 0; i < strings.Count; i++)
        totalSize += strings[i].Length;
            
    var hasSeparator = !string.IsNullOrEmpty(separator);
    if(hasSeparator)
        totalSize += (separator.Length * strings.Count - 1);

    var result = new string('\0', totalSize);

    fixed(char* resultPtr = result)                        
    {
        uint offset = 0;
        var separatorByteCount = (uint)separator.Length * sizeof(char);   
        for(int i = 0; i < strings.Count; i++)
        {
            var current = strings[i];
            var byteCount = (uint)current.Length * sizeof(char);                  
            fixed (char* strPtr = current)
                Unsafe.CopyBlockUnaligned(resultPtr + offset, strPtr, byteCount);
                    
            offset += (uint)current.Length;

            if(hasSeparator && i < strings.Count - 1)
            {
                fixed(char* separatorPtr = separator)
                    Unsafe.CopyBlockUnaligned(resultPtr + offset, separatorPtr, separatorByteCount);
                offset += (uint)separator.Length;
            }
        }
    }
    return result;
}
```

Then I implemented  ``string.Join()`` using the newer ``string.Create()`` method.
```cs
public static string JoinStringCreate(IReadOnlyList<string> strings, string separator = null)
{
    int totalSize = 0;
    for(int i = 0; i < strings.Count; i++)
        totalSize += strings[i].Length;
            
    if(!string.IsNullOrEmpty(separator))
        totalSize += (separator.Length * strings.Count - 1);

    //construct the resulting string
    return string.Create(totalSize, (strings, separator), (chars, state) =>
    {
        /*
        note that 'chars' parameter of the lambda
        is the Span<char> that is in fact a pointer to newly allocated string
        */
        var offset = 0;
                
        var separatorSpan = state.separator.AsSpan();
        for(int i = 0; i < state.strings.Count; i++)
        {
            var currentStr = state.strings[i];                    
            currentStr.AsSpan().CopyTo(chars.Slice(offset));
            offset += currentStr.Length;
            if(!string.IsNullOrEmpty(state.separator) && i < state.strings.Count - 1)
            {
                separatorSpan.CopyTo(chars.Slice(offset));
                offset += state.separator.Length;
            }
        }
    });
}
```

Here is the test I came up with.
```cs
[MemoryDiagnoser]
public class Program
{
    private List<string> _strings;

    private string _separator = "*";

    [Params(3, 25, 100)]
    public int ListSize { get; set; }

    [IterationSetup]
    public void Init()
    {
        _strings = new List<string>();
        for (int i = 0; i < ListSize; i++)
            _strings.Add("XYZ");
    }

    [Benchmark]
    public void StringCreateJoin()
    {
        var _ = StringUtils.JoinStringCreate(_strings, _separator);
    }

    [Benchmark]
    public void StringPtrJoin()
    {
        var _ = StringUtils.JoinWithPtrs(_strings, _separator);
    }

    [Benchmark]
    public void RegularStringJoin()
    {
        var _ = string.Join(_separator, _strings);
    }

    [Benchmark]
    public void StringConcatenation()
    {
        var result = string.Empty;
        for (int i = 0; i < _strings.Count; i++)
        {
            result += _strings[i];
            if (i < _strings.Count - 1)
                result += _separator;
        }
    }

    [Benchmark]
    public void ZStringJoin()
    {
        var _ = ZString.Join(_separator, _strings);
    }

    [Benchmark]
    public void StringBuilderJoin()
    {
        var sb = new StringBuilder();
        for (int i = 0; i < _strings.Count; i++)
        {
            sb.Append(_strings[i]);
            if (i < _strings.Count - 1)
                sb.Append(_separator);
        }

        var _ = sb.ToString();
    }

    internal static void Main(string[] args) => BenchmarkRunner.Run<Program>();
}
```

> Just to be thorough, I added to the test the [ZString library](https://github.com/Cysharp/ZString), an awesome library by the creator of equally awesome [Utf8Json](https://github.com/neuecc/Utf8Json). According to the code I saw in ZString repo, it should perform minimum allocations on managed heap (if any!)

The tests results were a bit surprising.

|              Method | ListSize |      Mean |     Error |    StdDev |    Median | Gen 0 | Gen 1 | Gen 2 | Allocated |
|-------------------- |--------- |----------:|----------:|----------:|----------:|------:|------:|------:|----------:|
|    **StringCreateJoin** |        **3** |  **1.436 μs** | **0.0589 μs** | **0.1511 μs** |  **1.350 μs** |     **-** |     **-** |     **-** |      **48 B** |
|       StringPtrJoin |        3 |  1.327 μs | 0.1638 μs | 0.4428 μs |  1.100 μs |     - |     - |     - |      48 B |
|   RegularStringJoin |        3 |  2.631 μs | 0.2790 μs | 0.7731 μs |  2.300 μs |     - |     - |     - |      88 B |
| StringConcatenation |        3 |  1.196 μs | 0.0576 μs | 0.1497 μs |  1.200 μs |     - |     - |     - |     160 B |
|         ZStringJoin |        3 |  2.619 μs | 0.2829 μs | 0.8118 μs |  2.200 μs |     - |     - |     - |      48 B |
|   StringBuilderJoin |        3 |  1.154 μs | 0.0674 μs | 0.1728 μs |  1.050 μs |     - |     - |     - |     152 B |
|    **StringCreateJoin** |       **25** |  **2.224 μs** | **0.2860 μs** | **0.8206 μs** |  **1.700 μs** |     **-** |     **-** |     **-** |     **224 B** |
|       StringPtrJoin |       25 |  2.277 μs | 0.3194 μs | 0.9366 μs |  1.700 μs |     - |     - |     - |     224 B |
|   RegularStringJoin |       25 |  3.296 μs | 0.3298 μs | 0.9302 μs |  2.800 μs |     - |     - |     - |     264 B |
| StringConcatenation |       25 |  2.529 μs | 0.0487 μs | 0.0891 μs |  2.500 μs |     - |     - |     - |    6144 B |
|         ZStringJoin |       25 |  4.727 μs | 0.3210 μs | 0.8949 μs |  4.300 μs |     - |     - |     - |     224 B |
|   StringBuilderJoin |       25 |  2.632 μs | 0.3303 μs | 0.9531 μs |  2.050 μs |     - |     - |     - |     768 B |
|    **StringCreateJoin** |      **100** |  **2.875 μs** | **0.0579 μs** | **0.0452 μs** |  **2.900 μs** |     **-** |     **-** |     **-** |     **824 B** |
|       StringPtrJoin |      100 |  4.034 μs | 0.4147 μs | 1.1964 μs |  4.200 μs |     - |     - |     - |     824 B |
|   RegularStringJoin |      100 |  4.677 μs | 0.0791 μs | 0.1561 μs |  4.700 μs |     - |     - |     - |    2320 B |
| StringConcatenation |      100 | 13.320 μs | 0.5394 μs | 1.5036 μs | 12.750 μs |     - |     - |     - |   84744 B |
|         ZStringJoin |      100 | 12.110 μs | 0.5239 μs | 1.3893 μs | 12.100 μs |     - |     - |     - |     824 B |
|   StringBuilderJoin |      100 |  4.233 μs | 0.3968 μs | 1.1575 μs |  4.350 μs |     - |     - |     - |    2280 B |



First, I was really surprised to find that .Net's ``StringBuilder`` does more allocations than I expected, especially in comparison to my implementations of ``Join()``. Yes, it is consistently more efficient in allocations and runtime than regular string concatenation (with the '+' operator), but since ``StringBuilder`` deals with pointers (as can be seen [in the reference source](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/text/stringbuilder.cs#L634)), I expected it to be doing less allocations.
After browsing ``StringBuilder`` reference source, I noticed the following code in its constructor (as can be seen [here](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/text/stringbuilder.cs#L146))
```cs
//capacity by default is 8000, if no capacity is specified.
m_ChunkChars = new char[capacity];
```

Interestingly enough, in the ``Append()`` method, if there is not enough capacity of ``m_ChunkChars``, the ``StringBuilder`` will allocate new buffer char array in ``ExpandByABlock()`` method (which can be seen [here](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/text/stringbuilder.cs#L1962)).

So, in order to see the best performance out of the ``StringBuilder`` I changed its test method to supply the resulting string length and re-run the test.
Here are the results.
|              Method | ListSize |      Mean |     Error |    StdDev |    Median | Gen 0 | Gen 1 | Gen 2 | Allocated |
|-------------------- |--------- |----------:|----------:|----------:|----------:|------:|------:|------:|----------:|
|    **StringCreateJoin** |        **3** |  **1.505 μs** | **0.0962 μs** | **0.2501 μs** |  **1.400 μs** |     **-** |     **-** |     **-** |      **48 B** |
|       StringPtrJoin |        3 |  1.647 μs | 0.2524 μs | 0.7282 μs |  1.200 μs |     - |     - |     - |      48 B |
|   RegularStringJoin |        3 |  2.901 μs | 0.3827 μs | 1.0667 μs |  2.400 μs |     - |     - |     - |      88 B |
| StringConcatenation |        3 |  1.524 μs | 0.2490 μs | 0.7062 μs |  1.200 μs |     - |     - |     - |     160 B |
|         ZStringJoin |        3 |  3.125 μs | 0.3665 μs | 1.0515 μs |  2.800 μs |     - |     - |     - |      48 B |
|   StringBuilderJoin |        3 |  1.743 μs | 0.2516 μs | 0.7219 μs |  1.350 μs |     - |     - |     - |     144 B |
|    **StringCreateJoin** |       **25** |  **2.337 μs** | **0.3120 μs** | **0.8902 μs** |  **1.800 μs** |     **-** |     **-** |     **-** |     **224 B** |
|       StringPtrJoin |       25 |  2.268 μs | 0.3398 μs | 0.9805 μs |  1.700 μs |     - |     - |     - |     224 B |
|   RegularStringJoin |       25 |  4.023 μs | 0.6376 μs | 1.8293 μs |  3.400 μs |     - |     - |     - |     264 B |
| StringConcatenation |       25 |  3.078 μs | 0.3658 μs | 1.0437 μs |  2.600 μs |     - |     - |     - |    6144 B |
|         ZStringJoin |       25 |  5.118 μs | 0.4363 μs | 1.2449 μs |  4.650 μs |     - |     - |     - |     224 B |
|   StringBuilderJoin |       25 |  2.474 μs | 0.3755 μs | 1.0894 μs |  1.800 μs |     - |     - |     - |     496 B |
|    **StringCreateJoin** |      **100** |  **4.506 μs** | **0.4507 μs** | **1.3217 μs** |  **4.500 μs** |     **-** |     **-** |     **-** |     **824 B** |
|       StringPtrJoin |      100 |  3.969 μs | 0.3715 μs | 1.0778 μs |  4.000 μs |     - |     - |     - |     824 B |
|   RegularStringJoin |      100 |  6.362 μs | 0.4833 μs | 1.3472 μs |  6.100 μs |     - |     - |     - |    2320 B |
| StringConcatenation |      100 | 13.838 μs | 0.6251 μs | 1.7425 μs | 13.400 μs |     - |     - |     - |   84744 B |
|         ZStringJoin |      100 | 10.792 μs | 0.1016 μs | 0.0793 μs | 10.800 μs |     - |     - |     - |     824 B |
|   StringBuilderJoin |      100 |  3.914 μs | 0.4024 μs | 1.1739 μs |  4.100 μs |     - |     - |     - |    1696 B |

This change did make a difference, especially for large size of the list to concatenate (because if the ``m_ChunkChars`` is initialized to proper length, the ``StringBuilder`` doesn't need to make additional allocations to grow its buffer)
Plain old string concatenation unsurprisingly makes lots of allocations, but for small amount of strings it is really fast.

``ZString.Join`` is works slower than anything except plain string concatenation, but its ``Join()`` method is as efficient as my implementations - which is good to know!

And finally, the ``StringCreateJoin()`` allocates the same amount of memory as the approach with pointers, but interestingly, ``StringCreateJoin()`` is a bit slower than ``StringPtrJoin()`` approach.  

> Note that the kind of code I have in ``StringPtrJoin()`` and ``StringCreateJoin()`` are micro-optimizations and while occasionally useful, should be used only when profiling shows a hot-spot in code.

Also, ``StringCreateJoin()`` being a bit slower than ``StringPtrJoin()`` shouldn't be surprsing. In order to support the "magic" of that ``SpanAction`` lambda, the compiler generates the following code. Extra method invocations and in general doing more work than ``StringPtrJoin()``, makes ``StringCreateJoin()`` slower.
```cs
[Serializable]
[CompilerGenerated]
private sealed class <>c
{
    public static readonly <>c <>9 = new <>c();

    [TupleElementNames(new string[] {
        "strings",
        "separator"
    })]
    public static SpanAction<char, ValueTuple<List<string>, string>> <>9__0_0;

    internal void <JoinStringCreate>b__0_0(Span<char> chars, [TupleElementNames(new string[] {
        "strings",
        "separator"
    })] ValueTuple<List<string>, string> state)
    {
        int num = 0;
        ReadOnlySpan<char> readOnlySpan = MemoryExtensions.AsSpan(state.Item2);
        for (int i = 0; i < state.Item1.Count; i++)
        {
            string text = state.Item1[i];
            MemoryExtensions.AsSpan(text).CopyTo(chars.Slice(num));
            num += text.Length;
            if (!string.IsNullOrEmpty(state.Item2) && i < state.Item1.Count - 1)
            {
                readOnlySpan.CopyTo(chars.Slice(num));
                num += state.Item2.Length;
            }
        }
    }
}

public string JoinStringCreate(List<string> strings, string separator = null)
{
    int num = 0;
    for (int i = 0; i < strings.Count; i++)
    {
        num += strings[i].Length;
    }
    if (!string.IsNullOrEmpty(separator))
    {
        num += separator.Length * strings.Count - 1;
    }
    return string.Create(num, new ValueTuple<List<string>, string>(strings, separator), <>c.<>9__0_0 ?? (<>c.<>9__0_0 = new SpanAction<char, ValueTuple<List<string>, string>>(<>c.<>9.<JoinStringCreate>b__0_0)));
}
```

Overall, I can say that this was an interesting excercise in micro-optimizations!
Also, my two attempts at string concatenation are not guaranteed to handle any characters than utf8. .Net's ``StringBuilder`` uses ``wstrcpy()`` to copy strings around into the buffer, which properly handles wide characters (reference to the source code [here](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/text/stringbuilder.cs#L356))
Since I use ``sizeof(char)``, I believe that my code should work for UTF16 characters as well, but I haven't tested it.