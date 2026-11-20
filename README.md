# Kagent Platform Guardian (Kagent 平台守护与 GitOps 自愈专家)

[![Repository](https://img.shields.io/badge/GitHub-SDB--DevOps%2FKagentGuardian-blue.svg)](https://github.com/SDB-DevOps/KagentGuardian)
[![Platform](https://img.shields.io/badge/Platform-Kagent%20v1alpha3-purple.svg)](https://github.com/kagent-dev/kagent)
[![GitOps](https://img.shields.io/badge/GitOps-ArgoCD%20Ready-orange.svg)](https://argoproj.github.io/)

`KagentPlatformGuardian` 是专为 [Kagent](https://github.com/kagent-dev/kagent) 企业级多智能体平台打造的原生平台 SRE 守护者。
它具备对 Substrate 运行时、AgentGateway 流量网关、WorkerPool 计算池、PostgreSQL 持久层及 A2A 协议的平台内核级认知，执行 4 级阶梯式健康巡检，基于时序指标与日志推导根本原因，并通过 **GitHub Pull Request + ArgoCD** 实施声明式安全自愈，坚决杜绝生产环境直接热修。

---

## 目录结构 (Repository Structure)

```text
KagentGuardian/
├── README.md                          # 仓库全景导览与运维规范
├── deploy/                            # Kubernetes 生产声明清单
│   ├── 00-prompt-configmap.yaml       # Kagent Web UI 挂载的 Prompts 库 ConfigMap
│   ├── 01-agenttemplate.yaml          # AgentTemplate 智能体装配规约清单
│   └── 02-agent.yaml                  # Agent 实例运行清单 (挂载 agentstage-harness)
├── skills/                            # Guardian 专业领域 Skill 规约
│   └── kagent-guardian/
│       └── SKILL.md                   # 4 级巡检 SOP、故障决策树与报告模板
├── docs/
│   ├── requirements/                  # 客户运维需求与痛点深度分析
│   │   └── 01-CUSTOMER_PAIN_POINTS_AND_REQUIREMENTS_ANALYSIS.md # 8 大生产痛点剖析与心智转化矩阵
│   ├── design/                        # 系统架构与规范设计
│   │   ├── 01-PLATFORM_GUARDIAN_GITOPS_DESIGN.md # 初始架构探索设计与 GitOps 状态机规范 (v1.0 基石)
│   │   ├── 02-OPERATIONS_GUARDIAN_HIGH_LEVEL_DESIGN.md # 总体高阶设计规范 (v2.0)：七大原则、六大宏观演进、七大模块与八大架构禁区
│   │   └── 03-OPERATIONS_GUARDIAN_LOW_LEVEL_DESIGN.md # 详细工程设计规范 (v2.0)：追溯矩阵、混合短路引擎、AgentStage MCP 网关 PEP、HITL 审批流、纯 GitHub 运营闭环与调度矩阵
│   ├── knowledge-base/                # SRE 专业知识库与排障 Runbooks
│   │   ├── 01-KAGENT_ARCHITECTURE_INTERNALS.md         # 控制面 CRD 与持久层拓扑
│   │   ├── 02-SUBSTRATE_RUNTIME_AND_ACTOR_LIFECYCLE.md # Substrate 沙箱与 CFS 算力机制
│   │   └── 03-TROUBLESHOOTING_RUNBOOKS.md              # 经典故障诊断决策树与自愈手册
│   └── discussions/                   # 关键技术专题讨论与决策记录
│       ├── 01-PROMPT_SEPARATION_AND_UI_INTEGRATION.md  # 讨论一：提示词解耦与 Web UI Prompts 集成
│       ├── 02-REPO_VS_KNOWLEDGE_BASE_TRADEOFFS.md      # 讨论二：源码全量塞入 vs 独立知识库辨析
│       ├── 03-SESSION_TIMEOUT_503_OPTIMIZATION.md      # 讨论三：会话 503 超时与 CFS 算力优化
│       ├── 04-CURRENT_PROGRESS_INSPECTION_AND_GITOPS_STATUS.md # 讨论四：Guardian 建设进度、巡检与 GitOps 现状评估
│       ├── 05-CUSTOMER_CENTRIC_REQUIREMENTS_AND_SRE_BLUEPRINT.md # 讨论五：客户运维痛点剥离与全栈 SRE 蓝图 (Prompt/CRD/MCP/Skills)
│       ├── 06-DESIGN_COMPARISON_GAP_ANALYSIS_AND_EVOLUTION.md # 讨论六：Doc 01 与 Doc 03 逐项对比审查、缺陷剖析与演进建议
│       ├── 07-DESIGN_DOCUMENTATION_REVISION_PLAN_AND_RATIONALE.md # 讨论七：设计文档修编规划与推演过程记录 (逐项整改考量与落地路线图)
│       ├── 08-GUARDIAN_PRODUCTION_IMPLEMENTATION_PLAN_AND_GAP_ANALYSIS.md # 讨论八：生产落地实施方案 (线上资产对比、增删改清单、依赖拓扑与实施动因)
│       ├── 09-SCHEDULED_INSPECTION_TRIGGER_AND_NOTIFICATION_ARCHITECTURE.md # 讨论九：定期巡检触发机制、定点通知、GitHub Issue自动建单与PR记录辨析
│       ├── 10-INSPECTION_EXECUTION_ENGINE_AGENT_ROLE_AND_TIERED_SHORT_CIRCUIT_LOGIC.md # 讨论十：L1-L4巡检执行内核设计：脚本与Agent角色定位、阶梯依赖拓扑与熔断短路机制
│       ├── 11-INTERACTIVE_DIAGNOSIS_KNOWLEDGE_FLYWHEEL_AND_SHIFT_LEFT_EVOLUTION.md # 讨论十一：交互式深度排障机制、知识沉淀飞轮与常态巡检左移体系
│       ├── 12-MCP_TOOLSET_COVERAGE_GAP_ANALYSIS_AND_SAFETY_PERMISSIONS_BOUNDARY.md # 讨论十二：MCP工具集场景覆盖度评估、Agent权限边界与防失误安全控制体系
│       └── 13-INTEGRATING_AGENTSTAGE_MCP_PROXY_GATEWAY_POLICY_AND_HITL_GOVERNANCE.md # 讨论十三：融入同事研发的 AgentStage MCP Proxy 与 Policy 架构：网关鉴权机制、HITL 审批流与安全融入改造方案
│   └── test/                          # 智能体测试计划与实测用例集
│       ├── 01-MANUAL_TESTING_PLAN_AND_CASES.md         # Web UI 手动验证测试计划与 6 大精选用例
│       └── 02-AUTOMATED_TEST_EXECUTION_REPORT.md       # 自动化测试执行与质量验收报告 (6/6 全部通过)


```

---

## 核心设计：三层解耦模型 (Three-Tier Architecture)

1. **AgentTemplate (声明与装配层)**：
   - 挂载模型配置 (`modelConfig: default-model-config`)。
   - 动态引用 Web UI 提示词库 (`systemPromptFrom: kagent-guardian-prompts`)。
   - 挂载 4 大受控工具组（通过 AgentStage MCP Gateway PEP 代理）：`mcp-proxy-kagent-tool`（127 工具全量，含 PromQL 指标与 K8s 诊断）、`mcp-proxy-github-ops`（分支/PR/Issue 流水线）、`mcp-proxy-k8sgpt-read`（AI 初筛）、`kagent-api`（瞬态 L4 合成探针）。
2. **Prompts 页面库 (人设与安全红线层)**：
   - 在 Kagent Web 控制台 `Prompts` 页面集中纳管与在线润色。
   - 明确常驻 SRE 人设与**生产严禁热改（No Hot-Patching Gate）**、**网关 HITL 审批拦截与轮询协议**。
3. **知识库与 Domain Skill (知识大脑层)**：
   - 沉淀平台内核内幕与四级阶梯巡检流水线（L1 基础资源 -> L2 运行时与网关 -> L3 控制面与持久化 -> L4 端到端合成探针）。
   - 固化故障诊断决策树与标准报告模板。

---

## GitOps 闭环状态机

```text
[每日 09:00 / 18:00 定点巡检 触发 Agent]
            │
            ▼
[巡检发现指标异常 / 潜在隐患 (健康分 < 90)]
            │
            ▼
[自动在 GitHub 创建 Issue 跟踪工单 (幂等查重防刷屏)]
            │
            ▼
[判定是否触发自动自愈：抢占排他锁，提 PR (Fixes #Issue)]
            │
            ▼
[人类审查 / CI 自动 Merge PR]
            │
            ▼
[ArgoCD 驱动集群完成声明式调谐 (Synced & Healthy)]
            │
            ▼
[GitHub 联动自动关闭关联 Issue ──> L4 探针复测结单]
```

> 📌 **企业零外部依赖实践（Pure GitHub-Centric Operations）**：针对企业没有钉钉/飞书、且 Slack/Teams 受限无 Webhook 权限的现实场景，系统完全以 GitHub 为单一运营中枢：
> - **日常绿色巡检**：日终生成签收 PR（`[Daily-Patrol] YYYY-MM-DD Ledger`），SRE 点击 Merge 完成运维日结，合规审计 100% 留痕；
> - **异常预警与工单**：自动创建 GitHub Issue 并 `@SDB-DevOps/sre-team`，通过 GitHub Mobile App 原生推送与企业邮件秒级直达工程师；
> - **自愈与变更**：自愈 PR 显式绑定 `Fixes #<Issue-ID>`，审查批准后自动关闭 Issue，无需离开 GitHub 即可完成 SRE 闭环。

---

## 协同规约与后续演进

1. **配置与知识库同步**：后续所有的 AgentTemplate 调优、Prompt 更新、知识库新增均统一沉淀并提交至本仓库。
2. **改动与发版**：后续对 Guardian Agent 的功能扩充与参数微调一律遵循 Pull Request 规范。
3. **问题追踪**：针对巡检发现的深层 Bug 或需求演进，统一在 [GitHub Issues](https://github.com/SDB-DevOps/KagentGuardian/issues) 中建单跟踪。
