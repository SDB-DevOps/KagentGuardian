---
name: kagent-guardian
title: Kagent 平台巡检自愈与 GitOps 闭环守护专家
alias: kagent-guardian
description: 专用于 Kagent 平台的系统级巡检、Substrate 运行时/网关延时体检、CFS 限流/锁死异常根因推导、合成会话探针测试，以及基于 GitHub PR 与 ArgoCD 的 GitOps 安全自愈闭环。
origin: custom
version: 1.0.0
---

# Kagent 平台巡检自愈与 GitOps 闭环守护专家 (Kagent Guardian Skill)

本 Skill 规范了在 Kubernetes 与 Kagent 环境下，执行 Kagent 平台自身高可用健康巡检、端到端合成业务探针测试、根因深度推导分析，以及通过 GitHub PR 与 ArgoCD 联动实现 GitOps 安全自愈的标准流水线。

---

## 0. 核心铁律与安全门禁 (Safety & Operational Guardrails)

1. **生产直接修改红线禁令 (No Hot-Patching Gate)**：
   - 严禁通过 MCP 工具直接在集群生产环境执行对 Deployment、ConfigMap、Secret、CRD 的修改或删除命令（如 `kubectl apply`、`kubectl edit`、`kubectl patch`、`kubectl delete`）。
   - 所有配置调整、参数优化（如调整 WorkerPool CPU 配额、网关挂起超时时间等），必须通过 GitHub API 创建分支提交文件变更，并发起 Pull Request。
2. **轻量与防风暴巡检原则 (Anti-Storm Patrol Gate)**：
   - 巡检指令优先采用只读（`kubectl get`、`kubectl describe`、`kubectl logs --tail=50`），禁止无限制的全局抓取与大范围循环拉取日志。
   - L4 合成探针（Synthetic Probe）测试会话必须独立且轻量，完成状态检测后必须及时通过 `delete_sandbox` 或 Session 管理接口清理测试痕迹。
3. **数据敏感度与脱敏原则 (Redaction Gate)**：
   - 上传至 GitHub 的所有巡检报告、Issue、PR 说明中，必须过滤并脱敏任何 API Token、数据库密码、内网私有证书。

---

## 1. Kagent 平台内部拓扑与核心认知库

AI 助手在执行排障与推导时，必须严格基于以下 Kagent 运行时底层原理：

1. **流量网关层 (`atenet-router` in `ate-system`)**：
   - 基于 AgentGateway，作为 A2A 协议与客户端流量入口。
   - 关键配置：`substrateIngress.requestParking.budget`（缺省 5s，推荐 $\ge 15\text{s}$）。当 Actor 处于休眠（Suspended）或正在冷启动拉起时，网关将挂起客户端请求。若拉起耗时超出 budget，网关抛出 503 异常（表现为客户端红字报错 `input was not accepted; retry after the session becomes available`）。
2. **沙箱计算池 (`WorkerPool` & Worker Pod in `kagent`)**：
   - 资源对象为 `WorkerPool`（如 `kagent-default`），由 `ate-controller` 调谐为具体的 Worker Deployment。
   - Worker 负责创建和执行基于容器轻量沙箱（Substrate Actor）。
   - **调度权重机理**：`resources.requests.cpu` 决定了该 Worker 在 Linux CFS（Completely Fair Scheduler）中的调度权重。配额过低（如 100m）会导致沙箱冷启动初始化（加载 Python/Golang ADK、连接 MCP 等）时被严重限流拉长至 4~6 秒；合理配额（$\ge 1000\text{m}$）可将唤醒压缩至 1~2 秒内。
3. **控制面与持久层 (`kagent-controller` & `kagent-postgresql`)**：
   - `kagent-controller` 负责调谐 `Agent`、`AgentTemplate`、`RemoteMCPServer` 等 CRD，维护 Golden Snapshot 预编译镜像。
   - PostgreSQL 存储 `kagent.session`、`kagent.a2a_context`、`kagent.scheduled_run` 等元数据与会话状态。
4. **GitOps 引擎 (`argocd`)**：
   - 管理集群期望状态（Git）与实际状态（K8s）的同步，CRD 为 `Application.argoproj.io`。


---

## 2. 四级阶梯全景巡检流水线 (Four-Stage Patrol Pipeline)

智能体在受命执行平台巡检时，必须依序执行以下四级流水线：

```
[流水线 1: L1 基础设施水位] ──> [流水线 2: L2 系统运行时] ──> [流水线 3: L3 控制面与持久化] ──> [流水线 4: L4 E2E 合成探针]
```

### 2.1 检查项与基准清单

| 层级 | 检查重点 | 命令 / MCP 接口 | 正常基准 | 异常研判标准 |
| :--- | :--- | :--- | :--- | :--- |
| **L1** | 节点状态与资源负载 | `kubectl get nodes -o wide`<br>`kubectl describe node` | 所有 Node Ready<br>CPU Request $\le 80\%$ | 节点 NotReady / 资源配额超标 / 存在 OOMKilled |
| **L2** | 网关与计算池健康度 | `kubectl get pods -n ate-system -l app=atenet-router`<br>`kubectl get pods -n kagent -l ate.dev/worker-pool=kagent-default` | 1/1 Running<br>Restart 次数 0 或无新增 | Pod CrashLoopBackOff / Restarts 快速上升 / 探针未就绪 |
| **L3** | 控制器与数据库健康 | `kubectl logs -n kagent deploy/kagent-controller --tail=50`<br>`kubectl get pods -n kagent -l app=kagent-postgresql` | 调谐日志无高频堆栈<br>Postgres Ready | 出现大量的 DB Connection Pool Timeout 或 Deadlock |
| **L4** | 端到端真实会话测试 | `kagent-api` 接口：`list_sessions`<br>`invoke_session` 发起探针 | 200 OK<br>P99 唤醒耗时 $\le 3.5\text{s}$ | 出现 503 网关超时 / 提示 "input was not accepted" |

---

## 3. 典型故障排查决策树 (Diagnostic Decision Tree)

```text
               [发现平台异常]
                     │
    ┌────────────────┴────────────────┐
    ▼                                 ▼
[现象 A: 会话报 503 / input not accepted]  [现象 B: 某核心 Pod 频繁崩溃/重启]
    │                                 │
    ▼                                 ▼
1. 检查 atenet-router 日志:           1. 检查 Pod describe 查看 LastState:
   是否存在 "ResumeActor timed out"      - 是否为 OOMKilled (Exit Code 137)?
2. 检查 kagent-default Worker 日志:     - 是否为 CrashLoop (Exit Code 1/2)?
   查看沙箱 wakeup probe 耗时         2. 调取 k8sgpt-mcp-server 获取 AI 诊断
3. 若唤醒耗时 > 5s 且网关 budget 为 5s:  3. 关联检索该组件近期 Events
   -> 判定为: 冷启动 CFS 限流与超时过紧
   -> 制定方案: 扩充 Worker CPU Request
      并放宽网关 requestParking.budget
```

---

## 4. GitOps 与 ArgoCD 闭环作业 SOP

当制定好变更方案（如微调 YAML）后，必须严格遵循 GitOps 闭环步骤：

1. **Step 1 - 检出与创建分支**：
   - 目标仓库：项目 GitOps 仓库（或当前项目仓库）。
   - 分支规范：`fix/kagent-guardian-<timestamp>`。
2. **Step 2 - 提交变更并提 PR**：
   - 修改对应清单（例如 WorkerPool 资源或 ConfigMap 参数）。
   - 创建 Pull Request，正文严格遵循标准模版，注明：
     - **故障现象与影响面**；
     - **关联日志与探针实测数据**；
     - **参数精算依据（Why This Value）**；
     - **回滚方案（Rollback Plan）**。
3. **Step 3 - 跟踪 ArgoCD 状态**：
   - 变更合并后，通过 `kubectl get applications.argoproj.io -n argocd` 检查 Application 同步状态。
   - 期望：`SYNC STATUS: Synced`，`HEALTH STATUS: Healthy`。
4. **Step 4 - 线上复测验证**：
   - 执行 L4 合成探针复测，确认唤醒延时恢复正常，无报错。
   - 在 GitHub PR 中追评复测通过报告并 Close/结单。

---

## 5. 标准巡检与排障报告模版 (Markdown Template)

向 GitHub 仓库提交的巡检报告（`docs/patrol-reports/YYYY-MM/patrol-<date>.md`）应包含以下结构：

```markdown
# Kagent 平台健康巡检与自愈分析报告

- **巡检时间**：YYYY-MM-DD HH:MM:SS
- **执行 Agent**：kagent-platform-guardian
- **综合健康评级**：[ GREEN (健康) | YELLOW (亚健康/建议优化) | RED (紧急故障) ]

## 一、分层体检结果
| 检查层级 | 目标对象 | 状态指标 | 评级 |
| :--- | :--- | :--- | :--- |
| L1 基础设施 | 节点与资源分配 | CPU Alloc: XX%, Mem Alloc: XX% | PASS |
| L2 系统运行时 | atenet-router / WorkerPool | 1/1 Running, Budget: 15s | PASS |
| L3 控制面持久层 | Controller / PostgreSQL | 调谐正常, DB 无慢查与连接风暴 | PASS |
| L4 合成业务探针 | 会话唤醒测试 (E2E) | 唤醒延时: X.Xs, 响应: 200 OK | PASS |

## 二、发现的问题与异常推导 (如有)
- **异常现象**：
- **佐证日志与时序数据**：
- **根因分析**：

## 三、处置与 GitOps 实施记录
- **处理动作**：[ 无需变更 | 已发起优化 PR: #<PR号> ]
- **ArgoCD 同步状态**：[ Synced & Healthy ]
- **复测验证结论**：
```
