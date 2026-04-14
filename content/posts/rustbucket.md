---
title: "Rustbucket"
date: 2026-04-13T20:38:18-05:00
draft: false
tags: [Fedora, io_uring, storage, performance, nvme, benchmark, historical]
---

Sorting a terabyte of data in the late 1990s meant serious hardware, serious
planning, and probably a serious budget approval process. Today you can do it
on a workstation before lunch. I wanted to know how fast, so I wrote
[rustbucket](https://github.com/tasleson/rustbucket) to find out.

It's a two-phase external sort implemented in Rust, built around `io_uring`,
and named for reasons that should be obvious to anyone who has spent time
with either Rust or storage systems.

## Some History Worth Knowing

The [Sort Benchmark](http://sortbenchmark.org/) came out of the database
research community in the late 1990s as a way to measure whole-system
performance — not just CPU, but memory, I/O scheduling, and the disk
subsystem together. A few variants got traction:

- **PennySort** — a "bang-for-the-buck" measure: Amount of data that can be sorted for a penny's worth of system time
- **MinuteSort** — sort as much data as possible in one minute
- **TerabyteSort** — sort 1 TB as fast as possible

To get a feel for what TerabyteSort actually required at the time, here's
a passage from an nsort white paper on a September 1997 run using an SGI
Origin2000:

```
In order to demonstrate Nsort's ability to sort large data sets, we
sorted a terabyte of data (10,000,000,000 100-byte records with random
10-byte keys) in 2.5 hours. The Origin2000 system for this September
1997 result included 32 processors, 8GB of main memory, and 559 4GB
disks:

  • 1 system disk
  • a 280-disk XLV volume for input and output files
  • 278 temporary disks
```

**559 disks.** For a single benchmark run. Aggregate throughput came out
around 110 MB/s, which was genuinely impressive given what they were
working with.

A single Gen4 NVMe today does 5–7 GB/s sequential. The gap is not subtle.

## The Tool

rustbucket has three subcommands. Nothing fancy.

**`create`** generates a test file of randomly-keyed 128-byte records:

```bash
rustbucket create 1tib_input.dat 1T
```

Each record is 128 bytes — 16-byte `u128` key followed by 112 bytes of
payload. Not byte-for-byte compatible with the original benchmark's 100-byte
ASCII format, but close enough to be a meaningful comparison. Getting closer
to the original spec is on the list.

**`sort`** does the actual work:

```bash
rustbucket sort [OPTIONS] <INPUT> <OUTPUT> <SCRATCH>...

Options:
  -m, --memory <MEMORY>    Total memory budget (e.g. 16G, 512M) [default: 4G]
  -t, --threads <THREADS>  Number of worker threads. 0 = use all available [default: 0]
      --remove-input       Remove input after scatter phase to free space
```

Multiple scratch directories can be spread across separate NVMe devices.
Do this. It matters.

**`verify`** checks that the output is actually sorted:

```bash
rustbucket verify <FILE>
```

Skipping verification is an option, technically.

## How It Works

Two phases: scatter, then gather.

### Scatter

The input is read sequentially. Each record gets binned by a prefix of
its key, buffered in memory, and flushed to a scratch file on disk once
the buffer is large enough. The number of bins is derived from the input
size and the memory budget, less memory means more bins, each covering a
narrower slice of the keyspace. The scatter is single-threaded; it's
I/O-bound and adding threads here doesn't help much. Scratch directories
are spread across devices to keep the write pressure distributed.

The memory budget is a soft limit. The tool estimates at runtime and uses
those estimates to drive flush decisions. It won't blow past the budget
wildly, but don't treat the number as a hard ceiling.

### Gather

Once everything is scattered, the gather phase processes each bin through
a three-stage pipeline running concurrently:

- **reader** — loads a bin from scratch into memory
- **sorter** — sorts it using rayon's `par_sort_unstable_by`
- **writer** — writes the sorted bin to the output file

Each stage hands off to the next, so reading, sorting, and writing are
all happening at the same time across consecutive bins. The per-bin
timing lines in the output show exactly how this plays out — watch the
`wait-reader` times on the sorter drop to near-zero once the pipeline
is full.

The I/O layer runs on `io_uring`. That was the other reason I wrote this:
I wanted to see what kind of throughput a straightforward Rust program
could pull from NVMe using Linux's async I/O interface without a lot of
heroics.

## The 1 TiB Run

First full run was at commit `6d5c30130d62694c23ac721b2363dfae100b2263`.

Hardware:

| Component | Detail |
|-----------|--------|
| CPU | AMD Threadripper Pro 5945WX |
| Memory | 128 GiB RAM |
| Storage | 3× Gen4 NVMe SSD |

One NVMe held the input file and later the sorted output. The other two
were scratch. The Threadripper system had proper cooling on all three
drives — large heatsinks and a fan. More on why that matters later.

Dataset: 1 TiB, 8,589,934,592 records, 16 GiB memory budget.

One complication: there wasn't enough free space to keep the input around
while writing the output, so the input had to go first, same situation
the nsort team ran into in 1997. That's what `--remove-input` is for.
Once you use it, the input is gone. Plan accordingly.

### Results

```
Sort: 1024.000 GiB input, 128 bins, 8.0 MiB per bin write-buffer, 16.0 GiB memory budget
Scatter done: 315.0s  3329 MiB/s
Phase 1 done: 8589934592 records scattered
Gather done: 354.2s  2960 MiB/s
Phase 2 done: 8589934592 records written
Total: 669.3s  effective throughput 1567 MiB/s (based on input size)
```

669 seconds. Just over 11 minutes. ~1567 MiB/s effective throughput on 1 TiB.

The 1997 SGI run with 559 disks took 2.5 hours at 110 MB/s. The hardware
situation has improved somewhat.

### What the Numbers Show

Scatter started around 4,100 MiB/s and finished at 3,329 MiB/s. That
drop is mostly flash physics. Most consumer NVMe drives use TLC (3 bits
per cell) or QLC (4 bits per cell) NAND — dense, cheap, and not
particularly fast to write. To hide this, manufacturers front the drive
with an SLC write cache: a portion of the NAND operated in single-bit
mode that absorbs burst writes quickly. The impressive sequential write
numbers on spec sheets? That's the cache. Push past it with a sustained
workload and throughput falls back to what the underlying NAND can
actually sustain. A 1 TiB write will exhaust the SLC cache. There's
no way around it.

Worth noting: some drives also include a dedicated DRAM cache, separate
from the SLC NAND cache, used mainly to hold the FTL mapping tables that
translate logical block addresses to physical NAND locations. It helps
with random I/O latency, but it doesn't change the sustained write
picture — SLC exhausted means slow writes, DRAM or not.  The last consumer
NVMe that has some really flat sustained sequential writes is the Samsung 970 Pro,
but they are quite old now with PCIe gen3 interface.  Last of the 2-bit MLC NAND.

The gather pipeline takes a few bins to hit its stride. Writer throughput
on bin 0 was ~1,042 MiB/s; by bin 5 it was past 2,000 MiB/s and
eventually leveled off around 2,950–2,960 MiB/s. The slow start is just
pipeline fill — reader finishes, waits on sorter, both wait on the first
write, then everything locks in.

Per-bin sort times ran 2.6–3.2 seconds on ~8 GiB bins, and sorter
wait-on-reader time was essentially zero after bin 0.
Rayon's parallel sort on 12 cores keeps up.

One result from earlier testing worth mentioning: a smaller memory budget
sometimes outperformed a larger one. It makes sense once you think about
how bins work — less memory means more bins, each with less data, so each
gather-phase sort is working on a smaller chunk and the pipeline cycle
completes faster. More memory isn't always the right move. Experiment.

## NVMe Thermals: A Cautionary Tale

The Threadripper setup had heatsinks and active cooling on all three
drives — no problems there. Earlier development used a different, smaller
machine. Two NVMe devices, no heatsinks.

NVMe drives have thermal monitoring and self-preservation. They throttle
when hot, shut down before damage. That's what they say. I knew this going in
and wasn't particularly worried about it.

After running a 1 TiB sort on that machine, I checked the SMART data on
the drive that handled both the input read and the final output write —
the one under the most sustained load. What it showed:

- **17+ minutes** of elevated temperature
- **17 seconds** of critical temperature

The drive is still alive. Thermal protection did engage. But there's a
real difference between "the safety net caught it" and "it stayed within
rated operating limits," and 17 seconds at critical temperature is not a
number you want to see. The throttling was late enough that the drive had
already been outside its thermal envelope for a meaningful stretch before
it reacted.

Heatsinks for M.2 drives are cheap. If you're running anything that
sustains high I/O for more than a few minutes, use them. Thermal
self-protection in storage hardware is a last resort, not a cooling
solution.

## Caveats

- **Not an exact reproduction.** Original benchmark: 100-byte records,
  10-byte ASCII key. rustbucket: 128-byte records, 16-byte `u128` key.
  Close enough to be interesting, not close enough for a direct
  apples-to-apples comparison. A proper reproduction is planned — I also
  still need to pin down the exact key format from the original spec.
- **Memory budget is a soft limit.** Reasonable in practice, but it's an
  estimate-driven ceiling, not a hard cap.
- **Linux only.** `io_uring` isn't portable.
- **NVMe strongly recommended.** Running on spinning disks is possible.
  The results will be humbling.

## Building and Running

```bash
git clone https://github.com/tasleson/rustbucket
cd rustbucket
cargo build --release
```

Quick run:

```bash
./target/release/rustbucket create test_input.dat 10G
./target/release/rustbucket sort test_input.dat test_sorted.dat /mnt/scratch
./target/release/rustbucket verify test_sorted.dat
```

With multiple scratch devices and a larger budget:

```bash
./target/release/rustbucket sort input.dat sorted.dat \
    /mnt/nvme1/scratch /mnt/nvme2/scratch \
    --memory 16GiB \
    --remove-input
```

## Wrap-up

The original question was simple: how fast can a Rust program sort a
terabyte using `io_uring` and modern NVMe? Fast enough that the benchmark
requiring 559 disks and 2.5 hours in 1997 now runs in 11 minutes on a
workstation.

The hard part isn't getting I/O anymore. It's keeping up with it.

Code is at [github.com/tasleson/rustbucket](https://github.com/tasleson/rustbucket).
If you have spare NVMe capacity and a free afternoon, give it a run.
Just put heatsinks on your drives first.

There is probably a **ton** of optimizations that can be done.  Pull requests are welcome.
