# 11 - EEVDF 与 CFS 核心差异及调度参数调优（openEuler OLK-6.6 实测）

> 本文所有行号以本地 `openEuler-6.6/` 实测为准；凡涉及"openEuler 自研还是社区本就有"的结论，均与 `linux-6.6.159/`（社区 stable）双侧 diff 验证过（方法见 docs/10）。
> 前置阅读：docs/03（调度决策全景）、docs/04（抢占与临界区）。本文聚焦 fair class 内部：CFS → EEVDF 的演化。

---

## 0. 五分钟调度基础（零背景补课）

| 概念 | 一句话解释 | 代码对应 |
|---|---|---|
| 运行队列 runqueue | 每 CPU 一个的"排队处"，就绪（可运行）的任务在这里等 CPU | `struct rq`，`rq->cfs_rq` |
| 调度类 sched_class | 内核把任务按实时性分成五档，从高到低：stop > deadline > RT > fair(普通) > idle | `kernel/sched/sched.h` 的 `sched_class_above()` |
| 调度策略 policy | 用户可见的档位：`SCHED_FIFO`/`SCHED_RR`（RT 类）、`SCHED_OTHER`/`SCHED_BATCH`/`SCHED_IDLE`（fair 类）、`SCHED_DEADLINE` | `task_struct->policy` |
| 上下文切换 | 从任务 A 切到任务 B：保存/恢复寄存器，换地址空间 | `__schedule()`（core.c:6617）→ `context_switch()` |
| 时间片 slice | 一个任务一次"上 CPU 合同"能连续跑多久 | `sched_entity->slice`（EEVDF 下由 `sysctl_sched_base_slice` 决定，fair.c:100） |
| 抢占 preempt | 不等当前任务跑完就换人。分两种：**自愿**（任务自己调用会阻塞的 syscall）和**非自愿**（内核标记 `TIF_NEED_RESCHED` 强制换） | `resched_curr()`；`preempt_enable()`/中断返回时检查 |
| nice 值 | fair 类内的优先级数字，-20（最高）~19（最低），本质是**权重**查表 `NICE_0_LOAD >> ...` | `prio_to_weight[]`（core.c） |
| vruntime | "虚拟运行时间"——按权重折算后的公平记账刻度，是 CFS/EEVDF 共用的地基 | `sched_entity->vruntime` |

公平调度器要回答的两个永恒问题：
1. **选谁跑**（pick）；
2. **跑多久**（slice / 何时该换人，preempt）。

CFS 和 EEVDF 的全部差异，都是对这两个问题的不同回答。

---

## 1. CFS 回顾：EEVDF 继承了什么、丢掉了什么

### 1.1 CFS 的记账与选人（6.6 之前）

CFS（Completely Fair Scheduler，2007~2023）的核心思想：**按权重把物理时间折算成虚拟时间，谁欠得多（vruntime 最小）谁先跑**。

- 记账（EEVDF 原样继承）：
  `vruntime += Δexec × NICE_0_LOAD / weight`（`calc_delta_fair()`，fair.c）
  nice 越低权重越大，同样 1ms 物理时间记的 vruntime 越少 → 能多跑。
- 选人：红黑树以 vruntime 为键，**永远选最左边的**（`__pick_first_entity()`），O(logN)。
- 睡眠唤醒的"补偿"：睡醒任务 vruntime 落后太多，若不处理会独占 CPU；CFS 的做法是唤醒时把 vruntime 拉到 `min_vruntime - 偏移量` 附近（`place_entity()` 旧版），并靠 `GENTLE_FAIR_SLEEPERS`、`wakeup_granularity_ns` 等一堆**启发式**防止睡眠任务反过来欺负 CPU 密集任务。

### 1.2 CFS 的三个设计痛点（EEVDF 的动机）

1. **交互任务靠猜**：内核不知道谁是"交互型"（睡得多、要得急），只能用 vruntime 补偿启发式近似。"多睡少记"可被恶意任务利用（sleep 后抢跑）；调保守了交互任务又卡。
2. **延迟保证是隐式的**：CFS 里每个任务的调度延迟 ≈ `nr_running × min_granularity`（近似 period），任务越多延迟越差，但**没有任何机制显式承诺"你最晚何时上 CPU"**。
3. **公平性只有记账、没有结算**：vruntime 只管"从现在起谁欠得多"，任务睡了/退出了，它历史上欠的账（该得而未得的 CPU 时间）就凭空消失——真正按权重比例的"加权公平"（WFQ 意义上的）并不严格成立。

### 1.3 6.6 是分水岭

- **6.6 起社区默认调度器即 EEVDF**（Peter Zijlstra 重写 fair.c，替换的正是"选最左 + 启发式唤醒"那套）；
- `sched_latency_ns` / `sched_min_granularity_ns` / `sched_wakeup_granularity_ns` 等 sysctl **被整体删除**（本仓双侧实测：openEuler 与社区 6.6.159 的 `sched_update_scaling()` 里都只剩 `sched_base_slice` 一个，fair.c:1345~1356）——**网上大量"调 latency_ns 优化延迟"的老文章在 6.6+ 内核上已是无效参数**，这是混部调优第一个要避的坑；
- CFS 时代的 buddy（next/last buddy 缓存）大幅弱化，唤醒抢占判定改为纯 EEVDF 公式（见 §2.4）。

---

## 2. EEVDF 原理与 openEuler 代码实现链路

EEVDF = **Earliest Eligible Virtual Deadline First**。名字里就是两个判据：

- **Eligible（有资格）**：按加权公平算，你还"亏着"（欠你的服务没给够）→ 可以跑；
- **Virtual Deadline（虚拟截止期）**：资格用完的时刻 = 本轮请求到期时刻 → 亏得最久、最急的先跑。

### 2.1 三个公式（全部来自 fair.c 注释与代码实测）

```
① 虚拟时钟   V = Σ(w_j·v_j) / Σ(w_j)          —— 队列的"加权平均 vruntime"
② 虚拟滞后  vl_i = V - v_i   （lag_i = w_i·vl_i）
              vl_i ≥ 0  ⇔  v_i ≤ V  ⇔ eligible  （fair.c:989-1012 注释）
③ 虚拟截止  vd_i = ve_i + r_i/w_i              （fair.c:1379, update_deadline()）
              r_i = 请求时长（slice），w_i = 权重
```

直观解读：
- **V 是"集体平均进度"**。你的 vruntime 比平均进度慢 → 你被亏欠了 → eligible。
- **deadline 是"这一轮服务的到期日"**：你上 CPU（ve_i 时刻）后，按你的权重折算，最晚应在此前获得 slice 的服务。所有 eligible 的任务里 **deadline 最早**的被选中。

### 2.2 实现链路一：O(1) 增量维护 V —— `avg_vruntime()`（fair.c:926）

计算加权平均看起来要遍历全队列，实际靠三个增量字段避免遍历（这是实现上的精彩点）：

- `sum_w_vruntime += key × weight`（入队/出队时更新，`sum_w_vruntime_add/sub()`，fair.c:885/895）；
- `zero_vruntime` 缓存基准点，`update_zero_vruntime()`（fair.c:904）每次只加差值；
- `avg_vruntime()` 由此 O(1) 返回 V，并顺带把基准点推进到最新（懒更新）。

三个固定调用点（fair.c:918-921 注释列明）：`place_entity()`（入队前）、`update_entity_lag()`（出队前）、`update_deadline()`（slice 到期）。

### 2.3 实现链路二：选人 —— `pick_eevdf()`（fair.c:1253）

红黑树现在按 **vruntime** 组织，但要找的是"eligible 中 deadline 最早"的实体，于是：

```c
fair.c:1281-1287  正在跑的 curr 若已不 eligible（且无 slice 保护）→ 直接出局
fair.c:1290-1293  最左节点（vruntime 最小）若 eligible → 必然是它（欠最多者 deadline 也最早）
fair.c:1296-1322  否则在树上做"堆搜索"：利用 augmented 的子树 min_vruntime 剪枝，
                  只走"子树里还有 eligible 实体"的分支，找到第一个 eligible 节点即停
```

关键结构：`struct sched_entity` 增加了 `vlag`（虚拟滞后，s64）与 `deadline`（u64）；树的每个节点缓存子树最小 vruntime（`min_vruntime` augmented RB-tree），`vruntime_eligible()`（fair.c:1013）用 128 位乘法避免溢出（`__int128`，fair.c:1040——是个值得读的工程细节）。

> openEuler 差异点（社区 6.6.159 无）：`entity_eligible()` 开头挂了 `bpf_sched_cfs_tag_entity_eligible()` 钩子（fair.c:1060-1065，CONFIG_BPF_SCHED）——允许 eBPF 程序直接改写单个实体的 eligible 判定，是 openEuler 面向混部场景的自研增强。

### 2.4 实现链路三：睡眠与唤醒 —— `place_entity()`（fair.c:5601）

这是 EEVDF 对 CFS 启发式的正式替代，两步：

**策略 #1：lag 结算（PLACE_LAG，features.h:7）** —— 睡眠不再送时间，而是"存取款"：

```c
fair.c:5618  lag = se->vlag;                      // 睡前记住你亏欠多少
fair.c:5688  lag *= load + w_i; lag /= load;      // 放大补偿 (W+w_i)/W —— 抵消
fair.c:5695  se->vruntime = V - lag;              // 新实体入队会拉低 V，先放大再放回
```

数学上保证：新任务入队把 V 往回拽时，它的有效 lag 不蒸发（fair.c:5632-5683 注释有完整推导）。于是：

- 短高频任务每睡一次，正 lag 保留 → 醒来立刻 eligible、deadline 靠前 → **快**；
- 恶意 sleep 不再有额外奖励：睡着的任务 vlag 不变，醒来只拿回"真正欠它的份额"。

**策略 #2：半片入场（PLACE_DEADLINE_INITIAL，fair.c:5708）**——新任务/醒来的任务首轮 vslice 减半（`vslice /= 2`）：老任务平均已跑完半个 slice，新来者 deadline 提前半片，平滑入场不震荡。

### 2.5 实现链路四：slice 到期与抢占 —— `update_curr()`/`update_deadline()`

- 每个 tick（以及入队、唤醒等记账点）都会调 `update_curr()`（fair.c:1569）：
  ```c
  fair.c:1583  curr->vruntime += calc_delta_fair(delta_exec, curr);  // 记账
  fair.c:1584  resched = update_deadline(cfs_rq, curr);              // slice 到期？
  fair.c:1594  if (resched || resched_next_slice(cfs_rq, curr))      // 或已不 eligible
  fair.c:1595      resched_curr(rq);                                  // 打 TIF_NEED_RESCHED
  ```
- `update_deadline()`（fair.c:1365）：`vruntime` 追上 `deadline` → 本轮"合同"履行完毕，按公式 ③ 立下一轮的约（新 deadline），并返回 true 触发重调度。
- `resched_next_slice()`（fair.c:1522，**openEuler 独有重构**，社区 6.6.159 无此函数）：slice 未满但 curr 已不 eligible 也提前让位。社区版把等价逻辑放在 `update_deadline()` 内部（社区 fair.c:1045-1048 的 `nr_running > 1 → resched_curr`），openEuler 拆出来并叠加 slice 保护，行为更平滑。

### 2.6 唤醒抢占 —— `check_preempt_wakeup_fair()`（fair.c:9848）

新任务 p 入队后要不要立刻踢掉 curr？EEVDF 的回答**不靠粒度参数，靠公式**：

```c
fair.c:9937  if (do_preempt_short(cfs_rq, pse, se))    // PREEMPT_SHORT 特性（默认关）
fair.c:9938      cancel_protect_slice(se);              // 短请求可取消 curr 的 slice 保护
fair.c:9943  if (pick_eevdf(cfs_rq, true) == pse)       // 直接问调度器：现在最该跑的是不是 p？
fair.c:9944      goto preempt;                          // 是 → resched_curr(rq)
```

即"唤醒抢占判定 = 重跑一次选人逻辑，看赢家是否变人"。CFS 时代的 `wakeup_preempt_entity()`（比较 vruntime 差与 wakeup_granularity）在 6.6 已删除。

> openEuler 差异点（社区无）：CONFIG_QOS_SCHED 的 offline 任务无条件让位（fair.c:9917-9920）；CONFIG_SCHED_SOFT_QUOTA 的组间软配额短路（fair.c:9891-9894）。均服务于混部 QoS 分级。

### 2.7 6.6 EEVDF 精简的另一个侧面：`entity_tick()`（fair.c:5952）

CFS 时代 tick 里有 `check_preempt_tick()`（比较 vruntime 差与 min_granularity）。6.6 里该函数**整体消失**，`entity_tick()` 只剩记账 + load_avg 维护——因为"该不该换人"已统一收敛到 `update_curr() → update_deadline()` 的公式判定，tick 不再是独立的抢占决策点。

### 2.8 openEuler OLK-6.6 vs 社区 6.6.159：fair.c 差异清单（双侧 diff 实测）

| 差异点 | 社区 6.6.159 | openEuler OLK-6.6 | 归因 |
|---|---|---|---|
| `resched_next_slice` / slice 保护重构 | 无（resched 在 update_deadline 内） | fair.c:1522 有 | openEuler 自研/回移 |
| `custom_slice`（per-task 自定义 slice） | 无 | fair.c:1375 有 | 回移社区 6.7+（sched_ext/prctl 系列） |
| PREEMPT_SHORT / do_preempt_short / protect_slice | 无 | fair.c:1209-1240, 1530-1549，**默认关闭**（features.h:19） | 回移社区 6.10 系列 |
| BPF_SCHED eligible 钩子 | 无 | fair.c:1060-1065 | openEuler 自研 |
| CONFIG_QOS_SCHED（qos_level 分级 + 节流 + SMT 驱逐） | 无 | fair.c:9952 起 | openEuler 自研（混部核心特性） |
| CONFIG_SCHED_SOFT_QUOTA（软配额） | 无 | fair.c sysctl `sched_soft_runtime_ratio`（fair.c:337） | openEuler 自研 |
| `sysctl_sched_base_slice` 默认值 | 700000（0.7ms，fair.c:78） | 700000（fair.c:100） | **相同**（两侧实测一致，勿信"社区 0.75ms"的旧说法） |

---

## 3. CFS vs EEVDF：设计差异总表与核心优势

| 维度 | CFS（≤6.5） | EEVDF（6.6+，本仓） |
|---|---|---|
| 理论模型 | 加权轮转的工程近似 | WFQ + Virtual Deadline（有严格公平定义） |
| 选人判据 | vruntime 最小（红黑树最左） | eligible 集合中 **deadline 最早**（pick_eevdf，fair.c:1253） |
| 公平记账 | vruntime 单变量 | vruntime + **vlag**（滞后量，可结算可继承） |
| 睡眠补偿 | vruntime 置到 min_vruntime-偏移（送虚拟时间） | **lag 保留**，醒来即还（place_entity 策略 #1） |
| 唤醒抢占 | vruntime 差 > wakeup_granularity_ns | `pick_eevdf()==p` 公式判定（fair.c:9943） |
| 时间片 | 隐式：period=latency，随 nr_running 动态切分 | 显式：`slice = base_slice`（0.7ms），vdeadline 即"合同到期" |
| 短任务偏好 | 无显式机制，靠启发式 | **天然**：短 slice → deadline 提前 → 优先获服务 |
| 可调参数 | latency/min_granularity/wakeup_granularity/… 一组 | 收敛到 base_slice + cgroup cpu.weight |
| nice 语义 | 权重 → vruntime 增速 | 同为权重，但**结算意义更强**（lag×w），高 nice 短任务受害更小 |

### EEVDF 的主要优势（对应"为什么 6.6 要换"）

1. **交互/短任务延迟天然更优，且无需启发式**：短任务请求小（slice 小）→ vd = ve + r/w 更近 → 一 eligible 立刻被 pick。CFS 需要用"睡眠补偿 + 粒度豁免"拼出来的效果，EEVDF 一个公式给出。实测上典型收益是**尾延迟（p99）改善**——恰恰是混部场景最痛的指标。
2. **公平可结算**：lag 明确记录"欠多少"，睡眠/唤醒/renice/进出 cgroup 都按 lag 结算，杜绝 sleep 作弊与 vruntime 赠送的不公平。
3. **参数模型极简**：砍掉 latency/min_granularity/wakeup_granularity 三件套，"跑多久"一个 base_slice 说清，跨负载行为可预测——对调优者是天大的好事（少三个互相打架的旋钮）。
4. **为 sched_ext（eBPF 调度器）铺路**：EEVDF 的请求/截止期模型让用户态可编程调度器有清晰的接入语义（openEuler OLK-6.6 已回移 sched_ext，见 `kernel/sched/ext.c`）。

一句话总结：**CFS 回答"谁欠得多"，EEVDF 同时回答"谁欠得多（eligible）"和"谁最急（deadline）"** —— 多出的第二个维度就是实时性与公平性兼得的关键。

---

## 4. 问题一：哪些常见系统调用会触发调度

分两大类记忆（对照 docs/03 的总链路图）：

### 4.1 主动让出（任务自己阻塞 → `schedule()`）

| 系统调用 | 场景 | 内核路径（入口→终点） |
|---|---|---|
| `nanosleep` / `clock_nanosleep` | sleep | `hrtimer_nanosleep()` → 入定时器队列 → `schedule()` |
| `futex`(WAIT) | 锁等待/条件变量（pthread_mutex/cond 底层） | `futex_wait()` → `futex_wait_queue()` → `schedule()` |
| `read`/`write`/`recv`/`send` | 磁盘/网络/管道无数据 | 驱动 `wait_queue` → `schedule()` |
| `poll`/`select`/`epoll_wait` | 多路等待 | `do_poll()` → `schedule()` |
| `wait4`/`waitid` | 等子进程 | `do_wait()` → `schedule()` |
| `sched_yield` | 主动让位 | `yield_task_fair()`（fair.c:10720） |
| `exit` | 线程退出 | `do_exit()` → `do_task_dead()` → `__schedule()` |
| `membarrier`/`rseq` 等 | 少见 | — |

### 4.2 唤醒他人 / 改变格局（产生"别人"的调度点）

| 系统调用 | 效果 | 关键函数 |
|---|---|---|
| `futex`(WAKE) / `pthread_cond_signal` | 唤醒等待者 | `try_to_wake_up()`（core.c:4093）→ `check_preempt_wakeup_fair()`（fair.c:9848） |
| `write`（解锁管道/套接字） | 唤醒读者 | 同上 |
| `kill`/`tgkill`（信号） | 唤醒或打断 | 信号投递路径 |
| `fork`/`clone` | 新任务入队 | `sched_fork()` → `wake_up_new_task()` → `check_preempt_wakeup_fair()`（带 WF_FORK） |
| `execve` | 置换地址空间，短暂让出 | `exec` 路径中 `schedule()` |
| `sched_setscheduler` / `setpriority` / `prctl`(nice) | 改类/改权重 | `check_class_changed()`（core.c:2091）→ 可能立刻 `resched_curr()` |
| `mmap` 缺页（第一次访存） | 磁盘 IO → 睡眠 | fault 路径 |
| `sendmsg` 到拥塞队列 | 睡眠等待 sndbuf | — |

### 4.3 非自愿抢占（不是 syscall，而是内核时机）

- **tick 到期**：`sched_tick()` → `task_tick_fair()`（fair.c:15266）→ `entity_tick()`（fair.c:5952）→ `update_curr()` → slice 耗尽 → `resched_curr()`；
- **唤醒抢占**：低延迟任务醒来，公式判 p 该跑 → 设 `TIF_NEED_RESCHED`；
- **切换执行点**：`TIF_NEED_RESCHED` 只是个"请求"，真正切换发生在下一次 `preempt_enable()`、中断/异常返回（`ret_to_user` 检查）或显式 `schedule()` 时——CONFIG_PREEMPT_DYNAMIC 决定内核态内可否即时切换（voluntary/full）。

> 实操观察：`perf sched record -- sleep 5 && perf sched latency` 能直接看到上面每条路径实际发生时的"排队耗时"。

---

## 5. 问题二：openEuler 可调调度参数全景（/proc、debugfs、cgroup）

### 5.1 `/proc/sys/kernel/sched_*`（注册点分散在 kernel/sched/ 各文件，非 sysctl.c）

| 参数（注册处） | 默认 | 作用 | 调整影响 / 低噪风险 |
|---|---|---|---|
| `sched_rt_runtime_us` / `sched_rt_period_us`（rt.c:44/35） | 950000/1000000 | RT 带宽节流：1s 里 RT 类最多占 0.95s | **低噪必调项**。防 RT 任务饿死全场（WATCHDOG/软锁）；调太大（-1 关闭）→ 一个失控 FIFO 任务可永久占核，背景噪声直接拉满 |
| `sched_rr_timeslice_ms`（rt.c:53） | 100 | SCHED_RR 轮转片 | 调小→RR 任务切换更频（上下文切换开销↑） |
| `sched_cfs_bandwidth_slice_us`（fair.c:246） | 5000 | cgroup 带宽配额的账期粒度 | 调大→限流任务成批被 throttle（毛刺）；调小→锁开销↑ |
| `sched_child_runs_first`（fair.c:238） | 0 | fork 后子进程先跑 | 安全类(沙箱)相关，调度性能影响小 |
| `sched_autogroup_enabled`（autogroup.c:14） | 1 | 按会话(session)自动分组 | **混部常关**：autogroup 会拆散手工 cgroup 分级，让 weight 配置失效；服务器建议 echo 0 |
| `sched_schedstats`（core.c:4610） | 0 | 打开 /proc/schedstat 统计 | 观测时开，常态关——记账有开销（约 1% 级） |
| `sched_util_clamp_min/max`（core.c:4621/4628） | 0/1024 | util 信号钳位（影响频率选择与选核） | min 调高→该任务倾向留在大核/高频；滥用则省电失效 |
| `sched_deadline_period_max/min_us`（deadline.c:31/39） | — | DEADLINE 类准入边界 | 仅 DL 任务相关 |
| `sched_migration_cost_ns`（fair.c:109，const_debug） | 500000 | 任务"热度"窗口： reciente 运行过的任务迁移时被认为 cache 还热 | 调大→唤醒选核更恋家（少迁移、cache 命中高，但负载不均）；调小→均衡更积极。**混部调优常动项** |
| `sched_soft_runtime_ratio`（fair.c:337，**openEuler**） | — | SOFT_QUOTA：组超配额后限速比例 | openEuler 混部特性 |
| `sched_prio_load_balance_enabled`（fair.c:297，**openEuler**） | — | 按优先级感知的负载均衡 | 同上 |
| ~~sched_latency_ns / sched_min_granularity_ns / sched_wakeup_granularity_ns~~ | **已删除** | EEVDF 移除 | **写进老文档/老脚本的这些参数在 6.6 不存在，调了报错或无效**。唯一"粒度"参数是内部变量 `sysctl_sched_base_slice`（fair.c:100，0.7ms），**没有 proc 接口**，不可在线调 |

### 5.2 debugfs：`/sys/kernel/debug/sched_features`（features.h，一次性多开关位图）

实测 openEuler-6.6 有效项（features.h）：

| 特性 | 默认 | 说明 |
|---|---|---|
| PLACE_LAG / PLACE_DEADLINE_INITIAL | 1 | EEVDF 唤醒放置两策略（§2.4）。关掉=回退到无 lag 结算（仅实验用） |
| RUN_TO_PARITY / RUN_TO_PARITY_WAKEUP | 1 | curr 的 slice 保护：slice 承诺未兑现时不被踢（pick_eevdf fair.c:1281-1286） |
| PREEMPT_SHORT | 0（**openEuler 回移，默认关**） | 短 slice 唤醒者可取消 curr 保护（do_preempt_short，fair.c:1530）。开启=交互延迟更低、吞吐略降 |
| NEXT_BUDDY | 0 | 唤醒时预标记 next buddy 缓存 |
| WAKEUP_PREEMPTION | 1 | 唤醒抢占总开关 |
| SCHED_AUTOGROUP... 等 | 见文件 | — |

用法：`echo NO_PREEMPT_SHORT > /sys/kernel/debug/sched_features`（NO_ 前缀=关）。**注意**：该接口需 CONFIG_SCHED_DEBUG，生产内核常常没开——先确认存在再纳入调优方案。

### 5.3 cgroup v2 `/sys/fs/cgroup/<grp>/cpu.*`（cpu 控制器，注册见 core.c:10640 起）

| 文件 | 语义 | 调整效果（对应代码） |
|---|---|---|
| `cpu.weight`（1~10000，默认 100）| 组间份额（原 v1 cpu.shares） | 直接进 `load.weight` 参与 lag/vruntime 折算。**混部核心旋钮**：提权 = 同组任务 vlag 结算更快、deadline 相对靠前 |
| `cpu.weight.nice` | 同上，nice 风格写法（-20~19） | 换算到 weight |
| `cpu.max`（`quota period`，默认 `max 100000`）| 硬带宽限流 | 超额整组被 throttle（CFS bandwidth，fair.c:5988 起）。**约束批处理/长任务的核心手段**；period 调小→限流更平滑但开销↑ |
| `cpu.max.burst` | 超额突发额度 | 允许短时攒额度，削 throttle 毛刺 |
| `cpu.idle`（0/1/2） | 1=SCHED_IDLE 组；2=idle 组不抢占 idle 任务 | **把离线/批处理打成"最后优先级"**，在线任务随时可抢——openEuler 混部推荐用法 |
| `cpu.stat` / `cpu.pressure` | 统计 / PSI 压力 | 观测 throttle 次数、some/full 延迟——**低噪观测金标准** |
| `cpuset.cpus` / `cpuset.cpus.partition` | 绑核 / CPU 分区 | 物理隔离，见 §7.4 |
| `cpu.qos_level`（**openEuler QOS_SCHED**，core.c:10350） | 组 QoS 分级 | 离线组被在线任务无条件抢占 + 节流 + SMT 驱逐 |

v1 路径（`cpu.cfs_quota_us` 等）等价，本文以 v2 为准。

### 5.4 每任务接口

- `nice`/`setpriority`：改 weight（CFS 与 EEVDF 语义一致，但 EEVDF 下高 nice 的**短**任务受损小于 CFS——lag×w 结算）；
- `sched_setattr()`：SCHED_DEADLINE（runtime/period/deadline 三元组）；openEuler 回移的 `custom_slice`（fair.c:1375）支持 per-task 请求片长（配合 prctl 扩展）；
- `sched_setaffinity()`：绑核。

---

## 6. 问题三：混布场景（短高频排队 + 长任务占核）的代码链路

设：CPU 核上跑着一个 CPU 密集长任务 L；一群短任务 S1..Sn 高频唤醒（网络包、tick 业务）。

### 6.1 长任务占核如何体现（L 为什么"占得住"）

EEVDF/CFS 下 fair 任务**没有时间片到点强切**这回事——L 只要还有资格（eligible）且 deadline 未被别人超过，就连续占用：

```
tick 中断
 └ sched_tick()                       core.c
    └ task_tick_fair(rq, L, 0)        fair.c:15266
       └ entity_tick(cfs_rq, se=L)    fair.c:5952
          └ update_curr(cfs_rq)       fair.c:1569
             ├ L->vruntime += Δ×NICE_0/w   (fair.c:1583)
             ├ update_deadline()      fair.c:1365
             │   └ vruntime < deadline → 返回 false（slice 没满，继续跑）
             └ resched_next_slice()   fair.c:1522
                 └ entity_eligible(L)？→ L 还 eligible → 不设 NEED_RESCHED
→ L 继续跑，直到 slice 耗尽（最多 0.7ms 级 base_slice）或 L 不再 eligible
```

即：**占核不是 bug 而是合同**——L 的 slice（0.7ms）内、且队列里没人 deadline 更早，它就该跑。真正的问题是核上有**多个 L 类任务叠加**时，S 类任务的 deadline 总排不到第一，调度延迟（schedule latency）累积。此时 L 的"占"通过 `nr_running↑ → V 推进快 → S 必须等 vruntime 相对滞后积累` 体现为排队。

若 L 进了内核态长临界区且内核是 PREEMPT_VOLUNTARY，用户态抢占点变少，占核时间还会拉长（这是"调 PREEMPT_DYNAMIC=full"的动机，docs/04 详述）。

### 6.2 短高频任务排队如何体现（S 的路径）

```
网卡/timer 唤醒 S_i
 └ try_to_wake_up(S_i)                  core.c:4093
    ├ select_task_rq_fair()             fair.c:9649   ← 选核：先找 idle 核
    │   └ select_idle_sibling()         （cache 亲和 vs 空闲 的折中；
    │       若所有核都被 L 占 → 只能挑 prev_cpu/最闲核，S_i 入队排队）
    └ ttwu_queue() → enqueue_task_fair()
       └ enqueue_entity()               fair.c:5723
          └ place_entity()              fair.c:5601   ← lag 结算：S_i 拿回亏欠
             └ S_i->deadline = V - lag + vslice/2    （半片入场）
    └ wakeup_preempt(rq, S_i)           core.c:2104
       └ check_preempt_wakeup_fair()    fair.c:9848
          └ pick_eevdf(cfs_rq, true)==S_i?           fair.c:9943
             ├ 是（S_i deadline 最早且 eligible）→ resched_curr：L 在下一个抢占点让位
             └ 否（前面还有更急的 S_j / L 受 slice 保护 RUN_TO_PARITY）→ S_i 继续排队
```

**排队的定量化**：队列上每多一个 runnable 任务，`avg_vruntime`（V）被推快，后来者的"最急"地位被稀释；`rq->cfs.nr_running` 增加 → `pick_eevdf` 遍历分支变多；更本质的是**调度延迟 = 从 enqueue 到首次 on_cpu 的间隔**拉长。观测：

```bash
perf sched record -a -- sleep 5 && perf sched latency   # 每任务的 wait/max delay
cat /proc/schedstat                                      # rq 级 runqueue 时间（需 sched_schedstats=1）
cat /sys/fs/cgroup/.../cpu.pressure                      # PSI some/full
tracepoint: sched:sched_wakeup ↔ sched:sched_switch      # 精确队列停留
```

### 6.3 多核侧的泄洪：负载均衡链路

单核排队后内核的自动泄洪（SMP）：

```
tick / idle 入口
 ├ sched_balance_rq()/load_balance      （周期均衡：fair.c:11313 注释块，
 │   └ detach_tasks()  fair.c:11318     从 busiest 拉任务 → attach_tasks()）
 └ newidle_balance()                    （CPU 一空闲就偷任务——短任务场景主力）
```

短任务风暴下 `newidle_balance` 的效果依赖 `sched_migration_cost_ns`（500000）：任务若"刚跑过"被判 cache 热，倾向不迁——这既保护了 cache 局部性，也可能让泄洪不够快（§5.1 的调优点）。openEuler 另有 `sched_prio_load_balance_enabled`（fair.c:297）让均衡感知优先级（高优任务优先被拉走/留下，混部向）。

### 6.4 EEVDF 在此场景的天然行为（相对 CFS 的差异）

- CFS：S 醒来拿 min_vruntime 补偿 → 与 L 的 vruntime 差决定能否抢占，但 L 若 vruntime 落后少，S 仍要排队整轮；恶意多睡者还能反复插队。
- EEVDF：S 每次睡眠保留 vlag，**排队越久亏欠越多 → 醒来 deadline 越提前 → 必然优先于 L 获得服务**；且 L 的 slice 只有 0.7ms、且必须 eligible 才能续跑——L 无法凭"在跑"欺负 S。理论结论：同样负载下 EEVDF 的短任务尾延迟优于 CFS，且无需调 wakeup_granularity 类启发参数。

### 6.5 阅读顺序建议（按此顺序精读一遍 ≈ 掌握 fair 类 80%）

1. `update_curr()` fair.c:1569 → `update_deadline()` 1365（记账与合同）
2. `avg_vruntime()` 926 → `entity_eligible()` 1058（V 与资格）
3. `pick_eevdf()` 1253（选人）
4. `place_entity()` 5601（睡眠/唤醒的 lag 结算，注释含完整推导）
5. `check_preempt_wakeup_fair()` 9848（唤醒抢占）
6. `try_to_wake_up()` core.c:4093 + `select_task_rq_fair()` 9649（选核）
7. `task_tick_fair()` 15266（tick 视角收尾）
8. 对照社区 6.6.159 同名函数 diff（差异清单见 §2.8）

---

## 7. 问题四：该场景下的参数优化（按层给出，均注明代码依据）

以下按"见效快→结构化"排序。目标：S 类（在线短任务）低延迟、低抖动；L 类（批处理）不饿死但让路。

### 7.1 cgroup 分级（首选，见效最大）

```bash
mkdir online batch
# 在线组：提权（weight 1~10000，默认100；等价份额 8:1）
echo 8000 > online/cpu.weight
# 批处理组：压限额 + 降为 idle 类
echo 50000 100000 > batch/cpu.max        # 最多 50% 单核时间（可再压）
echo 1    > batch/cpu.idle               # SCHED_IDLE 组：任何在线任务可随时抢占
```

- 依据：weight 进入 lag 公式的 w_i（§2.1①）——提权即加快结算；`cpu.idle=1` 对应 `se_is_idle()`，`check_preempt_wakeup_fair()` fair.c:9905-9915 给"idle 让位非 idle"的无条件通道，`pick_eevdf` 对 idle 实体也有抑制（se_is_idle 分支）。
- 低噪收益：L 类从"公平竞争者"降为"填洞者"，在线尾延迟直接与 L 解耦。
- 风险：batch 组饥饿需业务侧自监控（cpu.stat 的 nr_throttled）；勿对延迟敏感反向用 idle。
- openEuler 专属替代：`echo -1 > batch/cpu.qos_level` 启用 QOS_SCHED（比 cpu.idle 更强：节流 + SMT 驱逐，专为混部设计，fair.c:9952 起）。

### 7.2 全局 sysctl 微调

```bash
# 1) 观测先行（定位是"排队"还是"占核"）
echo 1 > /proc/sys/kernel/sched_schedstats      # 开 /proc/schedstat（观测完关掉）
# 2) 拆会话分组干扰（服务器几乎必做）
echo 0 > /proc/sys/kernel/sched_autogroup_enabled
# 3) 让唤醒任务更快迁移到空闲核（代价：cache 局部性）
echo 100000 > /proc/sys/kernel/sched_migration_cost_ns    # 500000→100000
# 4) RT 防护保持默认（950000/1000000）；确认无业务把它调没
```

低噪注意：schedstats 常开有 ~1% 开销；migration_cost 调太小会让短任务在核间弹跳（cache miss 噪声反升），建议 10 万~25 万之间二档实验。

### 7.3 debugfs 特性开关（存在才调）

```bash
# openEuler 已回移 PREEMPT_SHORT（默认关）——短任务唤醒更激进抢占
echo PREEMPT_SHORT > /sys/kernel/debug/sched_features
```

依据：do_preempt_short() fair.c:1530——短 slice 唤醒者取消 curr 的 slice 保护。适合"S 极短（<0.1ms 级）且 L 常在核上"的形态；副作用是吞吐略降（切换变多）。无该文件说明内核未开 CONFIG_SCHED_DEBUG，跳过。

### 7.4 拓扑级隔离（结构化手段，改启动参数/cpuset）

```bash
# 内核 cmdline：隔离核 + 抢占模型（docs/04 详述 preempt= 模型）
isolcpus=managed_irq,domain,2-3 nohz_full=2-3 rcu_nocbs=2-3 preempt=full
# 或 cgroup v2 动态分区
echo "2-3" > /sys/fs/cgroup/online/cpuset.cpus
echo root  > /sys/fs/cgroup/online/cpuset.cpus.partition
```

把 S 类独占 2-3 核：tick/RCU/内核线程噪声全部移走（nohz_full 是"低噪"的终极形态——减少 tick 本身）。代价：L 只能用剩余核，需与容量规划联动。

### 7.5 每任务层（少量关键线程）

- 关键 S 线程 `sched_setaffinity` 收敛到隔离核；
- openEuler 的 per-task custom_slice（fair.c:1375）：给极短关键任务更小请求片 → deadline 更靠前（需 prctl/sched_setattr 配套，见内核文档）；
- 绝不要把在线业务设成 RT（FIFO）来"抢"：RT 池有 950ms/1s 上限，超限被 throttle 反而制造毛刺；确有硬实时需求用 SCHED_DEADLINE 显式申请带宽。

### 7.6 参数速查（本场景）

| 旋钮 | 建议值 | 针对 | 代码依据 |
|---|---|---|---|
| cpu.weight | online 8000 / batch 100 | 份额结算 | lag 公式 w_i |
| cpu.max | batch 50% | 长任务限额 | fair.c:5988 带宽机制 |
| cpu.idle / cpu.qos_level | batch=1 / -1 | 无条件让位 | fair.c:9905 / 9952 |
| sched_migration_cost_ns | 100000~250000 | 泄洪速度 | select_idle_sibling |
| sched_autogroup_enabled | 0 | 分级不被拆散 | autogroup.c |
| PREEMPT_SHORT | 开（实验） | 短任务抢占 | fair.c:1530 |
| sched_rt_runtime_us | 保持 950000 | 防失控 | rt.c throttling |
| nohz_full+isolcpus | 关键核 | 低噪终极 | tick 移除 |

---

## 8. 答疑区

（待后续追问补充）

---

## 附：本文实测命令备忘

```bash
# EEVDF 核心函数行号（openEuler-6.6）
grep -n "static struct sched_entity \*pick_eevdf\|^place_entity\|static bool update_deadline\|^update_curr(struct cfs_rq" openEuler-6.6/kernel/sched/fair.c
# 双侧 diff 归因示例
diff <(sed -n '1569,1598p' openEuler-6.6/kernel/sched/fair.c) \
     <(sed -n '1201,1221p' linux-6.6.159/kernel/sched/fair.c)
# 有效 sched sysctl（在目标机上）
ls /proc/sys/kernel/ | grep sched
# sched_features
cat /sys/kernel/debug/sched_features 2>/dev/null
```
