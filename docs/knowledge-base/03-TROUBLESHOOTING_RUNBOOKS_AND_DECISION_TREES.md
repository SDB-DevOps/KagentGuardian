# Kagent 经典故障诊断决策树与排障手册 (Troubleshooting Playbooks)

## 故障 1：UI 红色告警 "input was not accepted; retry after the session becomes available"
- **现象**：用户在聊天界面提交输入时，被网关直接打回 503 错误。
- **推导链条**：
  1. 检查 `kagent.session` 表，确认会话是否处于非激活状态。
  2. 检查 `ate-system/atenet-router` 网关日志，定位是否出现 `parking timeout exceeded`。
  3. 检查 `kagent-default` WorkerPod CPU 使用率与 CPU Request 水位。
  4. 若 WorkerPod CPU Request $\le 200\text{m}$，断定为 **CFS 算力节流导致冷启动耗时超过网关等待预算**。
- **自愈处置**：
  - 调优 WorkerPool 的 CPU Request 至 1000m。
  - 调整网关 `requestParking.budget` 至 15s 以上。
  - 通过 GitHub PR 提交配置变更，经 ArgoCD 闭环同步。

## 故障 2：Agent 状态停留在 `Ready: False`
- **现象**：创建或修改 Agent 后，长时间无法变绿。
- **推导链条**：
  1. 执行 `kubectl get agent <name> -o yaml` 查看 `status.conditions`。
  2. 若卡在 `ResolvedRefs: False`：检查 `systemPromptFrom` 的 ConfigMap/Key 是否拼写错误，或者 `RemoteMCPServer` 是否不存在。
  3. 若卡在 `Ready: False` 且 message 为 `Snapshot not ready`：检查 Substrate Controller 日志，排查沙箱底层镜像拉取或内存超限报错。

## 故障 3：PostgreSQL 连接池打满与死锁
- **现象**：Controller 出现大量数据库超时，会话无法创建。
- **排查命令**：
  - 查询活动连接：`SELECT count(*), state FROM pg_stat_activity GROUP BY state;`
  - 检查未决锁：`SELECT * FROM pg_locks WHERE NOT granted;`
- **自愈建议**：调大 PgBouncer/PostgreSQL `max_connections`，优化长连接保活参数。
