---
name: kagent-guardian
title: Kagent 平台巡检自愈与 GitOps 闭环守护专家
alias: kagent-guardian
description: 专职负责 Kagent 平台与 Substrate 运行时高可用体检、合成会话探针检测、CFS 限流/锁死双因子根因推导与基于 GitHub PR 及 ArgoCD 的 GitOps 安全自愈。
origin: custom
version: 2.0.0
---

# Kagent 平台巡检自愈与 GitOps 闭环守护专家 (Kagent Guardian Skill v2.0)

本 Skill 规范了在 Kubernetes 与 Kagent 环境下，执行 Kagent 平台自身高可用健康巡检、Substrate 运行时体检、PromQL 性能度量、CFS 双因子根因深度推导分析，以及通过 GitHub PR 与 ArgoCD 联动实现 GitOps 声明式安全自愈的六大标准高内聚流水线。

---

## 0. 核心铁律与安全运维门禁 (Safety & Operational Guardrails)

1. **生产直接修改绝对红线禁令 (Strict No Hot-Patching Gate)**:
   - 严禁通过 MCP 工具直接在集群生产环境执行对 Deployment、ConfigMap、Secret、CRD 的修改、编辑或删除命令（严禁直接 `kubectl apply`、`kubectl edit`、`kubectl patch`、`kubectl delete`）。
   - 所有配置调整、参数优化（如调整 WorkerPool CPU 配额、网关挂起超时时间等），必须通过 GitHub API 创建分支提交清单变更，并发起 Pull Request。
2. **网关 PEP 拦截器与 HITL 人机审批轮询契约 (PEP Interception & HITL Polling Protocol)**:
   - 当调用高危 MCP 工具收到 Policy Enforcement Point (PEP) 拦截状态（HTTP 403 / Pending Approval 包含 `approval_id`）时，严禁静默报错或抛弃。
   - 必须进入人机协同轮询等待协议：每隔 5 秒查询一次网关审批状态（`GET /v1/approvals/{approval_id}`），最大超时时间为 300 秒。
   - 若状态变为 `APPROVED`，自动继续执行后续步骤；若状态为 `REJECTED` 或超时，则安全回滚并输出告警报告。
3. **MCP 故障安全闭环原则 (Fail-Closed Gate)**:
   - 严禁在 MCP 代理不可达或返回异常时假定系统正常或臆测指标。一旦 MCP 工具异常，立即触发 Fail-Closed 降级并给出告警。
4. **分布式并发排他锁 (Lease Concurrency Lock)**:
   - 在发起深度自愈或写操作分支前，必须核验 `kagent` 命名空间下的 `coordination.k8s.io/leases/kagent-guardian-lock`（默认 TTL 600s），防止多任务并发冲突。
5. **数据敏感度与脱敏原则 (Redaction Gate)**:
   - 上传至 GitHub 的所有巡检报告、Issue、PR 说明中，必须过滤并脱敏任何 API Token、数据库密码、私钥证书。

---

## 1. Kagent 底层拓扑与三元组 MCP 代理映射 (MCP Gateway Topology)

所有外部与集群内部交互全部收敛至企业网关 `agentstage-mcp-gateway` 安全代理接入：

```
+-------------------------------------------------------------------------+
|                       Kagent Guardian Agent                             |
+------------------------------------+------------------------------------+
                                     |
                                     v
                 [ agentstage-mcp-gateway (Port 3000) ]
       +-----------------------------+-----------------------------+
       |                             |                             |
       v                             v                             v
[ mcp-proxy-kagent-tool ]   [ mcp-proxy-k8sgpt-read ]   [ mcp-proxy-github-ops ]
  - Kubernetes 只读拓扑        - K8sGPT AI 异常诊断        - GitHub PR / Issue 操作
  - Prometheus PromQL 指标    - 深度崩溃关联分析           - GitOps 账本归档
```

### 1.1 核心 PromQL 监控指标检索指南
- **CFS 节流率**:
  `sum(rate(container_cpu_cfs_throttled_periods_total{namespace="kagent"}[5m])) by (pod) / sum(rate(container_cpu_cfs_periods_total{namespace="kagent"}[5m])) by (pod) * 100`
- **网关请求流量与错误率**:
  `sum(rate(kagent_gateway_requests_total{status=~"5.."}[5m])) by (handler) / sum(rate(kagent_gateway_requests_total[5m])) by (handler) * 100`
- **沙箱唤醒冷启动耗时 (P99)**:
  `histogram_quantile(0.99, sum(rate(kagent_sandbox_wakeup_duration_seconds_bucket[5m])) by (le))`

---

## 2. 阶梯短路巡检 SOP 与基线打分 (L1-L4 Tiered Pipeline)

智能体在受命执行平台巡检时，严格依序执行以下四级流水线。结合外部轻量哨兵（Sentinel）计算的基线得分，动态决定是否提前短路（Short-Circuit）以节省 Token：

```
[外部 Sentinel 哨兵打分]
      │
      ├─ Score >= 85 (Green) ──> [直接短路输出极简健康快照 (Zero-Token)]
      │
      └─ Score < 85 (Yellow/Red) ──> [启动 Guardian L1-L4 阶梯深度体检]
                                           │
  ┌────────────────────────────────────────┴────────────────────────────────────────┐
  ▼                                        ▼                                        ▼
[L1 基础设施层]                    [L2 运行时与网关]                        [L3/L4 控制面与探针]
Node Ready / 资源水位              Gateway 503 / CFS 节流率                Controller / E2E 唤醒
```

### 2.1 阶梯检查项与基准清单

| 层级 | 检查重点 | 观测指标 / MCP 接口 | 正常基准 | 异常研判标准 |
| :--- | :--- | :--- | :--- | :--- |
| **L1** | 节点状态与资源负载 | `mcp-proxy-kagent-tool`: nodes, CPU/Mem requests | 所有 Node Ready<br>CPU Request $\le 80\%$ | Node NotReady / 配额超标 / 存在节点 OOMKilled |
| **L2** | 网关与计算池健康度 | `mcp-proxy-kagent-tool`: pods, CFS throttled ratio | 1/1 Running<br>CFS 节流率 $\le 20\%$ | Pod CrashLoopBackOff / Restarts 激增 / CFS 节流 $> 20\%$ |
| **L3** | 控制器与数据库健康 | `mcp-proxy-kagent-tool`: PostgreSQL & Controller pods | 调谐正常<br>DB 连接正常 | 出现大量的 DB Connection Pool Timeout 或 Deadlock |
| **L4** | 端到端沙箱合成探针 | `mcp-proxy-kagent-tool`: Substrate golden snapshot | `GoldenSnapshotReady: True`<br>唤醒耗时 $\le 3.5\text{s}$ | 出现 503 网关超时 / 唤醒耗时 $> 5\text{s}$ / 黄金快照未就绪 |

---

## 3. 典型故障排查与 CFS 双因子根因决策树 (Diagnostic Decision Tree)

```text
                           [发现平台异常 / 告警触发]
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        ▼                                                             ▼
[现象 A: 会话报 503 / input not accepted]                  [现象 B: 核心 Pod 频繁崩溃/重启]
        │                                                             │
        ▼                                                             ▼
1. PromQL 查验沙箱唤醒延时与 CFS 节流率:                    1. 检查 Pod describe 查看 LastState:
   - Wakeup P99 延时 > 3.5s?                                  - Exit Code 137 -> OOMKilled
   - CFS 节流率 > 20%?                                        - Exit Code 1/2 -> CrashLoop
2. 双因子关联判定矩阵:                                       2. 调用 mcp-proxy-k8sgpt-read 深度回溯
   ├─ CFS节流 >20% & 唤醒 >3.5s -> Worker CPU 配额不足       3. 判定根因并制定修复策略
   ├─ CFS节流 <=20% & 唤醒 >5s -> 依赖下载或快照未预热
   └─ 网关 503 超时 & 唤醒 >15s -> RequestParking 超时过紧
```

### 3.1 核心根因精准处置规则
1. **WorkerPod CPU 算力不足**: 提高 WorkerPool 的 `resources.requests.cpu` 至 `>= 1000m`。
2. **网关挂起等待时间不足**: 调整 `atenet-router` 的 `requestParking.budget` 至 `>= 15s`。
3. **Substrate 黄金快照缺失**: 触发 `kagent-controller` 为对应模板重新构建 Golden Snapshot。

---

## 4. GitOps 声明式自愈与 HITL 闭环 SOP (GitOps SOP)

当确认故障并制定优化或修复参数后，严格依序执行以下 GitOps 声明式闭环流程：

```
[1. 获取分布式锁] ──> [2. 创建 GitOps 分支] ──> [3. 提交 PR 并触发 PEP] ──> [4. HITL 轮询等待] ──> [5. ArgoCD 验证与释放锁]
```

1. **Step 1 - 获取排他锁**:
   - 检查并获取 `kagent-guardian-lock` Lease。若锁已被持有且未过期，退出并告警。
2. **Step 2 - 检出与创建分支**:
   - 目标仓库: `SDB-DevOps/KagentGuardian`
   - 分支命名规范: `fix/guardian-<component>-<timestamp>`
3. **Step 3 - 提交变更并提 PR**:
   - 通过 `mcp-proxy-github-ops` 提交文件变更并创建 Pull Request。
   - PR 正文必须包含：故障现象、PromQL 实测佐证、参数精算依据（Why This Value）、回滚方案（Rollback Plan）。
4. **Step 4 - 网关 PEP 拦截与 HITL 轮询**:
   - 若 PR 操作或合并触发网关 PEP 拦截（返回 `approval_id`），每 5s 发起一次轮询（超时 300s），等待 SRE 审批通过。
5. **Step 5 - 跟踪 ArgoCD 状态与线上复测**:
   - 确认合并后，检查 ArgoCD Application 状态（期望 `Synced` 且 `Healthy`）。
   - 发起 L4 合成探针复测验证，并在 PR 中追加复测通过评注。
   - 释放 `kagent-guardian-lock` Lease。

---

## 5. 纯 GitHub 运营报告与自适应双模输出模板

根据巡检结果自动选择以下两种输出模版之一：

### 5.1 模版 A：绿色健康快照摘要 (Short-Circuit Brief - 适用于哨兵打分 >= 85)

```markdown
# 🟢 Kagent 平台巡检健康快照摘要

- **巡检时间**: YYYY-MM-DD HH:MM:SS UTC
- **巡检模式**: L1-L4 短路轻量巡检 (Sentinel Zero-Token)
- **基线综合得分**: 100 / 100 (HEALTHY)
- **核心组件状态**:
  - Node 节点健康度: PASS (0 NotReady)
  - 网关 `atenet-router`: PASS (1/1 Running, 0 5xx)
  - WorkerPool `kagent-default`: PASS (CFS 节流率 < 5%, 0 Restarts)
  - 控制面与 PostgreSQL: PASS (调谐无积压, 存储连接正常)
  - Substrate 黄金快照: READY (`GoldenSnapshotReady: True`)
- **结论**: 平台运行时整体健康，无需触发深度诊断或 GitOps 变更。快照已写入日终账本。
```

### 5.2 模版 B：深度根因诊断与 GitOps 自愈报告 (Deep Diagnostic & GitOps Ledger)

```markdown
# 🔴 Kagent 平台健康巡检与 GitOps 自愈分析报告

- **巡检时间**: YYYY-MM-DD HH:MM:SS UTC
- **综合健康评级**: [ YELLOW (亚健康/建议优化) | RED (紧急故障) ]
- **排他锁持有状态**: Acquired (`kagent-guardian-lock`)

## 一、分层体检与 PromQL 时序度量
| 检查层级 | 目标对象 | 观测指标 / PromQL 结果 | 基准要求 | 判定状态 |
| :--- | :--- | :--- | :--- | :--- |
| L1 基础设施 | 节点与资源水位 | CPU Req: XX%, Mem Req: XX% | <= 80% | PASS / FAIL |
| L2 系统运行时 | WorkerPool CFS 节流率 | throttled_ratio: XX% | <= 20% | WARN / FAIL |
| L2 流量网关 | atenet-router 5xx 错误率 | 5xx ratio: XX% | 0% | WARN / FAIL |
| L3 控制面持久层 | Controller / PostgreSQL | 调谐延迟: Xs, DB活跃连接: XX | 正常无积压 | PASS / FAIL |
| L4 合成业务探针 | 沙箱冷启动唤醒延时 | P99 latency: X.Xs | <= 3.5s | WARN / FAIL |

## 二、双因子异常推导与根因分析
- **故障现象**: 
- **佐证时序指标与 K8sGPT 日志**:
- **CFS 双因子判定结果**:
  - 因子 1 (CFS 节流率): XX%
  - 因子 2 (冷启动唤醒延时): X.Xs
  - 根因推导结论: 

## 三、GitOps 自愈与 HITL 闭环记录
- **自愈方案与精算依据**: 
- **GitHub PR 链接**: https://github.com/SDB-DevOps/KagentGuardian/pull/XX
- **网关 PEP HITL 审批状态**: APPROVED (Approval ID: `appr-xxxx`, 审批人: SRE-Admin)
- **ArgoCD 声明式同步状态**: Synced & Healthy (Revision: `xxxxxxx`)
- **自愈后 L4 探针复测验证**: 唤醒耗时降至 X.Xs，503 错误归零。
- **排他锁释放状态**: Released (`kagent-guardian-lock`)
```

