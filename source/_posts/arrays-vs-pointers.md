---
title: Is it faster to access arrays with pointer arithmetics?
date: 2020-02-23T08:39:23+02:00
tags:
  - C#
  - Performance
  - Optimization
categories:
  - Programming
author: Michael Yarichuk
top_img: /2020/02/17/data-locality/benchmarkdotnet.png
cover: /2020/02/23/arrays-vs-pointers/array.svg
---
After seeing the results of my [previous post](/2020/02/17/data-locality/) where I tested performance impact of data locality, one of my collegues theorized that perhaps the huge difference between C++ and C# performance (assuming adjacent memory iteration) is due to boundary checks of C# arrays - he said that if I would use pointers to access arrays, the code will be much faster. I was curious if he was right, so I tested it.


## The test
I used the following test to try and see how much performance usage of pointers over simple C# arrays would yield.

```cs
[DisassemblyDiagnoser(printSource: true, printDiff: true, printIL: true)]
public class Program
{
    [Params(64, 512, 1024, 4 * 1024, 8 * 1024, 16 * 1024, 32 * 1024, 64 * 1024)]
    public int Size { get; set; }

    private long[] Array;

    [IterationSetup]
    public void Init() => Array = new long[Size];

    [Benchmark(Baseline = true)]
    public void IterateArray()
    {
        for (int i = 0; i < Size; i++)
            Array[i] = i;
    }

    [Benchmark]
    public unsafe void IterateArrayWithPtr()
    {
        fixed (long* ptr = Array)
        {
            for (int i = 0; i < Size; i++)
                *(ptr + i) = i; 
        }
    }

    static void Main(string[] args) => BenchmarkRunner.Run<Program>();
}
```

## The Results
The results were a bit surprising (at least to me!)
I expected pointer access to be slightly faster, but apparently,it is not so (and sometimes pointer usage is a bit slower!)

|              Method |  Size |         Mean |        Error |       StdDev |      Median | Ratio | RatioSD |
|-------------------- |------ |-------------:|-------------:|-------------:|------------:|------:|--------:|
|        **IterateArray** |    **64** |     **84.34 ns** |     **13.70 ns** |     **36.57 ns** |    **100.0 ns** |     **?** |       **?** |
| IterateArrayWithPtr |    64 |    104.55 ns |     17.06 ns |     50.05 ns |    150.0 ns |     ? |       ? |
|                     |       |              |              |              |             |       |         |
|        **IterateArray** |   **512** |    **386.59 ns** |     **12.93 ns** |     **34.29 ns** |    **400.0 ns** |  **1.00** |    **0.00** |
| IterateArrayWithPtr |   512 |    380.21 ns |     13.88 ns |     40.05 ns |    400.0 ns |  1.00 |    0.16 |
|                     |       |              |              |              |             |       |         |
|        **IterateArray** |  **1024** |    **577.00 ns** |     **19.79 ns** |     **58.35 ns** |    **600.0 ns** |  **1.00** |    **0.00** |
| IterateArrayWithPtr |  1024 |    571.00 ns |     16.20 ns |     47.77 ns |    600.0 ns |  1.00 |    0.13 |
|                     |       |              |              |              |             |       |         |
|        **IterateArray** |  **4096** |  **2,229.29 ns** |     **51.38 ns** |    **150.68 ns** |  **2,300.0 ns** |  **1.00** |    **0.00** |
| IterateArrayWithPtr |  4096 |  2,263.33 ns |     48.00 ns |     71.84 ns |  2,250.0 ns |  1.03 |    0.07 |
|                     |       |              |              |              |             |       |         |
|        **IterateArray** |  **8192** |  **4,237.89 ns** |    **107.75 ns** |    **309.14 ns** |  **4,400.0 ns** |  **1.00** |    **0.00** |
| IterateArrayWithPtr |  8192 |  4,380.00 ns |     82.81 ns |     77.46 ns |  4,400.0 ns |  1.03 |    0.07 |
|                     |       |              |              |              |             |       |         |
|        **IterateArray** | **16384** |  **8,397.92 ns** |    **206.86 ns** |    **596.83 ns** |  **8,700.0 ns** |  **1.00** |    **0.00** |
| IterateArrayWithPtr | 16384 |  8,780.77 ns |    176.57 ns |    241.69 ns |  8,900.0 ns |  1.05 |    0.07 |
|                     |       |              |              |              |             |       |         |
|        **IterateArray** | **32768** | **17,067.68 ns** |    **416.37 ns** |  **1,221.14 ns** | **17,500.0 ns** |  **1.00** |    **0.00** |
| IterateArrayWithPtr | 32768 | 17,266.67 ns |    342.07 ns |    319.97 ns | 17,400.0 ns |  1.03 |    0.09 |
|                     |       |              |              |              |             |       |         |
|        **IterateArray** | **65536** | **75,184.00 ns** | **14,034.38 ns** | **41,380.68 ns** | **84,750.0 ns** |  **1.00** |    **0.00** |
| IterateArrayWithPtr | 65536 | 77,608.00 ns | 14,190.70 ns | 41,841.60 ns | 93,250.0 ns |  1.06 |    0.19 |


## Conculsion
Most likely there is more to it, and next I intend to take a look at boundary checks of .Net - how and when the JITter removes/optimizes them (which *should* happen in some cases, according to SO answers like [such as this one](https://stackoverflow.com/a/29269531/320103)). Is it possible that the boundary checks were optimized-away in this case? After some digging, I read through some code in .Net Core sources and found that the JIT optimizes array boundary checks when it can determine *statically* if the indexer *can* go out of boundaries of the array.
I expanded the existing test, adding some code to fool the JIT into *not* removing the boundary checks.
> If you are curious to read more about the magic CLR arrays do behind the scenes, take a look at [Matt Warren's awesome blog post](https://mattwarren.org/2017/05/08/Arrays-and-the-CLR-a-Very-Special-Relationship/)

## Second Test
```cs
[DisassemblyDiagnoser(printSource: true, printDiff: true, printIL: true)]
[HardwareCounters(HardwareCounter.CacheMisses, HardwareCounter.TotalCycles)]
public class Program
{
    [Params(64, 512, 1024, 4 * 1024, 8 * 1024, 16 * 1024, 32 * 1024, 64 * 1024)]
    public int Size { get; set; }

    private long[] Array;

    [IterationSetup]
    public void Init() => Array = new long[Size];

    [MethodImpl(MethodImplOptions.NoInlining)]
    public int GetSize() => Size;

    [Benchmark(Baseline = true)]
    public void IterateArray()
    {
        for (int i = 0; i < Size; i++)
            Array[i] = i;
    }

//this is the new test method
    [Benchmark()]
    public void IterateArrayWithGetSize()
    {
        //try and fool the JIT into NOT removing array boundary checks
        for (int i = 0; i < GetSize(); i++)
            Array[i] = i;
    }

    [Benchmark]
    public unsafe void IterateArrayWithPtr()
    {
        fixed (long* ptr = Array)
        {
            for (int i = 0; i < Size; i++)
                *(ptr + i) = i; 
        }
    }

    static void Main(string[] args) => BenchmarkRunner.Run<Program>();
}
```

## The Results
The expanded test yielded results which made much more sense. With boundary check, array iteration was noticeably slower, as the results show.

|                  Method |  Size |          Mean |         Error |        StdDev |       Median | Ratio | RatioSD | CacheMisses/Op | TotalCycles/Op |
|------------------------ |------ |--------------:|--------------:|--------------:|-------------:|------:|--------:|---------------:|---------------:|
|            **IterateArray** |    **64** |      **65.00 ns** |     **19.507 ns** |     **57.516 ns** |     **100.0 ns** |     **?** |       **?** |            **318** |              **-** |
| IterateArrayWithGetSize |    64 |     374.07 ns |     16.738 ns |     44.096 ns |     400.0 ns |     ? |       ? |            930 |        438,272 |
|     IterateArrayWithPtr |    64 |     100.00 ns |      0.000 ns |      0.000 ns |     100.0 ns |     ? |       ? |          1,170 |        994,093 |
|                         |       |               |               |               |              |       |         |                |                |
|            **IterateArray** |   **512** |     **332.97 ns** |     **26.074 ns** |     **73.113 ns** |     **300.0 ns** |  **1.00** |    **0.00** |          **1,187** |        **815,994** |
| IterateArrayWithGetSize |   512 |   1,927.50 ns |     40.263 ns |     71.567 ns |   1,900.0 ns |  5.86 |    1.29 |          1,187 |      1,076,951 |
|     IterateArrayWithPtr |   512 |     356.82 ns |     32.935 ns |     90.713 ns |     350.0 ns |  1.14 |    0.42 |          1,108 |      1,007,022 |
|                         |       |               |               |               |              |       |         |                |                |
|            **IterateArray** |  **1024** |     **783.33 ns** |     **49.699 ns** |    **143.392 ns** |     **700.0 ns** |  **1.00** |    **0.00** |          **1,130** |        **996,559** |
| IterateArrayWithGetSize |  1024 |   3,650.00 ns |     79.925 ns |    124.434 ns |   3,600.0 ns |  4.82 |    0.77 |          1,084 |      1,111,562 |
|     IterateArrayWithPtr |  1024 |     721.43 ns |     35.653 ns |     95.780 ns |     700.0 ns |  0.94 |    0.21 |          1,039 |        936,426 |
|                         |       |               |               |               |              |       |         |                |                |
|            **IterateArray** |  **4096** |   **2,526.97 ns** |    **141.950 ns** |    **393.343 ns** |   **2,400.0 ns** |  **1.00** |    **0.00** |          **1,193** |      **1,111,905** |
| IterateArrayWithGetSize |  4096 |  14,144.00 ns |    279.512 ns |    373.140 ns |  14,100.0 ns |  5.75 |    0.78 |          2,424 |      1,127,905 |
|     IterateArrayWithPtr |  4096 |   2,632.95 ns |    164.844 ns |    454.030 ns |   2,500.0 ns |  1.07 |    0.25 |          1,212 |      1,135,762 |
|                         |       |               |               |               |              |       |         |                |                |
|            **IterateArray** |  **8192** |   **5,022.09 ns** |    **259.323 ns** |    **705.508 ns** |   **4,800.0 ns** |  **1.00** |    **0.00** |            **879** |        **403,886** |
| IterateArrayWithGetSize |  8192 |  27,765.22 ns |    556.883 ns |    704.278 ns |  27,700.0 ns |  5.55 |    0.72 |          1,451 |        853,976 |
|     IterateArrayWithPtr |  8192 |   5,030.43 ns |    310.589 ns |    876.022 ns |   4,600.0 ns |  1.01 |    0.23 |            883 |        450,018 |
|                         |       |               |               |               |              |       |         |                |                |
|            **IterateArray** | **16384** |   **8,931.40 ns** |    **351.262 ns** |    **955.635 ns** |   **9,100.0 ns** |  **1.00** |    **0.00** |          **1,701** |      **1,100,102** |
| IterateArrayWithGetSize | 16384 |  56,023.53 ns |    966.160 ns |    992.175 ns |  56,100.0 ns |  6.22 |    0.74 |          2,048 |      1,107,106 |
|     IterateArrayWithPtr | 16384 |   9,634.88 ns |    603.972 ns |  1,643.151 ns |   8,800.0 ns |  1.10 |    0.24 |          1,325 |      1,026,168 |
|                         |       |               |               |               |              |       |         |                |                |
|            **IterateArray** | **32768** |  **19,279.79 ns** |  **1,193.483 ns** |  **3,405.075 ns** |  **18,500.0 ns** |  **1.00** |    **0.00** |          **1,796** |        **894,502** |
| IterateArrayWithGetSize | 32768 | 113,073.33 ns |  2,689.117 ns |  7,496.183 ns | 110,750.0 ns |  6.06 |    0.98 |          1,326 |        413,320 |
|     IterateArrayWithPtr | 32768 |  18,307.14 ns |    367.433 ns |    671.871 ns |  17,950.0 ns |  1.01 |    0.14 |          1,491 |        604,697 |
|                         |       |               |               |               |              |       |         |                |                |
|            **IterateArray** | **65536** |  **81,684.00 ns** | **14,972.693 ns** | **44,147.324 ns** |  **99,700.0 ns** |  **1.00** |    **0.00** |          **2,771** |        **322,993** |
| IterateArrayWithGetSize | 65536 | 265,269.00 ns | 16,331.343 ns | 48,153.334 ns | 283,250.0 ns |  4.20 |    1.99 |          1,809 |        778,339 |
|     IterateArrayWithPtr | 65536 |  84,369.00 ns | 16,073.756 ns | 47,393.832 ns |  92,150.0 ns |  1.07 |    0.31 |          2,583 |        296,627 |


## (Another) Conclusion
The results from the second, expanded test make much more sense; overall, delving this deep into C# internals was an awesome experience :)