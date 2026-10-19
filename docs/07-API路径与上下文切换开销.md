# 07 - 技术点5：API 路径与上下文切换开销（RISC-V 对RISC-V）

> 实时性代码对比系列。本系列：[02 形态](02-内核形态与启动流程.md) | [03 调度决策](03-调度决策机制.md) | [04 抢占与临界区](04-内核抢占与关中断窗口.md) | [05 中断路径](05-中断路径确定性.md) | [06 内存确定性](06-内存管理确定性.md) | **07 API路径与上下文切换** | [08 优先级继承](08-优先级继承.md) | [09 MICA核间通信](09-MICA混合部署与核间通信.md)

## 0. 本篇导读

**一句话结论**：一次 API 调用，Linux = ecall 陷入 + 31 个通用寄存器全量压栈（288B pt_regs）+ 进入/退出安全框架检查 + sret；UniProton = 一条 call 指令。一次上下文切换，Linux = 切地址空间（写 satp，无 ASID 硬件时还要 sfence.vma 全刷 TLB）+ 保存 14 个 callee-saved 寄存器；UniProton = 保存全部 30 GPR+mepc+mstatus（256B，帧自包含）+ 换栈指针，**零地址空间动作**。

**两个反直觉点**（本篇最有教学价值的部分）：
1. UniProton 单次切换保存的寄存器**更多**（32 槽 vs 14 个）——但总路径反而更短，因为省掉了 pt_regs 双份保存、TIF 循环、锁、satp/TLB、钩子开销。
2. Linux `__switch_to` 只存 14 个寄存器是安全的——切换发生在 `schedule()` 函数调用边界，caller-saved 寄存器由编译器按 ABI 放进了旧任务自己的栈帧；UniProton 的恢复点经 mret 任意，帧必须自包含。**这不是谁更聪明，是切换模型不同。**

本篇两边统一 RISC-V 视角（Linux S 态 vs UniProton M 态），同构可比。

## 1. 机制原理

**特权级差异是开销之源**。ecall 陷入要求：完整现场保存（内核不知道用户态会以什么寄存器状态进来）→ 安全框架（ptrace/seccomp/audit 是系统调用的必经关卡，不只是调试选项）→ 分发表间接跳转 → 退出前 TIF 标志循环复查（重调度/信号/resume 回调），全部清零才允许 sret。

**上下文切换的两层代价**：寄存器层（必付）+ 地址空间层（Linux 独有）。satp 写只是 CSR 写，但 TLB 失效的连带代价（sfence.vma 或 ASID 翻转时的全刷）才是大头——UniProton 整个体系里没有这层。

## 2. 架构设计对比

```
Linux syscall（S态）                          UniProton API（M态）
 用户态 ecall ──► handle_exception            应用 call PRT_TaskDelay
  sscratch 判来向                               └─ 普通函数：检查中断/锁状态
  31 GPR+4CSR → pt_regs (288B)                    ├ OsIntLock
  excp_vect_table[8]→do_trap_ecall_u              ├ 出就绪队/挂延时链
  ├ syscall_enter_from_user_mode                  └ OsTskScheduleFastPs
  │  (ptrace/seccomp/audit)                          └ OsTaskTrap 切换
  ├ sys_call_table[a7] → handler
  └ syscall_exit_to_user_mode                  Linux 切换（context_switch）
     exit_to_user_mode_loop                     ├ switch_mm_irqs_off
     (resched/信号/resume 循环)                  │   写 satp ± sfence.vma ★
  sret ◄── 恢复 31 GPR                          ├ FPU 懒保存(SR_FS 脏位)
                                                └ __switch_to：14 寄存器
UniProton 切换（两种入口汇聚）
 主动阻塞: OsTaskTrap(手工伪造 trap 语义)      UniProton: trap/OsTaskTrap 保存
 中断唤醒: trap(硬件入口)                        30 GPR+mepc+mstatus → TCB[0]=sp
   └──────► OsMainSchedule(纯C，选任务)          → OsTskContextLoad 载帧
              └► OsTskContextLoad → mret          → mret ★无地址空间动作
```

## 3. 关键数据结构：两种"帧"

**L1. `struct pt_regs`** — linux arch/riscv/include/asm/ptrace.h:15-54：36 个 unsigned long（288B）：epc + 全部 GPR + sstatus/badaddr/scause + **orig_a0**（syscall restart 回滚用）。这是"内核栈上的异常帧"，含诊断字段（badaddr/cause 供缺页处理）。

**U1. `struct TskContext`** — UniProton src/include/uapi/hw/riscv64/os_cpu_riscv64.h:30-79：

```c
struct TskContext {                 /* 就是任务栈顶的切换帧，256B；FPU 版 360B */
    U64 fcsr; U64 fs11...fs0;       /* OS_ARCH_SURPORT_F 时 12 个 F 寄存器 */
    U64 mepc;                       /* mret 的 pc——"伪造被中断过"的关键 */
    U64 mstatus;
    U64 t6..t3, s11..s2, a7..a0, s1, s0, t2..t0, tp, gp, ra;
};                                  /* 注意：sp 不占槽，由帧基址+帧长恢复 */
```

对比：pt_regs 面向异常处理（要诊断信息），TskContext 面向任务恢复（只需 mepc/mstatus 驱动 mret）。

**L2. `struct thread_struct`** — linux arch/riscv/include/asm/processor.h:53-62：只存 `ra/sp/s[12]` + FPU 状态——`__switch_to` 需要的全部。

**U2. `struct TagTskCb` 栈字段** — UniProton src/core/kernel/include/prt_task_external.h:106-135：`stackPointer` 是**第一个字段**（偏移 0，汇编 `sd sp, 0(t0)` 直写）、`topOfStack`（:135 水位检查）。最小栈 `OS_TSK_MIN_STACK_SIZE ≈ 0x1D0`（os_cpu_riscv64_external.h:47）。

## 4. 关键代码路径走读

### 4.1 Linux：一次 write() 的全程

**① 入口（汇编）** — arch/riscv/kernel/entry.S:21-110 `handle_exception`：`csrrw tp, CSR_SCRATCH, tp` 一次原子交换判来向（:27）；`addi sp, sp, -(PT_SIZE_ON_STACK)` 后 `REG_S` x1,x3,x5 + `save_from_x6_to_x31`（:42-49，**31 个 GPR 全存**）；再存 sstatus/sepc/stval/scause（:60-71）；按 scause 索引 `excp_vect_table`（:97-104，:295 的 8 号槽就是 `do_trap_ecall_u`）。

**② 分发与安全框架（C 层）** — arch/riscv/kernel/traps.c:331-359：

```c
regs->epc += 4;  regs->orig_a0 = regs->a0;  regs->a0 = -ENOSYS;
syscall = syscall_enter_from_user_mode(regs, syscall);   /* ptrace→seccomp→tracepoint→audit */
if (syscall >= 0 && syscall < NR_syscalls) {
        syscall = array_index_nospec(syscall, NR_syscalls);  /* 防推测执行 */
        syscall_handler(regs, syscall);                  /* sys_call_table[a7](regs) */
}
syscall_exit_to_user_mode(regs);
```

> ⚠ openEuler 差异勘误（两库 diff 复核）：`__trap_section` 标记并非 openEuler 特有——社区 6.6.159 的 do_trap_ecall_u 同样带此标记（traps.c:308-336）。真正的 openEuler 差异是新增 `CONFIG_LIVEPATCH_WO_FTRACE`（免 ftrace 的 livepatch 断点处理 `klp_breakpoint_handler`，traps.c:270-285），它使 do_trap_ecall_u 后移至 :331-359；分发逻辑本身两版一致。

**③ 退出循环** — kernel/entry/common.c:93-137 `exit_to_user_mode_loop`：`_TIF_NEED_RESCHED → schedule()`；`_TIF_SIGPENDING → arch_do_signal_or_restart()`（syscall restart 就在这）；`_TIF_NOTIFY_RESUME → resume_user_mode_work()`。**全部清零才 sret**——这是每次系统调用的固定保险费。

> ⚠ openEuler 差异：openEuler-6.6 回移了 mainline 6.12 的 entry 框架重构——enter_from_user_mode/syscall_enter_from_user_mode/exit_to_user_mode 等小函数移入 include/linux/entry-common.h 内联实现，kernel/entry/common.c 仅 407 行（社区 6.6 约 470 行），exit_to_user_mode_loop 位于 93-137；TIF 循环逻辑与社区 6.6 完全一致。

**④ 上下文切换** — kernel/sched/core.c:5283-5343 `context_switch`：`switch_mm_irqs_off`（:5323）→ `switch_to`（:5339）。地址空间切换在 arch/riscv/mm/context.c:193-206：`csr_write(CSR_SATP, pgd_pfn|asid|mode)`；**无 ASID 硬件时 `set_mm_noasid` 每次切换无条件 satp 写 + `sfence.vma` 全刷**（:201-206，local_flush_tlb_all 就是 sfence.vma，asm/tlbflush.h:20-23）。寄存器切换 `__switch_to`（entry.S:240-277）只存 **14 个**：ra/sp/s0-s11，恢复端 `move tp, a1` 重载 current。

### 4.2 UniProton：一次 PRT_TaskDelay 的全程

**① 任务栈初始化（伪造现场）** — `OsTskContextInit`（src/arch/cpu/riscv64/common/common/prt_port.c:74-131）：16 对齐栈顶 → 预留 TskContext 帧 → 填魔数（ra=0x1, t6=0x23 调试用）→ `context->mstatus = get_mstatus() | MPIE_S | MPP`（MPP=M 态、MPIE=开中断）→ `context->mepc = OsTskEntry`（**统一任务入口**，prt_task_init.c:164-185 从 TCB 取 args 调 taskEntry）。

**② 主动阻塞：OsTaskTrap（手工伪造 trap）** — prt_hw.s:329-460：先花十几条指令手工改写 mstatus（MIE→MPIE、MIE 清 0、MPP=0b11）——因为硬件只在真 trap 时自动做这件事，主动切换要"伪造得像被中断过"，恢复端 mret 逻辑才唯一。存全部寄存器，**mepc 槽写 `_tsk_trap_ret` 标号**而非真 mepc；TCB[0]=sp、换系统栈；`call OsMainSchedule`。若调度回原任务，mret 落在 `_tsk_trap_ret: ret`——像"调用了一个不返回的函数"。

**③ 中断驱动切换** — `trap`（prt_hw.s:109-322）：保存全部寄存器 → `g_runningTask->stackPointer = sp; sp = __os_sys_sp_end`（:203-208，中断余下处理跑在系统栈）→ C 链处理 → 非嵌套 `call OsMainSchedule`（:232）。

**④ 决策与恢复** — `OsMainSchedule`（prt_sched_single.c:55-81，纯 C 无锁——单核+中断已关）：查 `OS_FLG_TSK_REQ` → `RUNNING_TASK = g_highestTask` → `OsTskContextLoad`（prt_hw.s:12-104）载帧 → **mret**（:104）直达新任务。

**⑤ API 直调证据** — `PRT_TaskDelay`（src/core/kernel/task/amp/prt_task_minor.c:93-135）：校验（中断中/锁中调 Delay 报错）→ 出就绪队/挂延时链 → `OsTskScheduleFastPs`。pthread 适配同理：`pthread_create` 内部直调 `OsTaskCreateChkAndGetTcb/OsTskContextInit/PRT_TaskResume`（src/libc/litelibc/src/thread/pthread_create.c:193-220），全程无陷入。

### 4.3 寄存器保存集合逐项对比

| 寄存器组 | Linux __switch_to（entry.S:240-277） | UniProton trap/OsTaskTrap（prt_hw.s） |
|---|---|---|
| ra / sp | 保存 / 保存(thread.sp) | 保存 / 不占槽（帧基址重建） |
| s0-s11（12个） | 保存 | 保存 |
| a0-a7（8个） | **不保存**（ABI caller-saved，切换点=函数调用边界） | **保存**（mret 恢复点任意，帧须自包含） |
| t0-t6（7个） | 不保存（同上） | **保存** |
| tp / gp | 不保存（tp :275 重载 current；gp 内核统一） | 保存（每任务可有自己的 gp/tp） |
| pc / 状态 | 无（ra 驱动 ret） | mepc + mstatus（mret 驱动） |
| FPU | 懒保存（SR_FS 脏位才存） | 编译期 OS_ARCH_SURPORT_F 决定无条件全存 |
| 合计 | **14 个整型寄存器** | **30 GPR + 2 CSR（256B）** |
| 附带动作 | satp 切换 ± sfence.vma、mm 引用计数、FPU 懒存 | 写一个 TCB 字段 + 换栈指针 |

## 5. 实现差异总结表

| 维度 | Linux 6.6 (S态) | UniProton (M态) |
|---|---|---|
| API 进入方式 | ecall + 31 GPR/4 CSR 压栈（288B） | call 指令，0 额外压栈 |
| 进入安全开销 | user dispatch/ptrace/seccomp/tracepoint/audit | 无 |
| 退出安全开销 | rseq + TIF 循环复查（resched/信号/resume） | OsMainSchedule 一次判断后载帧 |
| 切换步骤 | mm 切换→FPU 懒存→14 寄存器→finish_task_switch | 存帧→TCB[0]=sp→选任务→载帧→mret |
| TLB/页表代价 | 有 ASID：satp 写（翻转时全刷）；无 ASID：每次全刷 | **零**（全库 grep satp 零命中） |
| 同步原语竞争 | futex：用户态自旋失败→又一次完整陷入+切换 | 信号量 Pend 即阻塞切换，无陷入前缀 |
| 返回指令 | sret（退出前 SPP 判定+TIF 清零） | mret（帧自带 mstatus） |

## 6. 代码阅读指导

**推荐路线**：

1. `openEuler-6.6/arch/riscv/kernel/entry.S:21-110`（入口与 scratch 技巧）→ `include/asm/ptrace.h:15-54`（pt_regs）
2. `openEuler-6.6/arch/riscv/kernel/traps.c:331-359`（C 层分发）→ `kernel/entry/common.c:28-83,93-137,183-224`（进/出通用框架三段）
3. `openEuler-6.6/kernel/sched/core.c:5283-5343`（context_switch 两层）→ `entry.S:240-277`（14 寄存器）→ `arch/riscv/mm/context.c:192-206,318-338`（satp/sfence.vma）
4. `openEuler-6.6/kernel/futex/syscalls.c:192-212` + `waitwake.c:323-328`（用户态同步进内核的对照）
5. `UniProton/src/include/uapi/hw/riscv64/os_cpu_riscv64.h:30-79`（TskContext 帧布局）
6. `UniProton/src/arch/cpu/riscv64/common/common/prt_port.c:74-131`（伪造初始栈帧）
7. `UniProton/src/arch/cpu/riscv64/common/common/prt_hw.s:12-104`（OsTskContextLoad）→ `:109-232`（trap）→ `:329-460`（OsTaskTrap，重点 :330-363 手工改 mstatus）
8. `UniProton/src/core/kernel/sched/prt_sched_single.c:55-81`（OsMainSchedule）
9. 对照收尾：`demos/riscv64virt/apps/hello_world/main.c:32-93`（应用直调形态）+ `src/libc/litelibc/src/thread/pthread_create.c:84-106`

**实验建议**：Linux 侧 `strace -T -e trace=all` 看一次 write 的耗时下限；再写程序循环 1e6 次 getpid（最薄系统调用）测平均/最坏耗时。UniProton 侧对照 rhealstone 的 task-switch 用例（`cd demos/riscv64virt/build && sh build_app.sh task-switch`）。

**思考题**：

1. Linux `__switch_to` 只存 14 个寄存器却安全——"另外那些寄存器"在什么路径上被保存了？（提示：ecall/中断边界 entry.S:42-49 与 schedule() 边界的分工——编译器在调用点已把 caller-saved 放进旧任务自己的栈帧。）
2. `OsTaskTrap` 为什么要手工改写 mstatus 的 MPIE/MIE/MPP（prt_hw.s:330-363），而 `trap` 入口不需要？
3. 若给 UniProton 加 MMU/U 态任务支持，prt_hw.s 的帧和 OsMainSchedule 至少要增加哪些动作？开销比例会发生什么变化？（对照 context.c:201-206 的"satp+无条件全刷"最坏路径，以及 Linux 为何引入 ASID。）

---

*本篇行号基于 openEuler-6.6 与 UniProton 当前 checkout 实测；6.6 中 RISC-V 的 __switch_to 已并入 entry.S（无独立 switch_to.S），syscall 分发也已从汇编移入 C 层 traps.c。（勘误：should_fail_futex() 故障注入并非 openEuler 特有，社区 6.6.159 的 syscalls.c:201 同样存在，两侧 SYSCALL_DEFINE6(futex) 逐行一致。）未确认项：arch_do_signal_or_restart 的 restart 细节、SMP 版 OsMainSchedule（riscv64virt 未启用）。*
