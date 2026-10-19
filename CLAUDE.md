# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目定位

这是一个**内核对比学习工作区**，目标是对比学习通用 Linux 内核（openEuler OLK-6.6 分支，基线 6.6 + openEuler 补丁）与 UniProton RTOS 在以下两个方向的实现差异：

1. **调度机制** — Linux 的 CFS/EEVDF、RT 调度类 vs UniProton 的优先级抢占式任务调度
2. **核间通信** — Linux 的 IPI/SMP/rpmsg 机制 vs UniProton 的 rpmsglite 及混合部署（MICA/mcs）方案

核心问题：RTOS 的实时性优势是通过什么技术手段实现的，内核层面的实现机制是什么。

## 工作约束

- **代码分析优先使用 codegraph**。两个代码库各自建立了索引，由于本会话没有默认项目，调用 `codegraph_explore` 时必须通过 `projectPath` 参数指定目标代码库（`.../learn-kernel/openEuler-6.6` 或 `.../learn-kernel/UniProton`）。没有索引的目录用常规工具。
- **所有问答讨论和文档记录一律使用中文**。
- **答疑文档**写入 `docs/`，命名格式：`NN-问题关键字.md`（NN 为两位序号，从 00 递增），如 `00-RTOS与linux内核的差异.md`。每次有价值的答疑讨论应整理归档到 docs/，而不是只留在对话里。
- `openEuler-6.6/` 和 `UniProton/` 是上游源码，**只读不改**（学习过程不做任何修改），且不纳入 git 提交。
- 远程仓库：https://github.com/dupengair/learn-kernel.git

## 工作区结构

根目录不是 git 仓库（git 仓库即本项目本身，学习笔记入库，两个内核源码仅本地存在）：

- `openEuler-6.6/` — Linux 内核 openEuler OLK-6.6 分支源码（基线 6.6 + openEuler 补丁集，与社区 6.6.y 存在行号偏移和局部代码差异），本身是 git 仓库，仅本地学习用
- `UniProton/` — openEuler UniProton RTOS 源码（上游在 AtomGit），仅本地学习用
- `docs/` — 学习答疑文档（入库内容）
- `CLAUDE.md`、`README.md`、`.gitignore` — 项目文件（入库内容）

重点阅读入口：

- **代码级教案系列（docs/02~09）**：已完成"实时性七大技术点"的行号级对比分析——02 内核形态与启动 / 03 调度决策 / 04 抢占与临界区 / 05 中断路径 / 06 内存确定性 / 07 API路径与上下文切换 / 08 优先级继承 / 09 MICA核间通信。回答相关问题时优先参考并保持一致口径
- Linux 调度：`openEuler-6.6/kernel/sched/`（`fair.c` EEVDF、`rt.c`、`core.c` 调度类框架）
- Linux 核间通信：`openEuler-6.6/arch/*/kernel/smp.c`（IPI）、`drivers/rpmsg/`、`drivers/virtio/`
- UniProton 调度：`UniProton/src/core/kernel/`（任务、TICK；注意本 checkout 无软中断模块）及 `doc/design/task.md`
- UniProton 核间通信：`UniProton/src/component/rpmsg-lite/`（RPMsg-Lite 库）、`UniProton/src/component/mica/`（MICA 混合部署，基于 OpenAMP/virtio + IPI）、`UniProton/doc/design/`（中文设计文档）
- UniProton demo/板级指南：`UniProton/doc/demo_guide/`；Linux 文档：`openEuler-6.6/Documentation/`

**硬件参考标准统一为 ARM64（AArch64）**：三份代码的架构级对比一律以 ARM64 为准——Linux 侧 `openEuler-6.6/arch/arm64/`，UniProton 侧 `src/arch/cpu/armv8/`，参考 demo 用 `demos/hi3093/`（GICv3；另有 raspi4 等 ARM64 demo）。openEuler 对 arm64（主力架构）的补丁远多于 riscv（entry.S/GIC v3 驱动/arch arm64 smp.c 均有大差异），跨架构平移行号不可靠。

版本陷阱提醒（写文档/回答时注意）：本工作区 Linux 侧为 openEuler OLK-6.6（非社区原生 6.6.y），与社区同版本存在 openEuler 补丁带来的行号偏移和局部代码差异，行号引用一律以本地 openEuler-6.6 实测为准（可用 `git -C openEuler-6.6 log` 追溯补丁来源）。基线 6.6 的通用陷阱仍成立：fair.c 是 EEVDF（非旧 CFS）；mm_init 已改名 mm_core_init、folio API 替换旧页函数；check_preempt_curr 已改名 wakeup_preempt；PREEMPT_RT 完整合入在 6.12；UniProton 本 checkout 无 SWI 模块、无 INT_NEST_DEPTH（armv8 无 20 层嵌套上限，该机制为 riscv64 独有）、hi3093 未启用 OS_OPTION_RR_SCHED 与 SMP、rpmsg-lite demo 仅在 riscv64virt 提供（ARM64 核间通信以 hi3093 openamp/MICA 为准）。

## UniProton 构建命令

入口是 `UniProton/build.py`，配置来自 `UniProton/config.xml`，工具链约定安装在 `/opt/buildtools` 下。官方推荐在 docker 镜像中编译（`swr.cn-north-4.myhuaweicloud.com/openeuler-embedded/uniproton:v004`）。

```bash
# 编译某平台内核静态库（cpu_type 取值见 config.xml：m4、raspi4、x86_64、rv64virt、hi3093、kp920 ...）
python3 build.py m4

# 清理
python3 build.py clean

# 一键编译 demo（含内核库 + 应用，产物在 demos/<plat>/build/out/）
cd demos/hi3093/build && sh build_app.sh                # ARM64 参考平台（openamp 应用，单板运行）
```

demo 通用结构：`apps/`（业务代码）、`bsp/`（板级驱动）、`build/`（构建脚本 + 链接脚本 `examples.ld`）、`config/`（`prt_config.c/h` 功能裁剪）。

## UniProton 架构

由五大子系统组成（详见 `doc/design/architecture_design.md`）：

- **Mem**（`src/mem/`）— 内存分区/块管理，FSC 算法
- **Arch**（`src/arch/`）— CPU 架构相关：硬中断、异常接管（armv7-m、armv7-r、armv8、riscv64、x86_64；ARM64 对应 **armv8** 目录）
- **Kernel**（`src/core/kernel/`）— 任务、软中断、TICK、软件定时器
- **IPC**（`src/core/ipc/`）— 事件、队列、信号量
- **OM**（`src/om/`）— cpup、hook、错误处理等调测功能

其他关键点：

- 对外 API 统一以 `PRT_` 前缀命名（如 `PRT_TaskCreate`、`PRT_TaskDelay`），头文件在 `src/include/uapi/` 与 `src/include/posix/`；`src/kal/` 是 POSIX 适配中间层。
- 每个平台的宏配置在 `build/uniproton_config/config_<arch>_<plat>/`，由 `cmake/common/build_auxiliary_script` 把 Kconfig 转成 `buildef.h`。
- 组件依赖源码（musl、lwip、littlefs、libboundscheck）不在仓库内，构建时按需下载/放置。
- `testsuites/` 含 posixtestsuite、libc-test、rhealstone 性能测试等。

## Linux 内核常用命令

```bash
cd openEuler-6.6
make defconfig                 # 或 make menuconfig / nconfig
make -j$(nproc)                # 完整构建
make htmldocs                  # 生成文档（需 sphinx）
```

标准内核布局：`arch/`（体系结构）、`kernel/`（核心子系统）、`mm/`（内存管理）、`drivers/`、`fs/`、`net/`。
