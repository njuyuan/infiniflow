# InfiniFlow 论文与代码对照解读，以及可行性论证

本文对照 SIGCOMM 2026 论文 *InfiniFlow: Decoupling Virtual Channel Scalability from Buffer Requirements in Lossless Datacenter Networks* 与本仓库 `ns-3` 实现，逐模块解读设计，并论证：在浅缓冲交换机上把 Virtual Channel（VC）规模与报文缓冲需求解耦，是算法上自洽、仿真上可复现、硬件上可落地的。

**结论先行：** InfiniFlow 的可行性成立。核心不在于“再多划几条 VC”，而在于把缓冲分配从下游静态预留改成上游按需发放。论文给出了缓冲下界，代码把该下界落成可运行的 hop-by-hop 协议，FPGA 原型证明 100 Gbps 线速下 16k+ VC 只需约 401 KB 片上内存。

---

## 1. 问题：无损网络里 VC 为什么扩不上去

RDMA 依赖 hop-by-hop 流控保证无损。现有机制都以 VC 为粒度：

| 机制 | 典型 VC 数 | 单 VC 保线速所需缓冲 | 扩 VC 的代价 |
| --- | --- | --- | --- |
| PFC | 约 8 | 至少 `2 * per-hop BDP`（Xon/Xoff 各吃 1 个 BDP） | 缓冲随 VC 线性增长 |
| CBFC | 约 16 | 至少 `1 * per-hop BDP`（`R_max <= B / tau`） | 缓冲随 VC 线性增长 |

冲突来自 VC 复用：多条流挤在同一 VC 上，拥塞会沿 VC 传播，形成 Head-of-Line Blocking（HoLB）。论文 Figure 2 的微基准把这件事说清楚了：

- **同 VC 碰撞：** victim 流被 congested 流拖死，无法达到公平份额（理想 75 Gbps）。
- **端到端拥塞控制补救：** DCQCN / HPCC 能抬高 victim 吞吐，但受端到端 RTT 限制，收敛慢、不准。
- **每流独立 VC：** 仅 hop-by-hop 流控就能把 victim 钉在 75 Gbps，不需要慢速探测。

Appendix A 进一步证明：每流独立 VC + 等权 DRR + 每 VC 缓冲足够保线速时，稳态速率向量是全局 max-min fair。这是 InfiniFlow 追求的理想工作点。

障碍也明确：100 Gbps、1 us 链路、32 口交换机、每口 1000 条 VC 时，CBFC 需要约 800 MB 报文缓冲，远超商用交换机几十 MB 的 SRAM。Figure 3 给出 U 型曲线：VC 太少则隔离不够，VC 太多则每 VC 缓冲不够，CBFC/PFC 都无法同时满足“细粒度隔离”和“无损线速”。

---

## 2. 关键洞察：缓冲需求Bound 在端口，不 Bound 在 VC

VCs 共享同一条物理链路，任意时刻 `sum_i r_i(t) <= C`。CBFC 里维持速率 `r_i` 至少需要 `r_i * tau` 缓冲，于是端口聚合缓冲：

```
B_agg(t) = sum_i r_i(t) * tau  <=  C * tau
```

也就是理论上每端口只需约 `1 * per-hop BDP`。线性增长来自**静态 per-VC 预留**，不是物理必然。论文实测最优工作点约 `3 * per-hop BDP`：多出来的 2 个 BDP 用来吸收传播时延和 BUCP 锁定期的短暂 credit 冻结。

这条不等式是整篇设计的支点。后面所有机制都在回答同一件事：如何在共享缓冲上把这条不等式变成可执行的、无溢出的、按需的调度约束。

---

## 3. 设计拆成三层，代码也按这三层组织

论文把 InfiniFlow 拆成三个可独立关闭的机制，仓库用三份 ns-3 实例一一对应：

| 机制 | 作用 | 仓库目录 | 消融实验角色 |
| --- | --- | --- | --- |
| UABD | 上游维护 per-port 共享 credit pool，立刻把下游缓冲分给活跃 VC | `ns-3-infiniflow-nothreshold` | 只有共享 credit，没有 per-VC 限额 |
| BUCP | 用下游 backlog 反馈动态收紧/放宽 per-VC threshold | `ns-3-infiniflow-notap` | 有限额，但每次 BCP 都调 threshold |
| BUCP locking | 一次调整生效前禁止下一次调整，避免振荡 | `ns-3-infiniflow-mmf` | 完整 InfiniFlow，RR 调度逼近 MMF |

基线对照：

- `ns-3-infiniband`：传统 CBFC
- `ns-3-roce`：PFC，可叠 DCQCN / HPCC
- `ns-3-xpass`：ExpressPass
- `ns-3-bfc`：BFC（NSDI 2022）
- `ns-3-dt`：Dynamic Threshold + credit，对应 Appendix C 对 DT 的反例

评估脚本按论文图号编号：`ns-3/simulator/sh-script/xx-*.sh` 跑实验，`py-script/xx-*.py` 出图。这不是“另写一套评测”，而是论文图表的可复现管线。

---

## 4. UABD：上游替下游分配缓冲

### 4.1 论文在做什么

传统流控由下游本地分配缓冲，再把每 VC 可用空间同步给上游。共享缓冲后，下游还必须知道上游哪些 VC 活跃，同步至少消耗 `1 * per-hop RTT`，激活 VC 会 stall。

UABD 的判断是：上游 egress 更早看见 VC 状态，并且本来就要做调度。把“分配下游缓冲”并进上游调度，就能立刻发 credit，不必等远程 grant。

每条链路在上游维护：

```
credit_pool = PT_FCCL - PT_TX
```

- `PT_TX`：上游累计发送字节
- `PT_RX`：下游累计接收字节
- `PT_FCCL = PT_RX + remaining_shared_buffer`
- 约束 1（端口级）：`Head_Pkt_Size <= credit_pool`

这是把 CBFC 的计数语义从 per-VC 抬到 per-port。无损性仍由 credit 覆盖保证：没 credit 就不发。

### 4.2 代码如何落地

**下游构造 BCP（论文里的 Buffer Control Packet，代码里叫 FCP）：**

```141:163:ns-3/simulator/ns-3-infiniflow-mmf/src/point-to-point/model/switch-mmu.cc
Ptr<Packet> SwitchMmu::ConstructFcp(uint32_t inDev, uint32_t qIndex){
	FcpHeader fcph = FcpHeader();
	fcph.SetVl(qIndex);

	uint32_t fccl_ =  abr[inDev] + (buffer_size[inDev] - used_bytes[inDev]);
	fcph.SetFccl(fccl_);
	fcph.SetQlen(ingress_bytes[inDev][qIndex]);
	fcph.SetFccr(accumu_drained_bytes[inDev][qIndex]);
	// ... TAP 回显 ...
	return p;
}
```

字段对照：

| 论文 BCP 字段 | 语义 | 代码字段 |
| --- | --- | --- |
| `VC_ID` | 这条反馈属于哪条 VC | `FcpHeader::m_vl` |
| `PT_FCCL` | 端口级 credit 上限 | `m_fccl = abr + remaining` |
| `VC_BKLG` | 该 VC 当前 backlog | `m_qlen = ingress_bytes` |
| `VC_DR` | 该 VC 累计 drained 字节 | `m_fccr = accumu_drained_bytes` |
| `TA_FLAG` | 锁定握手 | `m_flags`，以及数据报文 IP TOS 低 2 bit |

`abr` 只在入队时增加、从不回退，对应 `PT_RX`。`used_bytes` 是当前占用。因此：

- 报文到达：`abr += size`，`used_bytes += size`，`PT_FCCL` 不变。在途 credit 不会因为入队而被“重复发放”。
- 报文离开：`used_bytes -= size`，`PT_FCCL` 增加 `size`，credit 回到上游 pool。

**上游消费 credit：**

交换机 egress：

```419:442:ns-3/simulator/ns-3-infiniflow-mmf/src/point-to-point/model/qbb-net-device.cc
		if(sw->m_mmu->fccl[m_ifIndex] >= sw->m_mmu->fctbs[m_ifIndex]){
			credit = sw->m_mmu->fccl[m_ifIndex] - sw->m_mmu->fctbs[m_ifIndex];
		}
		// ...
		p = m_queue->DequeueRR(credit, inflight_counter, threshold);
```

主机 NIC 侧同一公式：`credit = perpt_fccl - perpt_fctbs`。`fctbs` 就是 `PT_TX`。

调度器先看 BCP 队列（index 0），**不消耗数据 credit**，保证反馈严格优先，对应论文 4.5 节 “BCP Transmission Priority”。

**无损兜底：** `CheckIngressAdmission()` 在 credit 逻辑之外再比较 `buffer_size - used_bytes < psize`。正常运行不应触发；它是仿真里的安全网，用来暴露 credit 计算错误，而不是第二条流控。

UABD 单独可运行：`ns-3-infiniflow-nothreshold` 的 `DoDequeueRR` 只检查 `credit >= pkt_len`，忽略 per-VC threshold。这正是论文 Figure 13 的 “w/o BUCP”：共享缓冲能保无损，但被卡住的 VC 会独占 buffer，victim 吞吐掉到 25 Gbps。

---

## 5. BUCP：按需求而不是按均分发 credit

### 5.1 为什么 DT 不够

共享缓冲后，被下游限速的 VC 会把 credit 堆在下游队列里，饿死无辜 VC。DT 能均分，但不能按需求分。Appendix C 的哑铃实验里，三条 10 Gbps 限速流和一条应跑 70 Gbps 的 victim 被 DT 当成四个“活跃 VC”均分 credit，victim 被压到 50 Gbps 以下。InfiniFlow 需要的是 **VC-demand-aware**：限速流只拿维持其服务速率所需的 credit，剩余全部给能跑满的流。

### 5.2 两个计数窗口

论文把 per-VC 状态做成两个 TCP 风格的窗口：

```
inflight_bytes = VC_TX - VC_DR     // 已占用但尚未回收的 credit
VC_BKLG        = VC_RX - VC_DR     // 下游该 VC 当前排队
```

约束 2（VC 级）：

```
Head_Pkt_Size <= threshold - inflight_bytes
```

代码里：

| 论文变量 | 交换机 MMU | NIC |
| --- | --- | --- |
| `VC_TX` | `accumu_used_credit[port][q]`，egress 出队时累加 | `perpq_accumu_used_credit[q]` |
| `VC_DR` | BCP 的 `fccr`，写入 `accumu_recycled_credit` | `perpq_accumu_recycled_credit` |
| `inflight_bytes` | 二者之差 | 二者之差 |
| `VC_BKLG` | `ingress_bytes`（入队加、出队减，等价于 `VC_RX - VC_DR`） | 由 BCP `qlen` 带回 |
| `threshold` | `inflight_threshold[port][q]` | `perpg_inflight_threshold[q]` |

主机调度：

```111:112:ns-3/simulator/ns-3-infiniflow-mmf/src/point-to-point/model/qbb-net-device.cc
			if( credit < pkt_size || (thresholds[qp->m_pg] <= pkt_size + inflight[qp->m_pg]) )
				continue;
```

交换机 RR：

```128:128:ns-3/simulator/ns-3-infiniflow-mmf/src/network/utils/broadcom-egress-queue.cc
					if (credit >= pkt_len && thresholds[cur_q_index] > (pkt_len + inflight[cur_q_index]))
```

两条约束同时成立才允许发送。这就是论文 Algorithm 1 的 `cond2` 和 `cond3`。

### 5.3 Set-point 调整，不是 AIMD 探测

threshold 初值等于整端口共享缓冲（`maxThreshold = per_port_buffer`）。之后只看 backlog：

| 条件 | 论文动作 | 代码 |
| --- | --- | --- |
| `VC_BKLG >= qmax` | 拥塞抑制：`threshold = inflight - BKLG + qmin` | `if (qlen > qlen_max) new = inflight + qlen_min - qlen` |
| `VC_BKLG < qmin` | 吞吐抬升：`threshold += qmin - BKLG`，上限为整段共享缓冲 | `else if (qlen <= qlen_min && threshold != maxThreshold)` |

物理含义：

- **抑制：** `BKLG - qmin` 是多占的 credit。从当前 inflight 里扣掉这部分，调度器会插入 bubble，直到 backlog 回到 `qmin`。
- **抬升：** backlog 低于目标，说明这条 VC 还没吃饱。把差额加进 threshold，让它把空闲带宽用掉。无下游拥塞时 threshold 会被推到最大值。

这不依赖链路速率或时延先验知识，只依赖实时 backlog。Figure 14 显示三条 10 Gbps 流的 threshold 收敛到约 `0.1 * BDP`，victim 的 threshold 停在最大值并吃掉剩余 70 Gbps；聚合分配略高于 `1 * BDP`，与第 2 节的不等式一致。

`notap` 变体保留这段调整逻辑，但把锁打开（见下一节）。`nothreshold` 变体在调度路径上直接丢掉 threshold 判断，只留 UABD。

---

## 6. Locking：一次调整必须看见效果

Set-point 公式是瞬时的。若每个 BCP 都改 threshold，管道里尚未排空的旧 backlog 会触发下一轮误调，速率振荡。论文用两台有限状态机：

```
上游:  Normal --调整--> ThresholdAdjusting --发出 TA 数据报文--> FeedbackWaiting --收到 TA-BCP--> Normal
下游:  Normal --收到 TA 数据报文--> FeedbackMarking --回显 TA-BCP--> Normal
```

数据报文的 TA 复用 IP ECN 低 2 bit（`Tos = 0x03`）：

```108:118:ns-3/simulator/ns-3-infiniflow-mmf/src/internet/model/ipv4-header.cc
void Ipv4Header::SetTap() // share bits with ecn
{
    m_tos &= 0xFC;
    m_tos |= 0x03;
}
```

下游入队时若看到 TAP，置 `need_ret_tap`；下一次为该 VC 构造 BCP 时回显。上游只有收到 TAP-BCP 才把 `rate_adjusting` 清掉，期间禁止再调 threshold：

```274:293:ns-3/simulator/ns-3-infiniflow-mmf/src/point-to-point/model/switch-node.cc
    if(tap){
        m_mmu->rate_adjusting[port][qIndex] = false;
    }

    if(!m_mmu->rate_adjusting[port][qIndex]){
        // ... 按 qlen 相对 qmax/qmin 调整 inflight_threshold ...
            m_mmu->rate_adjusting[port][qIndex] = true;
            m_mmu->need_set_tap[port][qIndex] = true;
    }
```

`notap` 的唯一关键差异是把 `if (!rate_adjusting)` 改成 `if (true)`：每次 BCP 都调。Figure 13 对应三种结果：

1. 无 BUCP：victim 被独占缓冲，卡在 25 Gbps。
2. BUCP 无锁：victim 能上去，但振荡剧烈。
3. 完整 InfiniFlow：振荡被压住，接近 max-min fair，并明显好于同场景下的 DCQCN/HPCC。

三份 ns-3 树不是复制粘贴后的装饰，而是论文消融的可执行定义。

---

## 7. 报文路径：一跳之内论文 Algorithm 1–4 全部闭合

以交换机 A（上游）到交换机 B（下游）为例。

```
A egress 调度
  |  同时满足 credit_pool 与 per-VC remaining quota
  |  fctbs += size, accumu_used_credit[vc] += size
  v
链路
  v
B ingress
  |  abr += size, used_bytes += size, ingress_bytes[vc] += size
  |  若数据报文带 TAP -> need_ret_tap[vc] = true
  |  转发到 B 的目的 egress
  v
B 该报文从共享缓冲出队（SwitchNotifyDequeue）
  |  RemoveFromIngress: used_bytes -= size, accumu_drained_bytes[vc] += size
  |  ConstructFcp: PT_FCCL / qlen / fccr / TAP
  |  从 B 的 ingress 口把 FCP 发回 A   （协议号 0x0086）
  v
A SwitchReceiveFcp
  |  更新 fccl、recycled_credit
  |  可能调整 threshold，并锁定
  |  TriggerTransmit，立刻尝试用新 credit 发送
```

对应关系：

- Algorithm 1 Upstream Packet Transmission = `QbbNetDevice::DequeueAndTransmit` + `BEgressQueue::DoDequeueRR` / `RdmaEgressQueue::GetNextQindex`
- Algorithm 2 Downstream Packet Processing = `SwitchNode::SendToDev` 入队与 TAP 检测
- Algorithm 3 Downstream BCP Generation = `SwitchMmu::ConstructFcp` + `SwitchNotifyDequeue`
- Algorithm 4 Upstream BCP Handling = `SwitchNode::SwitchReceiveFcp` 与 NIC 侧 `QbbNetDevice::Receive` 的 `0x0086` 分支

主机和交换机走同一套计数，只是状态一个放在 `QbbNetDevice` 成员里，一个放在 `SwitchMmu` 的 `[port][q]` 数组里。hop-by-hop 的 fate-sharing 单位就是一条链路的两端。

VC 映射：流的 `pg` 字段就是 VC index。大规模实验把 `Q_CNT`/`F_CNT` 设到 1000，接近论文选用的 1024 VC 工作点；FPGA 原型把同一数据通路扩到 16k–64k。

---

## 8. 论文与代码的细微差异（不影响可行性判断）

对照 Appendix B 伪代码，实现有几处工程化取舍：

| 点 | 论文 | 代码 | 影响 |
| --- | --- | --- | --- |
| BCP 线格式 | 8 字节紧凑字段（16-bit FCCL/DR，15-bit BKLG） | `FcpHeader` 16 字节，计数器用 `uint32` | 仿真不必做 wrap；硬件原型才需要 RFC 1982 模运算 |
| `qmax` 比较 | `BKLG >= qmax` | `qlen > qlen_max` | 差 1 个字节量级，参数以 MTU 计，行为等价 |
| `qmin` 比较 | `BKLG < qmin` | `qlen <= qlen_min` | 边界含等，略更积极地抬升 |
| 最小 threshold | 至少 1 MTU | 钳位到 0 | 极端抑制时可能把 VC 完全暂停一个 RTT，论文更保守 |
| 调度器 | 证明用等权 DRR | 实现是 packet RR（`DoDequeueRR`） | 不等长包时公平性近似 MMF，大规模实验仍接近理想基线 |
| 计数溢出 | 明确采用 RFC 1982 | `uint32` 减法，负数时打印 error 并清零 | 仿真时长达不到 4 GB wrap；硬件部署必须补模运算 |
| 每口缓冲 | 论文分析按端口静态划分 | `SetBufferPool` 把交换机总缓冲均分到 `P_CNT` 个端口 | 与论文 7.4 节“端口间静态隔离”一致，尚未做跨端口共享 |
| 故障处理 | 7.3 节描述 keepalive、冲队列、作废 credit | 仿真未实现链路故障状态机 | 不影响协议正确性论证，但是产品化缺口 |

这些差异属于实现精度，不推翻协议语义。消融、微基准、大规模 FCT 用的都是这份代码，所以论文声称的行为可以在本仓库复现。

---

## 9. 可行性：五条相互独立的证据

### 9.1 算法可行性：无损 + 立即分配 + 按需隔离

三条性质分别由可检查的不变量保证。

**无损。** 发送前 `Head <= credit_pool`，而 `credit_pool` 精确等于下游剩余共享缓冲减去已授权但未到达的在途字节。入队不增加 FCCL，出队才返还。只要 BCP 最终到达（丢失时计数器仍单调，CBFC 同类机制可恢复），缓冲不会被超额授权。`CheckIngressAdmission` 在仿真中充当断言。

**立即分配。** 活跃 VC 出现在上游调度器里的瞬间就可以从 pool 里取 credit，不必等下游重新划分。这是 UABD 相对“下游 DT + 同步”的决定性优势，也是 `nothreshold` 仍然能跑满线速的原因。

**按需隔离。** threshold 被 backlog 钉在“刚好维持该 VC 服务速率”的位置。限速 VC 的稳态占用约几个 MTU（`qmin` 到 `qmax`），而不是整个 BDP。无辜 VC 看到的是还留在 pool 里的 credit，而不是被冻在别人队列里的缓冲。Locking 保证这个钉扎过程不会自己打振荡。

Appendix A 的 max-min 证明依赖“每流独立 VC + 缓冲足够保线速 + 等权 work-conserving 调度”。InfiniFlow 用共享 pool 满足第二条，用大规模 VC 逼近第一条，用 RR 近似第三条。因此“接近理想 FCT”不是经验巧合，而是这条证明在缓冲约束被解除后的预期结果。

### 9.2 资源可行性：报文缓冲 O(1)，元数据 O(N_vc)

论文给出：

```
M_InfiniFlow  ≈  alpha * H_BDP          +  20 B * N_vc
                 报文缓冲，与 VC 无关        每 VC 元数据
```

典型 `alpha = 3`。100 Gbps、1.5 us per-hop RTT 时，`H_BDP ≈ 18.75 KB`，3 倍约 56–64 KB。16384 条 VC 的 metadata 约 320 KB，合计论文报告的 **401 KB/端口**。

对照 CBFC：`M_CBFC ≈ N_vc * H_BDP`。同一 40 MB SRAM、64 口交换机（每口 625 KB）只能放下约 32 条 CBFC VC 或 16 条 PFC VC。InfiniFlow 在同一预算下到 16k VC，即论文所说的 512x / 1024x。

本仓库 `py-script/11-evaluation-buffer-requirement.py` 把 InfiniFlow 画成一条不随 VC 数上升的 64 KB 水平线，与 FPGA Figure 11d 一致。`12-evaluation-memory-consumption.py` 使用 `result/fpga-data/` 中的实测利用率。

元数据内容与代码状态一一对应，规模是可信的：

| 每 VC 状态 | 作用 | 大致宽度 |
| --- | --- | --- |
| `VC_TX` / `VC_DR` | inflight | 4+4 B |
| `threshold` | BUCP 限额 | 4 B |
| `VC_RX` 或当前 qlen | backlog | 4 B |
| 链表头尾（硬件队列） | 非连续缓冲拼接 | 若干字节 |
| TAP / FSM 标志 | 锁定 | 1–2 B |

合计约 20 B/VC，与论文一致。控制逻辑是一条流水线时分复用，不按 VC 复制 ALU，所以 Figure 11c 的 LUT/FF 几乎不随 VC 数涨。

### 9.3 硬件可行性：U280 上 100 Gbps 线速

原型不是纸面面积估算：

- 平台：Xilinx Alveo U280，QSFP28 100 Gbps，单跳上下游两端。
- 调度器改编自开源 100G NIC Corundum，每周期一个调度决策。
- 数据通路 512-bit。即使 65536 VC 时 `F_max` 降到约 290 MHz，理论吞吐仍约 `290 MHz * 512 bit ≈ 148 Gbps`，高于 100 Gbps 线速。
- Figure 11b：并发流数拉到 65536，吞吐仍贴着 100 Gbps。
- 4096 VC 以内 `F_max > 400 MHz`；规模再大，瓶颈是布线与大 SRAM 译码，不是 InfiniFlow 算术。

本仓库 `result/fpga-data/FPGA_flow_control_data.csv` 记录 Credit / Threshold / Inflight / RemainingBS 的时间序列。论文 Appendix E 用同一实验设置对比 FPGA 与 ns-3，关键变量波动高度一致。这意味着：

1. ns-3 没有“仿真里偷偷做对、硬件做不到”的隐藏假设。
2. 硬件没有“只跑通线速、控制环是空壳”的简化。

`sh-script/21-validating-prototype.sh` 对应这条交叉验证。

### 9.4 系统可行性：1024 口规模、真实负载、对比完整

大规模设置与代码默认值对齐：

| 论文 6.4 节 | 代码 |
| --- | --- |
| 8 spine + 32 leaf，每叶 32 主机，共 1024 主机，4:1 过订阅 | `16-large-scale` 拓扑 |
| 64 口交换机，40 MB 共享缓冲 | `buffer_size = 40000000`，再按 `P_CNT=64` 均分 |
| 100 Gbps，1 us | 拓扑链路参数 |
| InfiniFlow 1024 VC，`qmin=5 MTU`，`qmax=10 MTU` | `Q_CNT≈1000`，`config-scale-common.txt` 中 `QMIN 5` / `QMAX 10` |
| WebSearch / Hadoop，40% 与 80% 负载 | `W3`/`W4` × `load40`/`load80` |
| 对照 PFC(8)、CBFC(16)、DCQCN+PFC、HPCC+PFC、ExpressPass、BFC(32) | `ns-3-roce` / `infiniband` / `xpass` / `bfc` 各跑同一脚本族 |

结果与机制对应：

- **短流尾部：** 大规模 VC 降低碰撞，InfiniFlow 的 P99 最好，尤其 Hadoop 80%。PFC/CBFC/BFC 在 VC 不够时尾时延爆炸。
- **长流平均：** hop-by-hop + 足够隔离让长流贴近公平份额；CC 为了控队列会牺牲平均吞吐。
- **缓冲占用：** InfiniFlow 的 CDF 明显左移。CC 队列可以更空，但 FCT 并不更好——说明“把队列打空”不是目标，目标是按需求占用。
- **参数鲁棒：** Table 1 与 Appendix D 扫描显示，平均 FCT slowdown 最多劣化 1.83%。`20-parameter-sweeping` 就是这张热力图的生成脚本。

敏感度实验还给出两个可部署的工作点：

- VC 数到 1024 后收益迅速饱和（Figure 15a）。
- 每口缓冲到 `3 * BDP`（约 75 KB @ 100G/1us）后接近理想（Figure 15b）。

这把“理论 1 BDP、实践 3 BDP”钉成可配置的工程参数，而不是事后拟合。

### 9.5 工程可行性：可复现的实现，而不是不可运行的伪代码

本仓库同时具备：

1. **可编译的协议实体：** MMU、QBB 设备、FCP 头、RR 调度、TAP/ECN 复用。
2. **可关闭的模块边界：** 三份树精确对应 UABD / BUCP / locking。
3. **与论文图号一一对应的实验入口：** `sh-script` + `py-script` + `result/`。
4. **FPGA 轨迹：** 不是只有面积数字，还有控制变量时间序列。

因此“论文是否可行”可以降级成三个可证伪的问题，而本仓库对三个问题都给出了肯定答案：

1. 协议能否在事件级仿真里保持无损且按需分配？能。
2. 同一控制环能否在 FPGA 上线速跑？能。
3. 在 1024 主机、真实流量下是否仍接近理想 FCT？能。

---

## 10. 仍然成立的边界，以及它们不否定可行性

论文第 7 节自己划了边界，代码也印证这些边界是真实的，而不是被藏起来的失败。

**InfiniFlow 替代的是 L4 拥塞控制的速率调节，不是可靠性。** 设备故障丢包仍要靠端主机重传。credit 对原始包、重传包、ACK 一视同仁。大规模实验里 `CC_MODE 0`、`HAS_WIN 1` 仍然保留 RDMA 可靠传输窗口，流控与可靠性正交。

**端口间缓冲是静态划分的。** 多个 per-link 控制环只通过共同的 egress 调度器间接耦合。突发拥塞消失时，过小的 `qmin/qmax` 可能短暂欠利用；突发拥塞出现时，过大的参数会让 credit 在慢队列里多停留一会儿。这解释了为什么实践要用 3 BDP 而不是 1 BDP，也解释了参数扫描的必要性。它不破坏“与 VC 数解耦”，只是给 `alpha` 一个大于 1 的常数。

**BDP 本身仍会随链路变长、速率变高而增长。** InfiniFlow 让缓冲不再乘以 VC 数，但还是乘以 `H_BDP`。未来更长的 DC 间光纤可能再次把 3 BDP 推过片上 SRAM。论文把它列为未来工作，而不是当前设计的内部矛盾。

**仿真 RR 不是严格 DRR。** 对 MTU 近似固定的 RDMA 负载，差异很小；若要在变长报文下给 Appendix A 的证明一个更硬的实现对应，把 `DoDequeueRR` 换成 deficit 计数即可，不需要改 credit 协议。

**故障状态机未在 ns-3 中实现。** 论文描述的是链路 keepalive、作废 credit、冲队列。这是产品化工作，不是“协议在正常数据面不可行”的证据。

---

## 11. 总评

InfiniFlow 可行，因为它把一个被当成硬件公理的线性关系拆开了：

```
旧假设:  无损线速缓冲  =  Theta(N_vc * BDP)
新事实:  无损线速缓冲  =  Theta(BDP)  +  Theta(N_vc) 字节级元数据
```

拆开的方法不是更激进的端到端控制，而是把分配权放到本来就在做调度的上游，再用几个 MTU 量级的 backlog 反馈防止共享被滥用。代码把这套方法写成了与论文 Algorithm 1–4 同构的事件处理；FPGA 证明算术和状态机撑得住 100 Gbps 和 16k VC；大规模仿真证明在真实负载下它确实逼近“每流独立 VC + 足够缓冲”的理想点。

因此，本仓库不仅是论文的附属实现，而是可行性论证本身：同一套控制环同时通过了协议不变量、硬件时序、以及系统级 FCT 三道检查。剩余问题是部署层面的（模运算、故障处理、跨端口共享、与负载均衡协同），不是“这个想法能否工作”。
