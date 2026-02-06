# 03_session_notes.py 逐函数详细说明

本文对应源码：`/Users/kirineko/papers/agent_paper/Mini-Agent/examples/03_session_notes.py`

这个示例的目标是展示 Session Note（会话记忆）工具如何工作，包括：
- 直接调用工具进行记录与回忆。
- 在 Agent 场景中跨会话保留上下文。

## 模块级结构

- `asyncio`：用于运行异步入口函数。
- `json`：用于读取和格式化记忆文件内容。
- `tempfile`：创建临时文件和临时目录，避免污染项目目录。
- `Path`：路径处理和文件读写。
- `LLMClient`：调用模型的客户端。
- `Agent`：负责多步推理和工具调用。
- `Config`：从 YAML 加载模型配置和 API 信息。
- `BashTool, ReadTool, WriteTool`：通用工具。
- `SessionNoteTool, RecallNoteTool`：会话记忆核心工具。

---

## `async def demo_direct_note_usage()`

函数职责：不经过 Agent，直接演示记忆工具的最小闭环。

执行流程：
1. 打印 Demo 标题。
2. 使用 `tempfile.NamedTemporaryFile(..., delete=False, suffix=".json")` 创建临时记忆文件，拿到 `note_file` 路径。
3. 在 `try` 块中初始化两个工具：
   - `record_tool = SessionNoteTool(memory_file=note_file)`
   - `recall_tool = RecallNoteTool(memory_file=note_file)`
4. 连续调用 `record_tool.execute(...)` 写入三条结构化记忆：
   - 用户背景（`user_info`）
   - 项目信息（`project_info`）
   - 用户偏好（`user_preference`）
5. 调用 `recall_tool.execute()`：不带过滤条件，回忆全部记忆。
6. 调用 `recall_tool.execute(category="user_preference")`：按类别过滤，只回忆偏好类。
7. 直接读取 JSON 记忆文件并 `json.dumps(..., indent=2, ensure_ascii=False)` 打印，便于观察底层存储格式。
8. 在 `finally` 中 `Path(note_file).unlink(missing_ok=True)` 删除临时文件，保证清理。

这个函数验证了三件事：
- 记忆可写入。
- 记忆可全量回忆。
- 记忆可按类别检索。

---

## `async def demo_agent_with_notes()`

函数职责：演示 Agent 如何借助 Session Note 实现跨会话记忆。

执行流程（前置阶段）：
1. 打印 Demo 标题。
2. 检查 `mini_agent/config/config.yaml` 是否存在；不存在就退出。
3. 加载 `Config.from_yaml(...)`。
4. 校验 API key 是否有效（空值或占位符 `YOUR_` 视为无效）。
5. 创建临时工作目录 `workspace_dir`，整个示例在该目录运行。
6. 加载系统提示词 `mini_agent/config/system_prompt.md`，不存在则回退默认提示词。
7. 追加 `note_instructions` 到 `system_prompt`，明确告诉模型如何使用：
   - `record_note` 保存重要信息。
   - `recall_notes` 恢复历史上下文。
   - 推荐类别如 `user_info / user_preference / project_info / decision`。
8. 初始化 `LLMClient`。
9. 定义记忆文件路径 `memory_file = Path(workspace_dir) / ".agent_memory.json"`。
10. 初始化工具列表，包含普通工具和记忆工具：
   - `ReadTool`、`WriteTool`、`BashTool`
   - `SessionNoteTool(memory_file=...)`
   - `RecallNoteTool(memory_file=...)`

执行流程（会话 1）：
1. 打印会话 1 标题：目标是“教会 Agent 用户偏好并要求记住”。
2. 创建第一个 Agent 实例 `agent1`：
   - `max_steps=15`，给更高步骤预算以容纳“理解+记录+写文件”。
3. 构造 `task1`：包含用户身份、项目、技术栈、编码偏好，并明确要求“记住这些信息”，同时创建 `README.md`。
4. `agent1.add_user_message(task1)` 后执行 `await agent1.run()`。
5. 运行成功后：
   - 打印 Agent 回复。
   - 检查 `memory_file` 是否存在。
   - 若存在，读取 JSON 并打印“记录条目数量+摘要”；不存在则给出提醒。
6. 若异常，打印错误并 `return` 提前结束该 Demo。

执行流程（会话 2）：
1. 打印会话 2 标题：用新实例模拟“新的对话轮次”。
2. 创建第二个 Agent 实例 `agent2`（不是复用 `agent1`），`max_steps=10`。
3. 发送 `task2`：询问“你还记得我是谁、在做什么项目、编码风格偏好吗”。
4. 执行 `await agent2.run()`。
5. 运行成功后打印回复，并总结关键结论：
   - 会话 1 记录了重要信息。
   - 会话 2 通过记忆工具恢复信息。
   - 持久化媒介是共享的记忆文件，而不是进程内变量。
6. 异常则打印错误。

这个函数的核心演示点：
- “跨会话记忆”依赖外部持久化文件，而不是单个 Agent 实例生命周期。
- 只要新 Agent 使用同一 `memory_file` 并拥有 recall 工具，就能恢复上下文。

---

## `async def main()`

函数职责：统一编排两个 Demo 的执行顺序。

执行流程：
1. 打印总标题和说明，强调 Session Notes 的价值。
2. 顺序执行：
   - `await demo_direct_note_usage()`
   - 打印空行
   - `await demo_agent_with_notes()`
3. 打印“全部完成”提示。

设计意图：
- 先讲“工具级能力”（直接调用），再讲“Agent 级能力”（跨会话协作）。

---

## `if __name__ == "__main__":`

函数职责：脚本入口控制。

执行流程：
1. 仅当该文件被直接运行时，执行 `asyncio.run(main())`。
2. 如果该文件被 import，不会自动运行 Demo。

---

## 这份示例体现的工程模式

1. 前置校验模式：配置文件和 API key 先校验，减少运行时歧义。
2. 隔离模式：临时工作区和临时记忆文件让示例可重复执行且无残留。
3. 提示词约束模式：在系统提示词中显式写入“何时记录/回忆”的行为规范。
4. 跨实例状态共享模式：通过 `memory_file` 把“记忆状态”从 Agent 实例中解耦。
