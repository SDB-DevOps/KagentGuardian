# 专题讨论三：会话 503 超时与 Linux CFS CPU 调度优化始末

## 一、故障现象复现
在 Kagent 早期使用中，前端 UI 频繁弹出红色报错：
```text
input was not accepted; retry after the session becomes available
```
该错误导致用户输入无法被沙箱正常接收，会话连接被中断，严重影响平台可用性。

## 二、根因定位全景推导
通过全链路追踪，我们定位出该问题的跨层级级联故障链条：
1. **网关层超时拦截 (`atenet-router`)**：
   网关为了防止客户端无限挂起，引入了 `requestParking.budget` 机制。默认情况下等待预算仅为 5 秒。
2. **Substrate 沙箱冷启动延时**：
   当用户唤醒一个处于休眠状态的 Agent 时，Substrate 需要调度 Worker Pod 挂载沙箱、初始化 Python/Go ADK 并恢复内存。
3. **致命瓶颈：Linux CFS 算力节流 (CFS Throttling)**：
   Kubernetes 中 `resources.requests.cpu` 决定了容器在宿主机 Linux Completely Fair Scheduler 中的时间片权重。
   - 原 WorkerPool 的 CPU Request 仅配置为 `100m`。
   - 在沙箱冷启动瞬间，由于 CPU 权重极低，容器遭遇严重的 CFS CPU 节流，冷启动耗时被强制拉长到 4~6 秒以上。
   - 沙箱耗时直接击穿网关 5s 的 `requestParking.budget`，网关触发熔断向前端打回 503 异常。

## 三、系统化调优与自愈标准
1. **WorkerPool 算力基线提升**：
   将 Worker Pod CPU Request 提升至 `1000m`，确保突发秒级算力充沛，冷启动唤醒耗时压减至 1~1.5 秒。
2. **网关挂起容错预算放宽**：
   将 `atenet-router` 的 `requestParking.budget` 调整为 `15s` 以上，提供充裕的安全缓冲。
3. **固化为 SRE Guardian 巡检指标**：
   将上述指标纳入 Guardian Agent 的 L2 巡检基线，定时检测并防患于未然。
