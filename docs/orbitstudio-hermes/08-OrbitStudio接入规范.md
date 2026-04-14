# 08-OrbitStudio 接入规范

## 1. 目标
定义 OrbitStudio 调用 Hermes（进程内直调 AIAgent）的接入规范，确保多助理隔离与可维护性。

## 2. 接入模式
本期采用：
- 进程内直调 `AIAgent`
- 不依赖 Hermes API Server `/v1/runs`

原因：
- 参数透传路径最短，可直接保证 `hermes_home` 生效。
- 调试与排障成本低于跨进程 API。

## 3. 目录计算契约

### 3.1 路径模板（冻结）
`~/.orbitstudio/assistants/{tenant_id}/{studio_id}/{assistant_id}/`

### 3.2 目录计算函数（示例）
```python
from pathlib import Path
import os


def hermes_home_for(tenant_id: str, studio_id: str, assistant_id: str) -> Path:
    base = Path(os.getenv("ORBITSTUDIO_DATA_DIR", "~/.orbitstudio")).expanduser()
    return base / "assistants" / tenant_id / studio_id / assistant_id
```

## 4. 单助理创建规范
```python
agent = AIAgent(
    hermes_home=hermes_home_for(tenant_id, studio_id, assistant_id),
    session_id=session_id,
    model=model,
    ephemeral_system_prompt=prompt,
)
```

要求：
- `session_id` 应包含 studio/assistant 语义，便于排障。
- 每个 assistant 独立创建 agent 实例，不复用跨 assistant 对象。

## 5. 协作组接入规范（冻结）

### 5.1 组运行接口
```python
def execute_group_run(
    tenant_id: str,
    studio_id: str,
    group_id: str,
    strategy: str = "sequential",  # sequential|parallel|coordinator
    coordinator_enabled: bool = False,
    timeout_ms: int = 120000,
) -> dict:
    ...
```

### 5.2 策略开关
- `strategy=sequential`：默认策略。
- `strategy=parallel`：并发成员执行。
- `strategy=coordinator`：启用组协调器决策执行计划。

### 5.3 协调器开关与降级
- `coordinator_enabled=False` 时忽略 `coordinator` 路径，回到 `sequential`。
- 协调器超时或失败时，自动回退 `sequential`。
- 降级事件必须进入审计日志。

## 6. 助理生命周期建议
1. 初始化：按 assistant 维度懒创建实例目录。
2. 执行：每次 run 使用对应实例。
3. 销毁：不主动清理目录，保留进化资产（memory/skills）。

## 7. 与组网同步关系
- 本地进化（memory/skills）先在本地目录生效。
- 是否同步到 Hub/Market 由 Orbit 同步策略控制，不由 Hermes 直接决定。

## 8. 语义边界
- 持久 CollabGroup 使用模型 B（多成员独立 AIAgent）。
- 模型 A 仅用于一次性临时协作说明。
- `delegate_task` 不作为跨成员协作编排机制。

## 9. 非本期范围
- Hub 远程执行器编排 Hermes 进程池。
- API 网关统一代理 Hermes runs。
