# 专题讨论四：Guardian 功能完成度盘点、巡检机制与 GitOps 闭环现状评估

## 一、阶段性目标与总体完成度评估
在完成了智能体设计、提示词解耦、知识库与代码仓库建立后，本专题针对用户关切的“整个 Agent 功能是否完成？巡检与 GitOps 实施到哪一步？”进行了系统化全景梳理与实操验证。

当前整体工程建设完成度约为 **85% ~ 90%**：
- **静态规约、模型装配与知识沉淀**：已 **100%** 实施就绪。
- **巡检机制 (Inspection Pipeline)**：编排与规则设计 **100%**，处于**待端到端实机验证 (90%)** 阶段。
- **GitOps 自愈机制 (GitOps Pipeline)**：门禁铁律与 PR 闭环规约 **100%**，处于**待 GitHub App 仓库授权打通 (80%)** 阶段。

---

## 二、各核心模块现状详细盘点

### 1. Agent 核心状态与 Substrate 运行时 (100%)
- **CRD 纳管就绪**：
  - `AgentTemplate` (`kagent-guardian-template`)：已定义并引用 `default-model-config` 与 `kagent-guardian-prompts`。
  - `Agent` (`kagent-guardian-agent`)：已挂载 `agentstage-harness`，状态达到 `Ready: True`。
  - `Substrate Golden Snapshot`：ActorTemplate 快照已成功烧录至集群，随时可供 WorkerPod 挂载秒级冷启动。

### 2. 提示词库与 Web UI 页面深度集成 (100%)
- **动态解耦**：系统提示词完全从内嵌代码中剥离，由 Kubernetes ConfigMap `kagent-guardian-prompts` 独立管理。
- **可视化纳管**：带有 `kagent.dev/prompt-library: "true"` 标签，在 Kagent Web 控制台（`http://34.69.136.108:31080/prompts`）中集中呈现，支持运维人员直接在线查看与热调。

### 3. 四大 MCP 工具链挂载 (100%)
通过控制面动态服务发现，成功为 Guardian 装配了全套工具集合：
- `kagent-tool-server`：126 个 Kubernetes 平台只读巡检工具（涵盖 Pods、Nodes、Logs、Events、CRD 检索）。
- `kagent-api`：18 个平台自身 API 工具（支持通过 A2A 协议发起合成会话探针、沙箱管理与快照检查）。
- `k8sgpt-mcp-server`：12 个 AI 智能初筛与异常分析工具。
- `github-mcp-server`：90 个 GitHub 操作工具（提供分支管理、Commit 提交、Issue 建单与 Pull Request 开启能力）。

---

## 三、巡检功能 (Inspection) 深度评估

### 1. 已就绪核心能力
- **4 级阶梯巡检流水线**已在 Prompt 与 Domain Skill 中全面固化并加载：
  - **L1 基础设施层**：宿主机 Node 状态、CPU/Memory Requests 申请水位（基准 $\le 80\%$）、排查节点级 OOMKilled。
  - **L2 运行时与网关**：`atenet-router` 网关健康度、`kagent-default` WorkerPod 状态与重启频次、Linux CFS CPU 算力权重与 `requestParking.budget` 超时容错匹配度。
  - **L3 控制面与持久化**：`kagent-controller` 调谐队列与堆栈报错、PostgreSQL 连接池与事务健康。
  - **L4 端到端合成探针**：调用 `kagent-api` 发送合成测试会话，实测沙箱冷启动延时（基准 $\le 3.5\text{s}$）。

### 2. 待验证动作
- 尚未在生产 Web 控制台触发首轮实机巡检。
- **验证手段**：在 Web UI（`http://34.69.136.108:31080`）中打开与 `kagent-guardian-agent` 的会话，发送巡检指令，实测模型调度 126 个只读工具输出巡检报告的准确性。

---

## 四、GitOps 自愈功能 (GitOps) 深度评估与授权排障

### 1. 架构与流程规范 (100%)
- **安全红线 (No Hot-Patching Gate)**：严禁 Guardian 通过 MCP 接口对生产 K8s 集群直接执行 `kubectl edit/patch/apply/delete`。
- **自愈闭环流水线**：
  1. 发现配置瓶颈（如 WorkerPod CPU 配额不足）；
  2. 自动在 `SDB-DevOps/KagentGuardian` 检出分支并提交配置 Diff；
  3. 创建结构化 Pull Request；
  4. 经人工审查/CI 合并后，由 ArgoCD 同步集群状态；
  5. Guardian 发起 L4 合成探针复测验证并闭环结单。

### 2. 关键阻断点：GitHub App 仓库访问授权
- **现状排查**：
  - 集群中的 `github-mcp-server` 挂载的是 GitHub App 凭证（Installation ID: `162644634`）。
  - 该 App 在 `SDB-DevOps` 组织下的授权模式为 **“Only select repositories”**，历史仅勾选了 `SDB-DevOps/AgentStage`。
  - 经 API 实测，GitHub App 当前无法读取或写入新创建的 `SDB-DevOps/KagentGuardian` 仓库。
- **解决方案**：
  - 组织管理员登录 GitHub，访问：`https://github.com/organizations/SDB-DevOps/settings/installations`
  - 找到对应的 GitHub App，在 **Repository access** 列表中将 `KagentGuardian` 勾选加入（或调整为 `All repositories`）。
  - 授权生效后，Guardian 即可自动向本仓库提 PR 并建 Issue。

---

## 五、后续行动指南

1. **管理员补全 GitHub App 授权**：勾选 `KagentGuardian`，彻底打通 GitOps 写权限。
2. **执行首轮端到端全景巡检实测**：通过 UI 发起对话验证。
3. **模拟自愈演练**：构造配置变更场景，验证 Guardian 自动提 PR 的完整链路。
