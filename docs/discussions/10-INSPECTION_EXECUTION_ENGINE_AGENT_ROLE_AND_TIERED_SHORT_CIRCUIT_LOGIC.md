# 10 - L1-L4 巡检执行内核设计：脚本与 Agent 角色定位、阶梯依赖拓扑与熔断短路机制 (Inspection Engine, Agent Role & Circuit-Breaker Architecture)

## 讨论背景与核心论题 (Context & Core Questions)

在深化生产落地实施方案时，运维架构团队提出了两个关乎系统底座命脉的本质问题：
> **论题一（角色与定位）**：“定时的 L1-L4 级别巡检，是不是就是一个 CronJob 自动化脚本，跟大模型/Agent 没关系，只是最后把巡检结果对接给 Agent，让它解读一下形成报告？”  
> **论题二（依赖与熔断）**：“L1 到 L4 巡检有先后顺序吗？有依赖关系吗？比如 L1 物理基础设施 (Infra) 层面就挂了，后面的巡检是不是就直接停了、短路报警了，不再跑后面的检查了？”

本文针对上述问题进行彻底的解构与技术定调，阐明**自动化脚本与大模型 Agent 的能力分工边界**，并给出**严格的四级依赖拓扑与熔断短路机制设计**。

---

## 一、核心论题一：L1-L4 巡检本质是脚本还是 Agent？——三流派辨析与系统定位

很多工程师初次接触 AgentOps 时，往往容易产生两种极端误区：
1. **极端一（虚无主义）**：“Agent 无用论，就是一堆 Bash 脚本跑完，最后丢给 ChatGPT 润色成漂亮的 Markdown 报告。”
2. **极端二（激进主义）**：“Agent 万能论，什么都靠大模型自己写 Prompt 动态去查，底层不用任何确定性工具。”

为了在生产环境中取得**极致的稳定性、低成本与真正的智能推理**，我们对比业界三种架构流派：

| 维度 | 流派 A：静态脚本 + 后置大模型润色<br>(Script-Probe & LLM-Summarizer) | 流派 B：纯 Agent 动态自主探查<br>(Pure Autonomous ReAct Agent) | 流派 C (本方案采用)：分层确定性探针 + Agent 深度推理决策引擎<br>(Hybrid Tiered Probe & Agent Decision Brain) |
| :--- | :--- | :--- | :--- |
| **触发方式** | CronJob 执行大段硬编码 Python/Bash | CronJob 仅作为极简“唤醒闹钟” (POST API) | **CronJob 仅作为极简“唤醒闹钟”** (发送巡检会话指令) |
| **数据采集者** | 硬编码脚本通过 curl/promql 拉取固定指标 | Agent 每一轮通过思维链自主调用工具获取指标 | **确定性 MCP 工具集** (`kagent-tool-server`, `k8sgpt`, `kagent-api`) |
| **动态下钻能力** | ❌ **完全无动态能力**。指标异常时无法自主追查具体 Pod/事件 | ✅ 极强。能像侦探一样按需深度追查 | ✅ **按需自适应下钻**。发现指标异常时，Agent 自主追加调用深层诊断工具 |
| **因果推导 (RCA)**| ❌ 无法做跨层关联。只能罗列孤立数据 | ✅ 具备关联推导，但多轮 Agentic 循环耗时较长 | ✅ **跨层因果图谱关联**。将 L1 资源、L2 节流、L3 数据库与 L4 探针结合推导 |
| **自愈代码生成** | ❌ 只能报错，无法智能生成修复 YAML Diff | ⚠️ 自由发挥可能格式漂移或出现幻觉 | ✅ **严守契约的代码生成**。结合当前状态与历史基准，直接输出合规自愈 PR |
| **Token 消耗** | 极低（仅最后一次总结） | 较高（每一步均消耗 Reasoning Tokens） | **优选平衡**（结构化上下文输入，关键节点高密度推理） |

### 1.1 结论定调：Agent 绝不仅仅是“后置报告解读员”
如果只用脚本跑完丢给大模型写报告，那系统充其量只是一个带皮肤的 Prometheus 告警器。在 Guardian 架构中，**脚本与 Agent 各司其职，形成了“感官神经”与“大脑中枢”的有机协同**：

```mermaid
graph TD
    classDef cron fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef tool fill:#ede7f6,stroke:#512da8,stroke-width:2px;
    classDef agent fill:#fff3e0,stroke:#f57c00,stroke-width:2px;
    classDef action fill:#e8f5e9,stroke:#388e3c,stroke-width:2px;

    Cron["1. 轻量 CronJob 触发器<br>(无业务逻辑，仅发送唤醒指令)"]:::cron -->|"POST /api/v1/sessions"| Agent["2. Guardian Agent 决策中枢 (大脑)<br>(加载常驻 SRE 知识库与推导逻辑)"]:::agent
    
    subgraph Senses ["确定性感官工具层 (MCP Tools)"]
        T1["kagent-tool-server (只读 K8s API)"]:::tool
        T2["Prometheus PromQL 引擎 (精准时序指标)"]:::tool
        T3["k8sgpt-mcp-server (AI 初筛异常事件)"]:::tool
        T4["kagent-api 合成探针服务 (执行 L4 黑盒体检)"]:::tool
    end

    Agent <-->|"动态调度与按需获取"| Senses
    
    Agent -->|"多层指标交叉推导<br>& 根因归因 (RCA)"| Decision{"3. 综合健康评估与决策"}:::agent
    
    Decision -->|"GREEN 正常"| Log["写入 Git 每日底账 & 开启签收 PR"]:::action
    Decision -->|"YELLOW 瓶颈"| Issue["自动创建 GitHub Issue 跟踪工单"]:::action
    Decision -->|"RED 复合故障"| PR["抢占 Lease 锁，生成精确自愈 PR<br>(代码级 YAML Diff + 标明 Fixes #Issue)"]:::action
```

* **MCP 探针工具是“手和眼”（确定性采集）**：底层指标计算（如 PromQL 计算 15 分钟节流率、Kubernetes API 获取 Pod 状态）绝不能交给大模型去脑补，必须由确定性的 MCP 服务严格执行；
* **Agent 大模型是“中枢大脑”（因果推导与行动决策）**：
  1. **跨层多维因果关联**：脚本只能并列输出“Worker 限流 28%”和“网关 503 超时”两条孤立日志；只有 Agent 能通过知识库理解“*这是由于冷启动排队造成网关 Parking 等待超时*”，并明确指出根因在 Worker Request 配置过小；
  2. **自适应动态补盲**：当发现某指标边缘异常时，Agent 能自主决定追加调用 `get_events` 或 `describe_pod` 进一步收集证据；
  3. **精确的代码合成（Code Synthesis）**：Agent 在判定需要自愈时，能基于当前运行的 CRD Spec，精准生成修改后的 GitOps YAML Diff 并提起 Pull Request。这是任何静态脚本无法灵活做到的。

---

## 二、核心论题二：L1 到 L4 的先后顺序、拓扑依赖与熔断短路机制

关于“L1 到 L4 是否有顺序？底层挂了后面的巡检是不是直接停了？”，答案非常明确：  
**L1 到 L4 存在严格的自底向上拓扑依赖关系，必须执行严格的“短路熔断机制（Short-Circuit / Circuit-Breaker Pattern）”！**

### 2.1 四级巡检的物理依赖链 (Strict Bottom-Up Dependency)

```mermaid
graph BT
    classDef l1 fill:#ffebee,stroke:#c62828,stroke-width:2px;
    classDef l2 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px;
    classDef l3 fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef l4 fill:#e1f5fe,stroke:#0277bd,stroke-width:2px;

    L1["<b>L1 基础资源层 (Infrastructure)</b><br>• Node Ready 状态<br>• 磁盘空间 / DiskPressure<br>• 内存 / OOM 压力<br>• 内核状态 & 网络连通性"]:::l1
    L2["<b>L2 运行时与网关层 (Runtime & Gateway)</b><br>• WorkerPool Pod 状态<br>• CPU CFS 节流率 (Throttling)<br>• 网关冷启动调度延时<br>• Parking 503 错误率"]:::l2
    L3["<b>L3 控制面与持久化层 (Control Plane & State)</b><br>• APIServer 连通与延迟<br>• Controller-Manager 调谐心跳<br>• PostgreSQL 连接池 & 慢查询<br>• CRD 同步状态"]:::l3
    L4["<b>L4 端到端黑盒合成探针 (E2E Synthetic Probe)</b><br>• 真实发起 API 创建探针会话<br>• 调度独立沙箱 Pod<br>• 执行端到端推理闭环<br>• 测量黑盒全链路耗时"]:::l4

    L1 -->|"L1 是 L2 的物理基石<br>(节点挂死则容器全灭)"| L2
    L2 -->|"L2 是 L3 的运行载体<br>(网关与Pod正常才能访问控制面)"| L3
    L3 -->|"L1~L3 正常才是 L4 的前提<br>(系统健康才能发起压力探针)"| L4
```

### 2.2 为什么必须短路？——严防“火上浇油（Cascading Failure）”
如果底层 L1 节点已经发生 `DiskPressure`、CPU 100% 锁死或大量 Pod 处于 Evicted 状态：
* **短路的必要性一（避免数据失真与误判）**：底座断裂时，上层 L2/L3/L4 的所有失败（如探针 503 超时、沙箱拉起失败）都是 L1 故障的**次生衍生现象**。如果不加短路，告警系统会瞬间涌出数十个涵盖 L2、L3、L4 的并发假告警，掩盖真实的 L1 宿主机故障；
* **短路的必要性二（严防加剧系统雪崩）**：**L4 探针是主动出击的黑盒测试**（会调用 API 创建真实的临时 Agent 会话与沙箱容器）。当底层宿主机或网关已经接近宕机时，继续强行执行 L4 探针，等于向濒临崩溃的集群强行注入新的调度与内存负载，形成致命的**级联击穿（Thundering Herd / Cascading Failure）**！

---

## 三、三级熔断短路状态机设计 (Circuit-Breaker State Machine)

```mermaid
flowchart TD
    Start(["开始巡检 (Cron 唤醒 Agent)"]) --> CheckL1["1. 巡检 L1 物理基础资源"]
    CheckL1 --> EvalL1{"L1 状态判定"}
    EvalL1 -->|"Fatal: Node NotReady / 磁盘打满"| ShortCircuitL1["🚨 触发 L1 致命熔断 (Fatal Short-Circuit)<br>• 立即中止巡检流程！严禁启动后续 L2/L3/L4！<br>• 综合健康分直置 0~10 分<br>• 自动在 GitHub 开 Critical Issue，直达 SRE 手机"]
    EvalL1 -->|"Warning: 节点磁盘>75% 但健康"| DegradeL1["标记 L1 扣分项 (黄色预警)<br>非致命状态，继续推进"] --> CheckL2["2. 巡检 L2 运行时与网关"]
    EvalL1 -->|"Pass: 基础资源全部正常"| CheckL2
    
    CheckL2 --> EvalL2{"L2 状态判定"}
    EvalL2 -->|"Fatal: 网关崩溃 CrashLoop / 503>50%"| ShortCircuitL2["🚨 触发 L2 运行时熔断<br>• 终止 L3/L4 探针，避免雪崩！<br>• 健康分置 20~30 分，自动建单报障"]
    EvalL2 -->|"Warning: CFS 节流超标但运行中"| DegradeL2["记录 L2 扣分项 & 记录节流指标<br>非致命崩溃，继续推进"] --> CheckL3["3. 巡检 L3 控制面与持久化"]
    EvalL2 -->|"Pass: 运行时平稳"| CheckL3
    
    CheckL3 --> EvalL3{"L3 状态判定"}
    EvalL3 -->|"Fatal: PostgreSQL 只读或离线"| ShortCircuitL3["🚨 触发 L3 状态层熔断<br>• 严禁发起 L4 探针 (防脏写/级联超时)！<br>• 健康分置 30~40 分，自动建单"]
    EvalL3 -->|"Pass / 仅有软性告警"| CheckL4["4. 安全进入 L4：端到端黑盒合成探针<br>(创建沙箱会话，验证全链路闭环)"]
    
    CheckL4 --> FinalReport["5. 汇聚全流程数据与打分<br>• GREEN: 写入日终底账 PR / Commit<br>• YELLOW: 开 Issue 跟踪<br>• RED (自愈类): 提 PR (Fixes #Issue)"]
```

### 3.1 熔断分级与处置矩阵 (Short-Circuit Policy Matrix)

| 巡检层级 | 触发熔断的硬性条件 (Fatal Breakers) | 熔断动作 (Action) | 跳过后续层级 | 最终分值区间 | 自动化处置动作 |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **L1 基础资源层** | • 任意核心 Node 处于 `NotReady`<br>• 节点存在 `DiskPressure` 或根分区爆满<br>• 核心网络组件 (CoreDNS) 全灭 | **强制熔断** | **跳过 L2, L3, L4** | **0 ~ 15 分** | 立即通过 GitHub 提 Critical Issue；手机端强推送；绝不启动合成探针。 |
| **L2 运行时层** | • Gateway Pod 处于 `CrashLoopBackOff`<br>• 网关 503 错误率 > 50% 且无法恢复<br>• 宿主机 Cgroup 驱动崩溃 | **强制熔断** | **跳过 L3, L4** | **15 ~ 30 分** | 判定网关不可用，跳过探针防止积压死连接；自动建单追踪。 |
| **L3 控制面层** | • PostgreSQL 宕机或变为只读<br>• K8s APIServer 响应超时率 > 60% | **强制熔断** | **跳过 L4** | **30 ~ 45 分** | 严禁发起 L4 探针（防止写入脏会话和引发级联锁超时）；建单报障。 |
| **L4 黑盒探针层** | • 合成探针沙箱冷启动超时 (>15s)<br>• 探针执行返回非 200 状态码 | **常规判定** | 无后续层级 | **45 ~ 60 分** | 判定系统可用性受损；触发根因追查；根据规则创建自愈 PR 或 Issue。 |

---

## 四、生产巡检交互时序完整拓扑 (Sequence Topology)

```mermaid
sequenceDiagram
    autonumber
    actor Cron as 定点 CronJob (09:00/18:00)
    participant GW as Kagent Gateway
    participant Agent as Guardian Agent
    participant MCP as MCP 工具集群
    participant GH as GitHub 仓库 (Issue / PR)

    Cron->>GW: POST /api/v1/sessions ("执行平台四级定点巡检")
    GW->>Agent: 唤醒常驻 SRE 会话实例
    
    rect rgb(255, 235, 238)
        Note over Agent,MCP: 【L1 基础层体检】
        Agent->>MCP: 调用 kagent-tool-server 获取 Node 状态与事件
        MCP-->>Agent: 返回 Node 状态
        alt 发现节点 NotReady 或 磁盘打满
            Agent->>Agent: 🚨 触发 L1 熔断！评分扣至 10分！立即中止后续体检！
            Agent->>GH: 创建 [CRITICAL] L1 Node Failure Issue，直达手机
            Agent-->>GW: 返回熔断巡检中断简报，会话完结
        end
    end

    rect rgb(255, 243, 224)
        Note over Agent,MCP: 【L2 运行时体检 (L1通过后方可进入)】
        Agent->>MCP: PromQL 采集 Worker CFS 节流率 & 网关 P99
        MCP-->>Agent: 返回指标数据 (发现节流 28%，网关延时 9.8s)
        Agent->>Agent: 判定为配置瓶颈 (非服务彻底死亡，不熔断，记严重扣分)
    end

    rect rgb(232, 245, 233)
        Note over Agent,MCP: 【L3 控制面体检】
        Agent->>MCP: 查询 APIServer 心跳 & PGSQL 连接池状态
        MCP-->>Agent: PGSQL 正常，连接池使用率 22% (Pass)
    end

    rect rgb(225, 245, 254)
        Note over Agent,MCP: 【L4 端到端黑盒探针 (前置层健康时方可执行)】
        Agent->>MCP: 调用 kagent-api 执行单次合成探针会话
        MCP-->>Agent: 探针完成，冷启动耗时 9.2s (印证了 L2 节流瓶颈)
    end

    Note over Agent: 【大脑综合归因与行动决策】
    Agent->>Agent: 综合评分: 45分 (RED)<br>推导: Worker CFS 导致冷启动劣化与网关超时
    Agent->>GH: 查重无 Open Issue ➔ 创建全新 Issue 跟踪单
    Agent->>GH: 满足自愈标准 ➔ 抢占 Lease 锁，提交配置修改自愈 PR (Fixes #Issue)
    Agent-->>GW: 巡检完成，结构化全链路日志归档于会话历史中
```

---

## 五、总结与核心共识 (Summary & Consensus)

1. **角色界限清晰明确**：
   - **自动化脚本 / CronJob**：仅仅是“没有思维的定时闹钟”，负责准点发出一条唤醒指令；
   - **MCP 工具集**：是“感官与手脚”，负责执行确定性的 PromQL 和 Kubernetes API 调用，保证底层数据的精确无误；
   - **Guardian Agent**：是“决策中枢与侦探大脑”，负责**因果链关联（RCA）、动态追查补盲、工单 Issue 生成、自愈代码合成与 PR 提交**。
2. **阶梯依赖与短路熔断坚如磐石**：
   - 严格遵循 `L1 ➔ L2 ➔ L3 ➔ L4` 的自底向上依赖链；
   - 遭遇底层致命故障（如物理节点故障或网关宕机）时，**无条件触发短路熔断（Fast-Fail）**，严禁触发 L4 黑盒探针，既保护故障集群不发生二次击穿雪崩，又确保报警精准直击根本原因。
