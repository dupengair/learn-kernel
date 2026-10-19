# learn-kernel — Linux 内核与 UniProton RTOS 对比学习

以 openEuler Linux kernel 6.6（[OLK-6.6 分支](https://gitee.com/openeuler/kernel/tree/OLK-6.6)，基线 6.6 + openEuler 补丁）与 openEuler [UniProton](https://atomgit.com/openeuler/UniProton) 两个内核代码为基础，对比学习**通用操作系统**与**实时操作系统（RTOS）**在调度机制和核间通信上的实现差异，深入理解 RTOS 实时性优势背后的技术手段与内核实现机制。

## 学习目标

1. **调度机制对比**
   - Linux：CFS / EEVDF（6.6 起）、实时调度类（SCHED_FIFO / SCHED_RR / SCHED_DEADLINE）、调度类框架
   - UniProton：基于优先级的抢占式任务调度、就绪队列管理、TICK 与软中断
2. **核间通信对比**
   - Linux：IPI（核间中断）/ SMP 支持、rpmsg / virtio 框架
   - UniProton：rpmsglite、混合部署（MICA / mcs）方案
3. **实时性专题**：RTOS 相比通用内核的实时性优势来自哪些技术手段（抢占粒度、中断管理、内存管理、锁与优先级反转处理等），在代码中如何实现

## 学习方法

- 精读两个内核的相关源码，配合 codegraph 建立符号级索引进行代码分析
- 每个专题以"对比"方式输出：同一问题在通用内核与 RTOS 中各自如何实现、为什么不同、差异带来什么
- 疑难问题以问答形式整理成文档沉淀在 `docs/` 目录

## 目录结构

```
learn-kernel/
├── docs/           # 学习答疑文档（NN-问题关键字.md，如 00-RTOS与linux内核的差异.md）
├── CLAUDE.md       # Claude Code 工作指引
└── README.md
```

> 两个内核源码（`openEuler-6.6/`、`UniProton/`）仅保留在本地，通过 `.gitignore` 排除，不入库。

## 本地源码版本

| 代码库 | 版本 | 获取方式 |
| ------ | ---- | -------- |
| Linux | openEuler OLK-6.6 分支（基线 6.6） | https://gitee.com/openeuler/kernel/tree/OLK-6.6 |
| UniProton | openEuler / AtomGit 主线 | https://atomgit.com/openeuler/UniProton |

## 环境说明

- 开发环境：WSL2（Ubuntu）
- 代码分析：CodeGraph 符号级代码索引（`codegraph init` 为代码库建立索引后，两个代码库均可进行符号级检索）
- UniProton 构建与运行参考其自带文档：`UniProton/doc/demo_guide/`（支持 riscv64 QEMU、树莓派 4B、x86_64 等环境）
