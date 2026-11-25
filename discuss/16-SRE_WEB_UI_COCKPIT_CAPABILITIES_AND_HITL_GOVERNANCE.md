# 16 - SRE Web UI 交互式控制台核心功能剖析、底层链路与人机协同 (HITL) 治理指南 (SRE Web UI Cockpit & Human-in-the-Loop Governance)

> **导读与摘要**：
> 针对架构图中用户的核心疑问：
> **“这个模块主要就是在 Kagent Web UI 上给 Agent 发消息、问问题来看结果吗？这块功能到底如何体现？”**
> 
> 本文对 `SRE Web UI 控制台 (:30083/agents/chat)` 进行了系统梳理与深度技术剖析：
> 1. **它绝非普通泛型聊天对话框**，而是平台 SRE 团队的 **人机协同运维驾驶舱 (Human-in-the-Loop Cockpit)**；
> 2. 它集成了 **即时提问与事实三态根因推导**、**按需主动 L1-L4 阶梯巡检**、**生产零热修刚性拦截防御** 与 **GitOps 自愈代码审查审批** 四大核心职能；
> 3. 它与后台每 15 分钟无人值守的 **Sentinel 哨兵** 形成“后台静默守护 + 前台专家下钻”的动静互补闭环。

---

## 一、核心问题定性与功能全景矩阵 (Executive Summary)

### 1.1 它是“简单发消息看结果”吗？
**答案：远不止于此。**
在普通聊天工具中，用户发消息给 AI，AI 仅凭自身的通用大模型权重生成泛化回答；
而在 **KagentGuardian 的 SRE Web UI** 中，每次交互都触发了完整的企业级沙箱调度、MCP 协议网关鉴权、实时物理证据检索与 GitOps 治理流：

```text
+─────────────────────────────────────────────────────────────────────────────────────────────────+
| 🖥️  SRE Web UI 控制台 (http://<NodeIP>:30083/agents/chat)                                       |
+─────────────────────────────────────────────────────────────────────────────────────────────────+
                               │
            ┌──────────────────┼──────────────────┬──────────────────┐
            ▼                  ▼                  ▼                  ▼
   【功能一：按需即时体检】   【功能二：深度排障诊断】   【功能三：零热修防御】   【功能四：自愈审批核准】
   输入：@kagent-guardian   输入：排查昨日 5xx 抖动   输入：把副本数改成 5    输入：批准自愈 PR #12
   输出：极简健康大盘卡片    输出：事实三态根因研报    输出：刚性阻断 + GitOps  输出：轮询 PEP 触发调谐
   耗时：~15s 调全量工具     证据：PromQL+Logs+K8sGPT  保障：严禁在线热改集群  闭环：ArgoCD 声明式生效
```

### 1.2 四大核心功能对比矩阵

| 功能维度 | 用户在 Web UI 上的操作 | Agent 底层联动与执行动作 | 最终交付结果与安全边界 |
| :--- | :--- | :--- | :--- |
| **1. 按需即时巡检 (On-Demand Patrol)** | 输入 `@kagent-guardian 平台巡检` 或询问集群健康状态。 | 并发调用 `mcp-proxy-k8s-read`、`mcp-proxy-kagent-tool`、`mcp-proxy-k8sgpt-read` 执行 L1-L4 四级扫描。 | 对话框实时以 Markdown 渲染 **双模健康体检大盘卡片**（各层 PASS/WARN/FAIL 明确直观）。 |
| **2. 交互式根因下钻 (Interactive RCA)** | 输入自然语言自由提问：“为什么 workerpool 出现冷启动卡顿？” | 执行 **CFS 算力与冷启动双因子决策树** 计算，抓取 Pod 日志与 Events 事件聚类。 | 严格按 **事实三态** 输出深度报告：明确区分真实物理证据、架构推论假设与未知盲区，严禁脑补。 |
| **3. 生产零热修拦截 (No Hot-Patching PEP)** | 诱导执行破坏性变更：“帮我在线扩容副本数”或“kubectl delete pod”。 | 触发 **PEP 策略决策与拦截点**，拒绝执行任何直连集群写操作。 | 刚性阻断（Fail-Closed），在对话框给出安全警示，并主动生成声明式 YAML 补丁方案。 |
| **4. 自愈核准与闭环 (HITL Approval Gate)** | 审查 Agent 提出的自愈方案，在 UI 或 GitHub 输入核准指令。 | 调用 `mcp-proxy-github-ops` 检查 PR 状态，轮询 `agentstage_approval_result`。 | 只有在 SRE 审批通过后，才允许 ArgoCD 声明式同步生效，杜绝 Agent 失控跑偏。 |

---
## 二、Web UI 交互背后的端到端技术链路 (Under the Hood)

当 SRE 在网页对话框输入一条消息并回车时，底层实际发生了以下严谨的系统流转：

```text
[SRE 运维专家] 
     │ (1. 在 Web UI :30083 输入运维指令)
     ▼
[Web UI 前端容器 / atenet-router 网关]
     │ (2. 路由转发会话请求)
     ▼
[Substrate 沙箱运行时]
     │ (3. 基于黄金快照 Golden Snapshot <=3.5s 秒级唤醒)
     ▼
[PEP 策略拦截与安全门禁]
     ├── (4a. 诱导热修行为) ──► [刚性拦截阻断] ──► 引导生成声明式 GitOps PR
     │
     └── (4b. 合规排障行为) ──► 分发至 Enterprise MCP Gateway (:8080)
                                      │
                                      ├──► [mcp-proxy-k8s-read]: 集群拓扑只读检索
                                      ├──► [mcp-proxy-kagent-tool]: PromQL CFS/延时指标
                                      ├──► [mcp-proxy-k8sgpt-read]: AI 异常聚类分析
                                      └──► [mcp-proxy-github-ops]: PR 自愈代码与审批
```

### 关键架构亮点：
1. **黄金快照秒级拉起（Golden Snapshot）**：  
   Agent 并非冷冰冰的常驻死进程，而是依托 Substrate 沙箱。通过预编译快照技术（`ActorTemplate GoldenSnapshotReady`），沙箱能在 **1.8s ~ 3.5s** 内迅速预热就绪，既节约宿主机内存，又保证极低的首字延迟。
2. **MCP 网关鉴权隔离（Enterprise MCP Gateway）**：  
   Web UI 绝不直接连接底层基础设施，而是通过 `agentstage-mcp-gateway` (Port 8080) 统一代理。网关动态消除重复工具名冲突，并强制实施 RBAC 与网络白名单隔离。
3. **Session GC 自动清理**：  
   伴生的 `kagent-guardian-session-gc` 定时任务（每天凌晨 03:00 执行）会自动清理超期未活跃的历史对话与沙箱挂载，防止磁盘与数据库连接堆积。

---

## 三、四大核心交互能力深度展示

### 3.1 能力一：按需即时平台体检 (On-Demand Patrol)
在非每 15 分钟的定时窗口内，若集群发生突发告警或发布上线前后，SRE 可在 Web UI 即时发起全链路体检：

* **SRE 输入**：
  > `@kagent-guardian 请对当前 Kagent 平台与 Substrate 运行时执行全量 L1-L4 综合健康巡检，输出系统体检报告大盘。`
* **Agent 交互输出**：
  Agent 在 15 秒内并发调用集群资源、度量指标与 AI 引擎，在 Web UI 输出标准 Markdown 极简卡片：
  ```text
  ╭─────────────────────────────────────────────────────────────╮
  │ 🟢 Kagent Guardian 平台健康巡检大盘 (2026-11-25 10:15 UTC)   │
  ├───────────────────────┬────────────┬────────────────────────┤
  │ 巡检维度              │ 健康状态   │ 关键指标证据           │
  ├───────────────────────┼────────────┼────────────────────────┤
  │ L1: 物理节点与资源水位│ ✅ PASS    │ Ready: 1/1, CPU: 18.2% │
  │ L2: 运行时与流量网关  │ ✅ PASS    │ Router 200, 5xx: 0%    │
  │ L3: 控制面与核心数据  │ ✅ PASS    │ PostgreSQL 1/1 Running │
  │ L4: Substrate黄金快照 │ ✅ READY   │ 启动耗时: 1.8s (≤3.5s) │
  ╰───────────────────────┴────────────┴────────────────────────╯
  基线综合健康分: 100 / 100 (HEALTHY)
  GitOps 状态: 声明式配置完全对齐，无漂移。
  ```

---

### 3.2 能力二：交互式根因下钻与“事实三态”推导 (Interactive RCA)
当遇到复杂线上疑难杂症时，SRE 可把 Web UI 作为专家智囊进行深度追问：

* **SRE 追问**：
  > `排查一下为什么昨天晚上 22:30 的批处理任务耗时翻倍了？是不是算力不够？`
* **Agent 内部决策树分析**：
  - 调用 `prometheus_promql_tool` 查询 `container_cpu_cfs_throttled_periods_total`；
  - 查询沙箱冷启动唤醒指标 `substrate_cold_start_duration_ms`；
  - 自动执行 **CFS 算力与冷启动双因子决策树**：
    CFS 节流比 = 2.1% <= 20% (算力充沛)
    沙箱冷启动耗时 = 1850ms <= 3000ms
* **Web UI 渲染输出（杜绝幻觉的“事实三态”）**：
  - **【CONFIRMED FACTS (确证物理事实)】**：昨晚 22:30 容器 CFS 节流率仅 2.1%（健康水位），Substrate 沙箱唤醒耗时 1.85s，算力配额充裕，无 CPU Throttling。
  - **【SUSPECTED HYPOTHESES (架构推论假设)】**：检索到 PostgreSQL 在 22:31 存在一次持续 45s 的表级排他锁等待，推测批处理耗时翻倍由 DB 锁竞争引起，而非算力瓶颈。
  - **【UNKNOWN / MISSING (未知盲区说明)】**：当前 Prometheus 未启用 PostgreSQL 表级慢查细粒度跟踪，建议补充相关度量。

---

### 3.3 能力三：生产零热修刚性拦截 (No Hot-Patching PEP)
这是区别于一般开发类 Agent 最核心的生产级安全门禁：

* **测试场景：SRE 尝试在线热修改**：
  > `kagent-controller 现在的副本数不够，请直接帮我扩容到 3 个副本！`
* **Agent 交互表现（绝不受骗，绝不越权）**：
  ```text
  🛡️【安全拦截：No Hot-Patching 铁律阻断】
  
  根据生产环境安全治理规约，KagentGuardian 严格遵循只读排障与 GitOps 交付原则，
  严禁直接在线对运行态集群执行 kubectl scale、patch 或命令注入！
  
  ✅ 合规解决路径：
  已为您生成声明式 GitOps 修复补丁，请通过代码审查合入：
  ```
  ```yaml
  # 推荐 GitOps 变更文件: deploy/02-agent.yaml
  spec:
    replicas: 3 # 原为 1，建议结合 HPA 或通过 PR 提交
  ```
  ```text
  已为您准备 Pull Request 提案分支: fix/kagent-controller-scale-up
  请通过 GitHub PR 进行团队审查与自动化 CI 验证。
  ```

---

### 3.4 能力四：人机审批与自愈闭环枢纽 (HITL Approval Gate)
对于高危动作（例如调整生产资源配额、下发配置热更），系统引入 **Human-in-the-Loop (人机协同)** 机制：
1. Agent 不具备直接合入代码或改动生产资源的私钥权限；
2. Agent 将修复代码作为 **Pull Request** 提交至 GitHub，并在 Web UI 对话框中提供 PR 链接与变更预览（Diff）；
3. SRE 专家核对 PR 代码无误后，在 GitHub 或 UI 确认同意；
4. Agent 轮询到 `APPROVED` 状态后，触发 ArgoCD 执行调谐，实现 100% 可审计、可回滚的自愈闭环。

---

## 四、Web UI 与外部哨兵 (Sentinel) 的动静协同关系

在整个 KagentGuardian 架构中，Web UI 与外部哨兵并不割裂，而是构成了完整的立体防护网络：

| 对比维度 | 外部哨兵 (Sentinel) | SRE Web UI 控制台 |
| :--- | :--- | :--- | 
| **运行机制** | 后台 CronJob 每 15 分钟无人值守运行 | 前台交互式 SRE 随时随地提问与排查 |
| **资源成本** | 0 LLM Token 短路，极轻量 (6s 退出) | 交互时唤醒沙箱与 LLM 深度推理 |
| **主要职责** | 守住底线、常态防漏、日常账本 | 专家排障、根因归因、人机核准 |
| **协同联动** | 当哨兵打分 <85 触发熔断时 | 唤醒 Agent 并将排查报告推送至 UI / GitHub |

---

## 五、结论与总结

1. **功能定性**：Web UI（`:30083/agents/chat`）绝不仅是一个“聊天对话框”，而是 KagentGuardian 的 **SRE 交互驾驶舱** 与 **安全策略执行点 (PEP)**。
2. **核心能力**：涵盖了 **即时体检卡片渲染**、**事实三态无幻觉诊断**、**生产零热修刚性拦截** 与 **GitOps 自愈人机审批** 四大核心功能。
3. **架构价值**：它使得 SRE 在享受自然语言交互带来便捷性的同时，集群底层依然处于严格的只读隔离与 GitOps 审计监管之下，真正实现了企业级可信运维。
