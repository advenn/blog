---
date: '2025-11-23T23:02:39+01:00'
draft: false
title: 'Pandas vs Polars in Production: Performance Comparison'
tags: ['pandas', 'polars', 'data-processing', 'data-pipeline']
---

When performance bottlenecks started affecting my production data pipeline, I decided to test whether Polars could deliver on its performance promises. This is what I learned from migrating a real production workload from Pandas to Polars.

## The Workload

The application was a data aggregation service running as a Kubernetes pod with the following constraints:
- **Resources**: 2 CPUs, 3 GB RAM
- **Execution frequency**: Every 2-2.5 minutes
- **Data volume**: 5,000-7,000 rows × 100-150 columns per run
- **Operations**: Multiple database calls, API requests, DataFrame merges, arithmetic operations (additions, multiplications), and group-by aggregations
- **Web server**: FastAPI with Uvicorn handling production traffic

All operations were properly vectorized-no row-by-row iteration. The pipeline combined data from various sources into a single DataFrame, transformed it, and output the results.

## The Performance Problem

The performance gap between development and production was significant:
- **Local machine** (powerful CPU, 24+ GB RAM): 25-30 seconds per run
- **Production server** (2 CPUs, 3 GB RAM): ~60 seconds per run

The pipeline was CPU-bound. With limited CPU resources and single-threaded Pandas operations, processing time doubled in production.

## Why Polars?

After researching alternatives, I narrowed it down to Polars and DuckDB. Both had positive feedback in forums and discussions. I chose Polars because:

1. **Parallel execution**: Polars can utilize multiple CPU cores automatically
2. **Lazy evaluation**: Query optimization happens before execution
3. **Better memory efficiency**: More predictable memory usage during operations
4. **Rust foundation**: Compiled code with fewer runtime overheads

My initial concern was whether the migration would be worth it for a relatively small dataset (5-7k rows). Would Polars' benefits apply at this scale, or would the overhead negate any gains?

## The Refactoring Process

Translating the Pandas code to Polars took about a week. The main challenges:

**API Differences**

Polars uses its own API with different naming conventions. Pandas feels intuitive because it leverages Python's dunder methods for `get` and `set` operations. Polars requires a mental shift.

**Expression System**

Polars uses expressions instead of method chaining. For example:
```python
# Pandas
df['total'] = df['price'] * df['quantity']

# Polars
df = df.with_columns(
    (pl.col('price') * pl.col('quantity')).alias('total')
)
```

The expression system took getting used to, but I found it more readable once I understood the pattern. Expressions are also what enable Polars' query optimizer to work its magic.

**Migration Strategy**

I took a gradual approach:
1. Refactored the code locally
2. Deployed to staging for testing
3. Monitored for a few days
4. Deployed to production with close monitoring

## Results

The improvements were dramatic and immediate.

**Test Server Performance**

![CPU Usage on Test Server](/photos/cpu.jpg)

![RAM Usage on Test Server](/photos/ram.jpg)

**Production Performance**

![CPU Usage on Production](/photos/cpu-prod.jpg)

![RAM Usage on Production](/photos/ram-prod.jpg)

**Metrics Summary**

| Metric | Pandas | Polars | Improvement |
|--------|--------|--------|-------------|
| Local processing time | 25-30s | 6-10s | 65-75% faster |
| Production processing time | ~60s | 25-30s | 50% faster |
| CPU usage | High, spiky | Stable, lower | Significantly reduced |
| CPU utilization | Single-threaded | Multi-threaded | Better resource usage |

The most impressive result was production performance matching local development. Polars' parallel execution extracted value from the 2-CPU constraint that Pandas couldn't utilize.

**CPU Usage**

The CPU spikes disappeared. Polars distributed the workload across both cores, resulting in more stable and predictable resource consumption. This was critical in a Kubernetes environment where resource spikes could trigger throttling or affect other services.

## When to Choose Polars vs Pandas

**Choose Polars when:**
- You're processing medium to large datasets (even 5-10k rows with many columns benefits)
- CPU usage is a concern, especially with limited resources
- You need parallel execution and have multiple cores available
- Processing time directly impacts user experience or throughput
- You're building new pipelines and can invest time in learning the API

**Stick with Pandas when:**
- Quick exploratory data analysis in Jupyter notebooks
- Working with libraries that require Pandas DataFrames (many ML libraries still expect Pandas)
- Very small datasets where performance isn't critical
- Your team is already proficient in Pandas and migration cost outweighs benefits
- You need the broader ecosystem and community resources

## Key Takeaways

1. **Polars delivers on its performance promises** - Even with relatively small datasets (5-7k rows), the performance gains were significant when CPU was the bottleneck

2. **The learning curve is manageable** - About a week to refactor a moderately complex pipeline. The expression system is different but logical once you understand it

3. **Benchmark on your actual workload** - Don't rely on synthetic benchmarks. Test with your real data, operations, and resource constraints

4. **Resource constraints matter** - Polars' parallel execution made the biggest difference in resource-constrained environments where Pandas couldn't utilize available cores

5. **Migration strategy matters** - Gradual rollout (local → staging → production) helped catch issues early and build confidence

The migration from Pandas to Polars was worth the investment. Processing time was cut in half, CPU usage became stable and predictable, and the application performed consistently across environments. For production workloads where performance matters, Polars is a strong choice.

---

**Next in this series:** While Polars solved the CPU performance issues, a memory problem persisted that led to architectural changes. Read about how I split the application into Go + Python services in [Part 2: When You Can't Find the Bug - Architecting Around Production Issues](../go-python-architecture/).
