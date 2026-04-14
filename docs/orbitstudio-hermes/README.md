# OrbitStudio Hermes 改造文档集

本目录用于承载 OrbitStudio 对 Hermes Agent 的 `per-instance hermes_home` 改造设计文档。

文档目标：
- 让 fork 开发者从 0 到 1 理解改造背景、边界、模型、链路和实施步骤。
- 在不修改 Hermes 既有默认行为的前提下，定义可落地的实例隔离方案。
- 为后续编码和测试提供冻结契约，避免实现阶段再次做高层决策。

建议阅读顺序：
1. `00-总览与阅读路径.md`
2. `01-目标与非目标.md`
3. `02-现状盘点与问题清单.md`
4. `03` 至 `07`（核心设计）
5. `08-OrbitStudio接入规范.md`
6. `12-协作组编排与协调器策略设计.md`
7. `10-测试计划与验收标准.md`
8. `11-实施任务分解与里程碑.md`

协作组专题说明：
- 持久 CollabGroup 主线固定为模型 B（多成员独立 AIAgent）。
- `coordinator` 是可选策略，不替代成员执行与成员记忆。
- 模型 A 仅保留为一次性临时协作概念，不进入持久协议。

说明：
- 本目录独立于 `docs/specs` 与 `website/docs`。
- 本轮只落设计文档，不包含代码改动。
- 术语默认与 OrbitStudio 侧一致：`AssistantRef{assistant_id, studio_id}`、`tenant_id`、`scope_type`。
