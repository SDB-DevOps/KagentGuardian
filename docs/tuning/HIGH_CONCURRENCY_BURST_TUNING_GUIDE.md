# Kagent 平台应对高并发突发流量调优指南与演练规范
(Kagent Platform High-Concurrency Burst Tuning Guide & SRE Specification)

---

## 1. 背景与调优目标

随着 Kagent 平台上承载的智能体实例、多 Agent 协作网络（A2A 通信）以及多租户会话规模的迅速增长，平台在流量洪峰和高并发会话唤醒场景下面临严峻的延时和吞吐考验。

### 核心调优目标
1. **秒级冷启动保障**：在突发大批量休眠 Agent 会话唤醒时，将沙箱 Cold-Start 延迟稳定控制在 **<= 1.5s** 以内。
2. **彻底消除 503 网关熔断**：杜绝因沙箱唤醒抖动击穿网关挂起预算而导致的 `503 Service Unavailable (input was not accepted; retry after the session becomes available)`。
3. **消除宿主机 CFS 算力节流**：通过合理的 CPU Request 算力分配，杜绝 Worker 容器在加载运行时和依赖时遭遇 Linux CFS 调度限流。
4. **数据库与控制面承载力提升**：保障 PostgreSQL 连接池与 Controller 调谐队列在高并发任务提交时不发生锁等待与队列拥塞。

---

## 2. 平台全景调优架构

```
[ Client Traffic / Web UI / A2A Cascade ]
                   │
                   ▼
       ┌────────────────────────┐
       │   atenet-router 网关   │  <-- 调优 1: requestParking.budget 扩展 (>= 15s)
       └────────────────────────┘
                   │ (HTTP/gRPC/Session Dispatch)
                   ▼
       ┌────────────────────────┐
       │  Substrate WorkerPool  │  <-- 调优 2: CPU Request >= 1000m 杜绝 CFS 节流
       │  (Worker Deployment)   │  <-- 调优 3: 黄金快照 (GoldenSnapshot) 预热加速
       └────────────────────────┘
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
┌──────────────────┐ ┌───────────────────┐
│kagent-controller │ │ kagent-postgresql │ <-- 调优 4: max_connections 连接池复用
│(调谐并发度优化)   │ │ (WAL 缓冲与慢查治理)│
└──────────────────┘ └───────────────────┘
```

---

## 3. 核心分层调优规范

### 3.1 网关流量与挂起容错层 (`ate-system/atenet-router`)

在高并发会话拉起时，多个请求可能并发请求处于休眠状态的沙箱，此时网关必须为沙箱预热留足排队等待窗口：
- **`requestParking.budget`**：
  - **默认基线**：5s（高并发下极易被长尾唤醒击穿）
  - **推荐生产值**：`15s` ~ `30s`
  - **作用机制**：当沙箱正在拉起或处于 GoldenSnapshot 载入阶段，网关将请求挂起在 Parking 队列中，待沙箱变为 Ready 立即投递，杜绝 503 报错。
- **并发连接与 Keep-Alive 策略**：
  - 上游空闲超时 (`idleTimeout`)：调整至 `65s`（保持长连接复用，降低频繁建立 TLS 握手的开销）。
  - 最大排队连接深度 (`maxQueueSize`)：根据节点内存与并发预期扩容至 `2048`。

### 3.2 计算工作池与 Substrate 沙箱运行时 (`kagent-default/WorkerPool`)

#### 彻底消除 Linux CFS 算力节流 (Completely Fair Scheduler)
- **底层原理**：Kubernetes 的 `resources.requests.cpu` 决定了容器在 Linux CFS 调度器中的 CPU 共享份额（`cpu.shares`）。配额过小（如 `100m`）会导致容器在初始化加载 Python 解释器、动态链接库及模型配置时，迅速耗尽单周期 CPU 时间片而被强行 Throttling。
- **调优基准**：
  ```yaml
  resources:
    requests:
      cpu: "1000m"       # 生产硬性基准：保证突发拉起时拥有完整核心算力
      memory: "2Gi"
    limits:
      cpu: "4000m"       # 允许短时爆发突增
      memory: "8Gi"
  ```
- **黄金快照预热 (`GoldenSnapshot`)**：
  - 对核心高频 `AgentTemplate` 开启快照预热能力，避免运行时拉取镜像层及重编译环境，将冷启动耗时从 4~6s 缩短至 1s 左右。

### 3.3 控制面协调器 (`kagent-controller`)

- **工作队列并发度**：
  - `--concurrent-workers`：将默认的调谐 Worker 协程数从 `5` 增加至 `20`，加快高并发创建 Session 和 Agent 时的状态机收敛速度。
- **Client-Go QPS 与 Burst**：
  - `kube-api-qps`: 调整为 `100`（默认 20）。
  - `kube-api-burst`: 调整为 `200`（默认 30），防止并发向 API Server 更新 Condition 时发生客户端限流。

### 3.4 调度存储与持久化层 (`kagent-postgresql`)

- **连接数调优**：
  - `max_connections`：从 `100` 提升至 `300`，适配控制器、API 服务以及会话持久化组件的并发写入需求。
  - 建议在高并发集群前置引入轻量级连接池代理（如 PgBouncer），开启事务级连接复用。
- **缓冲区与锁超时**：
  - `shared_buffers`: 调整为实例内存的 25%（如 1GB ~ 2GB）。
  - `lock_timeout`: 设置为 `5s`，避免慢事务长期持锁导致高并发队列雪崩。

---

## 4. GitOps 变更与安全门禁准则

根据平台安全红线要求：
1. **严禁热更**：任何上述参数调整严禁使用 `kubectl edit/patch` 在线上直接执行。
2. **声明式闭环**：所有变更必须在本仓库提交 Pull Request，通过代码评审后由 ArgoCD 自动同步至集群。
3. **合成探针验收**：变更生效后必须发起 L4 合成探针（Synthetic Session Probe）执行多轮并发会话验证，实测延迟达标方可关闭工单。
