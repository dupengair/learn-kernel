# 09 - 技术点7：MICA 混合部署与核间通信（物理隔离的实时性）

> 实时性代码对比系列。本系列：[02 形态](02-内核形态与启动流程.md) | [03 调度决策](03-调度决策机制.md) | [04 抢占与临界区](04-内核抢占与关中断窗口.md) | [05 中断路径](05-中断路径确定性.md) | [06 内存确定性](06-内存管理确定性.md) | [07 API路径与上下文切换](07-API路径与上下文切换开销.md) | [08 优先级继承](08-优先级继承.md) | **09 MICA核间通信**

## 0. 本篇导读

**一句话结论**：前六篇都在讲"单系统内怎么挤时延"，本篇讲"怎么让干扰根本不发生"——MICA 混合部署把实时负载放 UniProton 核、Linux 跑在另外的核，通过**物理隔离**消除跨域干扰；两个 OS 之间只共享两样东西：一块共享内存（vring）+ 一条中断通道（IPI/doorbell）。所有消息传递 = "写共享内存 + 打铃"两步。Linux 侧是五层框架（rpmsg 总线→virtio→rproc-virtio→remoteproc→mailbox，数万行）；UniProton 侧只有约 830 行后端 + 开源 OpenAMP 库。

**三个先澄清的事实**：
1. UniProton 的核间通信有**两条独立线**：`src/component/mica/`（OpenAMP 线，对接 Linux master，真实 SoC 场景）和 `src/component/rpmsg-lite/`（NXP rpmsg-lite 移植，自带精简 virtqueue，目前只用于 UniProton 双核自回环 demo）。别混。
2. **master 是 Linux**：remoteproc 加载 UniProton 固件 ELF、解析 resource table、分配 vring 并把地址写回去；UniProton 以 remote 身份自旋等 master 就绪。
3. 两套栈**共用同一 vring 内存格式**（desc/avail/used 三表）——这是互操作的全部基础。

## 1. 机制原理

**隔离的粒度**：Linux 内部的软隔离（isolcpus/cgroup/NOHZ_FULL）本质还是共享调度器、共享内核锁、共享内存带宽；AMP 混合部署里两个 OS 各占独立核、独立内存分区、独立中断控制器配置——干扰只剩 cache 污染和内存总线竞争，这是硬件级的边界。代价是失去单机 SMP 的编程模型，一切交互退化为显式消息传递。

**rpmsg 的核心抽象**：virtio 的 vring 环形缓冲 + rpmsg 的 endpoint 寻址（16B 头：src/dst/len/flags）。发送方从预分配 buffer 池取 buffer、填头+payload、挂 avail 环、打铃；接收方中断里遍历 used 环、按 dst 查 endpoint 表、回调。**拷贝次数：发送 1 次 memcpy，接收 0 次**（回调直接指向共享内存里的 payload）。

## 2. 架构设计对比

```
Linux 侧（五层框架，master 角色）              UniProton mica 组件（remote 角色）
 /dev/rpmsg* ← rpmsg 总线(设备/驱动模型)       rpmsg_service.c: tty/rpc/umt 三个 ept 服务
    │ virtio_rpmsg_bus.c                          │ PRT_SemPost/Pend（中断→任务）
    ▼ virtio 框架(virtio_ring.c: vring三表)       ▼ OpenAMP 库(rproc_virtio_*/rpmsg_virtio_*)
 rproc-virtio(kick转发mailbox)                  rpmsg_backend_rsc_table.c (~345行)
    │                                              ├ resource_table(.resource_table段)
    ▼ remoteproc(加载固件/解析rsc/上电)            ├ rpmsg_ipi_init(注册收铃中断)
 mailbox(平台打铃: 写MU寄存器)                     └ IPI handler: 只SemPost就返回
                                                  libmetal(IO抽象: 共享内存region)
 建链流程:
 Linux: rproc_boot→加载ELF→解析rsc table→
        分配vring→地址写回rsc->vring[i].da→上电远端核
 UniProton: 启动后读同一rsc table(用master写回的da)→
        rproc_virtio_wait_remote_ready(自旋等master)→rpmsg_init_vdev
```

## 3. 关键数据结构

**vring 三表** — 两边共用同一格式。Linux `include/uapi/linux/virtio_ring.h`：

```c
struct vring_desc {  __virtio64 addr; __virtio32 len;       // :107 16字节/个，next成链
                     __virtio16 flags; __virtio16 next; };
struct vring_avail { __virtio16 flags; __virtio16 idx; __virtio16 ring[]; };  // :114 "可用"
struct vring_used {  __virtio16 flags; __virtio16 idx;                       // :131 "已用"
                     struct vring_used_elem { __virtio32 id; __virtio32 len; } ring[]; };
```

一段连续内存依次是 `desc[num] → avail → 对齐填充 → used`（布局注释 ：170-192）。UniProton rpmsg-lite 的 `struct virtqueue`（src/component/rpmsg-lite/include/virtqueue.h:89-141）内嵌同一 `struct vring` 外加本地游标——**互操作的基础就是这份格式一致**。

**fw_rsc_vdev（resource table 条目）** — Linux `include/linux/rsc_table.h:294`：`id`(VIRTIO_ID_RPMSG=7)/`dfeatures`/`gfeatures`(host写回协商)/`status`(remote轮询它等master就绪)/`vring[]`(每项 da/align/num)。UniProton 对应 `struct fw_resource_table`（src/component/mica/resource_table.h:89-104），整个结构体放 `.resource_table` 段；vring 地址常量 `VRING_TX/RX_ADDRESS = -1 /* allocated by Master */`（openamp_common.h:32-33）——**remote 身份的直接证据**。

**rpmsg 消息头与 buffer 池** — Linux `drivers/rpmsg/virtio_rpmsg_bus.c:85` `struct rpmsg_hdr{src,dst,reserved,len,flags,data[]}`（16B 头）；`:130-131` `MAX_RPMSG_NUM_BUFS 512 / MAX_RPMSG_BUF_SIZE 512`（512B 定长池，含头 496B payload）。UniProton rpmsg-lite 对应 `rpmsg_std_hdr`（rpmsg_lite.h:110-117，字段相同）。

## 4. 关键代码路径走读

### 4.1 Linux：一次跨核发送 + 接收

**发送 6 步** — `rpmsg_send_offchannel_raw()`（virtio_rpmsg_bus.c:552-648）：

```c
msg = get_a_tx_buf(vrp);                    // :584 池取buffer，无则睡sendq(15s)
msg->len/src/dst = ...;                     // :613-615 填16B头
memcpy(msg->data, data, len);               // :618 ★唯一一次payload拷贝
virtqueue_add_outbuf(vrp->svq, &sg, 1, ...);// :632 挂avail环
virtqueue_kick(vrp->svq);                   // :644 打铃
```

**打铃的最后一米**（经三层转发）— `virtqueue_kick`（virtio_ring.c:2390，先查 used->flags 的 NO_NOTIFY 抑制位 :2345）→ `vq->notify` = `rproc_virtio_notify`（remoteproc_virtio.c:65-75 `rproc->ops->kick(rproc, notifyid)`）→ 以 i.MX 为例 `imx_mu_generic_tx`（drivers/mailbox/imx-mailbox.c:215-236）：`imx_mu_xcr_rmw(priv, IMX_MU_GCR, GIEn, 0)` **写 MU 寄存器触发对端核 IRQ**。

**接收** — vq 回调 `rpmsg_recv_done`（virtio_rpmsg_bus.c:777-806）：

```c
msg = virtqueue_get_buf(rvq, &len);         // 从used环取已完成buffer
while (msg) { rpmsg_recv_single(...); msg = virtqueue_get_buf(...); }  // 循环排空
if (msgs_received) virtqueue_kick(vrp->rvq);// 归还buffer通知对端可继续发
```

`rpmsg_recv_single`（:708-774）：按 `msg->dst` 在 idr 找 endpoint → 持 cb_lock 调 `ept->cb(...)`（:753，**零拷贝直指共享内存 payload**）→ buffer 重新挂回 avail。

**master 的生命周期管理** — `rproc_fw_boot`（remoteproc_core.c:1371-1445）：`request_firmware` → 定位固件内 resource table（:1403）→ `rproc_handle_resources` 逐条分发；vdev 条目走 `rproc_handle_vdev`（:475-526）注册 rproc-virtio 平台设备 → `rp_find_vq` 分配 vring 内存、**把地址写回 `rsc->vring[id].da`**（remoteproc_virtio.c:160-161）→ `rproc_start` 搬 ELF 段进对端内存（:1268）、上电放行（:1296）。**名字服务 NS**：remote 建 endpoint 时向端口 53 广播，Linux `rpmsg_ns_cb`（rpmsg_ns.c:31-72）动态创建 rpmsg 设备（rpmsg_tty 等）。

### 4.2 UniProton mica：remote 视角

**启动建链** — `rpmsg_backend_init()`（src/component/mica/rpmsg_backend_rsc_table.c:257-336）：

```c
rpmsg_ipi_init();                            // :266 注册收铃回调
metal_register_generic_device(&shm_device);  // :280 共享内存2-region(共享区+rsc table)
vdev = platform_create_vdev(...);            // :314
    rproc_virtio_create_vdev(VIRTIO_DEV_DEVICE, ...);   // :215 ★remote角色
    rproc_virtio_wait_remote_ready(vdev);    // :222 自旋等master置READY
    rproc_virtio_init_vring(vdev, 0, ..., (void *)vring_rsc->da, ...); // :227 用master写回的地址
rpmsg_init_vdev_with_config(&rvdev, vdev, ...);  // :321
```

**收铃与消费（中断-信号量模式，本篇核心对照点）** — `rpmsg_ipi_handler`（:134-158）：

```c
os_asm_invalidate_dcache_all();       // :144 共享内存cache非一致，先失效
status = rsc_table->reserved[0];      // :145 复用reserved字段带状态(PSCI风格)
PRT_SemPost(msg_sem);                 // :157 ★ISR只post信号量立即返回
```

消费任务 `rpmsg_listen_task`（rpmsg_service.c:220-228）死循环 `PRT_SemPend(msg_sem, OS_WAIT_FOREVER)` → `rproc_virtio_notified(vdev, VRING1_ID)`（backend:195）→ OpenAMP 遍历 used 环 → 按 hdr.dst 分发到 `rpmsg_rx_tty_callback`（rpmsg_service.c:110-121）：`rpmsg_hold_rx_buffer` **零拷贝持有** → 存指针 + `PRT_SemPost(tty_sem)` → tty 任务慢慢消费后 `rpmsg_release_rx_buffer`（:148-174）。**对照**：Linux 的 `rpmsg_recv_done` 在（线程化的）硬中断上下文直接回调 ept->cb；UniProton 把处理挪到任务上下文——回调里可以调阻塞 API，中断时延不受回调长度影响。

**发送打铃** — `virtio_notify`（backend:76-95）：SMP 配置走 `OsHwiMcTrigger(0, coreMask, OS_HWI_IPI_NO_07)`（核间 IPI）；milkv-duo 平台走 `rtos_cmdqu_send(&notify_cmd)`（CMDQU mailbox → Linux 侧 CSV 驱动收）。

**大数据通道 umt** — rpmsg_service.c:271-288：4 字节消息传地址 + 1MB 共享缓冲区，绕开 512B MTU 限制（对照思考题 1）。

### 4.3 rpmsg-lite 独立栈（demo 线）

`rpmsg_lite_send`（src/component/rpmsg-lite/rpmsg_lite.c:654）→ `rpmsg_lite_format_message`（:580-652）：`vq_tx_alloc` 从 used 环回收 buffer → 填头 → `env_memcpy` 单次拷贝（:642）→ `vq_tx` 入 avail + `virtqueue_kick`（:646-648）。kick 链（virtqueue.c:475-489）：`env_mb()` 内存屏障 → 查 NO_NOTIFY 位 → `platform_notify(vector_id)` → demo 平台 `send_ipi`（rpmsg_platform_riscv.c:81-86，RISC-V IPI）。接收：IPI ISR → `env_isr` → `virtqueue_notification(vq)` → `rpmsg_lite_rx_callback`（:113-170）按 dst 查 ept 链表调 `rx_cb`（RL_HOLD 返回值支持零拷贝持有）。demo 形态：demos/riscv64virt/apps/rpmsglite_env_test 在 QEMU 双核上核 0 master / 核 1 remote 互发计数校验（rpmsg_common.c:27-39）。

## 5. 实现差异总结表

| 维度 | Linux virtio_rpmsg | UniProton mica (openamp) | UniProton rpmsg-lite |
|---|---|---|---|
| 框架层级 | 五层框架+设备驱动模型+/dev | 库直调，RTOS 任务+信号量 | 单库自含（精简virtqueue） |
| master/remote | master：加载固件/分配vring/写status | remote：wait_remote_ready 自旋等待 | demo 内一核master一核remote |
| 发送拷贝 | 1 次 memcpy 进 512B 定长池 | 1 次；umt 大数据走地址传递 | 1 次 env_memcpy |
| 接收拷贝 | 0 次（cb 直指共享内存） | 0 次（hold_rx_buffer 持有） | 0 次 |
| 通知机制 | kick→mbox→写 MU 寄存器 | OsHwiMcTrigger(IPI) / cmdqu | platform_notify→send_ipi |
| 通道发现 | NS 动态（53 端口广播） | 静态（vendor ept_table + 固定名） | 无 NS，地址手工分配 |
| 回调上下文 | （线程化）硬中断上下文 | 任务上下文（信号量衔接） | 任务/中断（env 层定） |
| 代码量级 | 框架数万行 | 组件 ~830 行 + openamp 库 | ~3465 行 |

## 6. 代码阅读指导

**推荐路线**：

1. `linux-6.6.159/include/uapi/linux/virtio_ring.h:96-214`（三表结构+布局注释+vring_init）——**全部机制的物理基础，先读这个**
2. `linux-6.6.159/drivers/rpmsg/virtio_rpmsg_bus.c:868-1011`（probe）→ `:552-648`（发送）→ `:708-806`（接收）→ `drivers/rpmsg/rpmsg_ns.c:31-96`（NS）
3. `linux-6.6.159/drivers/virtio/virtio_ring.c:2228-2247,2345-2395`（add_sgs 与 kick 两段式）
4. `linux-6.6.159/drivers/remoteproc/remoteproc_core.c:1371-1445`（fw_boot）→ `remoteproc_virtio.c:65-75,103-164`（notify 与 vring 写回）→ `drivers/mailbox/imx-mailbox.c:215-236`（打铃最后一米）
5. `UniProton/src/component/mica/resource_table.c:21-50` + `resource_table.h:89-104`（remote 交给 master 的"资源清单"）
6. `UniProton/src/component/mica/rpmsg_backend_rsc_table.c:257-336`（建链）→ `:208-247`（wait_ready）→ `:76-95`（打铃）→ `:134-158`（**收铃只 SemPost——与 Linux 对照的核心**）
7. `UniProton/src/component/mica/rpmsg_service.c:290-351`（四任务）→ `:110-178`（tty 零拷贝）→ `:271-288`（umt 地址传递）
8. `UniProton/src/component/rpmsg-lite/rpmsg_lite.c:580-652` + `virtqueue.c:475-489` + `rpmsg_lite.c:113-170` + `demos/riscv64virt/apps/rpmsglite_env_test/rpmsg_common.c:18-50`

**实验建议**：跑 `demos/riscv64virt/build/build_app.sh rpmsglite_env_test`（QEMU 双核自回环），用 QEMU `-smp 2` 观察 master/remote 两个核的输出交错；有条件上 milkv-duo 真板体验 Linux master + UniProton remote 全链。

**思考题**：

1. Linux 用固定 512B buffer 池（代码注释自述"将来可做零拷贝"），mica 的 umt 通道用"4 字节传地址 + 1MB 共享缓冲区"绕开限制——两种设计各牺牲了什么？（提示：MTU 灵活性 vs buffer 生命周期与 cache 一致性管理。）
2. `rpmsg_ipi_handler` 只做 `PRT_SemPost` 就返回，处理挪到任务；Linux `rpmsg_recv_done` 在硬中断上下文直接回调 ept->cb。两种取舍对最坏中断延迟和回调里可调用的 API 各有什么约束？
3. vring 协议允许双方单方面抑制通知（avail 的 NO_INTERRUPT / used 的 NO_NOTIFY）。rpmsg_probe 里 `virtqueue_disable_cb(svq)`（:945）抑制了哪一方向、为什么安全？mica 的 `reset_vq`（backend:97-132）在 Linux 重启时只重置本地游标不清内存——这依赖了 vring 协议的什么性质？

---

*本篇行号基于 linux-6.6.159 与 UniProton 当前 checkout 实测。openamp 库本体在 thirdpart/openamp（CMakeLists.txt:154 引用，当前 checkout 未检出该目录，其内部行号未确认）。*
