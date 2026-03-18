# 04_full_agent.py 逐函数详细说明

本文对应源码：`/Users/kirinekoclaw/github/Mini-Agent/examples/04_full_agent.py`

这个示例的目标是演示一个“全量配置”的 Mini Agent，包括：
- 基础文件工具与 Bash 工具。
- Session Note 记忆工具。
- MCP 工具加载。
- 多轮对话模式。

它比前面的示例更接近真实 CLI 的完整能力组合，重点不再是单个工具，而是“完整 Agent 运行时”的装配方式。

---

## 模块级结构

- `asyncio`：作为脚本入口的异步运行器。
- `tempfile`：创建临时工作目录，避免污染仓库本身。
- `Path`：处理配置文件、提示词文件和工作目录中的文件路径。
- `LLMClient`：负责与底层模型接口通信。
- `Agent`：核心执行器，负责多步推理、消息管理和工具调用。
- `Config`：从 `config.yaml` 中读取模型和工具相关配置。
- `BashTool, EditTool, ReadTool, WriteTool`：构成基础工具集。
- `load_mcp_tools_async`：按配置加载 MCP 工具。
- `RecallNoteTool, SessionNoteTool`：提供会话记忆能力。

---

## `async def demo_full_agent()`

函数职责：演示“完整能力开启”的 Agent 如何被组装和运行。

执行流程（初始化阶段）：
1. 打印标题，说明当前是 Full Agent Demo。
2. 定位配置文件 `mini_agent/config/config.yaml`。
3. 如果配置文件不存在：
   - 打印错误；
   - 提示从 `config-example.yaml` 复制；
   - 直接返回。
4. 使用 `Config.from_yaml(config_path)` 加载配置。
5. 检查 API key 是否可用：
   - 如果为空或仍是占位值（以 `YOUR_` 开头），则提示并返回。
6. 通过 `tempfile.TemporaryDirectory()` 创建临时工作区。
7. 加载系统提示词 `mini_agent/config/system_prompt.md`：
   - 如果存在，则读取文件内容；
   - 否则回退到简单默认提示词。
8. 构造额外的 `note_instructions` 并拼接到系统提示词尾部。
   - 作用是明确告诉模型：有 `record_note` 和 `recall_notes` 两个工具；
   - 应该把重要事实、决策和上下文写入记忆。
9. 初始化 `LLMClient`，从配置读取：
   - `api_key`
   - `api_base`
   - `model`

执行流程（工具装配阶段）：
1. 初始化基础工具列表：
   - `ReadTool(workspace_dir=workspace_dir)`
   - `WriteTool(workspace_dir=workspace_dir)`
   - `EditTool(workspace_dir=workspace_dir)`
   - `BashTool()`
2. 打印“已加载 4 个基础工具”。
3. 计算记忆文件路径：`memory_file = Path(workspace_dir) / ".agent_memory.json"`。
4. 追加两个 Session Note 工具：
   - `SessionNoteTool(memory_file=str(memory_file))`
   - `RecallNoteTool(memory_file=str(memory_file))`
5. 打印“已加载 2 个记忆工具”。
6. 尝试加载 MCP 工具：
   - `mcp_tools = await load_mcp_tools_async(config_path="mini_agent/config/mcp.json")`
   - 如果成功且非空，则把这些工具追加到 `tools` 中；
   - 如果为空，打印 MCP 未配置或已禁用；
   - 如果抛异常，打印错误但不中断整个 demo。

这里有一个很重要的设计点：
- MCP 工具属于“可选增强能力”，示例允许它加载失败，但不影响基础 Agent 创建。

执行流程（Agent 创建与任务执行）：
1. 使用完整工具列表创建 `Agent`：
   - `llm_client=llm_client`
   - `system_prompt=system_prompt`
   - `tools=tools`
   - `max_steps=config.agent.max_steps`
   - `workspace_dir=workspace_dir`
2. 打印 Agent 当前拥有的工具总数。
3. 构造一个复杂任务 `task`，要求 Agent：
   - 创建 `calculator.py`
   - 创建 `README.md`
   - 用 Bash 测试脚本
   - 把项目关键信息记录到 Session Notes
4. 打印任务正文。
5. 调用 `agent.add_user_message(task)` 把任务加入消息历史。
6. `result = await agent.run()` 启动 Agent 执行。

执行流程（结果展示阶段）：
1. 成功时打印 Agent 最终回复。
2. 遍历工作目录里所有非隐藏文件，逐个打印内容预览：
   - 每个文件最多显示前 20 行；
   - 超出部分用 `... (truncated)` 表示截断。
3. 如果记忆文件存在，则读取 JSON 并打印：
   - 记录总条数；
   - 每条记录的类别和内容摘要。
4. 如果整个运行过程出错，则打印异常并输出 `traceback`。

这个函数整体在演示：
1. 一个完整 Agent 的标准装配顺序。
2. 基础工具、记忆工具、MCP 工具如何并存。
3. 复杂任务可以拆成“写文件 + 测试 + 记录上下文”三个能力层次。

---

## `async def demo_interactive_mode()`

函数职责：演示一个简化版的多轮交互，而不是一次性单任务执行。

执行流程（初始化阶段）：
1. 打印标题，说明当前是交互式 Demo。
2. 强调这里只是示意，生产环境应使用 `mini-agent` CLI。
3. 检查 `mini_agent/config/config.yaml` 是否存在，不存在则提示并返回。
4. 加载配置并校验 API key。

执行流程（Agent 装配阶段）：
1. 创建临时工作目录。
2. 这里使用了一个更简单的系统提示词：
   - `You are a helpful assistant with access to tools.`
3. 初始化 `LLMClient`。
4. 初始化一个更精简的工具集：
   - `WriteTool`
   - `ReadTool`
   - `BashTool`
5. 创建 `Agent`：
   - `max_steps=20`
   - 仍然传入 `workspace_dir`

这里的设计意图很明确：
- 这个 Demo 不是为了展示“所有能力”，而是为了展示“同一 Agent 在多轮消息历史下连续工作”。

执行流程（多轮对话阶段）：
1. 定义 `conversations` 列表，共三轮消息：
   - 第 1 轮：创建 `data.txt`，内容是 1 到 5。
   - 第 2 轮：读取刚才创建的文件并总结内容。
   - 第 3 轮：使用 Bash 统计文件行数。
2. 用 `for i, message in enumerate(conversations, 1)` 依次执行每一轮：
   - 打印当前轮次；
   - 打印用户消息；
   - `agent.add_user_message(message)` 把该轮输入加入已有上下文；
   - `result = await agent.run()` 让 Agent 在历史上下文基础上继续工作；
   - 打印 Agent 的回复。
3. 如果某一轮异常，则打印错误并 `break` 终止后续轮次。

这个函数展示的关键点：
1. Agent 实例被复用，消息上下文会累积。
2. 前一轮创建的文件可以成为后一轮的上下文和操作对象。
3. 工具调用能力和对话记忆可以在同一个 Agent 实例里自然叠加。

---

## `async def main()`

函数职责：统一编排 Full Agent 示例和多轮交互示例。

执行流程：
1. 打印总标题 `Full Agent Examples`。
2. 列出本脚本要展示的能力：
   - 基础工具
   - Session memory
   - MCP tools
   - Multi-turn conversations
3. 顺序执行：
   - `await demo_full_agent()`
   - 打印两个空行
   - `await demo_interactive_mode()`
4. 打印全部完成提示，并建议下一步直接运行：
   - `mini-agent`

设计意图：
- 前半部分展示“完整功能装配”。
- 后半部分展示“多轮上下文延续”。
- 最后把读者引导到真正的交互式 CLI。

---

## `if __name__ == "__main__":`

函数职责：脚本入口保护。

执行流程：
1. 当文件被直接执行时，调用 `asyncio.run(main())`。
2. 当它被其他模块导入时，不会自动运行 Demo。

---

## 这份示例体现的工程模式

1. 配置驱动模式：模型参数、MCP 超时、工作方式都来自配置文件，而不是硬编码。
2. 能力分层模式：基础工具、记忆工具、MCP 工具可以按层追加，逐步增强 Agent。
3. 降级容错模式：MCP 加载失败不会让整个示例不可运行。
4. 任务复合模式：一个任务里同时包含文件生成、命令执行和状态记录。
5. 上下文延续模式：通过复用同一个 Agent，在多轮对话中逐步推进任务。
