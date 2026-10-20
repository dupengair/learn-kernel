# 07 - 技术点5：API 路径与上下文切换开销（ARM64 对ARM64）

> 实时性代码对比系列。本系列：[02 形态](02-内核形态与启动流程.md) | [03 调度决策](03-调度决策机制.md) | [04 抢占与临界区](04-内核抢占与关中断窗口.md) | [05 中断路径](05-中断路径确定性.md) | [06 内存确定性](06-内存管理确定性.md) | **07 API路径与上下文切换** | [08 优先级继承](08-优先级继承.md) | [09 MICA核间通信](09-MICA混合部署与核间通信.md)

## 0. 本篇导读

**一句话结论**：一次 API 调用，Linux = svc #0 陷入 + 34 槽现场全量压栈（pt_regs 336B，其中 x0-x30/sp/pc/pstate 共 272B）+ 进入/退出安全框架检查 + eret；UniProton = 一条 bl 指令。一次上下文切换，Linux = 切地址空间（写 TTBR0_EL1/TTBR1_EL1 携带 ASID，ASID 代际回卷时 tlbi 全刷）+ 保存 12 个 callee-saved 寄存器+sp（FPSIMD 走 per-CPU 懒切换）；UniProton = 保存全部 31 个 GPR+xzr+ELR/SPSR/FAR/ESR（288B，帧自包含）+ 换栈指针，**零地址空间动作**（BSP 静态恒等映射、共享一张页表）。

**两个反直觉点**（本篇最有教学价值的部分）：
1. UniProton 单次切换保存的寄存器**更多**（36 槽 vs 12 寄存器+sp）——但总路径反而更短，因为省掉了 pt_regs 双份保存、TIF 循环、锁、TTBR0/TLB、钩子开销。
2. Linux `cpu_switch_to` 只存 12 个寄存器+sp 是安全的——切换发生在 `schedule()` 函数调用边界，caller-saved 寄存器（x0-x18，AAPCS64）由编译器放进了旧任务自己的栈帧；UniProton 的恢复点经 eret 任意，帧必须自包含。**这不是谁更聪明，是切换模型不同。**

本篇两边统一 ARM64（AArch64）视角。注意两侧异常级并不同构：Linux 是 EL0（用户）/EL1（内核）两级；UniProton 任务与内核同在 EL1——hi3093 从 EL2 引导后 `eret` 降 EL1h（demos/hi3093/bsp/start.S 的 OsElxState/OsEl2SwitchToEl1），任务 SPSR 初值 0x305 即 M[3:0]=0101=EL1h（prt_port.c:178）。因此开销差异的主因不是特权级跨越，而是"每任务地址空间+安全框架" vs "单一静态恒等映射+直调"。

## 1. 机制原理

**隔离模型差异是开销之源**（arm64 上双方内核同在 EL1）。svc #0 陷入要求：完整现场保存（内核不知道用户态会以什么寄存器状态进来）→ 安全框架（ptrace/seccomp/audit 是系统调用的必经关卡，不只是调试选项）→ 分发表间接跳转 → 退出前 TIF 标志循环复查（重调度/信号/resume 回调），全部清零才允许 eret。

**上下文切换的两层代价**：寄存器层（必付）+ 地址空间层（Linux 独有）。TTBR0_EL1/TTBR1_EL1 写只是两次系统寄存器写，真正的连带代价是 TLB 失效：ASID 代际回卷（rollover）时的 tlbi vmalle1 全刷——UniProton 整个体系里没有这层（任务切换路径零 TTBR 写）。

## 2. 架构设计对比

```
Linux syscall（EL0→EL1）                       UniProton API（EL1 直调）
 用户态 svc #0 ──► vectors EL0 sync 槽         应用 bl PRT_TaskDelay
  kernel_entry 宏(entry.S:260)                  └─ 普通函数：检查中断/锁状态
  stp 成对压 30 GPR + ELR/SPSR                     ├ OsIntLock(DAIFSet)
  → pt_regs (336B)                                  ├ 出就绪队/挂延时链
  el0t_64_sync_handler(entry-common.c:875)          └ OsTskScheduleFastPs
  ├ el0_svc: enter_from_user_mode                      └ OsTaskTrap 切换
  │  (ptrace/seccomp/audit 在 C 层)
  ├ do_el0_svc → el0_svc_common                 Linux 切换（context_switch）
  │  invoke_syscall: sys_call_table[x8]          ├ switch_mm_irqs_off
  └ syscall_exit_to_user_mode                    │   check_and_switch_context
     exit_to_user_mode_loop                      │   写 TTBR0/TTBR1(ASID) ★
     (resched/信号/resume 循环)                   │   ASID 回卷时 tlbi vmalle1
  eret ◄── 恢复 34 槽现场                        ├ FPU 懒切换(per-CPU fpsimd)
                                                └ cpu_switch_to：12 寄存器+sp
UniProton 切换（两种入口汇聚）
 主动阻塞: OsTaskTrap(手工拼 SPSR 伪造异常语义) UniProton: OsHwiDispatcher/OsTaskTrap 保存
 中断唤醒: IRQ→OsHwiDispatcher(硬件已压 SPSR)     31 GPR+xzr+ELR/SPSR/FAR/ESR
   └──────► OsMainSchedule(纯C，选任务)           → TCB[0]=sp
              └► OsTskContextLoad → eret           → OsTskContextLoad 载帧
                                                   → eret ★无地址空间动作
```

## 3. 关键数据结构：两种"帧"

**L1. `struct pt_regs`** — openEuler-6.6 arch/arm64/include/uapi/asm/ptrace.h:88 user_pt_regs：31 个 x + sp/pc/pstate = 34×8 = **272B**；内核版 arch/arm64/include/asm/ptrace.h:178-204 再加 **orig_x0**（syscall restart 回滚用）、syscallno、pmr_save、stackframe[2] 等 = **336B**（PT_REGS_SIZE）。这是"内核栈上的异常帧"，含诊断字段（far/esr 供缺页处理）。

**U1. `struct TskContext`** — UniProton src/include/uapi/hw/armv8/os_cpu_armv8.h:51-94：

```c
struct TskContext {                 /* 任务栈顶的切换帧，36 槽 288B；FPU 版 816B */
    __uint128_t q[32];              /* OS_OPTION_HAVE_FPU 时 q0-q31(512B) */
    uintptr_t fpcr, fpsr;           /*   加 FPCR/FPSR，共 528B */
    uintptr_t elr;                  /* eret 的 pc——"伪造被中断过"的关键 */
    uintptr_t spsr;                 /* PSTATE：DAIF/NZCV/M(异常级)/SP 选择 */
    uintptr_t far, esr;             /* 诊断槽（切换路径不填，仅异常路径用） */
    uintptr_t xzr;                  /* 对齐占位 */
    uintptr_t x30, x29, x28, ..., x01, x00;   /* 31 个 GPR */
};                                  /* 注意：sp 不占槽，由帧基址+帧长恢复 */
```

（288B 有旁证：prt_hw_exc.S `.equ OS_TSKCONTEXT_SIZE, 0x120`）

对比：pt_regs 面向异常处理（要诊断信息），TskContext 面向任务恢复（只需 ELR/SPSR 驱动 eret；far/esr 槽在切换路径上跳过不填）。

**L2. `struct thread_struct`** — openEuler-6.6 arch/arm64/include/asm/processor.h:140-157：`cpu_context` 只存 `x19-x28/x29/lr/sp`——`cpu_switch_to` 需要的全部；FPSIMD 状态另存于 thread，但切换走 per-CPU 懒路径，不进 cpu_switch_to。

**U2. `struct TagTskCb` 栈字段** — UniProton src/core/kernel/include/prt_task_external.h:106-135：`stackPointer` 是**第一个字段**（偏移 0，汇编直写）、`topOfStack`（:135 水位检查）。最小栈 `OS_TSK_MIN_STACK_SIZE = ALIGN(0x1D0+0x10+0x4, 16) = 0x1F0`（os_cpu_armv8_external.h:106）。

## 4. 关键代码路径走读

### 4.1 Linux：一次 write() 的全程

**① 入口（汇编）** — arch/arm64/kernel/entry.S:619 `vectors` 按 SP_EL0/SP_ELx × Sync/IRQ/FIQ/SError × AArch64/32 共 16 槽（EL0 sync 槽 :633）；命中槽 `kernel_ventry` 后进 :260 `kernel_entry` 宏：**15 对 stp**（:269-283）把 x0-x29 共 30 个 GPR 成对压入 pt_regs；EL0 入口把 current 装回 sp_el0（:287-289）；`mrs x22,elr_el1; mrs x23,spsr_el1`（:356-357）存入 S_PC/S_PSTATE 槽（:380）——硬件自动压 ELR/SPSR，软件补齐其余 34 槽；syscallno 预置 NO_SYSCALL（:382-386）。随后 entry_handler 宏（:762）`bl el0t_64_sync_handler`（entry-common.c:875）按 ESR_EL1.EC 分发，EC=0x15（SVC64）→ `el0_svc`（:828）→ `do_el0_svc`（syscall.c:175）。

**② 分发与安全框架（C 层）** — arch/arm64/kernel/syscall.c:76 el0_svc_common：

```c
regs->orig_x0 = regs->regs[0];
regs->syscallno = scno;                                   /* scno 来自 regs->regs[8] */
if (has_syscall_work(flags))                              /* _TIF_SYSCALL_WORK */
        scno = syscall_trace_enter(regs);                 /* ptrace→seccomp→tracepoint→audit */
invoke_syscall(regs, scno, sc_nr, syscall_table);         /* :135 → :41 */
    /* invoke_syscall 内：syscall_table[array_index_nospec(scno, sc_nr)](regs)  防推测执行 */
    syscall_set_return_value(current, regs, 0, ret);
syscall_trace_exit(regs);                                 /* 有 trace 需求才走 */
```

取号用 **x8**（do_el0_svc:190 `el0_svc_common(regs, regs->regs[8], ...)`，x0-x5 传参是 AAPCS64 约定）。

> ⚠ openEuler 差异勘误（两库 diff 复核）：arm64 系统调用分发逻辑两版一致；openEuler 差异一是行号整体后移（社区 6.6.159 syscall.c invoke_syscall:40/do_el0_svc:151，openEuler :41/:175；entry.S kernel_entry 198→260、vectors 520→619、cpu_switch_to 826→1072），二是新增 `CONFIG_FAST_SYSCALL`/`CONFIG_DYNAMIC_XCALL` 低开销系统调用特性——vectors 的 EL0 sync 槽改用 `sync_ventry`（entry.S:630-634），C 侧 `el0_fast_syscall`/`el0_xcall_syscall`（entry-common.c:850-870）用 fast_enter/fast_exit 简化进出场，且可配 `CONFIG_SECURITY_FEATURE_BYPASS` 绕过部分安全框架；syscall.c 另有 `do_el0_xcall`（:167）与 `hijack_syscall`（:186-189）动态改写 syscall 表入口。

**③ 退出循环** — kernel/entry/common.c:93-137 `exit_to_user_mode_loop`：`_TIF_NEED_RESCHED → schedule()`；`_TIF_SIGPENDING → arch_do_signal_or_restart()`（syscall restart 就在这）；`_TIF_NOTIFY_RESUME → resume_user_mode_work()`。**全部清零才 eret**——这是每次系统调用的固定保险费。

> ⚠ openEuler 差异：openEuler-6.6 回移了 mainline 6.12 的 entry 框架重构——enter_from_user_mode/syscall_enter_from_user_mode/exit_to_user_mode 等小函数移入 include/linux/entry-common.h 内联实现，kernel/entry/common.c 仅 407 行（社区 6.6 约 470 行），exit_to_user_mode_loop 位于 93-137；TIF 循环逻辑与社区 6.6 完全一致。

**④ 上下文切换** — kernel/sched/core.c:5283-5343 `context_switch`：`switch_mm_irqs_off`（:5323）→ `switch_to`（:5339）。地址空间切换在 arch/arm64/mm/context.c:218 `check_and_switch_context`：ASID 代际匹配走免锁 fastpath 直达 `cpu_do_switch_mm`（:354-382，mmu_context.h:57 的 `cpu_switch_mm` 是 inline 包装）——ASID 写 TTBR1_EL1（TCR.A1=1）、用户页表物理地址写 TTBR0_EL1 + isb，**常规切换无 TLB 失效**；ASID（8/16 位，ID_AA64MMFR0_EL1 查得）耗尽时 `new_context`（:158）代际回卷（ASID_FIRST_VERSION=1<<16）→ `flush_context`（:104）给全核挂 tlb_flush_pending，各核下次切换 `tlbi vmalle1` 全刷。寄存器切换 `cpu_switch_to`（entry.S:1072-1099）只存 **12 个寄存器+sp**：x19-x28、x29、lr、sp（:1077-1083），恢复端 `msr sp_el0, x1` 重装 current（:1093）；FPSIMD 一概不存——`__switch_to`（process.c:528）调 `fpsimd_thread_switch`（:533）做 per-CPU 懒切换。

### 4.2 UniProton：一次 PRT_TaskDelay 的全程

**① 任务栈初始化（伪造现场）** — `OsTskContextInit`（src/arch/cpu/armv8/common/prt_port.c:135-180）：栈顶下移一个 TskContext（288B）→ 填魔数（x1=0x01010101 … x29=0x29292929 调试用）→ `x30 = elr = funcTskEntry`（:172,:175，**统一任务入口** OsTskEntry，src/core/kernel/task/amp/prt_task_init.c:164 从 TCB 取 args 调 taskEntry）→ `spsr = 0x305`（:178，M[3:0]=0101 → EL1h、D/A mask——任务与内核同在 EL1）。

**② 主动阻塞：OsTaskTrap（手工伪造异常返回语义）** — prt_dispatch.S:128-166（OsTaskTrap 取 &g_runningTask->sp 后进入 OsTskTrapSmp）：GENERAL_REGS_SAVE（:25-41，x2-x30+xzr；x0/x1 先压）→ **手工拼接 SPSR**（:139-148）：`mrs x3,DAIF` + `mrs x2,NZCV` orr 出 PSTATE，再 `orr #(1<<2)`（M=EL1）、`orr #1`（选 SPx）——硬件只在真异常时自动压 SPSR_EL1，主动切换要"伪造得像被中断过"，恢复端 eret 逻辑才唯一；**ELR 槽写 bl 调用点的返回地址 x30**（:146-148，far/esr 槽跳过不填）；`str x1,[x0]` 写 TCB[0]=sp（:150-152）、`mov sp, g_sysStackHigh` 换系统栈（:154-163）、`B OsMainSchedule`（:164，其后 loop1 死循环——调度器不返回）。若调度回原任务，eret 落回帧内 x30——就像"调用了一个刚返回的函数"。

**③ 中断驱动切换** — IRQ 打在 EL1h 走向量表 0x280 槽（prt_vector.S:147-150 `b OsHwiDispatcher`，OsVectorTable :126 装入 VBAR_EL1，见 prt_reset_vector.S:43）：`OsHwiDispatcher`（prt_dispatch.S:174-194）GENERAL_REGS_SAVE（:182）+ `mrs spsr_el1/elr_el1`（:184-185，硬件已自动压好，直接读）+ 跳过 far/esr 槽 → `BL OsHwiDispatch`（C，prt_hwi.c:182）→ `OsHwiDispatchHandle`（:130）→ `OsHwiDispatchTail`（prt_sched_single.c:118）尾部 `OsMainSchedule`（:146）。

**④ 决策与恢复** — `OsMainSchedule`（prt_sched_single.c:55-81，纯 C 无锁——单核+中断已关）：查 `OS_FLG_TSK_REQ` → `RUNNING_TASK = g_highestTask` → `OsTskContextLoad`（prt_dispatch.S:202-204）载帧 → 恢复 ELR/SPSR_EL1 + `dsb sy; isb`（:210-215）→ **eret**（:232）直达新任务。

**⑤ API 直调证据** — `PRT_TaskDelay`（src/core/kernel/task/amp/prt_task_minor.c:93-135）：校验（中断中/锁中调 Delay 报错）→ 出就绪队/挂延时链 → `OsTskScheduleFastPs`。pthread 适配同理：`pthread_create` 内部直调 `OsTaskCreateChkAndGetTcb/OsTskContextInit/PRT_TaskResume`（src/libc/litelibc/src/thread/pthread_create.c:185-220），全程无陷入。

### 4.3 寄存器保存集合逐项对比

| 寄存器组 | Linux cpu_switch_to（entry.S:1072-1099） | UniProton OsHwiDispatcher/OsTaskTrap（prt_dispatch.S） |
|---|---|---|
| x19-x28（10个） | 保存（5 对 stp，:1077-1081） | 保存 |
| x29(fp) / x30(lr) | 保存（:1082-1083） | 保存 |
| sp | 保存（thread.cpu_context.sp 槽，:1076/:1082） | 不占槽（帧基址即恢复后 sp） |
| x0-x18（19个） | **不保存**（AAPCS64 caller-saved，切换点=函数调用边界） | **保存**（eret 恢复点任意，帧须自包含） |
| xzr | 不适用 | 占 1 槽（对齐占位，:40） |
| pc / 状态 | 无（lr 驱动 ret） | ELR_EL1 + SPSR_EL1（eret 驱动；主动切换时 :139-148 手工拼 SPSR） |
| 诊断字段 | 无 | FAR/ESR 占 2 槽（切换路径跳过不填） |
| FPU | **不在此存**：per-CPU 懒切换（__switch_to→fpsimd_thread_switch，process.c:533；内核不用 FPSIMD，仅 d8-d15 属 callee-saved） | OS_OPTION_HAVE_FPU 时 q0-q31+FPCR/FPSR 无条件全存（528B） |
| 合计 | **12 个寄存器 + sp（13 槽 104B）** | **31 GPR + xzr + 4 系统槽（36 槽 288B）** |
| 附带动作 | TTBR0/TTBR1+ASID 切换 ± tlbi（回卷时）、msr sp_el0 重装 current、PAC 密钥/SCS（:1094-1096） | 写一个 TCB 字段 + 换栈指针 |

## 5. 实现差异总结表

| 维度 | Linux 6.6 (EL0→EL1) | UniProton (EL1 单级) |
|---|---|---|
| API 进入方式 | svc #0 + 34 槽现场压栈（pt_regs 336B） | bl 调用，0 额外压栈 |
| 进入安全开销 | user dispatch/ptrace/seccomp/tracepoint/audit | 无 |
| 退出安全开销 | rseq + TIF 循环复查（resched/信号/resume） | OsMainSchedule 一次判断后载帧 |
| 切换步骤 | mm 切换→FPU 懒切换→12 寄存器+sp→finish_task_switch | 存帧→TCB[0]=sp→选任务→载帧→eret |
| TLB/页表代价 | 硬件 ASID（8/16 位）：常规仅写 TTBR；代际回卷时 tlbi vmalle1 全刷 | **零**（任务切换路径无 TTBR 写；仅异常诊断快照 mrs 读 ttbr0/ttbr1，prt_hw_exc.S:94-96） |
| 同步原语竞争 | futex：用户态自旋失败→又一次完整陷入+切换 | 信号量 Pend 即阻塞切换，无陷入前缀 |
| 返回指令 | eret（退出前 TIF 清零循环） | eret（帧自带 SPSR_EL1） |

## 6. 代码阅读指导

**推荐路线**：

1. `openEuler-6.6/arch/arm64/kernel/entry.S:260-386`（kernel_entry 宏）→ `:619-647`（vectors 16 槽）→ `include/uapi/asm/ptrace.h:88`（user_pt_regs）→ `include/asm/ptrace.h:178-204`（内核 pt_regs 336B）
2. `openEuler-6.6/arch/arm64/kernel/entry-common.c:828-838,875`（ESR 分发→el0_svc）→ `arch/arm64/kernel/syscall.c:36-190`（invoke_syscall:41/:135、do_el0_svc:175）→ `kernel/entry/common.c:28-83,93-137,183-224`（进/出通用框架三段）
3. `openEuler-6.6/kernel/sched/core.c:5283-5343`（context_switch 两层）→ `arch/arm64/kernel/entry.S:1072-1099`（cpu_switch_to 12 寄存器+sp）→ `arch/arm64/mm/context.c:42-60,104-215,218-280,354-382`（ASID 位数/回卷/check_and_switch_context/cpu_do_switch_mm 双 TTBR 写）
4. `openEuler-6.6/kernel/futex/syscalls.c:192-212` + `waitwake.c:323-328`（用户态同步进内核的对照）
5. `UniProton/src/include/uapi/hw/armv8/os_cpu_armv8.h:51-94`（TskContext 帧布局，288B/816B）
6. `UniProton/src/arch/cpu/armv8/common/prt_port.c:135-180`（伪造初始栈帧，spsr=0x305）
7. `UniProton/src/arch/cpu/armv8/common/hwi/prt_dispatch.S:202-232`（OsTskContextLoad→eret）→ `:128-166`（OsTaskTrap，重点 :139-148 手工拼 SPSR）→ `:174-194`（OsHwiDispatcher）
8. `UniProton/src/arch/cpu/armv8/common/hwi/prt_vector.S:126-177`（VBAR_EL1 向量表）+ `prt_reset_vector.S:43`（装 VBAR_EL1）+ `demos/hi3093/bsp/start.S`（OsElxState/OsEl2SwitchToEl1 降 EL1h）
9. `UniProton/src/core/kernel/sched/prt_sched_single.c:55-81`（OsMainSchedule）
10. 对照收尾：`demos/hi3093/apps/openamp/main.c:167`（应用直调 PRT_TaskDelay）+ `src/libc/litelibc/src/thread/pthread_create.c:84-106`

**实验建议**：Linux 侧 `strace -T -e trace=all` 看一次 write 的耗时下限；再写程序循环 1e6 次 getpid（最薄系统调用）测平均/最坏耗时。UniProton 侧对照 rhealstone 的 task-switch 用例（`cd demos/hi3093/build && sh build_app.sh` 生成 openamp 镜像，shell 内 reg 'rhealstone' 命令运行，见 apps/openamp/main.c:97；测试本体在 testsuites/rhealstone）。

**思考题**：

1. Linux `cpu_switch_to` 只存 12 个寄存器+sp 却安全——"另外那些寄存器"在什么路径上被保存了？（提示：svc/irq 边界 kernel_entry 的 15 对 stp（entry.S:269-283）与 schedule() 边界的分工——编译器在调用点已把 caller-saved 的 x0-x18 放进旧任务自己的栈帧。）
2. `OsTskTrapSmp` 为什么要手工拼接 SPSR 的 DAIF/NZCV/EL/SP 位（prt_dispatch.S:139-148），而 `OsHwiDispatcher` 入口不需要？（提示：硬件异常自动把 PSTATE 压入 SPSR_EL1，bl 调用不会。）
3. 若给 UniProton 加 MMU/每任务地址空间与 EL0 用户任务支持，prt_dispatch.S 的帧和 OsMainSchedule 至少要增加哪些动作？开销比例会发生什么变化？（对照 arm64 check_and_switch_context：常规切换写 TTBR0/TTBR1 带 ASID、免 TLB 失效，ASID 代际回卷才 tlbi vmalle1 全刷——硬件 ASID 正是为摊薄这层代价而设，context.c:158-215。）

---

*本篇行号基于 openEuler-6.6（arm64）与 UniProton（armv8，参考 demo hi3093）当前 checkout 实测；arm64 6.6 的寄存器切换 cpu_switch_to 在 arch/arm64/kernel/entry.S、地址空间切换在 arch/arm64/mm/context.c（check_and_switch_context/cpu_do_switch_mm）、系统调用分发在 arch/arm64/kernel/syscall.c。openEuler arm64 相对社区 6.6.159 行号整体后移，并新增 FAST_SYSCALL/XCALL（低开销系统调用）与 tlbid（延迟 TLB 失效，mm/tlbid.c）等特性。（futex 勘误：should_fail_futex() 故障注入并非 openEuler 特有，社区 6.6.159 的 syscalls.c:201 同样存在，两侧 SYSCALL_DEFINE6(futex) 逐行一致。）未确认项：arch_do_signal_or_restart 的 restart 细节、SMP 版 OsMainSchedule 行为细节（hi3093 已启用 OS_OPTION_SMP，defconfig:34）、UniProton IRQ 路径切换系统栈的精确位置（OsHwiDispatchHandle 内部处理）。*
