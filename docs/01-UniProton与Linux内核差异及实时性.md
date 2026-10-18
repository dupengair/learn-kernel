# 01 - UniProton 与社区 Linux 内核的主要差异、实时性技术点、UB 协议支持现状

> 问题：① UniProton 与社区 kernel 的主要差异是什么？② 实时性体现在哪里？有哪些主要技术点？③ 当前 UniProton 是否已支持华为 UB（灵衢）协议？

## 一、主要差异总览

**一句话定位**：Linux 6.6 是面向吞吐与公平的通用分时内核；UniProton 是面向确定性时延的硬实时内核，且以"静态库"形态嵌入应用，主打混合关键性部署（与 Linux 同芯片共存、各干各的）。

| 维度 | Linux 6.6 | UniProton |
| ---- | --------- | --------- |
| 设计目标 | 通用 OS：吞吐、公平、生态兼容 | 硬实时 RTOS：微秒级低时延、最坏情况可预测；工业控制/混合关键性 |
| 内核形态 | 单体内核，内核态/用户态两态分离，模块可动态加载 | **静态库**（libUniProton.a），与应用链接成单一 ELF 镜像运行，无两态之分 |
| API 路径 | 系统调用：陷入 → 权限切换 → 寄存器保存 → 分发表 → 返回 | **直接函数调用**（`PRT_*`），无陷入开销 |
| 地址空间 | 每进程独立虚拟地址空间、按需分页、可 swap | 平坦地址映射（按平台），无进程隔离、无 fork，**无页错误** |
| 任务模型 | task_struct，fork/clone 动态创建，资源随进程 | 轻量 Task（`PRT_TaskCreate` 显式指定栈与优先级） |
| 调度 | 5 个调度类（stop > dl > rt > fair(EEVDF) > idle），公平类按权重分配 | 固定优先级抢占，位图就绪队列 O(1) 取最高优先级 |
| 中断 | 通用 IRQ 子系统，可线程化；softirq/workqueue 底半部时长不定 | hwi 短 ISR，嵌套策略明确（`doc/design/hwi.md`） |
| 内存 | buddy + slab + vmalloc，页错误/回收/swap 引入不确定时延 | 分区内存池（FSC 算法），申请释放时间有界 |
| 时间 | hrtimer（纳秒级）、tickless/NOHZ_FULL | 固定周期 TICK，处理路径短而确定 |
| 同步原语 | mutex / rt_mutex / spinlock / rwsem / RCU 层次复杂 | 信号量/互斥/事件/队列，信号量支持优先级继承（PI）模式 |
| 文件/网络 | VFS + 内核网络协议栈（全套） | littlefs / lwip 适配层（`src/fs/`、`src/net/`） |
| 核间通信 | IPI + smp_call_function、rpmsg/virtio 标准框架 | rpmsg-lite 库 + IPI（详见下文源码结构） |
| POSIX | 完整 | 子集，经 `src/kal/` 适配 musl（litelibc） |

## 二、实时性体现在哪里 —— 七个主要技术点

**先立标准**：实时 ≠ 快，实时 = **确定性**（最坏情况有界）。以下每条都对应"Linux 为什么不确定 / UniProton 怎么做到确定"。

### 1. 固定优先级抢占调度 + O(1) 决策

- UniProton：就绪队列用**位图**组织，一条 CLZ（前导零计数）指令即可定位最高优先级就绪任务，决策时间与任务数无关且恒定；高优先级任务就绪的瞬间即抢占。
- Linux：EEVDF 要维护虚拟时间、lag、eligibility，公平类内部树操作 + 周期性负载均衡，**单次时延随系统负载浮动**——这正是通用内核"公平"的代价。

### 2. 内核全可抢占、关中断区极短

- UniProton 内核从设计上就是可抢占的，关键路径的关中断/关调度窗口有严格上界（可用 `src/om/hook/` 的 hook 统计验证）。
- Linux 默认配置（`PREEMPT_NONE`/`PREEMPT_VOLUNTARY`）下内核态长临界区不可抢占；完整 `PREEMPT_RT` 到 6.12 才合入主线，**6.6 主线没有**。Linux 实时化本质是在"补课"，UniProton 是"天生如此"。

### 3. 中断路径确定性

- UniProton：ISR 要求短小，嵌套策略可配置，底半部在任务上下文确定性执行（`src/arch/*/hwi`）。
- Linux：softirq 有 budget 循环（网络突发时可连续处理），IRQ 线程化后仍要过调度器，时延分布宽。

### 4. 无页错误 + 分区内存

- UniProton：内存预先分区（`src/mem/`，FSC 算法），申请/释放时间复杂度固定，**不存在缺页处理**——这是 Linux 无法给出的承诺（缺页 + 回收 + swap 可引入毫秒级毛刺）。
- 任务栈显式指定大小（`TskInitParam`），无动态增长的失败风险。

### 5. 无系统调用开销 + 短上下文切换路径

- API 即函数调用，省掉陷入/退出（syscall 指令 + 权限切换 + pt_regs 保存恢复）。
- 任务切换不切换地址空间（不重写页表基址寄存器），只保存必要的通用寄存器——这是 UniProton 宣称"极致微秒级时延"的主要来源之一。

### 6. 优先级继承防反转

- UniProton 信号量支持 PI 模式（`doc/design/sem.md`），高优先级任务等锁时提升持锁者优先级。
- 对应 Linux 的 rt_mutex PI（`kernel/locking/rtmutex.c`，仅 RT 路径使用）。优先级反转是固定优先级调度的经典缺陷，两边都必须处理，但 RTOS 里它是必答题。

### 7. 架构级隔离：混合部署（MICA）

- 实时负载放 UniProton 核，Linux 跑在另外的核，**干扰被物理隔离**——比 Linux 内部的软隔离（isolcpus / NOHZ_FULL / cgroup）彻底得多。
- 本仓库源码中可见证据：`src/component/mica/` 目录即 MICA 混合部署的 UniProton 侧支撑。

**量化验证手段**（后续阶段 2 实验）：Linux 用 `cyclictest`；UniProton 用自带 rhealstone 基准（`demos/riscv64virt/build/build_app.sh task-switch`，五项：切换/抢占/消息时延/死锁打破/信号量抖动）。

## 三、核间通信的源码结构（codegraph 实查）

UniProton 侧核间通信分两块，均在本仓库可见：

- `src/component/rpmsg-lite/` — RPMsg-Lite 轻量库本体（`rpmsg_lite.c`，endpoint 模型 `rpmsg_std_hdr` 头部）
- `src/component/mica/` — MICA 混合部署场景的服务层，基于 **OpenAMP 生态**（libmetal + rpmsg-virtio）：
  - `rpmsg_backend_rsc_table.c`：共享内存 + resource table + vring 初始化，`rpmsg_ipi_init()` 注册核间中断做通知（milkv-duo 平台走 CMDQU 命令队列中断）
  - `rpmsg_service.c`：tty/rpc/umt 等 endpoint 服务，用 `PRT_SemPost/Pend` 在中断与任务间传递消息
- 对比记忆点：Linux 的 rpmsg 是"内核框架 + 设备树 + virtio 驱动栈"的标准体系；UniProton 是"轻量库直连共享内存"，抽象层少、路径短，但生态依赖 Linux 侧配套（mcs 仓库）。

## 四、UB（灵衢/UnifiedBus）协议支持现状 —— 查证结论

**结论：当前 UniProton 不支持 UB 协议。** 依据三方面：

1. **本地源码 grep 零命中**：对 `UniProton/src`、`UniProton/doc` 全量搜索 `unifiedbus / unified_bus / 灵衢 / ub_protocol` 无任何结果，仓库中不存在 UB 组件或驱动。
2. **UB 的 OS 载体是 openEuler Linux**：灵衢（UnifiedBus，2025 年 9 月 HC 正式发布的超节点统一互联协议，RTT 从 7μs 压缩至 2μs、TB 级带宽、全局内存统一编址）的操作系统侧支持以 **UB OS Component** 形式开源，官方支持 openEuler 24.03 LTS SP3，涵盖设备管理、池化内存、通信、虚拟化模块——全部落在 Linux 内核生态。
3. **定位不同**：UB 面向数据中心超节点（Atlas 900 等 AI 基础设施）互联；UniProton 面向嵌入式端实时域（工业控制 MCU / 混合部署实时核），两者当前没有交集场景。

**展望**：两者同属 openEuler/华为开源生态（UniProton 由 sig-embedded 孵化，灵衢组件合入 openEuler）。若未来 UB 下沉到边缘/端侧或超节点内需要实时控制域，存在结合的可能性，但目前无公开计划。持续关注 [灵衢社区](https://www.unifiedbus.com) 与 openEuler sig-embedded 动态即可。

## 五、遗留问题（引导后续专题）

- [ ] 位图就绪队列的具体实现（`src/core/kernel/task/`，CLZ 指令用法）→ 归入调度专题
- [ ] UniProton 中断嵌套策略与关中断窗口实测 → 归入中断专题
- [ ] FSC 内存算法为什么时间有界 → 可作为内存小专题
- [ ] MICA 混合部署的完整链路（Linux mcs 驱动 ↔ UniProton rpmsg）→ 归入核间通信专题

---

**参考来源**
- [灵衢社区（unifiedbus.com）](https://www.unifiedbus.com)
- [华为 HC2025 主题演讲：超节点互联](https://www.huawei.com/cn/news/2025/9/hc-xu-keynote-speech)
- [灵衢操作系统组件下载页](https://www.unifiedbus.com/zh/software/ub-os-component)
- [openEuler 社区](https://www.openeuler.org/zh)
- 本地源码：UniProton `src/component/{rpmsg-lite,mica}/`、`doc/design/`；codegraph 索引实查
