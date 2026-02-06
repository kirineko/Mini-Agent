# 02_simple_agent.py 逐函数说明

本文说明文件：`/Users/kirineko/papers/agent_paper/Mini-Agent/examples/02_simple_agent.py`

这个示例的核心目标是：演示如何把 `LLMClient + Agent + Tools` 组装起来，让 Agent 根据自然语言任务自动选择并调用工具完成工作。

## 模块级内容

- `import asyncio`：用于运行异步入口函数 `main()`。
- `import tempfile`：创建临时工作目录，避免污染项目目录。
- `from pathlib import Path`：处理文件路径（检查存在、读文件、拼路径等）。
- `from mini_agent import LLMClient`：创建 LLM 调用客户端。
- `from mini_agent.agent import Agent`：Agent 主体，负责多步推理和工具调用。
- `from mini_agent.config import Config`：读取 YAML 配置（模型、API key、base URL）。
- `from mini_agent.tools import BashTool, EditTool, ReadTool, WriteTool`：给 Agent 的工具集。

---

## `async def demo_file_creation()`

函数职责：演示“Agent 驱动的文件创建任务”。

执行流程：
1. 打印标题区块，说明当前是文件创建 Demo。
2. 定位配置文件 `mini_agent/config/config.yaml`。
3. 如果配置文件不存在：
   - 打印错误提示；
   - 提示用户从 `config-example.yaml` 复制；
   - `return` 提前结束。
4. 读取配置：`config = Config.from_yaml(config_path)`。
5. 校验 API Key：
   - 条件是 key 为空，或以 `YOUR_` 开头（示例占位符）；
   - 不合法则提示并 `return`。
6. 创建临时工作区：`with tempfile.TemporaryDirectory() as workspace_dir`。
   - 这是整个 Agent 实际读写的目录。
7. 加载系统提示词 `mini_agent/config/system_prompt.md`：
   - 文件存在则读取 UTF-8 文本；
   - 不存在则使用默认提示词 `You are a helpful AI assistant that can use tools.`。
8. 初始化 LLM 客户端：
   - `api_key / api_base / model` 都来自配置。
9. 初始化工具列表：
   - `ReadTool(workspace_dir=workspace_dir)`：限制读取范围到临时工作区；
   - `WriteTool(workspace_dir=workspace_dir)`：限制写入范围到临时工作区；
   - `EditTool(workspace_dir=workspace_dir)`：限制编辑范围到临时工作区；
   - `BashTool()`：执行 bash 命令。
10. 创建 Agent：
   - 传入 `llm_client`、`system_prompt`、`tools`；
   - `max_steps=10` 限制最多推理/工具调用步数；
   - `workspace_dir` 提供工作目录上下文。
11. 定义任务 `task`：让 Agent 创建 `hello.py`，包含 `greet(name)` 并调用。
12. 打印任务文本，然后 `agent.add_user_message(task)` 把任务加入对话。
13. `result = await agent.run()`：启动 Agent 执行。
14. 成功时：
   - 打印 Agent 的最终回复；
   - 检查 `hello.py` 是否存在；
   - 存在则打印文件内容，不存在则提示“可能用了不同完成方式”。
15. 异常时：
   - 捕获异常并打印；
   - `traceback.print_exc()` 输出完整堆栈方便排错。

这个函数体现的设计点：
- 先做配置与凭据前置校验，避免无效调用。
- 用临时目录隔离运行副作用。
- 让 Agent 决定工具使用顺序，人只给任务目标。

---

## `async def demo_bash_task()`

函数职责：演示“Agent 驱动的 bash 信息收集任务”。

执行流程：
1. 打印标题区块，说明当前是 Bash Demo。
2. 检查 `config.yaml` 是否存在，不存在直接返回。
3. 读取配置并校验 API key；不合法直接返回。
4. 创建临时工作目录并打印路径。
5. 同样读取系统提示词文件，不存在则回退默认提示词。
6. 初始化 `LLMClient`。
7. 初始化工具列表（与上个函数不同点）：
   - 使用 `ReadTool`、`WriteTool`、`BashTool`；
   - 这里没有放 `EditTool`，因为任务是命令执行与结果汇总，不强调文本替换。
8. 创建 `Agent`（同样 `max_steps=10`）。
9. 定义任务 `task`：
   - 显示当前日期时间；
   - 列出当前目录所有 Python 文件；
   - 统计 Python 文件数量。
10. 把任务加入消息后调用 `await agent.run()`。
11. 成功时打印 Agent 回答；异常时打印错误。

这个函数体现的设计点：
- Agent 可用于“命令编排 + 结果解释”场景。
- 工具最小化按任务裁剪，不必每次挂全部工具。

---

## `async def main()`

函数职责：统一编排示例运行顺序。

执行流程：
1. 打印总标题与说明（强调 Agent 通过 LLM 决定工具调用）。
2. 顺序执行：
   - `await demo_file_creation()`
   - 打印两个空行分隔
   - `await demo_bash_task()`
3. 打印“全部完成”提示。

关键点：
- `main()` 只负责流程编排，不承担具体业务逻辑。
- 两个 demo 都是异步函数，所以用 `await` 串行执行。

---

## `if __name__ == "__main__":`

函数职责：脚本入口保护。

执行流程：
1. 当文件被“直接运行”时执行下面代码；
2. `asyncio.run(main())` 启动事件循环并运行 `main()`。

关键点：
- 如果此文件被其他模块 `import`，不会自动执行 demo。

---

## 补充理解：这个示例整体在演示什么

1. Agent 的标准装配顺序：`Config -> LLMClient -> Tools -> Agent -> add_user_message -> run`。
2. 工具权限边界：通过 `workspace_dir` 把文件工具限制在临时目录。
3. 任务表达方式：给目标和约束，不给具体命令，让 Agent 自主规划。
4. 最小可运行闭环：配置检查、任务执行、结果打印、异常处理都具备。
