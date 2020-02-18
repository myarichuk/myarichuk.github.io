---
title: Sequential memory access is... faster?
date: 2020-02-17T19:31:11+02:00
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
cover: https://upload.wikimedia.org/wikipedia/en/0/08/CachePrefetching_StreamBuffers.png
---
In gamedev articles about Entity-Component-System, data locality is often mentioned as a big reason to use such design pattern. The underlying data structures of the ECS are cache friendly, thus allowing much better performance for iterations of large amount of game objects.
I knew that cache-friendly usage of memory (sequential memory access, for example) would yield better performance, but I was curious, how much better it would be?
>In case you never heard about Entity-Component-System, [this article](http://gameprogrammingpatterns.com/data-locality.html) is a good place to read about it.

## The Tests

In order to test that, I decided to benchmark an iteration of 2d array, using C# and C++, using [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) and [Google Benchmark](https://github.com/google/benchmark) respectively.

In C# (running .Net Core 3.1), I used the following test:
```cs
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

namespace NetCoreDataLocality
{
    [HardwareCounters(HardwareCounter.CacheMisses)]
    public class Program
    {
        [Params(16, 512, 4096, 16 * 1024)]
        public int Size { get; set; }

        private long[,] Array2D;

        [IterationSetup]
        public void Init() => Array2D = new long[Size, Size];

        [Benchmark]
        public void Iterate2DArrayColumnFirst()
        {
            for (int y = 0; y < Size; y++)
            for (int x = 0; x < Size; x++)
                Array2D[x, y] = x + y;
        }

        [Benchmark]
        public void Iterate2DArrayRowFirst()
        {
            for (int x = 0; x < Size; x++)
            for (int y = 0; y < Size; y++)
                Array2D[x, y] = x + y;
        }

        static void Main(string[] args) => BenchmarkRunner.Run<Program>();
    }
}

```

And in C++, I used code as close as possible to C# - I was also curious how the two languages performance would compare
```cpp
#include <benchmark/benchmark.h>
#include <array>

using namespace std;

class array2d_benchmark
{
private:
	shared_ptr<vector<vector<unsigned long long>>> array2d;
	int _array_size;
public:
	array2d_benchmark(int array_size) : _array_size(array_size)
	{		
		array2d = make_shared<vector<vector<unsigned long long>>>(array_size , vector<unsigned long long> (array_size, 0));
	}
	
	void benchmark_column_first() const
	{
		for(unsigned long long y = 0; y < _array_size; y++)
		for(unsigned long long x = 0; x < _array_size; x++)
			(*array2d)[x][y] = x + y;
	}

	void benchmark_row_first() const
	{
		for(unsigned long long x = 0; x < _array_size; x++)
		for(unsigned long long y = 0; y < _array_size; y++)
			(*array2d)[x][y] = x + y;
	}
};


static void benchmark_array2d_column_first(benchmark::State& state) {
  const array2d_benchmark benchmark(state.range_x());
  for (auto _ : state) {
    benchmark.benchmark_column_first();
  }
}

static void benchmark_array2d_row_first(benchmark::State& state) {
  const array2d_benchmark benchmark(state.range_x());
  for (auto _ : state) {
    benchmark.benchmark_row_first();
  }
}
//16, 512, 4096, 16 * 1024
BENCHMARK(benchmark_array2d_column_first)->Arg(16);
BENCHMARK(benchmark_array2d_column_first)->Arg(512);
BENCHMARK(benchmark_array2d_column_first)->Arg(4096);
BENCHMARK(benchmark_array2d_column_first)->Arg(16 * 1024);

BENCHMARK(benchmark_array2d_row_first)->Arg(16);
BENCHMARK(benchmark_array2d_row_first)->Arg(512);
BENCHMARK(benchmark_array2d_row_first)->Arg(4096);
BENCHMARK(benchmark_array2d_row_first)->Arg(16 * 1024);

BENCHMARK_MAIN();

```

## The Results
I got... interesting results.

For C#, I got:
```results
|                    Method |  Size |               Mean |             Error |            StdDev |             Median | CacheMisses/Op |
|-------------------------- |------ |-------------------:|------------------:|------------------:|-------------------:|---------------:|
| Iterate2DArrayColumnFirst |    16 |           469.1 ns |          18.64 ns |          49.10 ns |           500.0 ns |            340 |
|    Iterate2DArrayRowFirst |    16 |           482.0 ns |          26.13 ns |          77.04 ns |           500.0 ns |            455 |
| Iterate2DArrayColumnFirst |   512 |     1,218,714.3 ns |     112,811.61 ns |     329,076.75 ns |     1,156,900.0 ns |         12,545 |
|    Iterate2DArrayRowFirst |   512 |       583,742.0 ns |      61,619.35 ns |     181,686.04 ns |       636,450.0 ns |          2,157 |
| Iterate2DArrayColumnFirst |  4096 |   197,656,800.0 ns |   3,124,528.79 ns |   2,922,686.17 ns |   198,075,200.0 ns |      6,301,816 |
|    Iterate2DArrayRowFirst |  4096 |    54,081,651.9 ns |   1,063,472.59 ns |   2,196,255.32 ns |    53,712,750.0 ns |        259,819 |
| Iterate2DArrayColumnFirst | 16384 | 5,995,155,895.9 ns | 324,258,680.47 ns | 945,877,790.80 ns | 5,567,680,450.0 ns |    150,143,526 |
|    Iterate2DArrayRowFirst | 16384 |   820,319,993.3 ns |   5,415,915.82 ns |   5,066,051.02 ns |   818,728,000.0 ns |      3,783,339 |
```
>Notice the the last ``CacheMisses/Op`` column - it highlights the effect of data locality on performance

For C++, I got:
```results
----------------------------------------------------------------------------
Benchmark                                     Time           CPU Iterations
----------------------------------------------------------------------------
benchmark_array2d_column_first/16           234 ns        231 ns    3446154
benchmark_array2d_column_first/512       496207 ns     488281 ns       1120
benchmark_array2d_column_first/4096   181828000 ns  179687500 ns          4
benchmark_array2d_column_first/16384 4868694600 ns 4859375000 ns          1
benchmark_array2d_row_first/16              210 ns        210 ns    3200000
benchmark_array2d_row_first/512          197220 ns     196725 ns       3733
benchmark_array2d_row_first/4096       13954918 ns   13750000 ns         50
benchmark_array2d_row_first/16384     213235700 ns  213541667 ns          3
```

Unsurprisingly, for "row-first" iteration I got much better results, because the memory access in this case is sequential, thus allowing much less cache misses, as can be seen from C# benchmark (really awesome feature of BenchmarkDotNet!)
What *did* surprise me is how much faster the sequential memory access in fact is, even for such a simple use-case. For 16384x16384 arrays, in C# it is x7 running time improvement and for C++ it is approximately x22 improvement!
  
Also, the run-time difference between C++ and C# in case of row-first iteration for 16384x16384 arrays is almost x3 - much more than I expected. Overall, this was an interesting experiment that proved to me the value of Entity-Component-System as a performance optimization.