---
layout: post
title:  "Memory-Mapped Arrays in NumPy: Lessons from Processing Large-Scale Datasets"
date: 2025-11-14 12:00:00
description: Hard-won lessons on using memory-mapped arrays to process datasets larger than RAM
tags: numpy python memory-mapping performance data-engineering big-data
categories: data-engineering python performance-optimization
---

> How we went from crashing servers to processing massive datasets efficiently

---


## Table of Contents
- [The Challenge](#the-challenge)
- [Key Learnings](#key-learnings)
  - [1. Memory-Mapped Arrays: Not Just for Big Data](#1-memory-mapped-arrays-not-just-for-big-data-)
  - [2. The dtype Mismatch Disaster](#2-the-dtype-mismatch-disaster-)
  - [3. flush() Saved Our Data](#3-flush-saved-our-data-)
  - [4. Sequential Access Beats Random Access](#4-sequential-access-beats-random-access-)
  - [5. Shape Must Be Exact](#5-shape-must-be-exact-)
  - [6. Parallel Processing with Shared Memory](#6-parallel-processing-with-shared-memory-)
  - [7. Mode Selection Matters](#7-mode-selection-matters-)
- [Real-World Performance](#real-world-performance)
- [When NOT to Use Memory Mapping](#when-not-to-use-memory-mapping)
- [The Bottom Line](#the-bottom-line)

## The Challenge

We were building a data processing pipeline for a large-scale analysis project. The total dataset was massive - far exceeding available RAM.

**The Problem**: Loading even a small batch of data would consume significant RAM. Our pipeline needed to process the entire dataset, extract features, and perform analysis - all on a machine with limited memory.

**Initial Approach**: Load batches of data, process, clear memory, repeat.

**Result**: Constant memory pressure, frequent garbage collection, and processing times that made iteration nearly impossible.

**The Breakthrough**: NumPy's memory-mapped arrays let us treat the entire dataset as if it were in memory, without actually loading it.

## Key Learnings

### 1. Memory-Mapped Arrays: Not Just for Big Data 🚀

**The Misconception**

We thought memory mapping was only for massive datasets. Wrong.

**What We Discovered**

Even with moderate-sized datasets, memory mapping provided unexpected benefits:

| Metric | Regular Array | Memory-Mapped | Improvement |
|--------|--------------|---------------|-------------|
| Startup time | Minutes | < 1 second | 100×+ faster |
| RAM usage | Most/all of RAM | Minimal | 10-100× less |
| Processing time | Hours | Significantly faster | 20-40% faster |

**The Learning**

Memory-mapped arrays provide benefits beyond just handling data larger than RAM:

- ✅ **Instant startup**: No waiting for data to load - access is immediate
- ✅ **Multi-process sharing**: Multiple workers access the same data without duplication
- ✅ **Persistent changes**: Modifications write directly to disk
- ✅ **Reduced memory pressure**: OS manages caching, fewer GC pauses

**When It Helped Us**

Processing pipeline with 3 stages:
1. Feature extraction (reads entire dataset)
2. Normalization (reads + writes entire dataset)
3. Training (reads dataset repeatedly during epochs)

With regular arrays: Each stage loaded large amounts into RAM  
With memory mapping: Single small mapping shared across all stages

**Code Example**

```python
# Before: Loading everything into RAM
data = np.load('data.npy')  # Large dataset loaded, long wait
results = process_data(data)

# After: Memory mapping
data = np.memmap('data.dat', dtype='float32', 
                  mode='r', shape=(n_samples, n_features))
results = process_data(data)  # Instant, minimal RAM
```

### 2. The dtype Mismatch Disaster 💥

**The Problem**

We created a memory-mapped array for our dataset, processed data over many hours, saved everything... and got complete garbage when we tried to read it back.

**The Root Cause**

```python
# Day 1: Creating the file
data = np.memmap('data.dat', dtype='float32', 
                  mode='w+', shape=(n_samples, n_features))
# ... process data for hours ...
del data

# Day 2: Opening the file
data = np.memmap('data.dat', dtype='float64',  # ❌ WRONG dtype!
                  mode='r', shape=(n_samples, n_features))
```

**What Happened**

The file doesn't store dtype information. When we opened with `float64` instead of `float32`:
- Each value was interpreted as 8 bytes instead of 4
- Shape became wrong (only loaded half of the data)
- Values were completely meaningless
- **Hours of processing lost**

**The Fix**

✅ **Document dtype in filename**

```python
# Encode dtype in the filename
filename = 'data_float32.dat'
data = np.memmap(filename, dtype='float32', mode='w+', 
                  shape=(n_samples, n_features))
```

✅ **Store metadata separately**

```python
import json

# Save metadata
metadata = {
    'dtype': str(data.dtype),
    'shape': data.shape,
    'created': '2025-01-15'
}
with open('data_meta.json', 'w') as f:
    json.dump(metadata, f)

# Load using metadata
with open('data_meta.json', 'r') as f:
    meta = json.load(f)
    
data = np.memmap('data.dat', 
                  dtype=meta['dtype'],
                  mode='r',
                  shape=tuple(meta['shape']))
```

✅ **Validate immediately after creation**

```python
# After creating
data = np.memmap('data.dat', dtype='float32', 
                  mode='w+', shape=(n_samples, n_features))
data[0, 0] = 42.0  # Write test value
data.flush()

# Verify immediately
test = np.memmap('data.dat', dtype='float32', 
                mode='r', shape=(n_samples, n_features))
assert test[0, 0] == 42.0, "dtype mismatch detected!"
```

**The Learning**

- ❌ Memory-mapped files contain **only raw bytes** - no metadata
- ❌ Wrong dtype = silent data corruption
- ✅ Always store dtype and shape separately
- ✅ Validate immediately, not hours later
- ✅ Use descriptive filenames that include dtype

### 3. flush() Saved Our Data 💾

**The Nightmare**

We were running a long-running experiment, writing results to a memory-mapped array every iteration. After many hours of computation, the power went out. When we restarted, **the file was empty**.

**What We Didn't Know**

Memory-mapped arrays buffer writes in memory. Without calling `flush()`, changes stay in RAM and aren't written to disk.

**The Problem**

```python
results = np.memmap('experiment.dat', dtype='float64',
                   mode='w+', shape=(n_samples, n_features))

for i in range(n_samples):
    results[i] = run_experiment()  # Data in RAM buffer
    # ❌ No flush() - data never hits disk

# Power failure = ALL DATA LOST
```

**The Fix**

```python
results = np.memmap('experiment.dat', dtype='float64',
                   mode='w+', shape=(n_samples, n_features))

for i in range(n_samples):
    results[i] = run_experiment()
    
    # ✅ Flush periodically
    if i % 1000 == 0:
        results.flush()
        print(f"Checkpoint: {i} iterations saved")
```

**Flush Strategy Comparison**

| Strategy | Pros | Cons | Use When |
|----------|------|------|----------|
| Every write | Max safety | Extremely slow (10× slower) | Critical data, small writes |
| Every N writes | Good balance | Lose max N rows | Long experiments |
| Every T seconds | Time-based safety | Variable data loss | Real-time streaming |
| End of program | Fastest | Lose everything if crash | Short scripts only |

**Our Production Strategy**

```python
import time

last_flush = time.time()
FLUSH_INTERVAL = 30  # seconds

for i in range(n_samples):
    results[i] = run_experiment()
    
    # Flush every 30 seconds OR every N iterations
    if time.time() - last_flush > FLUSH_INTERVAL or i % 10000 == 0:
        results.flush()
        last_flush = time.time()
        
# Always flush at the end
results.flush()
```

**The Learning**

- ❌ Memory-mapped writes are buffered - not immediate
- ❌ No flush() = data lives in RAM, vulnerable to crashes
- ✅ Flush periodically for long-running processes
- ✅ Balance safety vs performance (flush frequency)
- ✅ Always flush before program exit

### 4. Sequential Access Beats Random Access 🎯

**The Discovery**

We had two algorithms that both processed our large dataset:
- Algorithm A: Process records in order (sequential)
- Algorithm B: Process records in random order (random access)

Both should take similar time, right? Wrong.

**The Reality**

| Access Pattern | Processing Time | Disk I/O | Explanation |
|----------------|----------------|----------|-------------|
| Sequential | Fast | High throughput | OS prefetches data |
| Random | Very slow | Low throughput | Constant disk seeks |

**Sequential access was 5-10× faster** for the same amount of work!

**Why This Happens**

Memory-mapped arrays rely on the OS page cache:

Sequential access:
```
Read record 0 → OS loads nearby pages (prefetch)
Read record 1 → Already in cache! ✅
Read record 2 → Already in cache! ✅
...
```

Random access:
```
Read record N → OS loads pages around N
Read record M (far from N) → Cache miss! Load pages around M ❌
Read record K (far from M) → Cache miss! Load pages around K ❌
...
```

**The Code**

```python
data = np.memmap('data.dat', dtype='float32',
                  mode='r', shape=(n_samples, n_features))

# ✅ GOOD: Sequential access
for i in range(n_samples):
    process_record(data[i])
    
# ❌ BAD: Random access
indices = np.random.permutation(n_samples)
for i in indices:
    process_record(data[i])  # Cache miss every time!
```

**When You Need Random Access**

If your algorithm requires random access, consider:

```python
# Option 1: Sort your indices
indices = np.random.permutation(n_samples)
indices.sort()  # ✅ Now it's sequential!
for i in indices:
    process_record(data[i])

# Option 2: Process in batches
batch_size = 100
for batch_start in range(0, n_samples, batch_size):
    batch_indices = np.random.randint(batch_start, 
                                     batch_start + batch_size, 
                                     size=batch_size)
    batch = data[batch_indices]  # Load batch into RAM
    for record in batch:
        process_record(record)  # Now in RAM, fast random access
```

**The Learning**

- ✅ Sequential access: OS prefetches, high throughput
- ❌ Random access: Constant disk seeks, low throughput
- ✅ Sort indices if possible
- ✅ Load random batches into RAM for processing
- ✅ Design algorithms for sequential patterns

### 5. Shape Must Be Exact 📐

**The Problem**

Unlike regular NumPy arrays, you can't just open a memory-mapped file and inspect its shape. You must know it in advance.

**What Bit Us**

```python
# Created with wrong shape documentation
data = np.memmap('data.dat', dtype='float32',
                  mode='w+', shape=(n_samples, wrong_n_features))
# Actually different dimensions!

# Months later, trying to read...
data = np.memmap('data.dat', dtype='float32',
                  mode='r', shape=(n_samples, wrong_n_features))
# Only reads portion of data, rest of file ignored or crashes
```

**The Silent Failure**

Memory mapping doesn't validate shape against file size. It just maps what you tell it to map:

```python
# File is 100MB
# You specify shape that needs 400MB
# NumPy: "Sure, I'll map 100MB and pretend it's 400MB"
# Result: Accessing beyond file = crash or garbage
```

**The Solution: Always Store Metadata**

```python
# When creating
shape = (n_samples, n_features)
dtype = 'float32'

# Store metadata
import json
metadata = {
    'shape': shape,
    'dtype': str(dtype),
    'file_size_mb': os.path.getsize('data.dat') / (1024**2)
}
with open('data_meta.json', 'w') as f:
    json.dump(metadata, f)

# When loading
with open('data_meta.json', 'r') as f:
    meta = json.load(f)

# Validate file size matches expected
expected_size = np.prod(meta['shape']) * np.dtype(meta['dtype']).itemsize
actual_size = os.path.getsize('data.dat')
assert expected_size == actual_size, f"Size mismatch: {expected_size} vs {actual_size}"

# Now safe to load
data = np.memmap('data.dat', dtype=meta['dtype'],
                mode='r', shape=tuple(meta['shape']))
```

**Alternative: Use Structured Format**

```python
# Instead of raw .dat files, use formats that store metadata
# Option 1: NPY format (stores dtype and shape)
np.save('data.npy', array)  # Metadata included
loaded = np.load('data.npy', mmap_mode='r')  # Auto-detects shape!

# Option 2: HDF5 (better for complex structures)
import h5py
with h5py.File('data.h5', 'w') as f:
    f.create_dataset('data', data=array)
```

**The Learning**

- ❌ Raw .dat files store no metadata - shape is unknown
- ❌ Wrong shape = silent partial reads or crashes
- ✅ Always store shape and dtype separately
- ✅ Validate file size matches expected dimensions
- ✅ Consider NPY or HDF5 for built-in metadata

### 6. Parallel Processing with Shared Memory 🔄

**The Problem**

We needed to process our large dataset using multiple CPU cores. Initial approach: split the data and load separate copies for each worker.

**Memory Explosion**

```python
# ❌ BAD: Each worker loads its own copy
from multiprocessing import Pool

def process_chunk(chunk_id):
    # Each worker loads entire dataset!
    data = np.load(f'chunk_{chunk_id}.npy')  # Multiple copies in RAM!
    return process(data)

with Pool(8) as p:
    results = p.map(process_chunk, range(8))
```

Result: RAM exhausted, system froze

**The Solution: Memory-Mapped Sharing**

```python
# ✅ GOOD: All workers share the same file
def process_chunk(start_idx):
    # Each worker opens same file - no duplication!
    shared = np.memmap('data.dat', dtype='float32',
                      mode='r',  # Read-only!
                      shape=(n_samples, n_features))
    
    # Process a chunk
    chunk = shared[start_idx:start_idx+chunk_size]
    return analyze(chunk)

with Pool(8) as p:
    chunk_starts = range(0, n_samples, chunk_size)
    results = p.map(process_chunk, chunk_starts)
```

**Performance Comparison**

| Approach | RAM Usage | Processing Time |
|----------|-----------|-----------------|
| Load copies | RAM exhausted (OOM) | Crashed |
| Memory-mapped | Minimal | Fast |
| Improvement | 10-100× less RAM | Actually works! |

**Critical Details**

1. **Use mode='r' for workers**
   ```python
   # Workers should only read
   shared = np.memmap('data.dat', mode='r', ...)  # ✅ Safe
   
   # ❌ Don't use 'r+' or 'w+' in workers
   # Multiple workers writing = corruption
   ```

2. **Prepare data before spawning workers**
   ```python
   # Create and populate BEFORE multiprocessing
   data = np.memmap('data.dat', mode='w+', shape=(n_samples, n_features))
   data[:] = initialize_data()
   data.flush()
   del data
   
   # Now spawn workers
   with Pool(8) as p:
       results = p.map(worker_func, range(8))
   ```

3. **Each worker opens independently**
   ```python
   def worker(chunk_id):
       # Open in each worker - don't pass memmap objects!
       data = np.memmap('data.dat', mode='r', shape=(n_samples, n_features))
       # ... process ...
   ```

**The Learning**

- ✅ Memory-mapped arrays enable zero-copy sharing between processes
- ✅ Use mode='r' for read-only worker access
- ✅ Prepare data before spawning workers
- ❌ Never pass memmap objects to workers - open in each process
- ❌ Never use 'r+' or 'w+' with multiple workers (corruption risk)

### 7. Mode Selection Matters 🔐

**The Confusion**

We spent 2 hours debugging why our changes weren't persisting, only to discover we opened the file with the wrong mode.

**Mode Comparison**

| Mode | Creates File? | Reads? | Writes? | Persists? | Use Case |
|------|--------------|--------|---------|-----------|----------|
| `'r'` | ❌ No | ✅ Yes | ❌ No | N/A | Reading existing data |
| `'r+'` | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | Updating existing data |
| `'w+'` | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | Creating new data |
| `'c'` | ❌ No | ✅ Yes | ✅ Yes | ❌ No | Temporary modifications |

**What Bit Us: Copy-on-Write Mode**

```python
# We used 'c' mode thinking it would save changes
data = np.memmap('results.dat', dtype='float32',
                mode='c',  # ❌ Copy-on-write!
                shape=(10000, 100))

data[0] = expensive_computation()  # Takes 1 hour
data.flush()  # Does nothing in 'c' mode
del data

# Load again
data = np.memmap('results.dat', dtype='float32',
                mode='r', shape=(10000, 100))
print(data[0])  # All zeros! Changes were lost!
```

**When to Use Each Mode**

✅ **'r' - Read-Only**
```python
# Parallel processing workers
# Loading pretrained features
# Validation without modification risk
data = np.memmap('features.dat', mode='r', ...)
```

✅ **'r+' - Read and Write Existing**
```python
# Update specific rows in existing file
# Append to existing data
# Modify computed results
data = np.memmap('existing_data.dat', mode='r+', ...)
data[100:200] = new_values
data.flush()
```

✅ **'w+' - Create New File**
```python
# Initialize new dataset
# Overwrite old results
# Start fresh
data = np.memmap('new_data.dat', mode='w+', shape=(1000, 100))
data[:] = np.zeros((1000, 100))
data.flush()
```

✅ **'c' - Copy-on-Write (Rare)**
```python
# Testing modifications without affecting original
# Temporary transformations
# Safe experimentation
test_data = np.memmap('production.dat', mode='c', ...)
test_data[:] *= 2  # Doesn't affect file
```

**The Learning**

- ❌ 'c' mode changes don't persist - only in memory
- ❌ Wrong mode = lost work or runtime errors
- ✅ Use 'r' for read-only, 'r+' for updates, 'w+' for new files
- ✅ Always call flush() after writes (except in 'r' mode)
- ✅ Document which mode your code expects

## Real-World Performance

**Our Production Pipeline**

Before and after implementing memory-mapped arrays for our large-scale data processing:

| Metric | Before (Regular Arrays) | After (Memory-Mapped) | Improvement |
|--------|------------------------|----------------------|-------------|
| RAM usage | Near maximum | Minimal | 10-20× less |
| Startup time | Minutes | Seconds | 100×+ faster |
| Processing time | Many hours | Much faster | 2-4× faster |
| Can process dataset? | ❌ No (OOM) | ✅ Yes | Actually works! |
| Parallel workers | Few (memory limited) | Many (CPU limited) | 3-5× parallelism |

**What Made the Difference**

1. **Zero data duplication**: Multiple workers sharing data vs multiple copies
2. **Instant access**: No loading time, start processing immediately
3. **OS caching**: Operating system optimizes disk access patterns
4. **Lower memory pressure**: No GC pauses during critical processing

**Cost Savings**

- Avoided expensive RAM upgrades
- Reduced compute time significantly
- Enabled faster iteration cycles

## When NOT to Use Memory Mapping

Memory mapping isn't always the answer. We learned this the hard way.

**❌ Case 1: Small Datasets**

```python
# Dataset: Small (easily fits in RAM)
# Regular array: Fast
# Memory-mapped: Slower (overhead exceeds benefit)
```

Overhead of memory mapping + OS page management exceeds benefits for small data.

**Rule of thumb**: If it fits comfortably in RAM (< 50% of available), load it normally.

**❌ Case 2: Heavy Random Access**

We tried using memory-mapped arrays for a Monte Carlo simulation with random sampling:

| Pattern | Regular Array | Memory-Mapped | Winner |
|---------|--------------|---------------|--------|
| Sequential | Fast | Fast | Tie |
| Random (many samples) | Fast | Very slow | Regular (10-50× faster) |

Every random access = potential cache miss = disk seek

**Rule of thumb**: If your algorithm is inherently random-access heavy, load chunks into RAM.

**❌ Case 3: Network File Systems**

Tried running memory-mapped processing on network-mounted storage:

| Storage | Access Time | Reliability |
|---------|-------------|-------------|
| Local SSD | Fast | ✅ Excellent |
| Local HDD | Moderate | ✅ Good |
| NFS/Network | Slow & variable | ❌ Unreliable |

Network latency + memory mapping = unpredictable performance

**Rule of thumb**: Always use local storage for memory-mapped files.

**❌ Case 4: Frequent Full Modifications**

If you're constantly rewriting the entire array:

```python
# Bad use case: Updating everything repeatedly
for epoch in range(n_epochs):
    data[:] = transform(data)  # Full array rewrite
    data.flush()  # Slow disk write
```

**Better approach**: Load into RAM, process, write back once at the end.

**✅ When Memory Mapping Shines**

- Dataset > 50% of available RAM
- Sequential or semi-sequential access patterns
- Local storage (SSD preferred)
- Multiple processes reading same data
- Partial updates (not full rewrites)
- Long-running processes that can be checkpointed

## The Bottom Line

Memory-mapped arrays in NumPy solved our "dataset too large" problem and gave us unexpected benefits:
- Faster startup
- Better parallelism
- Lower costs

**Key Insights**

The biggest wins came from:
- ✅ Understanding access patterns (sequential > random)
- ✅ Proper flushing strategy (balance safety vs performance)
- ✅ Storing metadata separately (dtype + shape)
- ✅ Using right mode for each use case
- ✅ Validating immediately, not later

Not from:
- ❌ Blindly using memory mapping everywhere
- ❌ Ignoring the nature of our workload
- ❌ Assuming it would magically fix everything

**The Golden Rules**

1. **Profile first**: Measure if memory mapping actually helps your use case
2. **Design for sequential access**: Most speed comes from this
3. **Store metadata**: dtype and shape must be documented
4. **Flush strategically**: Balance data safety with performance
5. **Validate early**: Catch corruption immediately, not hours later
6. **Use local storage**: Network file systems perform poorly
7. **Know when to load into RAM**: Small or random-access data

**Recommended Starting Point**

If you're working with large datasets:

```python
import numpy as np
import json

# 1. Create with metadata
data = np.memmap('data.dat', dtype='float32', 
                mode='w+', shape=(n_samples, n_features))

# 2. Store metadata
with open('data_meta.json', 'w') as f:
    json.dump({
        'shape': data.shape,
        'dtype': str(data.dtype)
    }, f)

# 3. Populate with periodic flushing
batch_size = 1000
for i in range(0, len(data), batch_size):
    data[i:i+batch_size] = generate_batch()
    if i % (batch_size * 10) == 0:
        data.flush()

# 4. Final flush
data.flush()
del data

# 5. Load in workers
def worker(chunk_id):
    with open('data_meta.json') as f:
        meta = json.load(f)
    
    data = np.memmap('data.dat',
                    dtype=meta['dtype'],
                    mode='r',
                    shape=tuple(meta['shape']))
    
    # Process your chunk
    chunk_size = 1000
    return process(data[chunk_id*chunk_size:(chunk_id+1)*chunk_size])
```

**Further Reading**

- [NumPy memmap documentation](https://numpy.org/doc/stable/reference/generated/numpy.memmap.html)
- [Understanding OS page cache](https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html)
- [HDF5 for complex datasets](https://www.h5py.org/)

---