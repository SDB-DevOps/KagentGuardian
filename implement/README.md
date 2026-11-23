# Kagent Guardian 生产实施与升级方案导引 (Implementation Directory)

本目录 (`implement/`) 汇集了将 Kagent Guardian 从架构设计（HLD/LLD v2.0）落地至生产环境的所有实操指导、迁移路线图、升级步骤与应急预案。

---

## 核心文档索引

| 文档名称 | 说明 | 适用角色 | 状态 |
| :--- | :--- | :--- | :--- |
| **[01-PRODUCTION_AGENT_DEPLOYMENT_AND_MIGRATION_GUIDE.md](./01-PRODUCTION_AGENT_DEPLOYMENT_AND_MIGRATION_GUIDE.md)** | **生产级部署与存量 Agent 迁移升级实施方案**：详细阐明 6 大迁移实施阶段（Phase 0~5）、各阶段前后置门禁、核心注意事项、拓扑依赖图、实操指令、3.5 天工时预估与应急回滚预案。 | SRE 架构师<br>DevOps 工程师<br>平台研发 | **已就绪 (v2.0)** |
| **[02-PRODUCTION_MIGRATION_EXECUTION_LOG.md](./02-PRODUCTION_MIGRATION_EXECUTION_LOG.md)** | **生产级 Agent 改造实施执行全过程纪要与验收报告**：记录 6 个阶段的实操动作、执行命令、输出日志、Substrate 黄金快照就绪实测、Sentinel 零 Token 巡检验证及日终账本自动提交验收证据。 | SRE 架构师<br>DevOps 工程师<br>审计合规 | **已就绪 (v2.0)** |

---

## 核心实施原则

1. **绝对红线：生产严禁直接热打补丁 (No Hot-Patching Gate)**：任何集群配置变更一律经由 GitHub PR 与 ArgoCD 完成 GitOps 交付；
2. **依赖驱动：严禁跨阶段逆序推进**：必须严格遵循 `Phase 0 ➔ Phase 1 ➔ Phase 2 ➔ Phase 3 ➔ Phase 4 ➔ Phase 5` 的依赖单向拓扑；
3. **状态守卫：Substrate 黄金快照就绪门禁**：在更新 Agent 模板后，必须等待 `GoldenSnapshotReady: True` 与 `Ready: True`，方可开放接入生产流量；
4. **应急第一：5 分钟回滚保活预案**：遇阻塞性异常必须果断触发一键式清单回滚，优先保障集群控制面可用性。
