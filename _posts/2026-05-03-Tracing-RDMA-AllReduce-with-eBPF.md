---
layout: post
title: "Tracing RDMA AllReduce with eBPF"
date: 2026-05-03
categories: [Distributed Systems, Networking]
tags: [rdma, ebpf, nccl, infiniband, ai-infrastructure, networking]
---

Most explanations of NCCL and InfiniBand stay at a high level. I wanted to understand what actually happens at the kernel level during GPU-to-GPU data transfer, so I built an RDMA ring allreduce from scratch and traced it with eBPF. This is what I found.

## Why RDMA Matters for AI Training

During a training step, every GPU computes gradients for its slice of the model. Before the next step, all GPUs need to agree on the summed gradients: this is the AllReduce operation. At scale, this communication dominates training time.

RDMA (Remote Direct Memory Access) is what makes this fast. Unlike regular networking, RDMA lets one machine write directly into another machine's memory without involving the remote CPU. No kernel overhead, no copies, just direct memory-to-memory transfers at wire speed.

Real InfiniBand links run at 200 Gbit/sec with ~1 microsecond latency. For comparison, a regular 10GbE TCP connection has ~50-100 microsecond latency and burns CPU on every transfer.

## Setting Up the Environment

I used an Ubuntu 22.04 EC2 instance. Since I don't have real InfiniBand hardware, I used SoftRoCE, a kernel module that emulates RDMA on top of regular Ethernet. It's slower than real hardware but the programming model is identical.

```bash
# Install RDMA stack
sudo apt update && sudo apt install -y rdma-core libibverbs-dev ibverbs-utils infiniband-diags iproute2

# Load SoftRoCE kernel module
sudo modprobe rdma_rxe

# Create a virtual RDMA device on top of your ethernet interface
sudo rdma link add rxe0 type rxe netdev ens5

# Verify it's up
ibv_devices
rdma link show
```

When `ibv_devices` shows `rxe0` with state ACTIVE, you have a working RDMA device.

## The RDMA Verbs API

RDMA is programmed through the "verbs" API via `libibverbs`. The key objects you create, in order:

**Context**: your handle to the RDMA device. Everything else hangs off this.

**Protection Domain (PD)**: a security boundary. Memory regions and queue pairs in the same PD can interact; cross-PD access is rejected by hardware.

**Memory Region (MR)**: before RDMA can touch a buffer, the kernel must pin it in physical memory so it can't be swapped out mid-transfer. `ibv_reg_mr()` does this and gives you `lkey`/`rkey`: keys that authorize access to that memory.

**Completion Queue (CQ)**: RDMA operations are asynchronous. When one finishes, a Work Completion lands here. You poll it to find out what happened.

**Queue Pair (QP)**: the actual communication endpoint. A QP has a send queue and a receive queue. Two QPs connect to each other to form a channel.

## The QP Lifecycle

A QP starts in RESET state and must be walked through state transitions before it can transfer data:

```
RESET → INIT → RTR (Ready to Receive) → RTS (Ready to Send)
```

Each transition configures different things:
- **RESET → INIT**: attach to a port, set access permissions
- **INIT → RTR**: provide the remote QP's address (QPN, GID, PSN)
- **RTR → RTS**: set timeout and retry parameters

This is the part I found most surprising, there's a whole handshake before a single byte moves. The two sides exchange QP info out-of-band (over TCP, or in our case shared memory), then each side transitions its QP independently.

## Building a Ring AllReduce

Ring AllReduce is the algorithm NCCL uses for gradient aggregation. In a ring of N processes, each process sends to its right neighbor and receives from its left neighbor. After N-1 steps, every process has the sum of all values.

I implemented this with `fork()`, three processes sharing memory via `mmap(MAP_SHARED)`:

```
rank0 → rank1 → rank2 → rank0 (ring)

Initial values: rank0=1.0, rank1=2.0, rank2=3.0

Step 0: everyone sends their own value right
  rank0 receives 3.0, accumulates: 1+3 = 4.0
  rank1 receives 1.0, accumulates: 2+1 = 3.0
  rank2 receives 2.0, accumulates: 3+2 = 5.0

Step 1: everyone forwards what they received
  rank0 receives 2.0, accumulates: 4+2 = 6.0 ✓
  rank1 receives 3.0, accumulates: 3+3 = 6.0 ✓
  rank2 receives 1.0, accumulates: 5+1 = 6.0 ✓
```

The critical ordering: post recv *before* send, and use an atomic barrier to make sure all processes have armed their recv buffers before anyone sends. I got this wrong the first time and hit a deadlock.

## eBPF Tracing

I used bpftrace to hook into the kernel's RDMA operations. The SoftRoCE module (`rdma_rxe`) exposes fentry hooks for the key operations:

```
fentry:rdma_rxe:rxe_modify_qp    ← QP state transitions
fentry:rdma_rxe:rxe_post_send    ← send work request posted
fentry:rdma_rxe:rxe_post_recv    ← recv work request posted
fentry:rdma_rxe:rxe_poll_cq      ← completion harvested
```

The `modify_qp` hook is particularly useful, you can read `args->attr->qp_state` to see which state the QP is transitioning to:

```
fentry:rdma_rxe:rxe_modify_qp {
    $state = args->attr->qp_state;
    if ($state == 1) { printf("RESET->INIT pid=%d\n", pid); }
    else if ($state == 2) { printf("INIT->RTR  pid=%d\n", pid); }
    else if ($state == 3) { printf("RTR->RTS   pid=%d\n", pid); }
}
```

Running this against the ring allreduce, you can watch each process walk through the QP lifecycle in real time:

```
TIME(us)   EVENT       PID    DETAIL
2060272    modify_qp   7096   RESET->INIT
2060289    modify_qp   7096   RESET->INIT   # two QPs per process
2063676    modify_qp   7096   INIT->RTR
2063688    modify_qp   7096   RTR->RTS
2063773    modify_qp   7096   INIT->RTR
2063778    modify_qp   7096   RTR->RTS
2063791    post_send   7096   step=1
2063985    post_send   7096   step=2
```

### What SoftRoCE Can't Show

Here's something I hit: `rxe_poll_cq` and `rxe_post_recv` don't fire on SoftRoCE even though the symbols exist. These functions are called via function pointers in the `ib_device_ops` struct: fentry hooks on direct function calls, not indirect ones. The `rdma_core:cq_poll` tracepoint exists but also doesn't fire on SoftRoCE, it seems to require real HCA hardware.

This is actually an important insight for the blog post: **eBPF visibility depends on the hardware path**. On real InfiniBand with a Mellanox card, you'd get much richer tracepoints. SoftRoCE is great for learning but limits what you can observe.

## Latency Results

I fell back to `clock_gettime()` for measuring per-step latency since eBPF couldn't capture completions:

```
Step 0:
  rank2: 33 us
  rank0: 51 us
  rank1: 1664 us  ← OS scheduler jitter

Step 1:
  rank0: 31 us
  rank2: 34 us
  rank1: 859 us
```

rank0 and rank2 complete each step in 30-50 microseconds. rank1 shows high jitter, possibly OS scheduling on EC2. These numbers include barrier overhead, not just wire time. Real InfiniBand raw latency is ~1-2 us per message.

That gap matters at scale. With 1000 GPUs, even 1ms of extra latency per AllReduce step adds up to seconds per training iteration.

## What I'm Still Figuring Out

A few things I don't fully understand yet:

- **Congestion control on RoCE**: how ECN and PFC work to prevent packet drops on lossless fabrics. I know it matters but haven't dug into it.
- **NCCL's transport selection**: NCCL can use RDMA, shared memory, or TCP depending on topology. I haven't traced how it decides.
- **Multi-rail**, large clusters use multiple NICs per node for bandwidth aggregation. I don't know how NCCL manages QPs across multiple rails.

Next step is to get access to a real GPU cluster and run actual NCCL workloads. The SoftRoCE setup is useful for understanding the mechanics, but the real learning will come from tracing actual training runs.

## Code

Everything in this post is in my GitHub repo: [nccl-rdma-tracer](https://github.com/bkpathak/nccl-rdma-tracer)

The repo has:
- `rdma-primer/rdma_pingpong.c`: basic QP lifecycle
- `rdma-primer/ring_allreduce.c`: ring allreduce over RDMA
- `ebpf/nccl_tracer.bt`: bpftrace script

## References

- [RDMA Aware Networks Programming User Manual](https://docs.nvidia.com/networking/display/RDMAAwareProgrammingv17): Nvidia/Mellanox
- [Linux kernel rdma_rxe driver](https://github.com/torvalds/linux/tree/master/drivers/infiniband/sw/rxe)
- [NCCL source code](https://github.com/NVIDIA/nccl)
- [bpftrace reference guide](https://github.com/bpftrace/bpftrace/blob/master/docs/reference_guide.md)