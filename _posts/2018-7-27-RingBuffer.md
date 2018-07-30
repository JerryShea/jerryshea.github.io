---
layout: post
title: Reducing tail latencies with a Ring Buffer
---

## Background
[Chronicle Queue](https://github.com/OpenHFT/Chronicle-Queue/) is a pure Java low-latency unbounded persisted queue.
When benchmarking Chronicle Queue, latencies tend to be excellent up to high percentiles until sustained throughput limits are reached; these sustained throughput limits are determined by disk performance, OS version and configuration. Unfortunately it seems that - despite OS virtual memory pages (and thus the disk) being used in a predictable forward-only manner, sometimes the OS introduces a pause in processing. We can see these in the graphs below.

Chronicle Queue works by reading from and writing to memory-mapped files. These files are mapped into the process' address space in fixed size pages. Pages are loaded on-demand as the queue is read from or written to. The OS keeps these pages in the page-cache in case they will need to be accessed frequently. As system memory is used, it is necessary for the OS to free up space in the page-cache, by copying the contents of memory to disk.

> All timings in this article make use of the following benchmarking methodology: using [JLBH](https://github.com/OpenHFT/Chronicle-Core/#jlbh) we send a **sustained** number of 32 byte messages per second (accounting for coordinated omission) and these messages are timestamped then written in to a Chronicle Queue. Another thread waits for these messages to arrive and picks them up, recording the difference between the time after reading vs the timestamp from before writing and records in a histogram. All experiments were run on an un-tuned CentOS server.

### Chronicle Queue, 50K msgs/sec of 32 bytes
We execute 8 runs of 200sec each, each run writes ~320MB. Latencies increase with increasing percentiles but they remain reasonably low, and after the first two runs, the worst 99.99th percentile is under 25μs.
```
-------------------------------- SUMMARY (end to end microseconds)---------------------------------------------
Percentile   run1         run2         run3         run4         run5         run6         run7         run8      
50:             0.37         0.37         0.37         0.37         0.37         0.37         0.37         0.37 
90:             0.44         0.43         0.42         0.43         0.43         0.42         0.44         0.43
99:             3.13         0.52         0.73         0.52         0.51         0.51         3.05         0.51
99.7:           4.13         2.11        10.80         4.37         2.17         3.20         4.08         1.83
99.9:           6.29         8.30        13.69        13.66         7.67        11.39         6.75         4.09
99.97:         15.23        15.28        15.64        16.25        14.51        14.47        14.51        13.39
99.99:        510.59        97.57        24.15        21.58        17.06        18.18        22.54        17.67
```

### Chronicle Queue, 250K msgs/sec of 32 bytes
We execute 8 runs of 100sec each, each run writes ~800MB.  Here we see high percentile pauses appearing. We can see that latencies are low and stable beneath the 99.97th percentile but, once there, they
start to increase to the low milliseconds
```
-------------------------------- SUMMARY (end to end - microseconds)-------------------------------------------
Percentile   run1         run2         run3         run4         run5         run6         run7         run8     
50:             0.38         0.37         0.37         0.38         0.38         0.38         0.38         0.38 
90:             0.43         0.42         0.42         0.43         0.42         0.43         0.42         0.42
99:             2.43         2.54         2.46         2.26         2.46         2.43         2.58         2.29
99.7:           7.02         7.73         4.66         3.15         4.65         8.68         6.37         3.38
99.9:          14.88        14.76        12.89        12.91        11.71        14.80        14.40        12.16
99.97:        823.04      1065.47      1069.57      2122.75       793.34       852.22      1071.62      1090.05
99.99:       1839.62      2561.02      2565.12      3873.79      2356.22      2339.84      2421.76      2581.50
```

### Chronicle Queue with pretoucher, 250K msgs/sec of 32 bytes
If Chronicle-Queuer's pretoucher is used, this improves the high percentiles, but they are still sub-optimal.

> To minimise latency of writes, the [pretoucher](https://github.com/OpenHFT/Chronicle-Queue/blob/9a56a86bf7f489d838d030a6486570cfb1b5cb15/src/main/java/net/openhft/chronicle/queue/impl/single/Pretoucher.java) is designed to be run in a separate thread (or process) and aggressively 'pre-touch' the pages in a queue, so that they are resident in the page-cache (i.e. loaded from storage) before they are required by the application.

```
-------------------------------- SUMMARY (end to end - microseconds)-------------------------------------------
Percentile   run1         run2         run3         run4         run5         run6         run7         run8    
50:             0.37         0.37         0.37         0.37         0.37         0.37         0.37         0.37 
90:             0.42         0.42         0.42         0.42         0.42         0.42         0.41         0.42
99:             2.98         1.41         1.35         1.40         1.42         1.37         1.34         1.34
99.7:          12.63         2.75         2.65         2.79         3.34         3.73         3.08         2.88
99.9:          16.16        10.74        10.40        11.35        12.53        13.14        11.44        11.52
99.97:       1003.78       457.86       427.65       734.46       269.44       441.47       412.03       418.69
99.99:       2112.51      1862.14      1802.75      1954.30      1578.50      1783.30      1618.43      1696.26
```

## Why do we see these delays and how to fix

The short answer is that the OS is introducing the delays, because it is stalling the process while waiting for a page to be read from or written to disk.
* Making use of Chronicle Queue's pretoucher helps, and research continues into how to make the pretoucher work more effectively.
* Map the queue in to /tmpfs if it is small enough and you have a suitable replication or backup strategy.
* Tuning of the BIOS and OS is effective, although requires patience and expertise: power states, BIOS, kernel, RAID, even changing to an alternative file system.

But, a straightforward way to mitigate this (if you have Enterprise Chronicle Queue) is to set a couple of parameters when creating your queue:

```
    builder.readBufferMode(BufferMode.Asynchronous);
    builder.writeBufferMode(BufferMode.Asynchronous);
    builder.bufferBytesStoreCreator(RB_BYTES_STORE_CREATOR_MAPPED_FILE_TMPFS);
```

which configures Chronicle Queue to create a ring buffer over the top of the queue to absorb any latencies from the OS. In the above example,
the Ring Buffer is mapped to a file (stored in this case in /tmpfs which is a high-speed file system) so that other processes can see it. All reads and writes from/to the queue
go to the ring buffer, and a separate handler is registered to drain any writes made to the ring buffer to a the underlying queue.

### Enterprise Chronicle Queue backed with ring buffer, 250K msgs/sec of 32 bytes
With the ring buffer, pauses at high percentiles are drastically reduced by the above 3 lines of code.
```
-------------------------------- SUMMARY (end to end - microseconds)-------------------------------------------
Percentile   run1         run2         run3         run4         run5         run6         run7         run8   
50:             0.84         0.85         0.85         0.85         0.85         0.85         0.85         0.86
90:             1.21         1.23         1.21         1.23         1.22         1.21         1.23         1.21
99:            12.12        11.69        10.81        11.48        11.49        11.48        12.12        11.12
99.7:          15.17        14.15        13.51        14.08        14.06        13.96        14.58        14.21
99.9:          18.70        16.08        14.75        16.00        16.11        16.07        16.90        16.74
99.97:         30.14        21.98        16.62        21.88        27.72        24.31        24.58        25.19
99.99:        594.69        31.24        27.82        30.54        30.30        32.12        30.81        33.07
```
