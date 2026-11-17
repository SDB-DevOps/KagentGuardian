# 讨论十三：融入同事研发的 AgentStage MCP Proxy 与 Policy 治理体系深度解析

## 1. 核心背景与技术诉求

在《讨论十二》中，我们深入剖析了 Kagent Guardian 面向生产落地时的 MCP 工具覆盖盲区、权限边界与防大模型幻觉/失误的安全防线。当时我们提出了“集群物理只读 RBAC、协同中枢 GitOps 受控写入”的双轨设计，但仍面临一个工业级痛点：

> **“在 Agent 与下游真实的各类 MCP Server 之间，如果大模型调用工具是‘直连’的，一旦提示词越狱或大模型出现幻觉去调用高危写操作、或者传入了跨租户跨集群的越权参数，如何做到在协议链路层的物理拦截、参数级强约束与人工审批（Human-in-the-Loop）？”**

现同事已在集群中上线了 **AgentStage MCP Proxy**（`agentstage-mcp-gateway`）与 **MCP Policy** 控制中枢（Web UI 访问入口为 `http://34.69.136.108:30083/governance/mcp-policy`）。

本篇专题讨论旨在深度剖析同事的新系统源码与运行架构，明确回答**该设计能否切实起到权限控制作用**，并给出 **Kagent Guardian 融入该架构的完整落地方案**。

---

## 2. 同事 MCP Proxy 与 MCP Policy 架构深入剖析

通过对集群中实际运行的 `agentstage-mcp-gateway` 容器（镜像 `ghcr.io/sdb-devops/agentstage-mcp-gateway:sha-73027fd`）及源码 `/app/src/mcp_proxy/` 的全面探查，同事实现的这套系统是一套高度完备、符合工业级零信任架构的 **MCP 策略执行点（Policy Enforcement Point, PEP）与反向代理网关**。

### 2.1 整体架构与三层模型（Three-Level Model）

系统将 MCP 的访问控制抽象为清晰的三层模型：

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        MCP Policy 三层模型                             │
├─────────────────┬──────────────────────────────────────────────────────┤
│ 1. Upstreams    │ 真实下游 MCP 服务的物理连接定义（URL, Transport,     │
│   (物理上游)    │ 凭据托管 Credential）。不持有工具白名单与业务语义。 │
├─────────────────┼──────────────────────────────────────────────────────┤
│ 2. Tool Groups  │ 面向 Agent 暴露的虚拟逻辑服务（/mcp/{name}）。       │
│   (工具集合)    │ 在 Kubernetes 中直接注册为 RemoteMCPServer 资源。    │
├─────────────────┼──────────────────────────────────────────────────────┤
│ 3. Tool Rows    │ 工具集合内的具体工具行与参数规则（ParamPolicy）。    │
│   (规则与审批)  │ 定义每项工具的动作性质、审批档位（none/self/cross）  │
│                 │ 以及参数约束（Namespace/Repo 范围白名单）。          │
└─────────────────┴──────────────────────────────────────────────────────┘
```

```text
  ┌───────────────────────────┐
  │   Kagent Guardian Agent   │
  └─────────────┬─────────────┘
                │ JSON-RPC (tools/call)
                ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │              agentstage-mcp-gateway (Port 3000)                     │
  │                                                                     │
  │  [链路 1: 限流与熔断] RateLimiter (10.0 rps) + CircuitBreaker (5次) │
  │  [链路 2: 路由匹配]   ToolRouter (前缀最长匹配 -> ServerConfig)     │
  │  [链路 3: 策略引擎]   PolicyEngine.check_tool()                     │
  │                       ├─ Server 白名单校验                          │
  │                       ├─ Tool 白名单校验 (Fail-Closed)              │
  │                       ├─ 参数规则 ParamPolicy (owner/repo/ns 约束)  │
  │                       └─ 审批档位判定 (none / self / cross)         │
  │  [链路 4: HITL 审批]  PostgreSQL (proxy_approvals, HMAC-SHA256 签名)│
  │                       └─ 内联等待 (wait_seconds=60s) / 挂起单号     │
  │  [链路 5: 凭证注入]   CredentialManager (自动注入安全 PAT, 模型无感知)│
  │  [链路 6: L3 审计]    AuditLogger + Spool (投递至 audit-service)     │
  └─────────────┬───────────────────────────┬───────────────────────────┘
                │ 透传/执行                 │ 透传/执行
                ▼                           ▼
  ┌───────────────────────────┐ ┌───────────────────────────┐
  │  kagent-tools (8084)      │ │  github-mcp-server (9090) │
  │  (Kubernetes / PromQL)    │ │  (GitHub 官方协议服务)    │
  └───────────────────────────┘ └───────────────────────────┘
```

### 2.2 核心代码实现机理

探查到的关键技术组件包括：

1. **确定性本地策略判定（`core/policy.py`）**：
   - 彻底摒弃了远程逐次 RPC 鉴权的性能损耗，在网关进程内存中完成毫秒级判定；
   - 实行严格的 **Fail-Closed（默认拒绝）**：不在 `tools` 列表中的工具一律报 `TOOL_NOT_ALLOWED`；
   - 任何标记为 `action: write` 的写工具，如果未显式声明审批档位，网关直接拒绝执行（`HITL_REQUIRED`）。
2. **防篡改持久化 HITL 审批（`core/approval.py`）**：
   - 当调用命中需要审批的工具时，网关在 PostgreSQL 的 `proxy_approvals` 表中生成审批单；
   - 采用 `APPROVAL_SIGNING_KEY` 对不可变字段计算 **HMAC-SHA256 签名**，杜绝任何外部数据库直连篡改；
   - 计算参数摘要 `params_digest`，如果审批与最终执行时的参数发生漂移，立即以 `APPROVAL_PARAM_DRIFT` 拒绝执行。
3. **参数硬边界扫描门禁（`policy/coverage.py`）**：
   - 截图 2 中显示的 `210 unconstrained resource parameter(s) in 9 tool group(s)` 告警，来自于该模块；
   - 扫描 `HARD_DIMENSIONS`（`owner`、`repo`、`namespace`、`cluster`、`app`）；
   - 如果某个工具接受 `namespace` 或 `repo` 参数但未在 `paramPolicy` 中配置范围校验规则，网关发出警告（`MCP_COVERAGE_ENFORCE=warn`）或直接阻断（`block`），从根本上杜绝越权访问其他集群租户。
4. **内置审批结果拉取工具（`agentstage_approval_result`）**：
   - 在 `server/handlers.py` 中，网关作为原生只读工具向上暴露了 `agentstage_approval_result`；
   - 解决了一个关键痛点：当写操作进入审批流程后，HTTP 会话不会无限挂死；Agent 可以主动调用该工具，传入 `approval_id` 等待并查询审批是否完成以及真实返回的 Pod 执行结果。

---

## 3. 该设计能起到作用吗？能否控制大模型权限？

**结论明确：不仅能起到作用，而且这套架构彻底解决了生产级 AI Agent 权限控制的最核心难题，达到了工业级零信任（Zero Trust）与深度防御（Defense-in-Depth）的标准！**

### 3.1 为什么传统的方案做不到，而 MCP Proxy 能做到？

| 管控维度 | 传统 K8s RBAC / GitHub Token | 同事的 AgentStage MCP Proxy 体系 |
| :--- | :--- | :--- |
| **颗粒度** | **粗粒度全有或全无**：给写权限就能写所有对象，给只读就完全不能动 | **细粒度工具切片**：同一套底层 API，可按只读（Read）、日常维护（Ops）、高危管理（Admin）划分出不同 Tool Groups |
| **参数边界** | **无法感知上下文**：无法限制命令里具体执行的是什么 shell 参数 | **参数级校验（ParamPolicy）**：强制校验 `namespace` 必须在限定集合内，`repo` 必须在指定组织下 |
| **高危拦截** | **即调即执行**：模型一旦发错指令，下游立即执行，造成毁灭性破坏 | **状态机拦截与审批（HITL）**：对 `k8s_delete_resource`、`shell`、`merge_pull_request` 自动拦截转审批单，阻断即时破坏 |
| **凭据安全** | **凭据暴露给 Agent**：PAT 或 Token 注入 Agent 容器，存在 Prompt 越狱窃取风险 | **凭据物理隔离（Credential Isolation）**：PAT/Token 仅在网关层内存中组装，Agent 完全无感知，杜绝泄漏 |
| **可追溯性** | **分散的系统日志**：审计散落在 K8s Audit 和 GitHub Audit 中，无法对应 Prompt | **全链路 Prompt-to-Action 审计**：记录完整会话 ID、ULID、参数摘要与审批人签名，投递至统一审计服务 |

---

## 4. Kagent Guardian 融入该架构的完整技术方案

目前集群中已经预置并就绪了 14 个由 MCP Proxy 托管的 `RemoteMCPServer`（位于 `kagent` 命名空间）：
- `mcp-proxy-kagent-tool`：官方全量 127 个工具（集成 K8s 操作、Prometheus PromQL 探针、Cilium、Istio、Helm 等，高危操作挂接交叉审批）；
- `mcp-proxy-k8s-read`：严格的 9 个只读 Kubernetes 探针；
- `mcp-proxy-github-ops`：GitHub 日常读写集合（放行 Issue/PR/分支/文件修改，拦截 Merge 与 Delete）；
- `mcp-proxy-github-read`：GitHub 纯只读集合；
- `mcp-proxy-k8sgpt-read`：K8sGPT 纯只读初筛；
- `mcp-proxy-log-governor`：日志检索与治理。

为将 Kagent Guardian 全面融入该体系，我们实施以下四步演进改造：

### 4.1 第一步：修改 AgentTemplate 工具挂载清单

将此前直连未受管控的 MCP 替换为受 Proxy 策略管控的 `RemoteMCPServer`：

```yaml
# deploy/01-agenttemplate.yaml (改造后)
apiVersion: api.kagent.dev/v1alpha3
kind: AgentTemplate
metadata:
  name: kagent-guardian-template
  namespace: kagent
  labels:
    app: kagent-guardian-agent
    capability: platform-sre
    role: worker
spec:
  description: "Kagent 平台巡检自愈与 GitOps 闭环守护专家 (已接入 AgentStage MCP Proxy 策略管控)"
  modelConfig:
    name: default-model-config
  systemPromptFrom:
    name: kagent-guardian-prompts
    key: kagent-guardian-prompt
  tools:
  # 1. Kubernetes 核心监控与探针：接入受控的官方全量工具集（带 PromQL 时序查询能力，破坏性动作受审批保护）
  - mcp:
      server:
        kind: RemoteMCPServer
        name: mcp-proxy-kagent-tool
  # 2. AI 故障聚类初筛：只读
  - mcp:
      server:
        kind: RemoteMCPServer
        name: mcp-proxy-k8sgpt-read
  # 3. GitHub 运维操作：支持开 Issue、建自愈分支、提交 PR，拦截 Merge
  - mcp:
      server:
        kind: RemoteMCPServer
        name: mcp-proxy-github-ops
  # 4. L4 端到端黑盒合成会话探针
  - mcp:
      server:
        kind: RemoteMCPServer
        name: kagent-api
```

> **显著收益**：
> 1. 原先在《讨论十二》中提及的“缺乏 Prometheus PromQL 时序查询工具”的盲区直接被消除！因为 `mcp-proxy-kagent-tool` 中完整包含了 `prometheus_promql_tool`、`prometheus_query_range_tool`。
> 2. 原先直连的 GitHub PAT 从 Agent 中剥离，统一由网关托管。

### 4.2 第二步：Prompt 提示词适配网关 HITL 协同协议

当 Agent 触发了需要审批的动作（如提议并触发打 Tag、高危自愈操作），网关会返回拦截信息并生成审批单。我们需要在 `deploy/00-prompt-configmap.yaml` 中为 Guardian 补充与网关交互的规范协议：

```markdown
### MCP Gateway 与人机协同审批（HITL）交互协议：
1. **写操作受控**：你对生产集群和 GitHub 仓库的写操作受 AgentStage MCP Gateway 严格审计与策略保护；
2. **审批单感知**：当你执行某些敏感操作（如 delete、merge 或跨命名空间操作）时，网关可能会返回挂起响应：
   - 响应格式包含 `approval_id: ap-xxxxxxxxxxxx` 与提示 "write tool requires approval"。
3. **处理规范**：
   - 严禁反复重试被拦截的指令；
   - 立即记录 `approval_id`，在巡检报告/Issue 中注明：“【待人工审批】操作已进入审批流水线，审批单号: ap-xxxxxxxx，请管理员登录 AgentStage 审批中心 (http://34.69.136.108:30083/governance/approvals) 进行核准”；
   - 若当前任务需要等待审批结果，可调用网关内置工具 `agentstage_approval_result(approval_id="ap-xxxxxxxx", wait_seconds=60)` 轮询核准状态。
```

### 4.3 第三步：消除 210 个未受约束参数告警（ParamPolicy 闭环）

针对截图 2 中黄色的警告：
`210 unconstrained resource parameter(s) in 9 tool group(s)`，这是因为部分工具的 `namespace` 和 `repo` 参数未配置显式的 `paramPolicy` 校验规则。

为了提升集群整体的覆盖度评分并将网关的 `MCP_COVERAGE_ENFORCE` 从 `warn` 提升至 `block`（零信任强制阻断），我们为 Guardian 关联的 Tool Group 配置参数规则：

```yaml
# 在 mcp-proxy 策略数据库或 policy.seed 中注入 Guardian 的专属参数白名单：
paramPolicy:
  # 约束 GitHub 工具的仓库边界
  - tools: [create_pull_request, create_branch, create_or_update_file, push_files]
    rules:
      - field: owner
        operator: in
        values: ["SDB-DevOps"]
      - field: repo
        operator: in
        values: ["KagentGuardian", "AgentStage", "substrate", "agentstage-infra"]
  # 约束 Kubernetes 工具的命名空间边界
  - tools: [k8s_describe_resource, k8s_get_resources, k8s_get_pod_logs, k8s_get_events]
    rules:
      - field: namespace
        operator: in
        values: ["kagent", "substrate", "default", "logging"]
```
一旦完成上述参数边界的收口，Agent 将被彻底约束在指定的组织、仓库和 Kubernetes 命名空间内，就算模型被越狱欺骗去操作其他租户的资源，网关层就会直接在本地以 `PARAM_VIOLATION` 阻断。

---

## 5. 总结与推进建议

| 对比维度 | 原直连方案 (Direct MCP) | 融入 MCP Proxy 方案 (Gateway + Policy) |
| :--- | :--- | :--- |
| **鉴权点** | 终端服务单点判定 | 网关层统一集中判定 (PEP) |
| **模型安全防线** | 仅靠 Prompt 约束（易失误、易幻觉） | **网关物理拦截 + 强参数校验 + HMAC 防篡改审批单** |
| **人机协同** | 无法做到中间拦截审批 | **原生支持 UI 一键审批（`/governance/approvals`）** |
| **运维体验** | 散落各处，缺乏全局视图 | **在 `/governance/mcp-policy` 实时可视化管控与热加载** |

**建议立即落地的步骤：**
1. 更新 `deploy/01-agenttemplate.yaml`，将 Guardian 的工具定义切换为 `mcp-proxy-kagent-tool`、`mcp-proxy-github-ops` 与 `mcp-proxy-k8sgpt-read`；
2. 更新 Prompt ConfigMap 引入 `approval_id` 与审批流处理协议；
3. 将本次架构评估与融入设计同步推送到远程仓库。

