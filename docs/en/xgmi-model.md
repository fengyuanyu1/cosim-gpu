[中文](../zh/xgmi-model.md)

# xGMI Interconnect Model Design

## Overview

The xGMI (inter-chip Global Memory Interconnect) model provides GPU-to-GPU
communication within a cosim-gpu multi-GPU hive.  It attaches to each GPU's
L2 cache (TCC) egress and routes remote VRAM accesses through a modeled
xGMI link with configurable bandwidth, latency, and topology.

## Packet Format

| Field    | Type   | Description                        |
|----------|--------|------------------------------------|
| src_gpu  | uint8  | Source GPU ID                      |
| dst_gpu  | uint8  | Destination GPU ID                 |
| addr     | uint64 | Target VRAM address                |
| size     | uint32 | Payload size in bytes              |
| payload  | bytes  | Data (for write operations)        |

## Address Mapping

Each GPU owns a contiguous VRAM address range:

```
GPU 0: [0, vram_size)
GPU 1: [vram_size, 2 * vram_size)
GPU N: [N * vram_size, (N+1) * vram_size)
```

The bridge determines whether an address is local or remote by checking
which GPU's range it falls into.

## Topology Configuration

Launch-time parameter `--xgmi-topology`:

- **mesh**: Every GPU has a direct link to every other GPU.
  An 8-GPU mesh creates 28 bidirectional links.
- **ring**: Each GPU connects to its two neighbors.
  Lower link count but multi-hop for non-adjacent GPUs.

## Link Parameters

| Parameter           | Default  | CLI Flag            |
|---------------------|----------|---------------------|
| Per-link bandwidth  | 128 GB/s | `--xgmi-bandwidth`  |
| Per-hop latency     | 100 ns   | `--xgmi-latency`    |
| Lanes per link      | 16       | (SimObject param)   |
| Max links per GPU   | 7        | (SimObject param)   |
| Flow-control credits| 32       | (SimObject param)   |

## Flow Control

Credit-based back-pressure prevents data loss:

1. Each link starts with N credits (default 32).
2. Sending a packet consumes one credit.
3. The receiver returns a credit upon packet acceptance.
4. When credits reach zero, the sender stalls (never drops).

## Architecture Phases

### Path A (Milestones 1-3): Self-built xGMI model

- Single-process multi-GPU (Milestones 1-2): in-process function calls
- Multi-process 8-GPU hive (Milestone 3): IPC transport via shared
  memory ring buffers or Unix sockets

### Path B (Milestones 4-5): SST Merlin integration

- Replace xGMI transport with SST Merlin network engine
- Three-layer synchronization: QEMU (functional) ↔ gem5 (GPU timing) ↔
  SST (network timing)
- Supports arbitrary topologies (fat-tree, dragonfly)

## Key Source Files

- `gem5/src/dev/amdgpu/XGMIBridge.py` — SimObject definition
- `gem5/src/dev/amdgpu/xgmi_bridge.hh` — C++ header
- `gem5/src/dev/amdgpu/xgmi_bridge.cc` — C++ implementation
- `gem5/configs/example/gpufs/mi300_cosim.py` — Configuration and wiring
