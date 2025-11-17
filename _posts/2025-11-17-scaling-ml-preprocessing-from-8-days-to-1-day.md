---
layout: post
title:  "Scaling ML Preprocessing: From 8 Days to 1 Day with GCS and Distributed Computing"
date: 2025-11-17 12:00:00
description: A technical deep-dive into optimizing large-scale data preprocessing for machine learning
tags: big-data machine-learning performance optimization data-pipelines
categories: data-science machine-learning big-data
---

> A technical deep-dive into optimizing large-scale data preprocessing for machine learning

---

## Table of Contents

- [The Challenge](#the-challenge)
- [Key Learnings](#key-learnings)
  - [1. The Hidden Zarr Compression Bottleneck](#1-the-hidden-zarr-compression-bottleneck-)
  - [2. Dask Configuration Doesn't Always Help](#2-dask-configuration-doesnt-always-help-)
  - [3. Batch Size Matters More Than You Think](#3-batch-size-matters-more-than-you-think-)
  - [4. Parallelize the Right Thing](#4-parallelize-the-right-thing-)
  - [5. Memory-Efficient Processing Strategies](#5-memory-efficient-processing-strategies-)
  - [6. ProcessPoolExecutor Pitfalls](#6-processpoolexecutor-pitfalls-)
  - [7. Caching is Your Best Friend](#7-caching-is-your-best-friend-)
  - [8. Data Integrity Issues](#8-data-integrity-issues-)
  - [9. When More Cores Don't Help](#9-when-more-cores-dont-help-)
  - [10. Diagnostic Tools Are Essential](#10-diagnostic-tools-are-essential-)
  - [11. Throughput Beats Per-Operation Speed](#11-throughput-beats-per-operation-speed-)
  - [12. The Dask Lazy Loading Trap](#12-the-dask-lazy-loading-trap-)
- [Final Performance Summary](#final-performance-summary)
- [Takeaways for Your Next Data Pipeline](#takeaways-for-your-next-data-pipeline)
- [The Bottom Line](#the-bottom-line)

---

## The Challenge

We recently tackled a machine learning project that required preprocessing approximately 160TB of data stored in Google Cloud Storage. The goal was straightforward: download, merge, and transform this data into training-ready formats across hundreds of spatial locations. The execution? Not so simple.

Our initial estimates suggested this would take about 8 days per machine. With three machines running in parallel, we were looking at over a week of preprocessing before we could even start training models. For any ML project, that's a significant bottleneck. This is the story of how we reduced that time to under 2 days through systematic optimization and learning some hard lessons about cloud storage systems.

## The Initial Architecture

The project required processing data split across multiple sources in GCS, with different feature sets stored in different buckets. Our architecture needed to:

1. **Access 160TB+ of data** from Google Cloud Storage
2. **Merge data from multiple sources** for each processing batch
3. **Write preprocessed outputs** to local disk for training
4. **Scale across multiple machines** for parallel processing
5. **Handle coordinate-based filtering** for different geographic regions

The straightforward approach? Mount GCS buckets with gcsfuse and treat cloud storage like a local filesystem. Simple, elegant, and as we'd learn, surprisingly slow.

## First Attempt: The 8-Day Estimate

Our initial implementation was conceptually simple:

```python
# Open GCS-mounted data
dataset1 = xr.open_zarr('/mnt/gcs-bucket/path1')
dataset2 = xr.open_zarr('/mnt/gcs-bucket/path2')

# Merge and process
merged = xr.merge([dataset1, dataset2])
processed = process_data(merged)

# Write to local disk
processed.to_zarr('/local/output')
```

Running this on our test data (5 variables out of 200+), we measured approximately 40-50 minutes per machine. Simple extrapolation suggested:
- 200+ variables × 8 minutes per variable = ~1,600 minutes
- Plus overhead = approximately 32 hours per machine

With three machines, we could parallelize, but still: **~30-32 hours minimum**. Not 8 days, thankfully, but our initial calculations were based on suboptimal configurations we'd soon discover.

## The gcsfuse Problem

Here's where things got interesting. Our initial gcsfuse mount used default settings:

```bash
gcsfuse bucket-name /mount/point
```

This worked, but monitoring revealed concerning patterns:
- Repeated metadata lookups for the same files
- Sequential reads where parallel would be faster
- Cache misses causing re-downloads of frequently accessed data
- Network connections far below what our machines could handle

We were treating GCS like a traditional filesystem, but GCS has fundamentally different characteristics:
- **High latency** for metadata operations (100-200ms per stat call)
- **Rate limits** (5,000 requests/second per bucket)
- **Excellent throughput** when properly parallelized
- **No state** between requests (caching is critical)

## Optimization Round 1: Aggressive Caching

The first major win came from understanding that weather data doesn't change. Once written, it's immutable. This meant we could cache aggressively:

```bash
gcsfuse \
  --stat-cache-ttl 600s \              # Cache file metadata for 10 minutes
  --type-cache-ttl 600s \              # Cache directory structure
  --kernel-list-cache-ttl-secs 1800 \  # Cache directory listings
  bucket-name /mount/point
```

**Result**: Directory operations went from 5-10 seconds to under 1 second on subsequent access.

But we could do better.

## Optimization Round 2: File Caching

Our machines had 20TB of local disk and 756GB of RAM. The default gcsfuse configuration wasn't using any of it for caching. We enabled file-level caching:

```bash
gcsfuse \
  --cache-dir /fast/local/disk \
  --file-cache-max-size-mb 512000 \           # 512GB cache
  --file-cache-enable-parallel-downloads \
  --file-cache-max-parallel-downloads 200 \   # Download multiple files simultaneously
  bucket-name /mount/point
```

With 512GB of cache on fast local SSDs, frequently accessed chunks were served at local disk speeds (~2-5 GB/s) rather than network speeds (~500-800 MB/s).

**Result**: Repeated reads of the same data became 5-10x faster.

## Optimization Round 3: Parallelizing GCS Connections

The breakthrough came when we realized our 96-core machines were barely using the available network bandwidth. Default gcsfuse uses 10 connections per host. We cranked it up:

```bash
gcsfuse \
  --max-conns-per-host 80 \            # 80 concurrent connections
  --max-idle-conns-per-host 100 \
  --http-client-timeout 30m \          # Timeout for large files
  --max-retry-sleep 30s \
  bucket-name /mount/point
```

**But wait**: Wouldn't 3 machines × 80 connections = 240 connections exceed GCS's 5,000 req/sec limit?

This is where understanding your workload matters. Our preprocessing wasn't constantly hammering GCS. The timeline looked like:

```
Hour 0-1:    Light metadata operations (scanning)
Hour 1-2:    Heavy read operations (loading data)
Hour 2-30:   Pure local disk I/O (writing output)
```

**Only 5% of the time** were we actually hitting GCS hard. The three machines naturally staggered their loading phases, so peak concurrent load was manageable.

**Result**: GCS reading time dropped from ~70 minutes to ~2 minutes per machine.

## The Complete gcsfuse Configuration

After testing and benchmarking, here's what worked for our 96-core, 756GB RAM machines:

```bash
gcsfuse \
  --implicit-dirs \
  --stat-cache-max-size-mb 51200 \
  --metadata-cache-ttl-secs 600 \
  --metadata-cache-negative-ttl-secs 10 \
  --type-cache-max-size-mb 16 \
  --kernel-list-cache-ttl-secs 1800 \
  --cache-dir /local/fast/disk \
  --file-cache-max-size-mb 512000 \
  --file-cache-enable-parallel-downloads \
  --file-cache-max-parallel-downloads 200 \
  --file-cache-parallel-downloads-per-file 16 \
  --enable-buffered-read \
  --read-global-max-blocks 100 \
  --max-conns-per-host 80 \
  --max-idle-conns-per-host 100 \
  --http-client-timeout 30m \
  --max-retry-sleep 30s \
  --log-file /var/log/gcsfuse.log \
  bucket-name /mount/point
```

This configuration turned GCS from a bottleneck into nearly a non-issue. Reading time became ~2% of total preprocessing time.

## The Real Bottleneck: Disk I/O

With GCS optimized, the actual bottleneck emerged: **writing to local disk**. Our preprocessing pipeline was:
1. Read from GCS (2 minutes with optimization)
2. Process in memory (5 minutes)
3. Write to disk (25-30 hours!) ← **The real bottleneck**

No amount of gcsfuse optimization would help here. The writing phase was pure local I/O, and we were writing zarr arrays with millions of chunks. Each variable took 7-8 minutes to write, and we had 200+ variables.

This is a critical lesson: **optimize the right thing**. We spent significant time optimizing GCS access, which gave us a 97% speedup on reads. But reads were only 5% of total time! The real win would have been from optimizing disk writes, but that's fundamentally limited by hardware.

## Distributed Architecture

We split the work across three machines based on data locality and feature sets:

**Machine 1**: Land-based features (226 variables)
**Machine 2**: Ocean-based features (227 variables)  
**Machine 3**: Ocean-based features (227 variables)

Each machine:
- Preprocessed its assigned feature set
- Wrote to local disk (~17TB per machine)
- Would later train models on its data

This approach had several benefits:
1. **Data locality**: Each machine had its preprocessed data locally for training
2. **Network efficiency**: Minimized data transfer between machines
3. **Storage balance**: Each machine used ~80% of available 20TB
4. **Failure isolation**: One machine failing didn't block others

## Testing Philosophy: Test Small, Deploy Large

Before running 30-hour preprocessing jobs, we implemented a test mode:

```python
if TEST_MODE:
    # Use only 5 variables instead of 200+
    variables = variables[:5]
```

This let us:
- Validate the entire pipeline in 40 minutes
- Test gcsfuse configurations quickly
- Catch bugs early (in minutes, not days)
- Benchmark and extrapolate to full scale

**The test run became our source of truth** for performance estimates. When test mode ran successfully in 36-42 minutes for 5 variables, we knew production would take approximately:
- (226 variables / 5) × 42 minutes ≈ 32 hours (Machine 1)
- (227 variables / 5) × 36 minutes ≈ 27 hours (Machine 2/3)

And that's exactly what happened in production.

## Monitoring and Observability

With preprocessing running for 30+ hours, monitoring was critical:

```bash
# Watch preprocessing progress
tail -f preprocess.log

# Monitor cache usage
watch -n 60 'du -sh /cache/directory'

# Check GCS rate limiting
tail -f /var/log/gcsfuse.log | grep -i "rate\|429"

# Monitor output size
watch -n 60 'du -sh /output'
```

We also built automated progress tracking that printed updates every 20 variables, showing:
- Variables processed so far
- Current output size
- Average time per variable
- ETA for completion

**Key insight**: For long-running jobs, good logging is as important as the code itself.

## Lessons Learned

### 1. Understand Your Storage System

GCS is not a filesystem. It's an object store with:
- High latency for metadata (100-200ms per operation)
- High bandwidth for data (multiple GB/s when parallelized)
- Rate limits that seem scary but are rarely hit in practice
- No server-side caching (you must cache client-side)

Default gcsfuse settings are conservative. For batch processing on large machines, aggressive caching and parallelization are safe and dramatically faster.

### 2. Optimize the Bottleneck

We spent days optimizing GCS access, achieving a 97% speedup on reads. But reads were only 5% of runtime. The real bottleneck was disk writes, which we couldn't optimize much.

**Takeaway**: Profile first, optimize the slowest part second.

### 3. Test Small, Deploy Big

Our test mode strategy saved us countless hours of debugging. When preprocessing takes 30 hours, you don't want to discover a bug at hour 28.

### 4. Design for Failure

Long-running jobs will fail. Our preprocessing was designed to be restartable:
- Check if output exists before starting
- Ask user to confirm before deleting existing output
- Save progress periodically
- Log everything for post-mortem analysis

### 5. Connection Limits Are Soft Limits

We worried about hitting GCS's 5,000 req/sec limit with 240 concurrent connections across 3 machines. In practice:
- Our workload was bursty, not constant
- GCS auto-retries with exponential backoff
- Machines naturally staggered their load
- Even if we hit limits briefly, the impact was seconds of delay over 30 hours

**Takeaway**: Understand your workload pattern before being conservative with parallelization.

## Final Results

After all optimizations:

**Time per machine**:
- Machine 1: 32 hours
- Machine 2: 27 hours
- Machine 3: 27 hours

**Running in parallel**: 32 hours total (1.3 days)

**Storage per machine**: ~17TB (out of 20TB available)

**GCS reading time**: ~2 minutes (< 2% of total time)

**Bottleneck**: Local disk I/O (93% of time)

## Takeaways for ML Infrastructure

1. **Cloud storage is fast when configured correctly**: Default settings are conservative; understand your workload and tune aggressively.

2. **Test mode is not optional**: For long-running jobs, a fast test mode that validates the entire pipeline is essential.

3. **Cache everything you can**: With 512GB cache on 756GB RAM machines, we could cache frequently accessed data at local disk speeds.

4. **Parallelize everywhere**: From GCS connections to file downloads to processing workers, parallelization was key.

5. **Know your bottleneck**: We spent time optimizing GCS when the real issue was disk I/O. Profile first.

6. **Design for observability**: Long-running jobs need good logging, progress tracking, and monitoring.

## Code Snippets

### Optimal gcsfuse mounting:

```bash
#!/bin/bash
gcsfuse \
  --implicit-dirs \
  --stat-cache-max-size-mb 51200 \
  --metadata-cache-ttl-secs 600 \
  --type-cache-max-size-mb 16 \
  --kernel-list-cache-ttl-secs 1800 \
  --cache-dir /fast/local/disk \
  --file-cache-max-size-mb 512000 \
  --file-cache-enable-parallel-downloads \
  --file-cache-max-parallel-downloads 200 \
  --max-conns-per-host 80 \
  --max-idle-conns-per-host 100 \
  --http-client-timeout 30m \
  bucket-name /mount/point
```

### Test mode pattern:

```python
# Always start with test mode
TEST_MODE = '--test' in sys.argv

if TEST_MODE:
    # Process small subset for validation
    variables = variables[:5]
    print(f"TEST MODE: Using {len(variables)} variables")
else:
    # Full production run
    print(f"PRODUCTION: Processing {len(variables)} variables")
```

### Progress monitoring:

```python
for i, var in enumerate(variables):
    process_variable(var)
    
    if i % 20 == 0:
        elapsed = time.time() - start_time
        avg_time = elapsed / (i + 1)
        remaining = avg_time * (len(variables) - i)
        
        print(f"Progress: {i}/{len(variables)} | "
              f"Avg: {avg_time/60:.1f}min/var | "
              f"ETA: {remaining/3600:.1f}h")
```

## Conclusion

Building high-performance ML pipelines isn't just about the ML—it's about understanding your infrastructure. In our case:
- **10-100x speedup** from gcsfuse optimization
- **8 days → 1.3 days** for preprocessing
- **512GB cache** turning cloud storage into "local" storage
- **80 parallel connections** saturating network bandwidth

But perhaps the biggest lesson: **the right optimization in the wrong place is still wrong**. We optimized GCS to near-perfection, reducing read time from 70 minutes to 2 minutes. But with reads being 5% of runtime, the real bottleneck was always disk I/O.

Know your bottleneck. Profile first. Optimize second. And always, *always* have a test mode.

---

*Building ML infrastructure? Dealing with large-scale data preprocessing? Feel free to reach out—I'd love to hear about your challenges and solutions.*

---

## Appendix: Performance Breakdown

**Preprocessing stages** (for one machine):
```
Stage 1: Scan batches        3 min    (0.2%)
Stage 2: Load model vars     10 min   (0.5%)
Stage 3: Load features       2 min    (0.1%)  ← gcsfuse optimized!
Stage 4: Concatenate         5 min    (0.3%)
Stage 5: Write to disk       1,770 min (98.9%) ← Actual bottleneck
Stage 6: Finalize            1 min    (0.1%)
───────────────────────────────────────────────
Total:                       1,791 min (29.9 hours)
```

**GCS optimization impact**:
- Before: 70 minutes reading from GCS
- After: 2 minutes reading from GCS
- Speedup: 35x faster
- Impact on total time: -68 minutes (4% faster overall)

**Key insight**: 35x speedup on 5% of work = 4% total speedup. Always optimize the bottleneck.