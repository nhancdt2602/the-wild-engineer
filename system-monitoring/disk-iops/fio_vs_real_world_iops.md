# Why does fio report 89K–220K IOPS while iostat shows my disk barely doing 1,000 writes/sec at 90% utilization?

## Introduction
I started with a simple goal: measure the IOPS of my laptop's NVMe SSD. `fio` gave me a clean number, I moved on, and a few days later I happened to glance at `iostat` while doing normal work:

```
Device      w/s    w_await   aqu-sz   %util
nvme0n1   1092.00     0.84     1.00    90.70
```

90% utilization, but only ~1,100 writes/sec — nearly **100x lower** than the ~89K IOPS `fio` had just reported for the same drive. Was the benchmark lying? Was the disk somehow crawling despite looking "almost full"?

Neither. Both numbers are correct. They're measuring completely different things, and reconciling them turned into a much more interesting investigation than the original benchmark.

## The baseline benchmark
Standard 4K random I/O test, queue depth 32, direct I/O (bypassing page cache):

```bash
fio --name=randread --rw=randread --bs=4k --size=1G \
    --numjobs=1 --iodepth=32 --runtime=15 --time_based \
    --direct=1 --ioengine=io_uring --group_reporting
```

| Test | IOPS | Bandwidth |
|---|---|---|
| Random Read (QD32) | ~204,000 | 797 MiB/s |
| Random Write (QD32) | ~89,000 | 348 MiB/s |

Reads roughly 2x writes, which is typical for NAND flash — writes need an erase-then-program cycle, reads don't. Nothing surprising yet.

## The missing variable: queue depth
The `fio` test used `--iodepth=32`: 32 write requests in flight at once. NVMe drives get their huge IOPS numbers from **parallelism** — many internal channels and queues — not from being fast at any single request.

The real-world `iostat` snapshot above showed `aqu-sz ≈ 1.00` — only one request in flight at a time. That's synchronous I/O: issue a write, wait for it to finish, issue the next. No parallelism to exploit.

At queue depth 1, throughput is capped by pure latency. By Little's Law:

```
concurrency ≈ throughput × latency
IOPS ≈ 1 / w_await = 1 / 0.00084s ≈ 1,190 IOPS
```

That's almost exactly the observed ~1,092–1,355 w/s. The disk wasn't slow — the workload simply wasn't concurrent enough to expose the drive's real ceiling.

## Why %util was 90% anyway
`%util` on Linux measures **the percentage of time the device had *any* outstanding I/O** — a metric that dates back to single-spindle disks, where "busy 90% of the time" really did mean "near saturation," because a spinning disk could only serve one request at a time.

NVMe drives can serve thousands of requests concurrently. A workload issuing one request at a time can keep the device "busy" in the `%util` sense almost constantly, while using a tiny fraction of its real capacity.

> `%util` ≈ 100% tells you the disk was never fully idle. It tells you nothing about whether it's saturated.

## Capturing the benchmark itself
To see the mechanics directly, I re-ran the write benchmark with `iostat -x -d 1` sampling once per second throughout the run:

```bash
iostat -x -d 1 20 nvme0n1 > iostat.log &
fio --name=randwrite --rw=randwrite --bs=4k --size=1G \
    --iodepth=32 --runtime=15 --time_based \
    --direct=1 --ioengine=io_uring --group_reporting
```

The per-second trace revealed something the single summary number completely hid:

| Phase | w/s | aqu-sz | w_await |
|---|---|---|---|
| Ramp-up | 1K → 240K | 0.5 → 4.2 | ~0.02ms |
| Peak sustained | 220K–240K | ~4–6 | ~0.02–0.03ms |
| **Sudden collapse** | 6,959–29,028 | **18 → 31** | **3.7–4.5ms (150x jump)** |
| Recovery | 83,628 → ... | back to ~2.7 | back down |

For a few seconds, the queue backed up to nearly its full configured depth (`aqu-sz` ~30), throughput cratered, and latency spiked 150x — all at once. Full queue + high latency + low throughput is the signature of hitting a wall *downstream* of the queue, not a benchmark artifact.

This lined up with `fio`'s own reported variance for that run: `iops: min=6508, max=268854, stdev=97782` — a 40x spread hiding inside a single averaged number.

## The SLC cache cliff
A 15-second test wasn't long enough to understand the pattern, so I extended the write benchmark to 60 seconds with continuous `iostat` sampling. The dip wasn't a one-off — it repeated:

```
t=4–22s   : ~50-70K IOPS      (aqu-sz ~2,  w_await ~0.03ms)   — warming up
t=23–31s  : ~175-220K IOPS    (aqu-sz ~7,  w_await ~0.03ms)   — fast cache, healthy
t=32–36s  : ~7K IOPS          (aqu-sz ~30, w_await ~4ms)      — CACHE EXHAUSTED
t=37–53s  : ~150-220K IOPS    (aqu-sz ~7,  w_await ~0.03ms)   — recovered
t=54–56s  : ~7K IOPS          (aqu-sz ~30, w_await ~4ms)      — CACHE EXHAUSTED AGAIN
t=57–63s  : ~75-223K IOPS     (aqu-sz ~7,  w_await ~0.03ms)   — recovered
```

Splitting all steady-state samples by whether the queue was backed up (`aqu-sz > 15`):

| State | Avg w/s |
|---|---|
| Normal (cache has headroom) | **~137,000** |
| Cache exhausted (aqu-sz 27–31) | **~23,000** |

Most consumer NVMe SSDs front their slower native flash (TLC/QLC) with a fast SLC (or pseudo-SLC) write cache. Incoming writes land in the fast cache and get acknowledged quickly, while a background process flushes them to native flash. Under sustained heavy random writes, the cache fills faster than it drains — once it's full, writes have to wait on the slower native media directly, and both latency and throughput fall off a cliff for a few seconds until headroom is recovered.

This is why `fio`'s single-number average was noisy run to run (89K, then 114K, then 119K across three separate runs): it depends on how much of the run happened to land inside a cache-exhaustion dip versus the healthy plateau. A short benchmark can accidentally sample only the fast-cache regime and wildly overstate sustained performance.

## So how do you reliably tell if a disk is saturated?
This is the actual production question, and the investigation above answers it.

**Don't trust `%util` on NVMe/SSDs.** It's a legacy single-spindle metric. A device can show 90%+ util at a tiny fraction of its real IOPS ceiling if the workload isn't concurrent, as shown above.

**Watch the latency-vs-throughput relationship, not either alone.** By Little's Law, below saturation, throughput scales with added concurrency while latency stays near the device's baseline service time. At saturation, throughput flattens or drops while latency spikes — exactly the pattern captured above (`aqu-sz` high, `w_await` 100x+ baseline, `w/s` collapsing). The *combination* is the signal, not any single metric in isolation.

**Benchmark first to know the ceiling.** "Saturated" is meaningless without a reference. Run `fio` at a range of queue depths ahead of time to learn: (a) the device's peak IOPS, (b) its baseline unloaded latency, and (c) how long it can sustain peak before hitting a cache cliff. Only then can you tell whether a live number is "near the ceiling" or "just a low-concurrency workload."

**Use `/proc/pressure/io` (PSI) as the real system-level signal.** It's purpose-built for exactly this question and immune to the `%util` trap:

```bash
$ cat /proc/pressure/io
some avg10=93.95 avg60=89.95 avg300=44.47
full avg10=79.64 avg60=75.62 avg300=37.20
```

- `some` — % of time *at least one* task was stalled on I/O
- `full` — % of time *every* runnable task was stalled on I/O

`full` is the number that matters: it can't be fooled by a single queue-depth-1 process looking "busy." A sustained high `full avg10`/`avg60` means I/O is genuinely bottlenecking the whole system — a much stronger and more honest saturation signal than anything derivable from `%util`.

## Warning
A single IOPS number from a benchmark is a ceiling under specific conditions (queue depth, run length) — not a constant your drive always delivers. Most real-world I/O is far less concurrent than synthetic benchmarks, and NVMe write caches are finite, so short benchmarks can silently measure only the fast-cache regime. When judging whether a disk is actually the bottleneck in a real system, don't reach for `%util` — look at latency, queue depth, and throughput together, and prefer `/proc/pressure/io` for a system-wide answer.
