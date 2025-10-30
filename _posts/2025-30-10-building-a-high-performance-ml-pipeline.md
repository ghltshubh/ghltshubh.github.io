---
layout: post
title:  "Building a High-Performance ML Pipeline: Lessons from Processing 162TB of Weather Data"
date: 2025-10-30 12:00:00
description: Optimizing data pipelines for large-scale geospatial machine learning tasks
tags: big-data machine-learning performance optimization data-pipelines
categories: data-science machine-learning big-data
---

> Hard-won lessons from training a bias correction model on massive geospatial datasets

---

## Table of Contents

- [The Challenge](#the-challenge)
- [Key Learnings](#key-learnings)
  - [1. The Hidden Zarr Compression Bottleneck](#1-the-hidden-zarr-compression-bottleneck-)
  - [2. Coordinate Selection: The Silent Data Killer](#2-coordinate-selection-the-silent-data-killer-)
  - [3. Zarr v3 API Changes Will Break Your Code](#3-zarr-v3-api-changes-will-break-your-code-)
  - [4. Dask Configuration Doesn't Always Help](#4-dask-configuration-doesnt-always-help-)
  - [5. Batch Size Matters More Than You Think](#5-batch-size-matters-more-than-you-think-)
  - [6. Parallelize the Right Thing](#6-parallelize-the-right-thing-)
  - [7. Memory-Efficient Processing Strategies](#7-memory-efficient-processing-strategies-)
  - [8. ProcessPoolExecutor Pitfalls](#8-processpoolexecutor-pitfalls-)
  - [9. Caching is Your Best Friend](#9-caching-is-your-best-friend-)
  - [10. Data Integrity Issues](#10-data-integrity-issues-)
  - [11. When More Cores Don't Help](#11-when-more-cores-dont-help-)
  - [12. Diagnostic Tools Are Essential](#12-diagnostic-tools-are-essential-)
- [Final Performance Summary](#final-performance-summary)
- [Takeaways for Your Next Data Pipeline](#takeaways-for-your-next-data-pipeline)
- [The Bottom Line](#the-bottom-line)

---

## The Challenge

Training a machine learning model on a single coordinate's weather data from a **162TB dataset** distributed across **44 zarr batches**. Each batch contained ~3,930 features across multiple time dimensions. 

**Goal:** Extract, process, and train without running out of memory or time.

**Constraints:**
- 96 CPU cores available
- 768GB RAM
- Data stored in Google Cloud Storage (GCS)
- Local SSD for caching

---

## Key Learnings

### 1. The Hidden Zarr Compression Bottleneck 🐌

#### The Problem
Our initial pipeline took **4+ hours** to load data, despite having 96 CPU cores available.

#### The Root Cause
Zarr files compressed with `blosc(nthreads=1)` force **single-threaded decompression**. No amount of dask configuration or parallel workers can fix this.

#### What We Learned

- ❌ Zarr compression codec settings are **permanent** - they're baked into the files
- ❌ Single-threaded decompression = 1 core working while 95 cores sit idle
- ❌ Loading 10 file_ids took 10-15 minutes per batch, regardless of available cores
- ✅ This is the #1 performance bottleneck in our pipeline

#### The Fix (for future datasets)

When creating zarr files, use multi-threaded compression:

```python
import numcodecs

blosc_compressor = numcodecs.Blosc(
    cname='zstd', 
    clevel=3, 
    shuffle=2, 
    nthreads=12  # Use multiple threads!
)
```

**Expected improvement:** 8-12× speedup on decompression

---

### 2. Coordinate Selection: The Silent Data Killer 💀

#### The Bug That Cost Us Hours

We spent hours debugging why we had **100% NaN values** in our dataset, only to discover it was a single line of code.

#### The Mistake

```python
# WRONG - Using integer indexing
lat_idx = int(51.5 + 90)  # = 141
lon_idx = int(0.0)        # = 0

coord_data = merged.isel(lat=lat_idx, lon=lon_idx)  # ❌ WRONG!
# This selects the 142nd element in the array, which might not be UK!
```

#### What Was Happening

- `.isel()` uses **integer positions** in the array
- `lat_idx=141` selected the 142nd lat value (0-indexed)
- This might be lat=51°, lat=52°, or even a NaN-filled coordinate!
- The data existed, but we were looking at the wrong spot

#### The Fix

```python
# CORRECT - Using value-based selection
coord_data = merged.sel(lat=51.5, lon=0.0, method="nearest")  # ✅ CORRECT!
# This finds the coordinate VALUE nearest to 51.5°N, 0.0°E
```

#### Why This Matters

**Geographic coordinates are NOT always sequential integers!**

Your dataset might have:
```python
lat: [-90.0, -89.0, ..., 50.0, 52.0, 53.0, ..., 90.0]
#                           ↑
#                    Note: no 51.0 or 51.5!
```

- Using `isel(lat=141)` gives you whatever is at index 141
- Using `sel(lat=51.5, method="nearest")` gives you lat=52.0 (the closest actual value)

#### The Symptom: 100% NaN

```
NaNs found: 15,504,300 (100.00%)
  - In features: 15,504,300  ← All features NaN
  - In target: 14,564,790    ← 94% target NaN
After: 0 valid samples (0.0%)
```

This pattern meant we were extracting data from an **invalid coordinate** that had no actual data.

#### The Diagnostic Process

1. **Check the working code first**
   - Found a notebook that successfully loaded the same data
   - Compared the coordinate selection methods

2. **Spot the difference**
   - Working code: `.sel(lat=51.5, lon=0.0, method="nearest")`
   - Broken code: `.isel(lat=141, lon=0)`

3. **Understand the coordinate grid**
   - Inspected actual lat/lon values in the dataset
   - Realized lat might be `[..., 50.0, 52.0, ...]` not continuous

#### Best Practices

**Always use `.sel()` with `method="nearest"` for geographic coordinates:**

```python
# ✅ GOOD - Value-based selection
data = ds.sel(lat=51.5, lon=0.0, method="nearest")

# ❌ BAD - Index-based selection
lat_idx = int(lat + 90)
data = ds.isel(lat=lat_idx, lon=lon_idx)
```

**When to use each method:**

- `.sel()` - When you have coordinate **values** (lat/lon, timestamps, etc.)
- `.isel()` - When you have **integer positions** (first 10 items, every other row, etc.)
- `method="nearest"` - Handles non-exact coordinate matching

#### The Lesson

> **Never assume coordinate grids are regular or sequential.**

Always inspect your actual coordinate values:

```python
print(f"Lat values: {ds.lat.values}")
print(f"Lon values: {ds.lon.values}")
```

And use value-based selection with `method="nearest"` for geographic data.

---

### 3. Zarr v3 API Changes Will Break Your Code 🔧

#### The Error

```python
AttributeError: module 'zarr' has no attribute 'Blosc'
```

#### The Confusion

We were using `zarr_format=2` in our code, so why was Zarr v3 causing issues?

#### Understanding the Distinction

There are **two different things** called "zarr":

1. **Zarr Storage Format** (v2 or v3) - the file format on disk
2. **Zarr Python Package** (v2.x or v3.x) - the API you use in code

**You can write zarr v2 FORMAT files using zarr v3 PACKAGE API!**

#### The Breaking Change

**Zarr v2 Package (old):**
```python
import zarr

compressor = zarr.Blosc(cname='zstd', clevel=3, shuffle=2, nthreads=12)
```

**Zarr v3 Package (new):**
```python
import zarr
import numcodecs  # Separate package!

compressor = numcodecs.Blosc(cname='zstd', clevel=3, shuffle=2, nthreads=12)
```

#### The Fix

```python
import numcodecs  # Add this import

# This works with zarr v3 package, writing zarr v2 format files
coord_slice.to_zarr(
    path, 
    mode='w', 
    zarr_format=2,  # ← Storage format (v2)
    encoding={
        var: {
            'compressor': numcodecs.Blosc(  # ← API (v3)
                cname='zstd', 
                clevel=3, 
                shuffle=2, 
                nthreads=12
            )
        }
        for var in data_vars
    }
)
```

#### Why This Happens

- Many environments now default to zarr v3 package
- Zarr v3 moved compression codecs to the separate `numcodecs` library
- `zarr_format=2` only controls the **file format**, not the **API**

#### Best Practice

**Check your zarr version and adjust imports:**

```python
import zarr
print(f"Zarr version: {zarr.__version__}")

# If zarr >= 3.0:
import numcodecs
compressor = numcodecs.Blosc(...)

# If zarr < 3.0:
compressor = zarr.Blosc(...)
```

Or better yet, just always use `numcodecs.Blosc()` - it works in both versions:

```python
# Works in zarr v2 and v3
import numcodecs
compressor = numcodecs.Blosc(cname='zstd', clevel=3, shuffle=2, nthreads=12)
```

---

### 4. Dask Configuration Doesn't Always Help ⚙️

#### What We Tried

```python
dask.config.set(scheduler='threads', num_workers=32)
coord_data.compute(scheduler='threads', num_workers=32)
```

#### What Happened
Nothing. Still single-threaded. CPU usage remained at 100% (1 core).

#### The Learning

- Dask parallelizes **computation graphs**, not I/O-bound codec operations
- If the bottleneck is in native C libraries (like blosc), dask can't help
- Always profile to find the **actual** bottleneck before optimizing

#### When Dask DOES Help

- ✅ Loading multiple files in parallel
- ✅ CPU-intensive transformations on lazy arrays
- ✅ Rechunking operations
- ✅ Distributed computing across multiple machines

#### When Dask DOESN'T Help

- ❌ Single-threaded codec decompression
- ❌ Sequential I/O operations
- ❌ Operations already optimized in C/Fortran

---

### 5. Batch Size Matters More Than You Think 📦

#### The Evolution

| Approach | Batches | Files per Batch | Result |
|----------|---------|----------------|---------|
| Initial | 23 | 3 | Too much overhead |
| Optimized | 7 | 10 | Sweet spot |
| Aggressive | 1 | 69 | Risk of OOM |

#### The Impact

- **67% fewer `.compute()` calls** (23 → 7 batches)
- Each call has overhead (lazy graph construction, memory allocation)
- Total time reduced from ~3 hours to ~1.5 hours

#### The Rule of Thumb

- **Too small:** Death by a thousand cuts (overhead dominates)
- **Too large:** RAM exhaustion and crashes
- **Sweet spot:** Size batches to use 10-20% of available RAM

#### How to Calculate Optimal Batch Size

```
1. Check single file size: ~15GB per file
2. Available RAM: 768GB
3. Target usage: ~20% = 150GB
4. Batch size: 150GB / 15GB = 10 files per batch
```

---

### 6. Parallelize the Right Thing 🎯

#### The Breakthrough
Separating I/O from CPU work

#### Before: Everything Serial

```
Load data (single-threaded, 600s)
  ↓
Extract features (serial loop, 600s)
  ↓
Total: ~1,200 seconds/batch
```

#### After: Strategic Parallelization

```
Load data (single-threaded, 600s)  ← Can't parallelize (codec limitation)
  ↓
Extract features (32 workers, 15s)  ← Highly parallelized
  ↓
Total: ~615 seconds/batch
```

#### The Learning

**I/O-bound operations:**
- ❌ Can't parallelize: zarr decompression, disk reads, network I/O
- Accept the limitation and optimize elsewhere

**CPU-bound operations:**
- ✅ Highly parallelizable: feature extraction, transformations, calculations
- Use `ThreadPoolExecutor` for already-loaded data
- Scales linearly with cores (up to a point)

#### Time Saved
**40% reduction** in total time by parallelizing just the extraction step

---

### 7. Memory-Efficient Processing Strategies 💾

#### Three Approaches We Tested

##### ❌ Approach 1: Load All at Once
```
Load entire 336GB → Extract features → Train
```

**Pros:**
- Fastest IF you have enough RAM
- Simplest code

**Cons:**
- High risk of OOM (Out of Memory) errors
- RAM usage spikes during load
- No fault tolerance

---

##### ❌ Approach 2: Tiny Batches
```
23 batches of 3 files each
```

**Pros:**
- Very safe on memory
- Fault tolerance (can resume)

**Cons:**
- Death by overhead (23× the setup cost)
- Much slower overall

---

##### ✅ Approach 3: Balanced Batches
```
7 batches of 10 files each
```

**Pros:**
- Peak RAM: ~220GB (safe on 768GB machine)
- Minimal overhead (only 7 batches)
- Good balance of speed and safety

**Cons:**
- Slightly more complex logic

---

#### Memory Usage Progression

| Stage | RAM Used | Notes |
|-------|----------|-------|
| Initial | 10GB | Script loaded, data lazy |
| Batch 1 loading | 85GB | Decompressing zarr |
| Batch 1 extraction | 75GB | Features being written |
| After cleanup | 50GB | Old batch freed |
| Batch 2 loading | 85GB | Process repeats |
| Final matrices | 220GB | All features in memory |

#### The Takeaway
Profile your RAM usage and find the largest batch size that fits comfortably. Leave 20-30% headroom for OS and other processes.

---

### 8. ProcessPoolExecutor Pitfalls 🕳️

#### The Mystery
Worker processes would hang indefinitely with no error message. No logs, no timeout, no crash - just eternal silence.

#### The Investigation

```
Batch 5: Starting...
Batch 5: Opened zarr stores...
Batch 5: Writing to cache...
[HANGS FOREVER - NO OUTPUT]
```

#### The Culprit

- Zarr's `to_zarr()` with `compute=True` in subprocess
- Some internal threading conflict between zarr, blosc, and multiprocessing
- No timeout = no way to detect the hang

#### The Solution

**Always set timeouts:**
```python
future.result(timeout=1800)  # 30 minutes max per batch
```

**Use `as_completed()` to handle results:**
```python
for future in as_completed(futures):
    try:
        result = future.result(timeout=TIMEOUT)
        # Process result
    except TimeoutError:
        # Handle timeout
        print(f"Batch {idx} timed out")
```

**For I/O heavy tasks, consider alternatives:**
- ThreadPoolExecutor (if GIL isn't an issue)
- Sequential processing with good progress bars
- Async I/O with asyncio

#### The Learning
Multiprocessing + complex I/O libraries = potential deadlocks. Always add timeouts and monitoring. Better to fail fast than hang forever.

---

### 9. Caching is Your Best Friend 💰

#### The Strategy

```python
CACHE_DIR = f"/local_ssd/batch_cache_{lat}_{lon}"

# Check cache first
if os.path.exists(f"{CACHE_DIR}/batch_{i}.zarr"):
    return load_from_cache()

# Download if missing
download_batch()
save_to_cache()
```

#### The Payoff

| Run Type | Time | Notes |
|----------|------|-------|
| First run | ~2 hours | Download all batches from GCS |
| Second run | ~4 minutes | Validate cache only |
| Nth run | ~4 minutes | Same coordinate |
| Different coordinate | ~2 hours | New cache directory |

**Speed improvement:** 30× faster for subsequent runs

#### Critical: Match Coordinate Selection in Cache

**Bug we found:**

```python
# Download phase - WRONG coordinate!
cached_data = merged.isel(lat=141, lon=0)  # ❌ Gets wrong coordinate

# Training phase - CORRECT coordinate!
model_data = ds.sel(lat=51.5, lon=0.0, method="nearest")  # ✅ Gets UK

# Result: Cache has data from lat index 141, not UK!
# Everything downstream gets 100% NaN
```

**The fix:**

```python
# BOTH phases must use the same method:
cached_data = merged.sel(lat=51.5, lon=0.0, method="nearest")  # ✅
model_data = ds.sel(lat=51.5, lon=0.0, method="nearest")       # ✅
```

#### Best Practices

1. **Include parameters in cache name**
   ```
   batch_cache_51.5_0.0  ← Good (coordinate-specific)
   batch_cache           ← Bad (conflicts)
   ```

2. **Validate cache files**
   ```python
   try:
       ds = xr.open_zarr(cache_path)
       if len(ds.data_vars) == 0:
           return False  # Corrupted
   except:
       return False  # Invalid
   ```

3. **Use local SSD if available**
   - NVMe SSD: 3-7 GB/s read speed
   - Network storage: 100-500 MB/s
   - **10-70× faster** for local SSD

4. **Invalidate cache when code changes**
   - Coordinate selection method changed? Force redownload!
   - Add version numbers to cache directory names
   - Set `FORCE_REDOWNLOAD = True` temporarily

---

### 10. Data Integrity Issues 🔍

#### The Surprise
After concatenating 22 batches, we expected 95 unique entries but found only 69.

#### The Diagnosis

```
Batch 0: entries [0, 1, 2, 3]
Batch 1: entries [0, 1, 2, 3]  ← DUPLICATE!
Batch 2: entries [4, 5, 6]
...
```

**Result:** 26 duplicate entries across batches

#### The Impact

- Concatenating created duplicate rows
- Silent data corruption (no error raised)
- Models would train on inflated datasets
- Results would be biased toward duplicated time periods

#### The Fix

```python
# 1. Detect duplicates
unique_ids, unique_indices = np.unique(
    dataset.index.values, 
    return_index=True
)

# 2. Count them
n_duplicates = len(dataset.index) - len(unique_ids)

# 3. Remove duplicates (keep first occurrence)
if n_duplicates > 0:
    unique_indices = np.sort(unique_indices)
    dataset = dataset.isel(index=unique_indices)
    print(f"Removed {n_duplicates} duplicates")
```

#### Prevention Strategies

1. **Validate during data creation**
   - Ensure batches have non-overlapping indices
   - Add assertions in batch creation code

2. **Check after concatenation**
   - Always validate dimension uniqueness
   - Log warnings for duplicates

3. **Add metadata**
   - Include index ranges in batch metadata
   - Document expected ranges

#### The Lesson
**Never trust data integrity implicitly.** Always validate dimension uniqueness after concatenation, especially with multiple data sources.

---

### 11. When More Cores Don't Help 🚫

#### The Question
"We have 96 cores. Why not use 64 or 96 workers instead of 32?"

#### The Reality Check

**Time breakdown per batch:**
- Load time: 600 seconds (98% of time, single-threaded)
- Extract time: 15 seconds (2% of time, 32 workers)

#### Performance Analysis

| Workers | Extract Time | Improvement | % of Total |
|---------|--------------|-------------|------------|
| 1 (serial) | 600s | baseline | - |
| 8 | 75s | 525s saved | Significant |
| 16 | 38s | 562s saved | Good |
| 32 | 15s | 585s saved | Excellent |
| 64 | 10s | 590s saved | Minimal gain |
| 96 | 8s | 592s saved | Minimal gain |

#### Increasing Workers to 96

**Time saved:** 7 seconds per batch
**Total saved (7 batches):** 49 seconds out of 5,000 seconds total
**Improvement:** <1%

#### The Hidden Costs

- Thread creation overhead
- Context switching
- Memory contention
- GIL (Global Interpreter Lock) contention
- Diminishing returns after ~CPU_count/2

#### The Principle

> **Optimize the bottleneck, not the fast parts.**

**Where to focus:**
1. ✅ Fix the 600s load time (98% of time)
2. ❌ Not the 15s extract time (2% of time)

#### Amdahl's Law in Practice

```
If 98% of time is serial (load):
Maximum speedup with infinite parallel cores = 1.02×

If you could parallelize the load time:
Maximum speedup = 50×
```

**Lesson:** Attack the bottleneck, not everything else.

---

### 12. Diagnostic Tools Are Essential 🔧

#### The Toolkit

##### 1. `top` - Real-time System Monitor

```bash
top -u username
```

**What to watch:**
- **%CPU:** Is parallelism working?
  - `100%` = 1 core active
  - `3200%` = 32 cores active
  - `9600%` = 96 cores active (theoretical max)
- **RES:** Actual RAM usage (watch for memory leaks)
- **TIME+:** How long the process has been running
- **S (State):** R=Running, S=Sleeping, D=Disk wait

##### 2. `htop` - Interactive Process Viewer

```bash
htop
```

**Benefits:**
- See all cores individually (visual bars)
- Identify if work is distributed or concentrated
- Sort by CPU, memory, time
- Tree view for process hierarchy

##### 3. Strategic `print()` Statements

```python
import time

start = time.time()
print(f"Starting batch {i}...", flush=True)

# Do work...

elapsed = time.time() - start
print(f"Completed in {elapsed:.1f}s", flush=True)
```

**Key additions:**
- `flush=True` - Force immediate output (critical for long operations)
- Timestamps for each operation
- Memory usage after major operations
- Progress indicators for long loops

##### 4. Memory Profiling

```python
import psutil
import os

def log_memory():
    process = psutil.Process(os.getpid())
    mem_gb = process.memory_info().rss / (1024**3)
    print(f"Memory: {mem_gb:.1f}GB")

# Call at key points
log_memory()
```

##### 5. Data Inspection

**Check coordinate values:**
```python
# Always inspect what you're actually getting
print(f"Lat values: {ds.lat.values}")
print(f"Lon values: {ds.lon.values}")
print(f"Lat range: {ds.lat.min().values} to {ds.lat.max().values}")

# Check NaN percentages
nan_pct = np.isnan(data).sum() / data.size * 100
print(f"NaN percentage: {nan_pct:.2f}%")
```

##### 6. Zarr Store Inspector

```python
import zarr
import numcodecs

# Check compression settings
z = zarr.open('path/to/store.zarr')
print(z.info)  # Shows codec, chunks, dtype

# Check compressor
for key in z.arrays():
    arr = z[key]
    print(f"{key}: {arr.compressor}")
    
# Check zarr version
print(f"Zarr version: {zarr.__version__}")
print(f"Numcodecs available: {'numcodecs' in dir()}")
```

#### The Learning

> **You can't optimize what you can't measure.**

**Best practices:**
1. Instrument early and often
2. Log timestamps at every major operation
3. Monitor resource usage in real-time
4. Profile before optimizing
5. Measure improvements quantitatively
6. **Inspect your data - never assume it's structured as expected**

---

## Final Performance Summary

### Metrics Comparison

| Metric | Initial Naive | Final Optimized | Improvement |
|--------|--------------|-----------------|-------------|
| **Batch count** | 23 | 7 | 3.3× fewer |
| **Files per batch** | 3 | 10 | 3.3× larger |
| **Feature extraction** | Serial (1 core) | 32 parallel workers | 32× faster |
| **Load time/batch** | 250s | 750s | - (limited by codec) |
| **Extract time/batch** | 600s | 15s | 40× faster |
| **Total time/batch** | 850s | 765s | 1.1× faster |
| **Total batches** | 23 batches | 7 batches | - |
| **Total pipeline** | ~5.5 hours | ~1.5 hours | **3.7× faster** |
| **Subsequent runs** | ~5.5 hours | ~4 minutes | **82× faster** |

### Bug Fixes Impact

| Bug | Time Lost | Impact |
|-----|-----------|--------|
| Wrong coordinate selection (.isel vs .sel) | 3+ hours debugging | Critical - 100% NaN |
| Zarr v3 API (zarr.Blosc vs numcodecs.Blosc) | 1 hour debugging | Critical - code crash |
| Cache invalidation after coordinate fix | 2 hours re-download | Medium - one-time cost |

### Key Optimizations Impact

| Optimization | Time Saved | Impact |
|--------------|------------|--------|
| Fewer batches (23→7) | ~1.5 hours | High |
| Parallel extraction | ~2.0 hours | Very High |
| Efficient caching | Infinite (subsequent runs) | Critical |
| Better batch sizing | ~0.5 hours | Medium |
| Fixing coordinate selection | Infinite (enabled pipeline) | Critical |

### What We Couldn't Fix

- **Zarr decompression bottleneck:** Limited by codec settings
  - Current: 600-900s per batch (single-threaded)
  - Potential: 75-110s per batch (with multi-threaded codec)
  - **Requires re-creating zarr files**

---

## Takeaways for Your Next Data Pipeline

### Top 12 Lessons

1. **🎯 Use .sel() not .isel() for coordinates**
   - Geographic coordinates are rarely sequential integers
   - Always use `method="nearest"` for value-based selection
   - Inspect coordinate values before assuming structure
   - **This one bug caused 100% NaN - check it first!**

2. **🔧 Understand Zarr v3 API changes**
   - `zarr_format` (storage) ≠ zarr package version (API)
   - Use `numcodecs.Blosc()` not `zarr.Blosc()`
   - Works in both zarr v2 and v3 packages

3. **🔍 Profile first, optimize second**
   - Find the actual bottleneck with `top`, `htop`, profilers
   - Don't assume - measure
   - Check data validity before blaming the pipeline

4. **💾 Compression settings matter**
   - Multi-threaded codecs save hours
   - Settings are permanent - choose wisely
   - Test decompression speed during prototyping
   - Use `nthreads=8-12` for multi-core systems

5. **📦 Batch size is critical**
   - Balance overhead vs memory
   - Target 10-20% of available RAM
   - Profile memory usage to find sweet spot

6. **🎯 Parallelize the right operations**
   - CPU-bound work: highly parallelizable
   - I/O-bound work: usually can't parallelize
   - Focus on CPU-bound bottlenecks

7. **💰 Cache aggressively (but validate!)**
   - Local SSD storage is cheap
   - 30× speedup for repeated operations
   - Include parameters in cache keys
   - **Invalidate cache when code changes!**

8. **🔧 Add timeouts everywhere**
   - Prevent silent hangs
   - Fail fast, not never
   - Always use `future.result(timeout=X)`

9. **✅ Validate data integrity**
   - Check for duplicates after merging
   - Verify dimension uniqueness
   - Check NaN percentages at each step
   - Add assertions liberally

10. **📊 Monitor in real-time**
    - Use `top`/`htop` to verify assumptions
    - Add logging with timestamps
    - Track memory usage
    - **Print NaN percentages after each operation**

11. **🚫 Know when NOT to parallelize**
    - Some operations can't be sped up
    - Thread overhead can make things slower
    - Amdahl's Law: optimize the bottleneck

12. **⚙️ Dask isn't magic**
    - Can't fix codec-level bottlenecks
    - Only helps with computation graphs
    - Profile to see if it actually helps

---

## The Bottom Line

Working with massive datasets requires understanding the full stack - from compression codecs to multiprocessing semantics to coordinate selection methods.

### Key Insights

**The biggest performance gains often come from:**
- ✅ Fixing fundamental bugs (coordinate selection)
- ✅ Fixing fundamental design choices (compression settings)
- ✅ Strategic caching with proper invalidation
- ✅ Right-sizing batches

**Not from:**
- ❌ Adding more parallelism everywhere
- ❌ Complex distributed computing frameworks
- ❌ Throwing more hardware at the problem

### The Golden Rules

> **The most expensive operation is the one you don't need to do.**

- Cache it (but invalidate when needed)
- Batch it smartly
- Parallelize what actually benefits from parallelism
- Fix the bottleneck, not everything else

> **The most dangerous bug is the silent one.**

- 100% NaN can come from a single wrong function call
- Always validate data at each pipeline stage
- Use `.sel()` with `method="nearest"` for coordinates
- Never assume - always inspect

---

## Additional Resources

### Recommended Reading

- [Xarray Indexing Documentation](https://docs.xarray.dev/en/stable/user-guide/indexing.html)
- [Zarr v3 Migration Guide](https://zarr.readthedocs.io/en/stable/migration.html)
- [Zarr Documentation - Compression](https://zarr.readthedocs.io/en/stable/tutorial.html#compression)
- [Dask Best Practices](https://docs.dask.org/en/stable/best-practices.html)
- [Amdahl's Law Explained](https://en.wikipedia.org/wiki/Amdahl%27s_law)
- [Python Multiprocessing Guide](https://docs.python.org/3/library/multiprocessing.html)

### Tools Mentioned

- `top` / `htop` - System monitoring
- `psutil` - Python system utilities
- `memory_profiler` - Memory usage profiling
- `cProfile` - CPU profiling
- `zarr` - Array storage library
- `numcodecs` - Compression codecs for zarr
- `dask` - Parallel computing library
- `xarray` - Multi-dimensional labeled arrays

---

**Questions or feedback?** Feel free to reach out - I learned these lessons the hard way so you don't have to! 🚀