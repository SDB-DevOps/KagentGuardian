# Kagent 平台巡检自愈与 GitOps 闭环专家智能体 (Platform Guardian) 设计规范

## 1. 背景与目标

Kagent 作为一个运行于 Kubernetes 与 Substrate 之上的企业级多智能体协同与沙箱执行平台，其自身的高可用、低延时唤醒与稳定性是支撑上层各类业务 Agent 稳定运行的基础。

本规范设计并实现一个原生运行在 Kagent 平台内部的平台守护专家智能体：**`kagent-platform-guardian`**。
该 Agent 具备以下核心目标：
1. **深度认知平台内幕**：具备对 Substrate 运行时、AgentGateway 网关、WorkerPool 沙箱池、PostgreSQL 持久层及 A2A 协议的平台内核级认知。
2. **分级阶梯全景巡检**：定时执行从基础资源、网关连通性到真实会话端到端合成探针（Synthetic Probe）的多维体检。
3. **确定性根因推导与故障精算**：当出现异常时，基于时序指标与系统日志进行归因分析，评估调优参数。
4. **GitOps 与 ArgoCD 安全闭环**：杜绝生产环境直接热改集群，所有参数优化与故障自愈均以 GitHub Pull Request 形式提交到配置仓库，经审核/自动化合并后由 ArgoCD 驱动集群同步，并自动进行健康回访验证。

---

## 2. 架构体系：三层解耦模型 (Three-Tier Architecture)

为避免单体 Prompt 膨胀导致的上下文丢失与高 Token 消耗，系统采用严格的职责分离架构：

```
┌────────────────────────────────────────────────────────────────────────┐
│ 1. AgentTemplate (声明与装配层 - Spec Layer)                           │
│    - 声明元数据 (Name, Labels, Description)                            │
│    - 挂载底层大模型配置 (ModelConfig)                                  │
│    - 动态引用 Prompt 库 (systemPromptFrom: kagent-guardian-prompts)    │
│    - 声明并绑定 MCP 工具集 (K8s 只读、Kagent API、GitHub、ArgoCD、监控)│
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
┌───────────────────────────────────┴────────────────────────────────────┐
│ 2. Prompts 库 (Kagent UI 可视化提示词层 - ConfigMap: kagent.dev/prompt) │
│    - 注册于 Kagent 控制台 Prompts 页面，支持在线可视化维护与版本管理   │
│    - 角色定位：Kagent 平台 SRE 专家与 GitOps 自动化守护者               │
│    - 核心铁律：严禁直接在生产集群执行破坏性命令；任何配置变更必须走 PR   │
│    - 调度指令：激活调用 Guardian Skill 对应的标准作业流水线            │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
┌───────────────────────────────────┴────────────────────────────────────┐
│ 3. Domain Skill (领域知识与作业大脑 - Skill Layer, SKILL.md)           │
│    - Kagent 运行时架构拓扑知识库 (Substrate/WorkerPool/Gateway/Postgres)│
│    - 4 级阶梯巡检标准作业规程 (SOP)                                    │
│    - 故障排查推导树 (Decision Tree: 唤醒超时、CFS 限流、连接池等)       │
│    - GitOps 与 ArgoCD 联动发版规范 (分支命名、PR 格式、Sync 状态核验)   │
│    - 标准巡检报告与 Incident Markdown 模版                             │
└────────────────────────────────────────────────────────────────────────┘

---

## 3. 工具链装配矩阵 (MCP Tools Matrix)

智能体通过绑定以下已部署在集群内的 RemoteMCPServers 获取确定性执行能力：

| 模块类别 | RemoteMCPServer 资源名 | 访问端点 (集群内 Service DNS) | 核心用途与能力 |
| :--- | :--- | :--- | :--- |
| **集群资源只读** | `kagent-tool-server` | `kagent-tools.kagent.svc:8084/mcp` | 执行 `kubectl get`, `describe`, `logs`，只读获取集群对象状态 |
| **Kagent 控制面** | `kagent-api` | `kagent-controller.kagent.svc:8083/mcp` | 调用 `list_sessions`, `invoke_session`, `get_sandbox` 执行 E2E 合成探针 |
| **AI 异常初筛** | `k8sgpt-mcp-server` | `k8sgpt-mcp-server.kagent.svc:8089/mcp` | 自动化获取 K8s 集群 Events 异常分析与 AI 建议 |
| **代码与配置提交** | `github-mcp-server` | `github-mcp-server.kagent.svc:9090/mcp` | 读取仓库源码、创建代码分支、推送巡检报告、提交 Pull Request |
| **GitOps 调谐核验**| `kagent-tool-server` | 同上 | 检查 ArgoCD `Application` CRD 的 `status.sync` 与 `status.health` |

---

## 4. 4 级阶梯巡检矩阵与指标基准

```
[L1 基础设施层] ──> [L2 系统运行时] ──> [L3 控制面与持久化] ──> [L4 E2E 合成探针]
```

### 4.1 L1：宿主机与基础资源水位 (Node & Resources)
- **Node Ready 状态**：所有节点状态必须为 `Ready`。
- **CPU Allocatable & Request**：宿主机 CPU 分配率 $\le 80\%$；单节点剩余空闲 CPU 算力 $\ge 500\text{m}$。
- **Memory Pressure**：内存分配率 $\le 75\%$，检查是否存在近期 `OOMKilled` 事件。

### 4.2 L2：系统服务与运行时拓扑 (Substrate & Network)
- **流量网关 (`atenet-router` in `ate-system`)**：
  - 检查 Pod 状态是否为 `1/1 Running`，无异常 Crash 重启。
  - 验证 `requestParking.budget` 是否保持在安全阈值（$\ge 15\text{s}$）。
- **计算工作池 (`kagent-default` in `kagent`)**：
  - 验证 Worker Pod 处于 `Running` 状态，探针探活正常。
  - 检查 CPU Request 是否达到基线（$\ge 1000\text{m}$），确保 CFS 时间片调度充裕。

### 4.3 L3：平台控制面与持久层 (Controller & DB)
- **控制器 (`kagent-controller`)**：
  - 检查近 10 分钟内是否有持续输出的 `ERROR` / `FATAL` 堆栈。
  - 检查 CRD 调谐队列延迟与未决任务数。
- **数据库 (`kagent-postgresql`)**：
  - 连接健康度与死锁检测。
  - 检查 `kagent.session` 表最近写入与状态更新频次。

### 4.4 L4：端到端业务合成探针 (E2E Synthetic Probe)
- **探针动作**：通过 `kagent-api` 向测试 Agent 发起一次轻量会话唤醒测试（"ping" / "healthcheck"）。
- **指标基线**：
  - **冷启动拉起延时**：$\le 3.5\text{s}$（超过 $5\text{s}$ 判定为性能亚健康警告）。
  - **网关响应状态**：HTTP 200 / 成功返回 A2A 消息，无 503 / "input was not accepted"。

---

## 5. GitOps 与 ArgoCD 闭环自动化作业流

当 Agent 在巡检或诊断过程中发现确需调整的平台配置时，必须严格执行以下 **GitOps 闭环状态机**：

```
                ┌───────────────────────────────┐
                │ 发现平台配置亚健康 / 性能瓶颈 │
                └───────────────┬───────────────┘
                                │
                                ▼
                ┌───────────────────────────────┐
                │ 步骤 1: 生成 Git 分支与改动   │
                │ branch: fix/kagent-tune-xxxx  │
                │ file: manifests/...           │
                └───────────────┬───────────────┘
                                │
                                ▼
                ┌───────────────────────────────┐
                │ 步骤 2: 提交 GitHub PR        │
                │ 包含: 现象、推导证据、Diff    │
                └───────────────┬───────────────┘
                                │
                                ▼
                ┌───────────────────────────────┐
                │ 步骤 3: 人类审批 / CI 合并     │
                │ Merge commit 进入 main 分支   │
                └───────────────┬───────────────┘
                                │
                                ▼
                ┌───────────────────────────────┐
                │ 步骤 4: ArgoCD 自动同步调谐   │
                │ Application 变为 Synced       │
                └───────────────┬───────────────┘
                                │
                                ▼
                ┌───────────────────────────────┐
                │ 步骤 5: Guardian 回访复测验证 │
                │ 确认健康度指标达标，PR 结单   │
                └───────────────────────────────┘
```

---

## 6. 安全防护栏铁律 (Safety Guardrails)

1. **绝对禁止生产直接修改**：严禁在未经过 Git 提交与 PR 审核的情况下，直接使用 MCP 对线上生产 Deployment、ConfigMap、Secret 执行 `kubectl apply / edit / delete`。
2. **轻量只读优先**：日常巡检只使用只读查询指令（`kubectl get`, `kubectl describe`, `kubectl logs --tail`）。
3. **合成测试隔离**：L4 端到端合成探针必须使用专用的独立轻量 Session，并在测试完成后主动回收沙箱资源。
4. **敏感信息脱敏**：上传至 GitHub 的报告中，严禁包含明文 Token、数据库密码或私有密钥。

```
