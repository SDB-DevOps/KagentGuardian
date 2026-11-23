# Kagent Guardian 生产验收测试中心 (Test Center)

| 属性 | 详情 |
| :--- | :--- |
| **所属模块** | Kagent Platform Guardian (`kagent-guardian-agent`) |
| **测试目标** | 全面验证生产改造（v2.0）实施成果与架构设计预想的一致性 |
| **遵循规范** | HLD (Doc 02) / LLD (Doc 03) / Implementation Guide (DOC-IMP-01) |
| **当前状态** | **就绪 (Ready for Full-Scale Acceptance)** |

---

## 目录与文档索引

本目录收录了保障 Kagent Guardian 具备生产级可用性、安全合规性与自动化运维能力的全套测试资产：

| 文档名称 | 核心内容与目的 | 适用阶段 | 状态 |
| :--- | :--- | :---: | :---: |
| **[01-PRODUCTION_VERIFICATION_TEST_PLAN.md](./01-PRODUCTION_VERIFICATION_TEST_PLAN.md)** | **生产级全量验收测试计划与设计预实验证方案**：涵盖 10 大核心测试用例（`TC-V2-01` ~ `TC-V2-10`），全量覆盖 No Hot-Patching 拦截、网关 PEP HITL 轮询、零 Token 哨兵短路、CFS 双因子诊断、GitOps 闭环、Lease 并发排他锁、日终账本自动归档、Session GC 清理及 Substrate 黄金快照验证。 | 生产验收 / 巡检审计 | **已发布 (v2.0)** |
| **[02-COMPREHENSIVE_TEST_CASES_SPECIFICATION.md](./02-COMPREHENSIVE_TEST_CASES_SPECIFICATION.md)** | **生产级全量测试用例详细设计规约 (功能/安全/边界/容错/性能)**：涵盖 5 大测试领域、28 个细分实测用例，包含深度提示词注入攻防、84/85 分与 299/300 秒极限临界测试、断网与时钟漂移混沌演练、以及 FinOps Token 成本核算审计。 | 深度测试 / 自动化执行 / 准出核签 | **已发布 (v2.0)** |
| **[03-MANUAL_UI_INTERACTION_TEST_CASES.md](./03-MANUAL_UI_INTERACTION_TEST_CASES.md)** | **网页端交互式手动测试用例集 (Manual UI Test Cases)**：涵盖 5 大分类、21 个可直接在 Web UI 复制粘贴的实战用例。覆盖 L1-L4 全链路与分层阶梯巡检、四元组 MCP 工具调用、生产环境零热修安全防御、越狱与注入攻防、以及 GitOps 自愈闭环。 | Web 控制台手动实操 / 巡检 / 权限审计 | **已发布 (v2.0)** |

---

## 测试分层与核心覆盖矩阵

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   Layer 5: GitOps 闭环与日终自动化审计                  │
│       - TC-V2-06: Issue 查重幂等与自愈 PR  - TC-V2-08: 日终账本自动归档 │
├────────────────────────────────────────────────────────────────────────┤
│                   Layer 4: 网关安全治理与 PEP HITL 审批                │
│       - TC-V2-01: No Hot-Patching 拦截    - TC-V2-02: PEP 5s/300s 轮询 │
│       - TC-V2-03: Fail-Closed 降级断言                                  │
├────────────────────────────────────────────────────────────────────────┤
│                   Layer 3: 诊断决策与阶梯巡检引擎                      │
│       - TC-V2-04: Sentinel 零 Token 短路  - TC-V2-05: CFS 双因子决策树 │
├────────────────────────────────────────────────────────────────────────┤
│                   Layer 2: 并发排他锁与数据生命周期                    │
│       - TC-V2-07: Lease 分布式排他锁      - TC-V2-09: Session GC 自动清理│
├────────────────────────────────────────────────────────────────────────┤
│                   Layer 1: 基座运行时与黄金快照                        │
│       - TC-V2-10: Substrate 黄金快照就绪与秒级冷启动                    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 测试准入与通过标准

- **准入标准 (Entry)**：
  1. `deploy/` 下所有清单（Lease, Sentinel, GC, Archiver, ConfigMap, AgentTemplate, Agent）均已通过 `kubectl apply` 并在集群内生效；
  2. `agent.kagent.dev/kagent-guardian-agent` 的 `Ready` 状态为 `True`，且 `ActorTemplate golden snapshot is ready`；
  3. GitHub App 凭证解析正常，具备目标仓库的读写权限。
- **通过标准 (Exit)**：
  1. 10 项核心测试用例执行结果 100% 达到预期，且无阻塞性（P0）缺陷；
  2. 哨兵健康状态下 0 Token 消耗通过；
  3. 生产严禁直接热修改线上工作负载铁律有效拦截率 100%。
