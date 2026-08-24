# AGENTS.md — Sepia 开发辅助说明

本文件供 AI 编程代理与后续维护者使用，用于快速理解项目约定、构建方式与验收口径。详细执行计划见 `README.md`。

---

## 1. 项目摘要

Sepia 验证“KV Cache 逻辑地址固定 + 物理显存动态重映射”的推理方案。核心模块：

- `VMMAllocator`：封装 CUDA VMM API，管理虚拟地址预留与物理内存映射。
- `KVBlockManager`：在固定逻辑槽位上分配/释放 KV block。
- 推理 Demo：用 CUDA Graph 重放实现端到端推理。

## 2. 环境

| 项 | 要求 |
|----|------|
| 操作系统 | Ubuntu 22.04 |
| GPU | NVIDIA RTX 4070（12 GB） |
| 驱动 | 535+ |
| CUDA | 12.x |
| 构建系统 | CMake |
| 性能分析 | Nsight Systems |
| 测试模型 | GPT-2（首选验证）、量化 LLaMA3-8B（可选） |

## 3. 构建与测试

常用流程：

- 配置：在项目根目录执行 `cmake -S . -B build`。
- 编译：执行 `cmake --build build -j`。
- 单元测试：执行 `ctest --test-dir build --output-on-failure`。
- 显存监控：执行 `watch -n 0.5 nvidia-smi --query-gpu=memory.used --format=csv`。
- 性能分析：使用 Nsight Systems 抓取 CUDA/NVTX/OS trace。

约束：不提交 `build/`、模型权重、测试生成的 trace 文件与日志。

## 4. 仓库结构

| 路径 | 用途 |
|------|------|
| `include/sepia/` | 公共头文件 |
| `src/` | 实现与可执行程序 |
| `tests/` | 单元测试与集成测试 |
| `benchmarks/` | 性能测试脚本、配置与结果归档 |
| `docs/` | 环境检查、基线数据、API 笔记、周报、决策记录 |
| `tools/` | 辅助脚本 |

## 5. 开发顺序与依赖关系

1. 先完成 `VMMAllocator` 并通过单元测试，再开始 `KVBlockManager`。
2. 先跑通纯 PyTorch + `cudaMalloc` 的 CUDA Graph 基线，再替换为 Sepia 内存管理。
3. 性能测试必须在功能验证全部通过后进行。
4. 每完成一个阶段，按 `README.md` 中的验收标准逐项检查。

## 6. VMM API 使用规则

- 物理内存分配大小必须对齐到 `cuMemGetAllocationGranularity` 返回的粒度。
- 映射完成后必须调用 `cuMemSetAccess`，否则内核访问会报地址空间错误。
- `cuMemAddressReserve` 只预留虚拟地址，不占用物理显存。
- 解映射前必须确保对应流上的访问已完成（同步或事件）。
- 释放顺序：先解映射，再释放物理句柄，最后释放虚拟地址。
- 逻辑地址一经预留，在进程生命周期内保持固定。

## 7. 编码约定

- 所有 CUDA Driver API 调用必须经过统一错误检查宏，失败时打印 API 名、参数与返回值。
- 分配/释放路径必须可重复执行，单元测试至少覆盖 1000 次循环。
- 调试信息通过日志输出，默认关闭；涉及映射表的打印要可控。
- 公共接口使用清晰的命名，避免缩写；核心数据结构要能序列化输出当前状态。

## 8. 完成定义

一个阶段算完成，需满足：

1. 该阶段所有验收标准勾选完成。
2. 新增单元测试通过。
3. `nvidia-smi` 无持续显存增长。
4. 相关文档已更新到 `docs/`。

## 9. 长上下文测试约束

- 测试长度不能随意超过模型原生/缩放后的有效上下文长度，见 `README.md` 3.5。
- OOM 预期必须基于显存预算计算，不得直接假设。
- 显存收益只来自“按需映射/换出”，软件 MMU 本身不降低显存占用，见 `README.md` 3.6。

## 10. 常见问题排查

| 症状 | 优先检查 |
|------|----------|
| `cuMemCreate` 返回错误 | 大小是否对齐到分配粒度 |
| 内核访问映射地址报错 | 是否遗漏 `cuMemSetAccess` |
| 显存持续增长 | 是否只解映射未释放物理句柄，或释放顺序错误 |
| CUDA Graph 重放失败 | 逻辑地址是否发生变化，或物理映射是否在重放前完成 |
| 性能低于预期 | 是否频繁映射/解映射；优先实现延迟解映射策略 |

## 11. 与 README 的分工

- `README.md`：项目目的、实验设计、按周执行计划、测试矩阵、风险。
- `AGENTS.md`：开发代理的工程约定、构建测试方式、API 规则与排查表。

后续若 AGENTS 内容与 README 冲突，以 README 的验收标准为准。
