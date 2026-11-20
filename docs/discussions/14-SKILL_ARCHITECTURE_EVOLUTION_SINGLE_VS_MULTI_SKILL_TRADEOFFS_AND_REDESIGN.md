# 技术专题讨论十四：Guardian 领域 Skill 架构演进评估 —— 单一高内聚 Skill 升级 vs 细粒度多 Skill 拆分的技术辨析与重构蓝图

> **文档标识**：`DOC-DIS-14`  
> **归属工程**：Kagent Guardian 平台运维与自愈智能体  
> **前序依赖**：`DOC-DES-02` (HLD v2.0)、`DOC-DES-03` (LLD v2.0)、`DOC-DIS-10` (混合执行引擎)、`DOC-DIS-12` (MCP覆盖与边界)、`DOC-DIS-13` (网关与HITL治理)  
> **讨论主题**：在 HLD/LLD v2.0 确立了 AgentStage MCP 网关 PEP、HITL 审批流、纯 GitHub 运营闭环与阶梯短路混合引擎的新架构下，Guardian 领域 Skill（`skills/kagent-guardian`）是否需要更新？应拆分为多个细粒度 Skill 还是保持单一内聚 Skill 并进行模块化升级？

---

## 一、背景与论题提出 (Context & Core Dilemma)

### 1.1 现状审视：现有 Skill (v1.0) 与 v2.0 架构的“认知代差”
当前仓库中的 `skills/kagent-guardian/SKILL.md`（Version 1.0.0）编写于 Guardian 探索初期。通过逐项比对，现有 Skill 存在严重的“设计与认知断层”：
1. **执行载体认知断层**：v1.0 通篇充斥着裸 Bash/`kubectl` 物理命令。但在严肃生产架构中，Guardian 作为 Kagent 托管智能体，没有直接的主机 Bash 终端，所有能力必须 100% 映射到 MCP 工具调用；
2. **监控指标获取盲区**：v1.0 无法获知 Discussion 12/13 中由 `mcp-proxy-kagent-tool` 代理的 Prometheus PromQL 工具（`prometheus_promql_tool`、`prometheus_query_range_tool`），排查延时只能依赖“脑补”或日志文本截取；
3. **安全网关与 HITL 盲区**：v1.0 完全没有 AgentStage MCP Gateway 作为 PEP 的概念，不知道写操作会被网关拦截为审批单（`approval_id`），也缺乏调用内置工具 `agentstage_approval_result` 轮询审批结果的 SOP；
4. **纯 GitHub 运营断层**：v1.0 仍在使用过时的单次临时巡检报告文件，缺乏 Discussion 09 确立的**日终账本 PR 签收机制**（`[Daily-Patrol] YYYY-MM-DD Ledger`）与 **Title Hash 幂等 Issue 告警推送机制**；
5. **混合执行与短路感知缺失**：v1.0 默认巡检由 Agent 从头到尾包揽，缺乏 Discussion 10 确立的“轻量 Sentinel 脚本极速打基线 + 异常短路熔断唤醒 Agent”的混合拓扑认知。

**结论毫无悬念：当前 Skill 必须进行彻底重构与升级！**

### 1.2 核心架构分歧：拆分为多 Skill 还是保留单 Skill？
在升级方向上，架构团队内部存在两种典型思路：
- **方案 A（细粒度多 Skill 拆分）**：将 Guardian 的职能按阶段拆分成 3~4 个专属 Skill（如巡检打分 Skill、深度排障 Skill、自愈与审批 Skill）；
- **方案 B（单一内聚高密度 Skill 模块化升级）**：保留唯一的 `kagent-guardian` Skill，在单一文件中构建清晰解耦的“高内聚子流水线”（Guardrails ➔ Tool Mapping ➔ Patrol SOP ➔ RCA Trees ➔ GitOps/HITL SOP ➔ Templates）。

---

## 二、方案 A：细粒度多 Skill 拆分的深度解构与缺陷剖析 (Multi-Skill Option)

### 2.1 方案 A 的设想与理论优势
多 Skill 方案通常遵循单一职责原则：单个 Skill 文件小，理论上按需加载，避免单次 Prompt 偏长。

### 2.2 生产级 SRE 场景下方案 A 的致命缺陷
然而，在深入结合企业级生产运维场景、Kagent 运行时机制以及大模型认知特性后，方案 A 暴露出了四大不可调和的矛盾：

```
[故障触发: 节点 CFS 截流超时]
            │
            ▼
[Skill 1: 巡检探针] ──(上下文中断/跨技能跳转)──> [Skill 2: 根因归因] ──(上下文中断/参数丢失)──> [Skill 3: GitOps PR]
      ▲                                               ▲                                            ▲
      └────────────────────────────────── 意图路由错误 / 证据丢失 / 延迟剧增 ───────────────────────┘
```

#### 缺陷 1：意图路由漂移与技能切换延迟（Intent Routing & Skill Handoff Penalty）
SRE 运维排障是一个**高度时间敏感、环环相扣的连续确定性闭环**。
- 当用户提问“目前网关偶发 503，帮我排查并修复”时，大模型在第一轮必须做元决策（Meta-Reasoning）：“我应该激活巡检 Skill、排障 Skill 还是自愈 Skill？”；
- 模型极易在 Skill 边界发生**意图震荡**；跨 Skill 调度会引入额外的推理轮次，白白浪费 10~20 秒黄金排障时间。

#### 缺陷 2：证据链截断与认知孤岛（Broken Evidence Chain）
HLD 原则 1 确立了**“证据第一原则（Evidence-First）”**：自愈 PR 的 Body 中必须完整嵌入巡检阶段获取的物理指标。
- 在多 Skill 架构下，巡检 Skill 收集的指标属于临时执行上下文。当模型切换到自愈 Skill 时，模型必须完全依赖对话历史中的 Token 记忆进行重构，极易发生**指标细节丢失、参数精算误差甚至幻觉拼凑 Diff**。

#### 缺陷 3：HITL 审批状态机跨 Skill 无法连贯闭环
AgentStage Gateway HITL 审批流具有即时拦截特性：当发起写操作时，网关立即挂起并返回 `approval_id`。如果将自愈与审批治理拆分，跨 Skill 状态机跳转显著放大了 LLM 状态机迷失的概率。

#### 缺陷 4：Kagent CRD 装配与 GitOps 交付复杂化
在 Kagent 框架中，智能体装配定义在 `AgentTemplate`（`01-agenttemplate.yaml`）中。若拆分为 4 个 Skill，CRD 声明必须引用多项，增加了配置漂移（Config Drift）与遗漏引用的运维负担。

---

## 三、方案 B：单一内聚高密度 Skill 模块化升级的深度优势 (Single-Skill Option)

### 3.1 方案 B 的设计内核
保持 `skills/kagent-guardian/SKILL.md` 作为全平台**唯一的、终极的 SRE 领域专家 Skill**。在该 Skill 内部，通过严谨的 Markdown 目录结构与条件门禁，划分为 6 大高内聚子流水线：
- **Section 0: 核心铁律与安全门禁**（No Hot-Patching + Gateway PEP + HITL 审批拦截协议）
- **Section 1: 平台内幕认知与 MCP 工具映射表**（`mcp-proxy-kagent-tool` PromQL、`mcp-proxy-github-ops` 等）
- **Section 2: 阶梯短路全景巡检流水线**（Sentinel 基线对接 + L1-L4 穿透 + 健康打分）
- **Section 3: 典型故障根因决策树**（CFS 双因子、网关 503、Pod Crash、内部 vs 外部 LLM 判别）
- **Section 4: GitOps 声明式自愈与 HITL 轮询闭环 SOP**（Lease 锁 ➔ 查重 ➔ 提 PR ➔ HITL 拦截轮询 ➔ ArgoCD 验收）
- **Section 5: 纯 GitHub 闭环运营规范**（日终巡检账本 PR + Title Hash 幂等 Issue 预警）
- **Section 6: 自适应双模输出模板**（Fast Inquiry 卡片 vs Standard 5-Section RCA）

### 3.2 方案 B 的四大核心工程优势

```
┌────────────────────────────────────────────────────────────────────────┐
│             方案 B: 单一高内聚全流程 Skill (Zero Context Switch)        │
├────────────────────────────────────────────────────────────────────────┤
│ [输入: 告警/提问] ➔ [阶梯短路检测] ➔ [根因决策树推导] ➔ [GitOps PR/HITL]│
│                                                                        │
│   ✓ 物理证据链零损耗贯穿全程                                           │
│   ✓ 零技能切换时延，意图 100% 聚焦                                     │
│   ✓ 完美承接 AgentStage MCP Gateway 6 步策略与 HITL 轮询契约           │
│   ✓ 全文约 2,000 Tokens，在 128k 上下文窗口中仅占 1.5%，开销微乎其微     │
└────────────────────────────────────────────────────────────────────────┘
```

1. **端到端证据链零损耗贯通（Zero-Loss Continuity）**：
   从 PromQL 指标抓取、CFS 截流判定，到自愈 PR 生成与 ArgoCD 状态复测，智能体在一个完整的连续工作流中认知推导，物理证据直接作为 PR Body 事实源，彻底杜绝幻觉；
2. **零调度决策开销（Zero Routing Overhead）**：
   模型无需进行“调用哪一个 Skill”的元调度推理，进入任务即按内置阶段门禁（Phase Gate）逐步向下推进，响应迅速、逻辑确定；
3. **极简交付与单一事实源（Single Source of Truth）**：
   `deploy/01-agenttemplate.yaml` 仅需挂载 `kagent-guardian`，平台运维团队只需维护一个权威的 `SKILL.md`，任何架构演进在一个文件中即可完成原子更新；
4. **Token 开销高度可控且缓存友好**：
   结构紧凑的高密度 Skill 文件仅需约 2,000 Tokens。在现代大模型普遍支持 128k+ 上下文的背景下，2,000 Tokens 仅占上下文容量的不到 1.5%，且能被 Prefix Cache（前缀缓存）高效命中，首 Token 推理成本几乎可以忽略不计。

---

## 四、两方案全维度工程对比矩阵 (Tradeoff Evaluation Matrix)

| 评价维度 | 方案 A：细粒度多 Skill 拆分 | 方案 B：单一高内聚 Skill 模块化 (推荐) | 深度胜出理由与场景考量 |
| :--- | :---: | :---: | :--- |
| **端到端流程连贯性** | 差 (跨 Skill 上下文易丢失) | **极优 (全程零断点、证据链完整)** | 运维任务是“巡检-归因-修复”不可分割的连续线，单 Skill 保证证据 100% 传导 |
| **首 Token 推理时延 (TTFT)** | 差 (需要元推理选择 Skill) | **优 (直接进入阶段流水线)** | 避免多轮元调度，紧急排障时节约 10~20 秒黄金时间 |
| **证据与参数保真度** | 中 (依赖长历史重构，易漂移) | **极高 (同一认知空间内精算)** | CFS 调优参数（2500m / 30s）由判定算法直接注入 PR 模板 |
| **HITL 审批状态机协同** | 极难 (拦截与轮询逻辑割裂) | **天然支持 (同一 SOP 内发起并轮询)** | 审批拦截后顺畅调用 `agentstage_approval_result` 轮询并驱动后续 CI/CD |
| **Token 消耗与缓存效率** | 理论单次少，但交互轮次增多 | **单次高密度，极易命中 Prefix Cache** | 2,000 Tokens 极小，且在会话生命周期内缓存稳定，综合更便宜 |
| **Kagent CRD 适配度** | 复杂 (`AgentTemplate` 挂载多项) | **极简 (单项引用，零配置漂移)** | 符合云原生声明式简洁原则，与现有部署清单无缝吻合 |
| **团队维护与版本迭代** | 高 (多文件同步、可能出现版本错配)| **低 (单一文件、原子提交、语义化版本)**| 一个 PR 即可完成从门禁、工具到报告模板的全部升级 |
| **意图误判率 (Hallucination)** | 偏高 (可能在排障中途误跳技能) | **极低 (严谨 Phase 门禁步进控制)** | 阶段门禁确保“基础未过不测合成，证据不足不提自愈” |


---

## 五、Skill v2.0 重构蓝图与实操规范 (Skill v2.0 Blueprint)

根据上述决议，`skills/kagent-guardian/SKILL.md` 将升级至 **v2.0.0**。本次升级将注入以下 6 项关键能力与重构要素：

### 5.1 门禁升级：引入 AgentStage 网关 PEP 与 HITL 拦截协议
- **No Direct Hot-Patching**：严禁在生产集群直接执行写操作，所有变更高风险工具均受网关 PEP 拦截；
- **Fail-Closed 故障兜底**：任何未注册工具调用或策略评估异常均静默拒绝；
- **HITL Intercept Protocol**：Agent 当且仅当调用写工具（如自愈 PR 合并、重启 Pod）收到 `status: PENDING_APPROVAL` 时，必须提取 `approval_id` 与 `review_url` 打印给 SRE，并进入有限退避轮询（每 10 秒调用一次 `agentstage_approval_result`，上限 18 次 / 3 分钟）。

### 5.2 接口换装：全面废除裸 kubectl，适配 AgentStage 代理工具组
全面废止命令式 `kubectl` 描述，替换为标准化 MCP 工具集：
- **`mcp-proxy-kagent-tool`**：
  - 指标透视：`prometheus_promql_tool(query="...")`、`prometheus_query_range_tool(query="...", start="...", end="...", step="...")`；
  - 资源状态只读：`k8s_get_resources`、`k8s_describe_resource`、`k8s_get_pod_logs(limit_bytes<=1048576, tail_lines<=200)`；
  - 事件关联：`k8s_get_events`；
- **`mcp-proxy-github-ops`**：
  - PR 闭环：`create_pull_request`、`merge_pull_request`（触发 HITL）；
  - 告警 Issue：`create_issue`、`list_issues`（基于 Title Hash 防重）；
  - 内容操作：`get_file_contents`、`create_or_update_file`；
- **`mcp-proxy-k8sgpt-read`**：`k8sgpt_analyze` 提取 AI 初筛异常；
- **`kagent-api`**：`invoke_session`、`list_sessions` 执行 L4 合成探针。

### 5.3 混合执行引擎与阶梯短路感知
Skill 明确赋予 Agent 对两套执行入口的自适应能力：
1. **Sentinel 极速打基线承接**：当 Agent 由 CronJob 唤醒并附带 Sentinel 检查结果（GREEN 或已打分）时，若全绿则直接生成日终账本摘要，跳过耗时探针；若被短路熔断（Score < 85），立即深入执行 L3/L4 穿透；
2. **交互式排障入口**：当人类 SRE 在 Web/CLI 发起即时排障时，优先执行“快速查验模式（Fast Inquiry）”，一轮输出诊断摘要；仅当被明确授权或存在严重故障时进入“深度 RCA + 自愈 PR 流程”。

### 5.4 纯 GitHub 运营闭环 SOP 规范化
1. **日终巡检账本 PR（Daily Ledger PR）**：
   - 每日 23:55 生成 `docs/patrol-ledgers/YYYY-MM/patrol-YYYY-MM-DD.md`；
   - PR 标题规范：`[Daily-Patrol] YYYY-MM-DD Health Ledger & Stability Sign-off`；
   - 包含 L1-L4 24小时综合得分走势、异常事件统计与次日维护建议；
2. **Title Hash 幂等 Issue 告警**：
   - 格式：`[ALERT][${Severity}][${Component}] ${Summary} #${TitleHash}`；
   - 发起前调用 `list_issues(state="open")` 检查是否存在同 Hash，存在则追加 Comment，禁止刷屏。

---

## 六、分阶段演进路径 (Evolutionary Roadmap)

```
[阶段一 (当前实施)] ───────────────> [阶段二 (未来规划)]
单一高内聚领域 Skill (v2.0)            跨集群 / FinOps 多智能体集群 (Swarm)
- 维持单个权威 SKILL.md                 - Guardian 演化为主控协调者 (Orchestrator)
- 模块化内部流水线                      - 引入 kagent-finops, kagent-chaos 独立智能体
- 零调度开销，证据链 100% 完整           - 基于 A2A 协议进行智能体间通信
```

1. **阶段一（当前落地期）：单一高内聚领域 Skill 升级（v2.0.0）**
   - 保持 `skills/kagent-guardian/SKILL.md` 唯一定位；
   - 彻底重构并引入 PromQL、网关 PEP、HITL 审批流、纯 GitHub 运营及短路逻辑；
   - 同步更新 `deploy/00-prompt-configmap.yaml` 与 `deploy/01-agenttemplate.yaml`，保障全流程零断点。

2. **阶段二（未来扩展期）：跨域多智能体协同（Multi-Agent Swarm）**
   - 只有当平台治理范围跨出 Kagent 本身，横向拓展到跨集群多联邦治理、异构多云 FinOps 成本优化、混沌工程（Chaos Mesh）故障注入演练等**异构非重叠领域**时，才适宜按智能体边界拆分。
   - 届时不是拆分单个 Guardian 的 Skill，而是通过 Kagent A2A 协议由 Guardian 协同调用专职的 FinOps Agent 或 Chaos Agent。

---

## 七、总结与后续行动项 (Summary & Next Steps)

### 7.1 核心决议
- **无需拆分多个 Skill，坚定采用单一高内聚、模块化流水线的 Skill 架构**；
- **立即将 `skills/kagent-guardian/SKILL.md` 从 v1.0.0 升级至 v2.0.0**，全面贯彻 HLD/LLD v2.0 规范。

### 7.2 后续行动项
1. **更新 Skill 文件**：编辑 `skills/kagent-guardian/SKILL.md`，注入 v2.0 规范；
2. **同步 Prompt ConfigMap**：更新 `deploy/00-prompt-configmap.yaml` 中的 Agent 系统提示词与 HITL 协议；
3. **提交与推送**：将 Discussion 14 及 Skill v2.0 提交并推送到 GitHub 远程仓库 `SDB-DevOps/KagentGuardian`。

