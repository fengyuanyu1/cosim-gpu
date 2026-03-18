[English](../en/xgmi-model.md)

# xGMI 互连模型设计

## 概述

xGMI（芯片间全局内存互连）模型提供 cosim-gpu 多 GPU hive 中的 GPU 间通信。
它挂载在每个 GPU 的 L2 缓存（TCC）出口端口上，将远程 VRAM 访问通过可配置的
带宽、延迟和拓扑的 xGMI 链路模型进行路由。

## 数据包格式

| 字段     | 类型   | 描述                               |
|----------|--------|------------------------------------|
| src_gpu  | uint8  | 源 GPU ID                          |
| dst_gpu  | uint8  | 目标 GPU ID                        |
| addr     | uint64 | 目标 VRAM 地址                     |
| size     | uint32 | 负载大小（字节）                   |
| payload  | bytes  | 数据（写操作时）                   |

## 地址映射

每个 GPU 拥有连续的 VRAM 地址范围：

```
GPU 0: [0, vram_size)
GPU 1: [vram_size, 2 * vram_size)
GPU N: [N * vram_size, (N+1) * vram_size)
```

桥接器通过检查地址落入哪个 GPU 的范围来判断本地或远程访问。

## 拓扑配置

启动参数 `--xgmi-topology`：

- **mesh**：每个 GPU 与所有其他 GPU 直连。8 GPU mesh 创建 28 条双向链路。
- **ring**：每个 GPU 连接其两个邻居。链路数更少但非相邻 GPU 需多跳。

## 链路参数

| 参数           | 默认值   | CLI 标志            |
|----------------|----------|---------------------|
| 每链路带宽     | 128 GB/s | `--xgmi-bandwidth`  |
| 每跳延迟       | 100 ns   | `--xgmi-latency`    |
| 每链路通道数   | 16       | （SimObject 参数）  |
| 每 GPU 最大链路 | 7       | （SimObject 参数）  |
| 流控信用       | 32       | （SimObject 参数）  |

## 流量控制

基于信用的背压机制防止数据丢失：

1. 每条链路初始 N 个信用（默认 32）。
2. 发送一个数据包消耗一个信用。
3. 接收方在接受数据包后归还信用。
4. 信用归零时发送方阻塞（永不丢弃）。

## 架构阶段

### Path A（里程碑 1-3）：自建 xGMI 模型

- 单进程多 GPU（里程碑 1-2）：进程内函数调用
- 多进程 8-GPU hive（里程碑 3）：通过共享内存环形缓冲区或 Unix socket 的 IPC 传输

### Path B（里程碑 4-5）：SST Merlin 集成

- 用 SST Merlin 网络引擎替换 xGMI 传输
- 三层同步：QEMU（功能仿真）↔ gem5（GPU 时序）↔ SST（网络时序）
- 支持任意拓扑（fat-tree、dragonfly）

## 关键源文件

- `gem5/src/dev/amdgpu/XGMIBridge.py` — SimObject 定义
- `gem5/src/dev/amdgpu/xgmi_bridge.hh` — C++ 头文件
- `gem5/src/dev/amdgpu/xgmi_bridge.cc` — C++ 实现
- `gem5/configs/example/gpufs/mi300_cosim.py` — 配置和连线
