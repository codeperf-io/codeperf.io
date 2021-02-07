---
layout:	post
title:	"Hands-on profile driven redudant computation removal"
date:	2021-02-07
---
r
![](/img/perf-hdr_value_at_percentile_chart.jpeg)
  
# Introduction

As you will see during this post, even highly optimized software is very likely to perform “redundant” computation during single or consecutive executions. 
This is either due to way we've designed it (algorithms) or the nature of the values assumed as inputs (dataset). 
Therefore, to fully understand if removing redudant computation represents an opportunity for improving the efficiency of our programs we need to:
- have visibility to where threads are spending CPU cycles while running on-CPU, and wether those cycles are effectively being used for computation.
- understand if we can reuse that same computation as a mean for improving performance -- and this can only be made with a deep understand of the domain and problems our program is solving ans how is it being used.

This post idea came after I've started a POC on my Redis fork [(link)](https://github.com/redis/redis/compare/unstable...filipecosta90:latencystats.per.cat) to extended the current latency metrics Redis Provides. To do so I'm using the C version of the [HdrHistogram](https://github.com/HdrHistogram/HdrHistogram_c) to have a per command histogram and calculate both the cumulative latency distribution and precompute some "commonly used" percentiles used for doing latency analysis. If you haven't yet come across this type of highly optimized skecthing data-structure I suggest you look into one of the many Gil Tene's excelent presentations on [youtube](https://www.youtube.com/watch?v=lJ8ydIuPFeU).

Now getting back to our problem, and thinking on the way we do latency percentile analysis, we normally require more than one percentile to be calculated for the given histogram at any given moment (example of p50, p95, p99, p999). 

If you look at the C function that provides the percentile calculation implementation you'll notice that we iterate over the counts array starting from 0 up until the we've reached a cumulative count of samples that represents the requested percentile. 

```c
int64_t hdr_value_at_percentile(const struct hdr_histogram* h, double percentile)
{
    struct hdr_iter iter;
    int64_t total = 0;
    double requested_percentile = percentile < 100.0 ? percentile : 100.0;
    int64_t count_at_percentile =
        (int64_t) (((requested_percentile / 100) * h->total_count) + 0.5);
    count_at_percentile = count_at_percentile > 1 ? count_at_percentile : 1;

    hdr_iter_init(&iter, h);

    while (hdr_iter_next(&iter))
    {
        total += iter.count;

        if (total >= count_at_percentile)
        {
            int64_t value_from_index = iter.value;
            return highest_equivalent_value(h, value_from_index);
        }
    }

    return 0;
}
```

## Defining the experimental setup and getting a grasp of on-cpu time

To realize computation reuse, it is necessary to capture run-time behavior of a program.
Given the above function definition we should start by preparing an example that will allow us to profile code execution and determine which functions are consuming the most time and thus are targets for optimization. 


To do so, we'll use Google's microbenchmarking framework ([google/benchmark](https://github.com/google/benchmark)), that the C version of the HdrHistogram already incomporates, to simulate a dataset representative of real-life scenarios ( random deterministic gamma distribution to mimic latencies ) and calculate 4 very common percetiles used on any analysis (p50,p95,p99,p999).

The bellow micro-benchmark can be consulted and directly compiled from the following [GH repo](https://github.com/RedisBloom/HdrHistogram_c/blob/value_at_percentiles/test/hdr_histogram_benchmark.cpp#L87).

```c++
static void BM_hdr_value_at_percentile_given_array(benchmark::State &state) {
  /////////////////////
  // benchmark setup //
  /////////////////////
  srand(12345);
  const int64_t precision = state.range(0);
  const int64_t max_value = state.range(1);
  const double percentile_list[4] = {50.0, 95.0, 99.0, 99.9};
  std::default_random_engine generator;
  // gama distribution shape 1 scale 100000
  std::gamma_distribution<double> latency_gamma_dist(1.0, 100000);
  struct hdr_histogram *histogram;
  hdr_init(min_value, max_value, precision, &histogram);
  for (int64_t i = 1; i < generated_datapoints; i++) {
    int64_t number = int64_t(latency_gamma_dist(generator)) + 1;
    number = number > max_value ? max_value : number;
    hdr_record_value(histogram, number);
  }
  benchmark::DoNotOptimize(histogram->counts);
  int64_t items_processed = 0;
  ///////////////////
  // benchmark run //
  ///////////////////
  for (auto _ : state) {
    for (auto percentile : percentile_list) {
      benchmark::DoNotOptimize(hdr_value_at_percentile(histogram, percentile));
      // read/write barrier
      benchmark::ClobberMemory();
    }
    items_processed += 4;
  }
  state.SetItemsProcessed(items_processed);
}
```

To clone the HdrHistogram repo and build the microbenchmark locallly do as follows:
```
git clone https://github.com/RedisBloom/HdrHistogram_c
git checkout value_at_percentiles
mkdir build && cd build
CC=gcc-10 CXX=g++-10 cmake -DHDR_HISTOGRAM_BUILD_BENCHMARK=ON -DCMAKE_BUILD_TYPE=ReleaseWithDebugInfo ..
make
```

To run it:
```

~/HdrHistogram_c/build$ ./test/hdr_histogram_benchmark --benchmark_filter=BM_hdr_value_at_percentile_given_array/3/86400000000 --benchmark_min_time=60
2021-02-07 14:54:39
Running ./test/hdr_histogram_benchmark
Run on (40 X 1240.4 MHz CPU s)
CPU Caches:
  L1 Data 32K (x20)
  L1 Instruction 32K (x20)
  L2 Unified 1024K (x20)
  L3 Unified 28160K (x1)
Load Average: 0.02, 0.16, 0.20
---------------------------------------------------------------------------------------------------------------
Benchmark                                                     Time             CPU   Iterations UserCounters...
---------------------------------------------------------------------------------------------------------------
BM_hdr_value_at_percentile_given_array/3/86400000000     344046 ns       344042 ns       244556 items_per_second=11.6265k/s
```

## Where are those CPU cycles being spent
As seen above, it takes us 344 microseconds of CPU time to compute the 4 percentiles, leading to a max of approximately 11.6K calculations per second. It is now time to understand where are those CPU cycles being spent on the above defined function definition. To do so we'll collect, report and visualize hotspots using perf. There are severall tools that can be used to achieve the same outcome like bcc/BPF tracing tools, or Intel Vtune Profiler, among others...

The following steps rely uppon Linux perf_events (aka ["perf"](https://man7.org/linux/man-pages/man1/perf.1.html)). I'll assume beforehand you have installed the perf tool on your system. Most Linux distributions will likely package this as a package related to the kernel. More information about the perf tool can be found at perf [wiki](https://perf.wiki.kernel.org/).

### Profiling build prerequisites

For a proper On-CPU analysis of our example we're
required that stack traces to be available to tracers. 

By default our example is compiled with the `-O2` switch ( which we intent to keep during profiling). This means that compiler
optimizations are enabled. 

Many compilers ommit the frame pointer as a way of 
runtime optimization ( saving a register ), thus breaking frame pointer-based 
stack walking. This makes the executable faster, but at the
same time it makes it (like any other program) harder to trace, potentially 
wrongfully pinpointing on-CPU time to the last available frame pointer of a call
 stack that can get a lot deeper ( but impossible to trace ).

It's important that you ensure:
- debug information is present: compile option `-g`
- frame pointer register is present: `-fno-omit-frame-pointer`
- we still run with optimizations to get an accurate representation of production run times, meaning we will keep: `-O2`

To do, and within the project's `build` folder, configure and build the project as follows:
```
CXXFLAGS="-g -fno-omit-frame-pointer -O2" CFLAGS="-g -fno-omit-frame-pointer -O2" CC=gcc-10 CXX=g++-10 cmake -DHDR_HISTOGRAM_BUILD_BENCHMARK=ON -DCMAKE_BUILD_TYPE=ReleaseWithDebugInfo ..
```

## Sampling stack traces using perf

Now that we have our executable properly build for profiling, and to profile both user and kernel-level stacks for a duration of 60 seconds, at an sampling frequency of 999 samples per second run the following command:

```
~/HdrHistogram_c/build$ sudo perf record -g -F 999 ./test/hdr_histogram_benchmark --benchmark_filter=BM_hdr_value_at_percentile_given_array/3/86400000000 --benchmark_min_time=60
2021-02-07 14:54:39
Running ./test/hdr_histogram_benchmark
Run on (40 X 1240.4 MHz CPU s)
CPU Caches:
  L1 Data 32K (x20)
  L1 Instruction 32K (x20)
  L2 Unified 1024K (x20)
  L3 Unified 28160K (x1)
Load Average: 0.02, 0.16, 0.20
***WARNING*** Library was built as DEBUG. Timings may be affected.
---------------------------------------------------------------------------------------------------------------
Benchmark                                                     Time             CPU   Iterations UserCounters...
---------------------------------------------------------------------------------------------------------------
BM_hdr_value_at_percentile_given_array/3/86400000000     344046 ns       344042 ns       244556 items_per_second=11.6265k/s
[ perf record: Woken up 91 times to write data ]
[ perf record: Captured and wrote 22.751 MB perf.data (126180 samples) ]
```

### Displaying the recorded profile information using perf report

By default perf record will generate a perf.data file in the current working directory. 

You can then report with a call-graph output (call chain, stack backtrace), with a minimum call graph inclusion threshold of 0.5%, with:

```
perf report -g "graph,0.5,caller"
```

Note: See the [perf report](https://man7.org/linux/man-pages/man1/perf-report.1.html) documention for advanced filtering, sorting and aggregation capabilities.


If we filter by the `hdr_value_at_percentile` symbol and annotate it we'll notice that 99% of the CPU cycles of `hdr_value_at_percentile` are spent on the iterator calculating the cumulative count up until we reach or surpass the count that represents the percentile.

![](/img/perf-hdr_value_at_percentile.png)
![](/img/perf-hdr_value_at_percentile-annotated.png)


This means that if we want to compute p50 and p99, we could re-use the already precomputed comulative count that gives us the p50 and start from that value ( instead of starting again at 0 ) for calculating the p99, thus eliminating the redundant computation. 

## Adding the value_at_percentiles API

As seen above, when we want to compute multiple percentiles for a given histogram, we've identified a reusable partial results in `hdr_value_at_percentile` that is also where threads are spending most CPU cycles while running on-CPU. 

With that in mind, by adding a new API with the following signature and implementation, and assuming the user will follow the API pre-requesites (sorted percentiles array input) we should be able to deeply reduce redundant work and improve the overall function performance.

### new api function signature
```c
/**
 * Get the values at the given percentiles.
 *
 * @param h "This" pointer.
 * @param percentiles The ordered percentiles array to get the values for.
 * @param N Number of elements in the arrays.
 * @param values Destination array containg the values at the given percentiles.
 * @return 0 on success, ENOMEM if malloc failed.
 */
int hdr_value_at_percentiles(const struct hdr_histogram* h, const double* percentiles, const size_t N, int64_t** values);
```

### new api function definition
```c
int hdr_value_at_percentiles(const struct hdr_histogram* h, const double* percentiles, const size_t N, int64_t** values)
{
    *values = (int64_t*) calloc((size_t) N, sizeof(int64_t));
    if (!*values)
    {
        return ENOMEM;
    }
    int64_t* count_at_percentiles = (int64_t*) calloc((size_t) N, sizeof(int64_t));
    if (!count_at_percentiles)
    {
        free(*values);
        return ENOMEM;
    }
    struct hdr_iter iter;
    const int64_t total_count = h->total_count;
    for (size_t i = 0; i < N; i++)
    {
        const double requested_percentile = percentiles[i] < 100.0 ? percentiles[i] : 100.0;
        int64_t count_at_percentile =
        (int64_t) (((requested_percentile / 100) * total_count) + 0.5);
        count_at_percentiles[i] = count_at_percentile > 1 ? count_at_percentile : 1;
    }

    hdr_iter_init(&iter, h);
    int64_t total = 0;
    size_t at_pos = 0;
    while (hdr_iter_next(&iter) && at_pos < N)
    {
        total += iter.count;
        while (total >= count_at_percentiles[at_pos] && at_pos < N)
        {
            (*values)[at_pos] = highest_equivalent_value(h, iter.value);
            at_pos++;
        }
    }
    free(count_at_percentiles);
    return 0;
}
```

### benchmarking the new API

Now that we've defined the new API, using the same setup code as the above microbenchmark, we should be able to deterministically and properly compare the optimization provided by eliminating the redundant computation.

New microbenchmark:

```c
static void BM_hdr_value_at_percentile_given_array(benchmark::State &state) {
  /////////////////////
  // benchmark setup //
  /////////////////////
  srand(12345);
  const int64_t precision = state.range(0);
  const int64_t max_value = state.range(1);
  const double percentile_list[4] = {50.0, 95.0, 99.0, 99.9};
  std::default_random_engine generator;
  // gama distribution shape 1 scale 100000
  std::gamma_distribution<double> latency_gamma_dist(1.0, 100000);
  struct hdr_histogram *histogram;
  hdr_init(min_value, max_value, precision, &histogram);
  for (int64_t i = 1; i < generated_datapoints; i++) {
    int64_t number = int64_t(latency_gamma_dist(generator)) + 1;
    number = number > max_value ? max_value : number;
    hdr_record_value(histogram, number);
  }
  benchmark::DoNotOptimize(histogram->counts);
  int64_t items_processed = 0;
  ///////////////////
  // benchmark run //
  ///////////////////
  for (auto _ : state) {
    for (auto percentile : percentile_list) {
      benchmark::DoNotOptimize(hdr_value_at_percentile(histogram, percentile));
      // read/write barrier
      benchmark::ClobberMemory();
    }
    items_processed += 4;
  }
  state.SetItemsProcessed(items_processed);
}
```

```
~/HdrHistogram_c/build$ ./test/hdr_histogram_benchmark --benchmark_filter=BM_hdr_value_at_percentile --benchmark_min_time=60
2021-02-07 16:55:23
Running ./test/hdr_histogram_benchmark
Run on (40 X 1227.32 MHz CPU s)
CPU Caches:
  L1 Data 32K (x20)
  L1 Instruction 32K (x20)
  L2 Unified 1024K (x20)
  L3 Unified 28160K (x1)
Load Average: 0.83, 0.37, 0.14
---------------------------------------------------------------------------------------------------------------
Benchmark                                                     Time             CPU   Iterations UserCounters...
---------------------------------------------------------------------------------------------------------------
BM_hdr_value_at_percentile_given_array/3/86400000        342337 ns       342335 ns       245263 items_per_second=11.6845k/s
BM_hdr_value_at_percentile_given_array/3/86400000000     343298 ns       343295 ns       245245 items_per_second=11.6518k/s
BM_hdr_value_at_percentile_given_array/4/86400000       3064091 ns      3064067 ns        27417 items_per_second=1.30545k/s
BM_hdr_value_at_percentile_given_array/4/86400000000    3062938 ns      3062914 ns        27379 items_per_second=1.30595k/s
BM_hdr_value_at_percentiles_given_array/3/86400000        107060 ns       107059 ns       786567 items_per_second=37.3625k/s
BM_hdr_value_at_percentiles_given_array/3/86400000000     107040 ns       107039 ns       783051 items_per_second=37.3694k/s
BM_hdr_value_at_percentiles_given_array/4/86400000       1043602 ns      1043592 ns        80462 items_per_second=3.83292k/s
BM_hdr_value_at_percentiles_given_array/4/86400000000    1043322 ns      1043312 ns        80458 items_per_second=3.83394k/s
```

As expected, by simply reusing intermediate computation and consequently reducing the redudant computation we've moved from taking 342 microseconds of CPU time to compute the 4 percentiles to 107 microseconds, and from a max of approximately 11.6K percentile calculations per second to 37.36K calculations per second, representing an 3.4X speedup ( dependant on both the number of percentiles to calculate and also the percentile and data distribution ).

The following chart visualize demonstrates the above optimization.

![](/img/perf-hdr_value_at_percentile_chart.jpeg)

#### Further notes

At the moment I wrote this blog we were discussing this optimization PRs I've opened both on the C and Go versions of the HdrHistogram.

Contributions and corrections to the above PRs are gratefully accepted. 

* [C version PR](https://github.com/HdrHistogram/HdrHistogram_c/pull/90)
  
* [Go version PR](https://github.com/HdrHistogram/hdrhistogram-go/pull/44)

#### Used platform details

The performance results provided in this blog were collected using:
- **Hardware platform**: A physical HPE ProLiant DL380 Gen10 Server, with one Intel(R) Xeon(R) Gold 6230 CPU @ 2.10GHz, Maximum memory capacity of 3 TB, disabling CPU Frequency Scaling and with all configurable BIOS and CPU system settings set to performance.
- **OS:** Ubuntu 18.04 Linux release 4.15.0-123. 
- **Installed Memory:** 64GB DDR4-SDRAM @ 2933 MHz
- **Compiler:** gcc-10 (Ubuntu 10.1.0-2ubuntu1~18.04) 10.1.0
