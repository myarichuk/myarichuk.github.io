---
title: arrays-vs-pointers
date: 2020-02-23T08:39:23+02:00
tags:
  - C#
  - Performance
  - Optimization
categories:
  - Programming
author: Michael Yarichuk
top_img: benchmarkdotnet.png
cover: /2020/02/23/arrays-vs-pointers/array.svg
---
After seeing the results of my [previous post](/2020/02/17/data-locality/) where I tested performance impact of data locality, one of my collegues theorized that perhaps the huge difference between C++ and C# performance (assuming adjacent memory iteration) is due to boundary checks of C# arrays - he said that if I would use pointers to access arrays, the code will be much faster.


## The test
I used the following test to try and see how much performance usage of pointers would yield.

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
The results were a bit surprising. Apparently, pointer access is not much faster (and sometimes a tiny bit slower!) than using arrays as they are.

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
Probably there is more to it, and next I intend to take a look at boundary checks of .Net - how and when the JITter removes/optimizes them (which *should* happen in some cases, according to SO answers like [such as this one](https://stackoverflow.com/a/29269531/320103))