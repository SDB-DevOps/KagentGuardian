# 15 - 外部轻量哨兵 (Sentinel) 自动化巡检机制与结果可视途径全景指南 (Sentinel Patrol Execution & Result Visibility)

> **导读与摘要**：
> 本文针对用户在架构图中关注的核心疑问进行专项技术论证与实证解答：
> 1. **“这块的功能怎么体现？他是自动巡检吗？”** —— 是的，系统具备 100% 全自动、无人值守的 7×24 巡检机制，由原生 Kubernetes CronJob 驱动。
> 2. **“每十五分钟巡检一次，结果我在哪里看？”** —— 结果分为三大查看途径：日常集群 Pod 实时日志、GitHub 官方仓库的日终健康账本（Markdown 归档），以及异常劣变触发自愈时自动创建的 GitHub Issue / PR 与 SRE Web UI 对话卡片。
> 3. **“现场物理实证”** —— 本文附带当前生产集群实时运行的 `kagent-guardian-sentinel` 任务状态与实测日志，确保事实可信、证据保真。

---

## 一、核心问题解答总览 (Executive Summary)

| 用户核心疑问 | 结论与技术定性 | 物理载体与实现位置 |
| :--- | :--- | :--- |
| **1. 他是自动巡检吗？** | **完全自动化、无人值守**。依托 Kubernetes 原生调度器，每 15 分钟准时拉起轻量级容器执行物理基线采集，全程无需人工干预。 | 命名空间 `kagent` 下的 `CronJob/kagent-guardian-sentinel`（配置见 `deploy/guardian/sentinel-cronjob.yaml`）。 |
| **2. 架构图中该功能如何体现？** | **阶梯短路防护 (Tiered Short-Circuit)**：采用“低成本哨兵探活 + 高智能 Agent 自愈”的两级架构。平时由轻量容器做 0 Token 物理检查，只有健康分 < 85 分时才抢占分布式排他锁唤醒重型大模型 Agent。 | 位于架构图 **Layer 1（触发与周期调度层）**，并向下连接 **Layer 2 的 Lease 分布式锁** 与 **Layer 3 的认知引擎**。 |
| **3. 巡检结果在哪里查看？** | **三维可视矩阵**：<br>① **日常实时日志**：Kubernetes 容器日志（秒级实时）；<br>② **日常健康账本**：GitHub 仓库 `docs/patrol-reports/`（每日终了 23:55 自动归档）；<br>③ **异常劣变工单**：GitHub Issues & 自愈 PR（故障时自动生成）；<br>④ **即时交互报告**：SRE Web 控制台（`:30083`）对话卡片。 | • 集群：`kubectl logs -n kagent job/<job-id>`<br>• GitHub：`docs/patrol-reports/YYYY-MM/patrol-YYYY-MM-DD.md`<br>• GitHub：Issues / Pull Requests<br>• Web UI：`http://<NodeIP>:30083/agents/chat` |

---

## 二、外部轻量哨兵 (Sentinel) 的底层工作原理

### 2.1 为什么必须设计“外部哨兵”而非让 Agent 一直轮询？

在企业级 AI Agent 架构中，大语言模型（LLM）调用具有以下两大固有特性：
1. **Token 成本高**：如果每 15 分钟让 LLM Agent 完整读取 PromQL、调用 4 个 MCP Remote 工具并推理一遍，一天 96 次巡检将消耗数百万 Token，产生极其昂贵的推理费用。
2. **Substrate 沙箱冷启动特性**：Kagent 平台的 Agent 运行在 Substrate 安全沙箱内，无外部请求时处于休眠挂起（Suspended）状态，沙箱本身没有常驻轮询触发器。

**设计方案**：
引入 Kubernetes 原生超轻量容器哨兵（`kagent-guardian-sentinel`），由 K8s `batch/v1 CronJob` 驱动（资源限制极低：50m CPU / 64Mi 内存），每次仅耗时 5~6 秒，执行确定性的 L1~L3 物理指标只读检查：

- **L1 物理节点健康检测**：`kubectl get nodes --no-headers | grep -v ' Ready'`。若存在未就绪节点，一次性扣 30 分。
- **L2 核心容器异常检测**：扫描 `kagent` 命名空间下是否存在 `CrashLoopBackOff`、`Error`、`OOMKilled` 容器。若存在，一次性扣 25 分。
- **L3 控制面与中间件检测**：检查 `kagent-postgresql` 状态。若未达 1/1 Running，一次性扣 35 分。
- **阈值分流**：
  - **>= 85 分（健康常态）**：以状态码 0 退出，**0 LLM Token 开销**，守护成本为零。
  - **< 85 分（劣变异常）**：以状态码 1 熔断退出，抢占 `coordination.k8s.io/leases/kagent-guardian-lock` 分布式锁，唤醒 Agent 执行 CFS 双因子诊断与 GitOps 自愈。

---

## 三、巡检结果全景查看指南 (Where to View Results)

根据平台的运行状态，SRE 和开发人员可以通过以下不同渠道查看对应结果：

### 途径 1：日常实时运行日志 (秒级查看)
Kubernetes 自动保留最近的巡检 Job 执行记录（`successfulJobsHistoryLimit: 3`）：

```bash
# 查看最近运行的哨兵 Job 列表
kubectl get jobs -n kagent -l app.kubernetes.io/name=kagent-guardian-sentinel

# 查看最近一次巡检 Pod 的实时日志输出
kubectl logs -n kagent -l app.kubernetes.io/name=kagent-guardian-sentinel --tail=50
```

实测输出：
```text
[2026-11-25T08:45:01Z] [SENTINEL] Starting L1-L3 Tiered Short-Circuit Baseline Patrol...
[SENTINEL] Health Baseline Score: 100/100
[SENTINEL] STATUS: GREEN (Healthy). Zero Token consumed. Direct ledger snapshot logged.
```

---

### 途径 2：GitHub 官方代码仓库日终账本 (永久可追溯)
每天 23:55，伴生 CronJob `kagent-guardian-ledger-archiver` 会使用 GitHub App 私钥签名换取短期安全 Token，将当天巡检的健康数据总结归档到 GitHub 仓库。

- **查看路径**：GitHub 仓库 `SDB-DevOps/KagentGuardian` 下的 `docs/patrol-reports/YYYY-MM/patrol-YYYY-MM-DD.md`
- **价值**：研发负责人、安全合规审计员无需登录内网 K8s 集群，打开 GitHub 即可透明查看平台每一天的运行体检历史。

---

### 途径 3：异常隐患触发自愈时的报警工单与自愈 PR
当巡检打分 < 85 分（或发生 Pod 崩溃、CFS 严重节流）时：
1. **GitHub Issue 告警工单**：Agent 自动创建工单，内附物理证据（抓取的 Pod 最近 100 行 Panic 日志、Prometheus CFS 节流时序曲线），具备 Hash 幂等防风暴特性。
2. **GitHub Pull Request 自愈代码**：Agent 基于真实物理证据计算出推荐修复方案，自动拉取分支并向 `main` 发起 Pull Request，关联 `Fixes #IssueID`。待 SRE 人工合并后，由 ArgoCD 自动调谐推向集群生效。

---

### 途径 4：SRE Web UI 控制台即时交互 (按需主动调取)
在需要即刻核查平台健康状态时，无需等待 15 分钟定时周期：
1. 打开 SRE 控制台：`http://<NodeIP>:30083/agents/chat`。
2. 在对话框中输入：`@kagent-guardian 执行平台巡检` 或 `检查当前集群健康状态`。
3. KagentGuardian 立即启动完整的 L1-L4 阶梯式巡检，并在 Web 界面上实时以 Markdown 渲染出结构化健康卡片。

---

## 四、生产集群实机核查证据 (Live Cluster Verification)

以下数据直接采集自当前运行中的 Kubernetes 生产集群：

### 1. 自动化 CronJob 注册状态
```text
NAME                              SCHEDULE       TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
kagent-guardian-ledger-archiver   55 23 * * *    <none>     False     0        8h              2d6h
kagent-guardian-sentinel          */15 * * * *   <none>     False     0        1m              2d7h
kagent-guardian-session-gc        0 3 * * *      <none>     False     0        5h45m           2d7h
```
**证实**：`kagent-guardian-sentinel` 准确按照 `*/15 * * * *` 调度执行，已连续稳定守护 2 天以上。

### 2. 最近 3 次 15 分钟巡检任务执行记录
```text
NAME                                STATUS     COMPLETIONS   DURATION   AGE
kagent-guardian-sentinel-29926575   Complete   1/1           6s         31m
kagent-guardian-sentinel-29926590   Complete   1/1           6s         16m
kagent-guardian-sentinel-29926605   Complete   1/1           6s         1m
```
**证实**：每 15 分钟触发一次，每次耗时仅 6 秒即可完成全部物理基线排查，执行状态均为 `Complete`。

---

## 五、总结与架构精髓

通过**外部轻量哨兵 (Sentinel)** 与 **KagentGuardian 认知引擎** 的分工协作，平台实现了：
1. **平稳期 0 Token 零成本开销**：在系统健康的绝大多数时间里，每 15 分钟的巡检只消耗微量容器 CPU（50m/6s），不耗费任何大模型 Token 费用。
2. **异常期秒级精准自愈**：一旦哨兵发现指标劣变（< 85 分），毫秒级熔断短路并升级激活 Agent，通过 Lease 排他分布式锁保证单实例安全排查，生成 GitHub 自愈 PR。
3. **全链路透明可追溯**：SRE 既可在集群内查实时日志，也可在 GitHub 上查每日历史账本，故障自愈走标准 GitOps 审批流程，达成生产级运维治理闭环。
