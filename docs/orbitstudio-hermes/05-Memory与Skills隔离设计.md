# 05-Memory 与 Skills 隔离设计

## 1. 目标
确保最核心的两类“可进化数据”在多助理并发下不串写：
- Memory（MEMORY.md / USER.md）
- Skills（SKILL.md + supporting files）

## 2. Memory 设计

### 2.1 MemoryStore
改造点：
1. `MemoryStore.__init__` 增加 `hermes_home` 可选参数。
2. 新增实例方法 `_get_memory_dir()`，目录来源为：
- `self._hermes_home / "memories"`（优先）
- `get_hermes_home() / "memories"`（回退）
3. `_path_for` 从静态方法改为实例方法。
4. `load_from_disk/save_to_disk/update` 全链路改用实例目录方法。

### 2.2 AIAgent Memory 初始化
- AIAgent 初始化 MemoryStore 时必须透传 `self._hermes_home`。
- 不允许通过全局单例 MemoryStore 复用跨实例状态。

## 3. Skills 设计

### 3.1 skill_utils
下列函数新增可选 `hermes_home` 参数并透传：
- `get_disabled_skill_names`
- `get_external_skills_dirs`
- `get_all_skills_dirs`
- `discover_all_skill_config_vars`
- `resolve_skill_config_values`

读取配置时规则：
- 传入 `hermes_home` 时读取 `{hermes_home}/config.yaml`
- 否则保持原 `get_config_path()` 行为

### 3.2 skills_tool 与 skill_manager_tool
问题：模块级 `SKILLS_DIR = get_hermes_home() / "skills"` 会固定路径。

改造要求：
1. 移除对模块级 `SKILLS_DIR` 作为唯一路径源的依赖。
2. 所有读写路径改为运行时解析（可接收 `hermes_home`）。
3. 保留测试兼容：若现有测试 monkeypatch `SKILLS_DIR`，需提供平滑过渡。

### 3.3 skill_commands
- 不再假设全局单一 `SKILLS_DIR`。
- 调用 `skill_view`、扫描命令时支持实例目录上下文。

## 4. 隔离边界
本文件冻结的边界：
- 任何技能创建、修改、删除均只作用于当前实例 `hermes_home`。
- 禁止“找不到实例目录时自动写全局 skills 目录”。

## 5. 安全性与并发
- Memory 文件写入继续保留文件锁与原子写，避免并发损坏。
- Skills 文件写入继续保留 path security 校验，避免路径穿越。
