# 专题讨论一：提示词解耦与 Kagent Web 控制台 Prompts 页面集成

## 一、背景与问题提出
在初版实现中，智能体的 System Prompt 是直接硬编码在 `AgentTemplate.spec.systemPrompt` 内嵌字符串中的。
这种方式存在以下痛点：
1. **运维体验差**：若 SRE 需要润色人设或微调安全规则，必须修改 Kubernetes Manifest YAML 并重新执行 `kubectl apply`。
2. **缺乏可视化管理**：Kagent Web UI 提供了专门的 `Prompts` 页面（`http://<IP>:31080/prompts`），支持提示词片段的可视化查看与热编辑，但内嵌式 Prompt 无法在 UI 页面中呈现。
3. **复用性低**：其他 SRE 相关的子 Agent（如专项网络探针、巡检报告通知 Agent）无法共享同一套提示词规则。

## 二、架构演进与技术方案
通过分析 Kagent 控制面源码与现有 `finops-prompts`、`cicd-prompts` 的实现机制，我们明确了 Kagent 的原生 Prompt 库集成规范：
1. **ConfigMap 专属标签机制**：
   在 Kubernetes ConfigMap 的 `metadata.labels` 中添加 `kagent.dev/prompt-library: "true"`。Kagent 控制器监听到该标签后，会自动将该 ConfigMap 注册进平台全局 Prompt 库并在前端 Web UI 展示。
2. **多片段（Fragment）支持**：
   ConfigMap `data` 下的每个 key 对应一个可复用的 Prompt 片段（Fragment）。
3. **AgentTemplate 动态引用**：
   将 `AgentTemplate` 中的内联字段替换为：
   ```yaml
   systemPromptFrom:
     name: kagent-guardian-prompts
     key: kagent-guardian-prompt
   ```

## 三、调谐状态机流转
当在集群中创建 `kagent-guardian-prompts` 并更新 `AgentTemplate` 后，Kagent 控制面与 Substrate 自动完成以下状态迁移：
- `ResolvedRefs: True` (All runtime references resolved)
- `Compatible: True` (Resolved configuration is compatible with the Harness)
- `Ready: False -> True` (Substrate 自动依据新 Prompt 重新预热并烧录 ActorTemplate Golden Snapshot)
- Web UI 页面立即呈现 `kagent-guardian-prompts`，支持在线查看与实时编辑。
