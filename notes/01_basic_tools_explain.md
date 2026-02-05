# 01_basic_tools.py 说明

下面是 `examples/01_basic_tools.py` 里每个函数的详细说明（含关键步骤与行为），按函数顺序说明。

## demo_write_tool()
- 目的：演示 `WriteTool` 如何创建一个新文件并写入内容。
- 关键流程：
  - 打印分隔线和标题，标识 Demo 1。
  - 使用 `tempfile.TemporaryDirectory()` 创建临时目录，退出 `with` 块时自动清理。
  - 组装临时文件路径 `hello.txt`。
  - 初始化 `WriteTool()`，调用 `await tool.execute(path=..., content=...)` 写入内容。
  - 根据 `result.success` 判断：
    - 成功：打印创建路径与文件内容。
    - 失败：打印 `result.error`。
- 重点：展示“用工具写文件 + 验证写入结果”的完整流程。

## demo_read_tool()
- 目的：演示 `ReadTool` 如何读取文件内容。
- 关键流程：
  - 打印分隔线和标题，标识 Demo 2。
  - 用 `tempfile.NamedTemporaryFile(..., delete=False)` 创建临时文件并写入三行文本。
  - 通过 `ReadTool().execute(path=...)` 读取内容。
  - 成功打印 `result.content`，失败打印 `result.error`。
  - `finally` 中删除临时文件，避免残留。
- 重点：展示“读文件 + 成功/失败分支 + 资源清理”的标准用法。

## demo_edit_tool()
- 目的：演示 `EditTool` 如何在已有文件中替换文本。
- 关键流程：
  - 打印分隔线和标题，标识 Demo 3。
  - 创建临时文件并写入两行包含 `Python` 的文本。
  - 打印原始内容。
  - 调用 `EditTool().execute(path=..., old_str="Python", new_str="Agent")` 进行替换。
  - 成功打印替换后内容，失败打印 `result.error`。
  - `finally` 删除临时文件。
- 重点：展示“替换文本”的最小流程与前后对比。

## demo_bash_tool()
- 目的：演示 `BashTool` 执行系统命令。
- 关键流程：
  - 打印分隔线和标题，标识 Demo 4。
  - 初始化 `BashTool()`。
  - 依次执行三条命令并检查 `result.success`：
    - `ls -la`：列出目录文件（输出截取前 200 个字符）。
    - `pwd`：输出当前工作目录。
    - `echo 'Hello from BashTool!'`：输出一行文本。
- 重点：展示“执行命令 + 判断成功 + 读取输出”的模式。

## main()
- 目的：串联并依次执行所有 demo。
- 关键流程：
  - 打印整体标题与说明。
  - 依次 `await` 调用四个 demo：
    - `demo_write_tool()`
    - `demo_read_tool()`
    - `demo_edit_tool()`
    - `demo_bash_tool()`
  - 打印“全部完成”的结束提示。
- 重点：这是异步入口，用于统一运行全部示例。

## if __name__ == "__main__":
- 目的：确保脚本被直接运行时才执行 demo。
- 行为：调用 `asyncio.run(main())` 启动异步主函数。
