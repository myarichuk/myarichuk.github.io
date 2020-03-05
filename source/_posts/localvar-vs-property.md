---
title: Local variables vs properties. No suprises here?
date: 2020-03-03T20:02:14+02:00
tags:
  - C++
  - C#
  - Performance
  - Optimization
categories:
  - Programming
  - C++
author: Michael Yarichuk
top_img: benchmarkdotnet.png
cover: /2020/03/03/localvar-vs-property/code_analysis.jpg
---
Can the benchmark that compares array iteration vs. pointer based iteration be optimized further? Yep.
In a post I wrote earlier about [performance comparison between array access with pointers and the usual C#'s  way](http://www.graymatterdeveloper.com/2020/02/23/arrays-vs-pointers/), I saw an interesting comment that suggested a way to squeeze some more performance out of the scenario. 
Using a local variable instead of a call to a property (which is in fact a method call) made sense, though I wondered, just how *much* of a performance boost it would provide.
So I decided to test it. The results turned out to be very... interesting. But before taking a look at the results, here is the test code: 

```cs
public class Program
{
    [Params(512, 1024, 4 * 1024, 8 * 1024, 16 * 1024, 32 * 1024, 64 * 1024)]
    public int Size { get; set; }

    private long[] Array;

    [IterationSetup]
    public void Init() => Array = new long[Size];

    [MethodImpl(MethodImplOptions.NoInlining)]
    public int GetSize() => Size;

//new test method
    [Benchmark(Baseline = true)]
    public void IterateArrayWithLocalVar()
    {
        var size = Size;
        for (int i = 0; i < size; i++)
            Array[i] = i;
    }

    [Benchmark]
    public void IterateArray()
    {
        for (int i = 0; i < Size; i++)
            Array[i] = i;
    }

    [Benchmark]
    public void IterateArrayWithBoundaryChecks()
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

// new test method
    [Benchmark]
    public unsafe void IterateArrayWithPtrLocalVar()
    {
        fixed (long* ptr = Array)
        {
            var size = Size;
            for (int i = 0; i < size; i++)
                *(ptr + i) = i;
        }
    }

    static void Main(string[] args) => BenchmarkRunner.Run<Program>();
}
```

Here are the results of the test:

|                         Method |  Size |         Mean |        Error |       StdDev |       Median | Ratio | RatioSD |
|------------------------------- |------ |-------------:|-------------:|-------------:|-------------:|------:|--------:|
|       **IterateArrayWithLocalVar** |   **256** |     **215.6 ns** |     **30.72 ns** |     **88.65 ns** |     **200.0 ns** |     **?** |       **?** |
|                   IterateArray |   256 |     200.0 ns |      0.00 ns |      0.00 ns |     200.0 ns |     ? |       ? |
| IterateArrayWithBoundaryChecks |   256 |   1,081.2 ns |     26.49 ns |     61.40 ns |   1,050.0 ns |     ? |       ? |
|            IterateArrayWithPtr |   256 |     243.3 ns |     23.29 ns |     67.56 ns |     200.0 ns |     ? |       ? |
|    IterateArrayWithPtrLocalVar |   256 |     214.3 ns |     20.88 ns |     60.92 ns |     200.0 ns |     ? |       ? |
|                                |       |              |              |              |              |       |         |
|       **IterateArrayWithLocalVar** |   **512** |     **272.8 ns** |     **16.99 ns** |     **44.76 ns** |     **300.0 ns** |  **1.00** |    **0.00** |
|                   IterateArray |   512 |     342.6 ns |     21.50 ns |     61.33 ns |     300.0 ns |  1.32 |    0.37 |
| IterateArrayWithBoundaryChecks |   512 |   4,942.9 ns |     96.07 ns |     85.16 ns |   4,950.0 ns | 19.43 |    4.17 |
|            IterateArrayWithPtr |   512 |     372.8 ns |     20.43 ns |     57.61 ns |     400.0 ns |  1.42 |    0.37 |
|    IterateArrayWithPtrLocalVar |   512 |     367.7 ns |     22.20 ns |     64.06 ns |     400.0 ns |  1.40 |    0.38 |
|                                |       |              |              |              |              |       |         |
|       **IterateArrayWithLocalVar** |  **1024** |     **558.3 ns** |     **22.30 ns** |     **64.35 ns** |     **600.0 ns** |  **1.00** |    **0.00** |
|                   IterateArray |  1024 |     578.2 ns |     16.10 ns |     41.55 ns |     600.0 ns |  1.05 |    0.14 |
| IterateArrayWithBoundaryChecks |  1024 |   4,037.5 ns |     84.28 ns |    109.59 ns |   4,000.0 ns |  7.36 |    0.79 |
|            IterateArrayWithPtr |  1024 |     313.6 ns |     27.62 ns |     76.08 ns |     300.0 ns |  0.57 |    0.16 |
|    IterateArrayWithPtrLocalVar |  1024 |     667.1 ns |     18.58 ns |     47.30 ns |     700.0 ns |  1.21 |    0.16 |
|                                |       |              |              |              |              |       |         |
|       **IterateArrayWithLocalVar** |  **4096** |   **2,118.9 ns** |     **45.70 ns** |     **77.60 ns** |   **2,100.0 ns** |  **1.00** |    **0.00** |
|                   IterateArray |  4096 |   1,935.2 ns |     51.41 ns |    108.43 ns |   1,900.0 ns |  0.91 |    0.08 |
| IterateArrayWithBoundaryChecks |  4096 |  15,557.1 ns |    308.82 ns |    273.76 ns |  15,400.0 ns |  7.27 |    0.31 |
|            IterateArrayWithPtr |  4096 |   2,475.7 ns |     52.68 ns |     89.46 ns |   2,500.0 ns |  1.17 |    0.05 |
|    IterateArrayWithPtrLocalVar |  4096 |   2,413.8 ns |     53.86 ns |     78.94 ns |   2,400.0 ns |  1.14 |    0.06 |
|                                |       |              |              |              |              |       |         |
|       **IterateArrayWithLocalVar** |  **8192** |   **4,050.0 ns** |     **84.81 ns** |     **90.75 ns** |   **4,050.0 ns** |  **1.00** |    **0.00** |
|                   IterateArray |  8192 |   9,935.7 ns |    152.36 ns |    135.06 ns |   9,950.0 ns |  2.46 |    0.08 |
| IterateArrayWithBoundaryChecks |  8192 |  33,253.3 ns |  3,421.63 ns |  5,121.34 ns |  31,250.0 ns |  8.25 |    1.28 |
|            IterateArrayWithPtr |  8192 |   4,780.8 ns |     75.50 ns |     63.04 ns |   4,750.0 ns |  1.18 |    0.04 |
|    IterateArrayWithPtrLocalVar |  8192 |   4,813.6 ns |    101.79 ns |    191.19 ns |   4,800.0 ns |  1.18 |    0.06 |
|                                |       |              |              |              |              |       |         |
|       **IterateArrayWithLocalVar** | **16384** |   **8,158.3 ns** |    **149.15 ns** |    **116.45 ns** |   **8,100.0 ns** |  **1.00** |    **0.00** |
|                   IterateArray | 16384 |   8,200.0 ns |    166.60 ns |    155.84 ns |   8,200.0 ns |  1.01 |    0.02 |
| IterateArrayWithBoundaryChecks | 16384 |  69,796.8 ns |  4,058.70 ns | 11,579.70 ns |  62,650.0 ns |  8.56 |    1.31 |
|            IterateArrayWithPtr | 16384 |  10,777.0 ns |  1,089.97 ns |  2,460.24 ns |   9,700.0 ns |  1.27 |    0.27 |
|    IterateArrayWithPtrLocalVar | 16384 |  12,319.8 ns |  1,417.59 ns |  4,090.08 ns |   9,700.0 ns |  1.49 |    0.60 |
|                                |       |              |              |              |              |       |         |
|       **IterateArrayWithLocalVar** | **32768** |  **16,333.3 ns** |    **443.18 ns** |    **527.57 ns** |  **16,300.0 ns** |  **1.00** |    **0.00** |
|                   IterateArray | 32768 |  16,937.5 ns |  1,494.17 ns |  1,942.84 ns |  16,500.0 ns |  1.04 |    0.13 |
| IterateArrayWithBoundaryChecks | 32768 | 187,506.0 ns | 26,906.30 ns | 79,333.84 ns | 144,100.0 ns | 19.13 |    1.95 |
|            IterateArrayWithPtr | 32768 |  24,892.8 ns |  2,918.12 ns |  8,465.99 ns |  19,400.0 ns |  1.64 |    0.58 |
|    IterateArrayWithPtrLocalVar | 32768 |  18,850.0 ns |    338.88 ns |    264.58 ns |  18,700.0 ns |  1.15 |    0.05 |
|                                |       |              |              |              |              |       |         |
|       **IterateArrayWithLocalVar** | **65536** |  **83,671.0 ns** | **16,469.96 ns** | **48,562.04 ns** |  **91,300.0 ns** |  **1.00** |    **0.00** |
|                   IterateArray | 65536 | 131,365.0 ns | 28,646.45 ns | 84,464.72 ns | 104,050.0 ns |  1.68 |    0.75 |
| IterateArrayWithBoundaryChecks | 65536 | 328,035.1 ns | 20,180.46 ns | 58,547.17 ns | 328,400.0 ns |  5.68 |    3.60 |
|            IterateArrayWithPtr | 65536 |  89,072.4 ns | 17,466.58 ns | 50,950.85 ns |  75,300.0 ns |  1.16 |    0.48 |
|    IterateArrayWithPtrLocalVar | 65536 |  83,553.0 ns | 15,204.40 ns | 44,830.51 ns |  96,700.0 ns |  1.08 |    0.33 |

Tsing a local variable for the ``for`` loop *tends* to be faster, but not by much. Single-digit percentages at best. Notice that it doesn't happen in all cases. Test case for 256 items is too fast for accurate measurement, so we can discard it. There is also a couple of cases where usage of local variable is less performant than using a property, which is a bit surprising and probably warrants some more investigation - I'd expect the results to be more consistent.