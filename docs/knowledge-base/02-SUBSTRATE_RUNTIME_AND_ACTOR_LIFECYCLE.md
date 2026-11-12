# Substrate 运行时、沙箱调度与 CFS 算力机制

## 1. Substrate 架构原理
Substrate 是 Kagent 底层的 Actor 模型沙箱执行引擎，负责：
- 隔离不同智能体运行环境（进程/容器级安全隔离）。
- 支持极速唤醒（通过 Golden Snapshot 快照机制）。
- 内存与执行上下文的休眠冻结与毫秒级恢复。

## 2. 黄金快照 (Golden Snapshot) 机制
- 当 `AgentTemplate` 或引用的 `systemPromptFrom` 发生变更时，Kagent Controller 自动向 Substrate 发送重编译请求。
- Substrate 在计算节点预热拉起一个专属沙箱，加载所有系统依赖、初始化 Python/Go ADK 运行时环境并执行一次内存快照。
- 只有快照制作成功且标志位写入数据库（`latest_successful_revision` 更新），Agent 状态才会跃迁为 `Ready: True`。

## 3. WorkerPool 与 Linux CFS 算力配额调度
- Worker Pod 由 `WorkerPool`（如 `kagent-default`）产生，承担具体的计算任务。
- **CFS (Completely Fair Scheduler) 机制**：
  - Kubernetes 中 `resources.requests.cpu` 决定了 cgroup 的 `cpu.shares` 权重。
  - **严重陷阱**：若将 Worker Pod 的 CPU requests 设得过小（如 100m），虽然日常闲置能节约资源，但每次冷拉起沙箱初始化加载多模块时会发生剧烈的 CFS CPU 节流（Throttling），拉起耗时从 1 秒飙升到 5 秒以上，直接撞上网关的 Request Parking 超时。
  - **黄金基准**：Worker Pod CPU Request 应配置 $\ge 1000\text{m}$，确保突发秒级唤醒。
