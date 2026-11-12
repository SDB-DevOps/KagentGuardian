# Kagent 平台底层架构与控制面内核白皮书 (Architecture Internals)

## 1. 核心控制面架构 (Control Plane)
Kagent 控制面由 `kagent-controller`（基于 Go 与 controller-runtime 编写）主导，管理以下 5 大核心 CRD：

### 1.1 核心 CRD 拓扑
- **`Agent`** (`api.kagent.dev/v1alpha3`):
  - 智能体定义实体，声明绑定的 `agentTemplate` 与 `agentStage`。
  - Status 经历 `Accepted` -> `EgressPoliciesApplied` -> `ResolvedRefs` -> `Compatible` -> `GoldenSnapshotReady` -> `Ready`。
- **`AgentTemplate`** (`api.kagent.dev/v1alpha3`):
  - 智能体装配规约，绑定底层大模型配置 `modelConfig`、Prompt（通过 `systemPrompt` 或 `systemPromptFrom` 引用 ConfigMap/PromptLibrary）及挂载的 MCP 工具（`RemoteMCPServer`）。
- **`AgentStage` / `Harness`**:
  - 智能体运行时沙箱环境基座（如 `agentstage-harness`），定义底座镜像与 Python/Go ADK 执行环境。
- **`RemoteMCPServer`**:
  - MCP 工具端点注册，控制器将其转化为沙箱可调用的 JSON-RPC / SSE 工具协议。
- **`ModelConfig`**:
  - 大模型网关配置（包含 Base URL, API Token, 默认采样参数与上下文窗口超参）。

---

## 2. 持久层与调度模型 (`kagent-postgresql`)
Kagent 使用 PostgreSQL 维护全局状态与多轮对话拓扑：
- `kagent.agent_definition`:
  - 记录各 Agent 的 `desired_revision` 与 `latest_successful_revision`。若两值不一致，表明控制器正在调谐或 Substrate 正在生成镜像。
- `kagent.session`:
  - 记录会话唯一 ID、归属 Agent UID、创建时间与生命周期状态。
- `kagent.task` & `kagent.message`:
  - 存储任务执行计划与 A2A（Agent-to-Agent）跨智能体交互日志。
- `kagent.checkpoint`:
  - 会话快照存储，支持对话的挂起（Suspend）、恢复（Resume）与分支分叉（Fork）。

---

## 3. 流量网关与 A2A 协议 (`atenet-router` in `ate-system`)
- 网关拦截所有入口流量并负责路由至具体的 Worker 沙箱。
- 关键参数 `requestParking.budget`:
  - 当智能体沙箱处于非活跃或冷拉起状态时，网关执行 Request Parking（请求挂载等待）。
  - 若冷启动耗时超过该阈值（如默认 5s 过于激进），网关抛出 503 错误（前端表现为红色告警 `input was not accepted; retry after the session becomes available`）。
  - 生产推荐值：保持在 $\ge 15\text{s}$。
