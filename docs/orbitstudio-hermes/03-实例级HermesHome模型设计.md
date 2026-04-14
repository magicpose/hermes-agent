# 03-实例级 HermesHome 模型设计

## 1. 目标
定义 Hermes 在 Orbit 场景下的实例隔离模型，包括：身份、目录、回退规则、数据边界。

## 2. 核心实体

### 2.1 AssistantRef
- 结构：`AssistantRef{assistant_id, studio_id}`
- 归属：可选 `tenant_id`, `owner_user_id`

### 2.2 Agent Runtime Instance
- 结构：`AIAgent(..., hermes_home=Path)`
- 含义：单个 AIAgent 实例对应一个独立数据根目录。

### 2.3 CollabGroup（持久协作组）
- 核心字段：`group_id`, `tenant_id`, `studio_id`, `members[]`, `strategy`
- `strategy` 枚举冻结：`sequential | parallel | coordinator`
- 默认策略冻结：`sequential`

### 2.4 GroupMemberBinding
- 结构：`{group_id, assistant_ref, role, weight, enabled}`
- 语义：协作组成员绑定持久身份，不是一次性角色文本。

### 2.5 GroupCoordinator
- 结构：`{group_id, assistant_ref, hermes_home}`
- 语义：可选协调器代理，仅用于“步骤决策/上下文裁剪/路由”。
- 约束：协调器不替代成员执行。

## 3. 目录契约（冻结）

### 3.1 成员目录
Orbit 侧为每个成员助理分配目录：

`~/.orbitstudio/assistants/{tenant_id}/{studio_id}/{assistant_id}/`

在该目录下：
- `memories/MEMORY.md`
- `memories/USER.md`
- `skills/`
- `config.yaml`
- `.env`
- `sessions/`
- 其他 Hermes 子目录（如 logs/checkpoints）按需创建

### 3.2 协调器目录（可选）
如果组启用协调器：

`~/.orbitstudio/groups/{tenant_id}/{studio_id}/{group_id}/group_coordinator/`

用途：
- 存储组级协作偏好与调度经验。
- 不存储成员私有记忆。

## 4. 优先级规则（冻结）
路径解析遵循：
1. `AIAgent` 实例传入的 `hermes_home`（最高优先级）
2. 全局 `get_hermes_home()`（默认回退）

说明：
- 不引入“动态环境变量覆盖实例参数”的机制。
- 实例参数优先级高于环境变量中已有 `HERMES_HOME`。

## 5. 接口契约（冻结）

### 5.1 AIAgent
```python
AIAgent(
    ...,
    hermes_home: str | Path | None = None,
)
```

约束：
- 新参数必须可选。
- 不传时保持旧行为。

### 5.2 Skills Prompt
```python
build_skills_system_prompt(
    available_tools: set[str] | None = None,
    available_toolsets: set[str] | None = None,
    hermes_home: Path | None = None,
)
```

### 5.3 MemoryStore
```python
MemoryStore(
    memory_char_limit: int = 2200,
    user_char_limit: int = 1375,
    hermes_home: Path | str | None = None,
)
```

### 5.4 CollabGroup Strategy（冻结）
- 持久组必须使用成员独立 `hermes_home`。
- 共享成员记忆目录被明确禁止。
- `coordinator` 失败时回退 `sequential`。

## 6. 边界定义
- 本次只保证 AIAgent 嵌入运行时隔离。
- CLI profile 仍保留原有流程，不在本次文档内重构。
- 模型 A（单 Agent 角色扮演）仅用于临时协作概念说明，不纳入持久 CollabGroup 协议。

## 7. 失败策略
- `hermes_home` 无法创建目录：抛出明确异常并停止实例启动。
- 禁止静默回退到全局目录，避免失败时串写。
- `coordinator` 运行超时或失败：自动降级到 `sequential`。
