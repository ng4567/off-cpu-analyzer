# eBPF Off-CPU and Blocked-Time Profiler

A Linux scheduler profiler written in C and eBPF/libbpf. It measures how long
threads spend away from a CPU and how much of that time is spent blocked before
being returned to a run queue.

The profiler was also used to examine the startup of an LLM inference server on
a dual-H100 system. The resulting distributions were consistent with an
I/O-bound model-loading phase dominated by host-to-GPU data movement.

![Illustration of blocked and off-CPU time](images/fig1.png)

## What it measures

- **Off-CPU time:** elapsed time from a thread being scheduled out to being
  scheduled back onto a CPU.
- **Blocked time:** elapsed time from a thread entering a sleeping state to
  being woken and re-enqueued.

Comparing the two distributions helps distinguish time spent sleeping on I/O
from time spent runnable but waiting for CPU time.

## Implementation

The kernel-side program attaches to the following scheduler tracepoints:

- `sched:sched_switch`
- `sched:sched_wakeup`
- `sched:sched_wakeup_new`

It uses BPF hash and array maps to track scheduling timestamps and blocked-time
histograms. Completed off-CPU samples are sent to userspace through a BPF ring
buffer. The userspace libbpf program resolves thread IDs to process IDs,
aggregates samples, and periodically prints log2 latency histograms.

Key files:

- [`cpu_analyzer.bpf.c`](cpu_analyzer.bpf.c): kernel-side tracepoint programs
  and BPF maps.
- [`cpu_analyzer.c`](cpu_analyzer.c): libbpf loader, ring-buffer consumer,
  aggregation, and histogram output.
- [`logs-moe.txt`](logs-moe.txt): measurements from the LLM model-loading
  experiment.

## Requirements

- Linux with BTF information available at `/sys/kernel/btf/vmlinux`
- Clang
- libbpf development headers and libraries
- bpftool
- Root privileges to load and attach BPF programs

The project was developed and tested on Debian 12.

## Build

```bash
make
```

The Makefile generates `vmlinux.h`, compiles the BPF object, and links the
userspace program against libbpf.

## Usage

```bash
sudo ./cpu_analyzer <interval_seconds> [pid]
```

Examples:

```bash
# Print system-wide histograms every five seconds
sudo ./cpu_analyzer 5

# Filter userspace off-CPU aggregation to process 1234
sudo ./cpu_analyzer 5 1234
```

### PID-filter limitation

The optional PID currently filters the userspace off-CPU aggregation only. The
blocked-time histogram is maintained in one system-wide BPF map and is not
filtered by process. Supporting per-process blocked-time histograms would
require filtering by TGID in the BPF program or keying the blocked-time map by
TGID.

## Example: profiling LLM model loading

The profiler was run during startup of the LLM inference server used for the
[Harvest](https://arxiv.org/abs/2602.00328) research prototype. The test
system was an Azure NC80adis H100 v5 VM with 80 vCPUs, 640 GB of RAM, and two
NVIDIA H100 GPUs connected by NVLink.

During the measurement window, the server was loading model weights from host
memory into GPU memory over PCIe. The off-CPU and blocked-time histograms were
nearly identical across the dominant latency buckets. This is consistent with
threads spending most of their off-CPU time blocked on I/O rather than waiting
runnable for a CPU.

The experiment traced all processes, so the result should be interpreted as a
system-level workload characterization rather than attribution to one process.
The complete five-second interval output is available in
[`logs-moe.txt`](logs-moe.txt).

## Current limitations

- Blocked-time measurements are system-wide even when a PID is supplied.
- Thread-to-process resolution happens in userspace through `/proc`, so very
  short-lived threads may exit before they can be resolved.
- The profiler reports latency distributions but does not collect stack traces
  or attribute blocking to individual kernel functions.
- The current implementation provides process and system views, not cgroup
  filtering.

## Attribution

The project uses Troy D. Hanson's
[`uthash`](https://github.com/troydhanson/uthash). The vendored `uthash.h` file
is not original project code.
