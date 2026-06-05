# Plan: 撰写 Hermes Agent 工具系统深度代码分析文档

## Summary
撰写关于 Hermes Agent 工具系统的深度代码分析文档，保存到 `/workspace/ReadCode/03-工具系统.md`。文档需遵循总分总结构，包含至少4张Mermaid图表，引用关键代码片段和文件路径，深入分析设计原因。

## Current State Analysis
- 目标文件 `/workspace/ReadCode/03-工具系统.md` 不存在（之前生成失败）
- 已阅读所有关键参考文件，理解了工具系统的完整架构
- `ReadCode/` 目录已存在，包含其他编号的分析文档

## Key Files Read
1. `/workspace/tools/registry.py` — 工具注册中心：ToolEntry、ToolRegistry、AST自动发现、check_fn TTL缓存、dispatch分发
2. `/workspace/model_tools.py` — 工具编排层：get_tool_definitions()、handle_function_call()、_run_async异步桥接、coerce_tool_args类型强制转换、schema动态覆盖
3. `/workspace/toolsets.py` — 工具集系统：TOOLSETS字典、_HERMES_CORE_TOOLS、resolve_toolset递归解析、平台特定工具集
4. `/workspace/tools/terminal_tool.py` — 终端工具：多后端支持(local/docker/modal/ssh/daytona/singularity)
5. `/workspace/tools/delegate_tool.py` — 子代理委派：DELEGATE_BLOCKED_TOOLS、ThreadPoolExecutor并发
6. `/workspace/tools/file_tools.py` — 文件操作：read_file/write_file/patch/search_files
7. `/workspace/tools/web_tools.py` — Web工具：多后端(Exa/Firecrawl/Parallel/Tavily)
8. `/workspace/tools/browser_tool.py` — 浏览器工具：多后端(Browserbase/Browser Use/本地Chromium)
9. `/workspace/tools/lazy_deps.py` — 懒加载依赖：LAZY_DEPS白名单、ensure()按需安装、供应链安全
10. `/workspace/tools/approval.py` — 审批机制：DANGEROUS_PATTERNS、HARDLINE_PATTERNS、smart approval
11. `/workspace/tools/environments/base.py` — 终端后端基类：BaseEnvironment ABC

## Proposed Changes

### File: `/workspace/ReadCode/03-工具系统.md`
Write a comprehensive Chinese-language analysis document with the following structure:

**【总】开篇**
- 一段话概括工具系统三层架构
- 全景架构Mermaid图（注册中心→编排层→工具集）

**【分】展开 6 个分析模块**

1. **工具注册中心** (registry.py)
   - ToolEntry 数据结构设计（__slots__优化）
   - registry.register() 注册接口与冲突防护（override机制）
   - AST扫描自动发现：_module_registers_tools + discover_builtin_tools
   - check_fn TTL缓存机制（30秒、线程安全）
   - generation计数器与快照隔离
   - Mermaid流程图：工具注册与发现流程

2. **工具编排层** (model_tools.py)
   - get_tool_definitions()：schema生成、缓存策略、动态schema覆盖
   - handle_function_call()：调用分发、参数类型强制转换、Tool Search桥接
   - _run_async异步桥接：持久事件循环、worker线程隔离、超时取消
   - 错误净化与安全防护
   - Mermaid时序图：工具调用完整流程

3. **工具集系统** (toolsets.py)
   - TOOLSETS字典定义与组合机制（includes递归解析）
   - _HERMES_CORE_TOOLS默认工具集
   - 平台特定工具集（hermes-cli/telegram/discord/...）
   - resolve_toolset递归解析与环检测
   - 插件/MCP工具集动态发现

4. **核心工具实现分析**
   - terminal_tool.py：6种后端、中断处理、磁盘警告
   - delegate_tool.py：子代理委派、DELEGATE_BLOCKED_TOOLS、审批回调
   - file_tools.py：文件读写/搜索、大小限制、敏感路径保护
   - web_tools.py：多后端搜索(Exa/Firecrawl/Parallel/Tavily)
   - browser_tool.py：多后端浏览器(Browserbase/Browser Use/本地)
   - Mermaid分类图：核心工具分类

5. **终端后端系统** (environments/)
   - BaseEnvironment ABC统一接口
   - 6种后端实现：local/docker/ssh/modal/daytona/singularity
   - 会话快照与CWD持久化
   - 活动回调与中断检测
   - Mermaid架构图：终端后端层次结构

6. **懒加载依赖** (lazy_deps.py)
   - LAZY_DEPS白名单与供应链安全模型
   - ensure()按需安装策略
   - _spec_is_safe安全校验
   - ensure_and_bind全局重绑定模式
   - 审批系统(approval.py)关键设计

**【总】收束**
- 三层架构的设计哲学总结
- 关键设计决策回顾
- 扩展性分析

## Assumptions & Decisions
- 文档使用中文撰写
- Mermaid图表至少4张（实际计划5张）
- 代码片段从已读文件中精确引用
- 分析"为什么这样设计"而非仅描述"是什么"

## Verification Steps
1. 写入文件后，使用 Read 工具确认文件存在且内容完整
2. 检查文件大小是否合理（预期 > 10KB）
3. 验证Mermaid图表语法正确性
