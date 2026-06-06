# Hermes Agent 使用指南 09：高级功能篇 - 自动化任务与工作流

## 一、自动化任务概述

### 1.1 自动化任务的价值

自动化任务将重复性的工作交给 Hermes Agent 处理，释放人类的时间和精力。在日常工作中，很多任务是规则化的重复操作：定期生成报告、定时执行检查、响应特定触发条件等。这些任务虽然不复杂，但会消耗大量时间，而且人工执行容易出错或遗漏。Hermes Agent 的自动化功能让你可以用自然语言定义任务规则，然后系统会按照设定的计划自动执行这些任务。自动化不仅提高了效率，还确保了任务执行的及时性和一致性。通过合理的自动化设置，你可以将精力集中在更需要创造力和判断力的工作上，而将机械性的事务性工作交给 Agent 处理。

### 1.2 任务类型分类

Hermes Agent 支持多种类型的自动化任务。按触发方式分类，定时任务按设定的时间表执行，如每天早上生成日报；事件触发任务在特定条件满足时执行，如收到特定邮件时自动处理；手动任务由用户随时触发执行。按任务性质分类，数据收集任务从多个来源汇总信息；处理转换任务对数据进行转换或分析；报告生成任务基于收集的数据生成格式化的报告；通知提醒任务在特定条件下发送通知。按执行频率分类，单次任务只执行一次；周期任务按固定间隔重复执行；条件满足任务在条件满足时执行一次然后可以重置。

### 1.3 任务与工作流的区别

理解任务和工作流的区别有助于选择正确的自动化方式。任务是独立的执行单元，一次执行完成一个相对简单的操作，如发送通知、生成报告等。任务可以设置触发条件，但执行过程相对线性。工作流是多个任务的组合，支持条件分支、循环和并行执行等复杂逻辑，适合编排多步骤的业务流程。工作流中的每个步骤可以是一个任务或子工作流，通过连接形成完整的流程。在 Hermes Agent 中，任务使用简化的任务定义格式，工作流使用更强大的流程定义语言。选择哪种方式取决于需求的复杂度：简单任务用任务，需要流程控制时用工作流。

## 二、定时任务配置

### 2.1 基础定时任务

定时任务基于时间计划自动执行。首先定义任务：

```yaml
scheduled_tasks:
  - name: daily_report
    description: 每日生成销售报告
    schedule: "0 9 * * *"  # 每天早上9点执行
    action:
      type: chat
      prompt: |
        请基于昨天的数据生成销售报告，包括：
        1. 销售额统计
        2. 热门产品分析
        3. 销售趋势图表描述
    output:
      type: file
      path: ~/reports/daily_{date}.md
    notification:
      enabled: true
      channel: email
```

这个配置创建了一个定时任务，每天早上 9 点自动执行，生成销售报告并保存到文件。

### 2.2 Cron 表达式详解

Cron 表达式是定时任务配置的核心。标准 Cron 格式由5个字段组成：分（0-59）、时（0-23）、日（1-31）、月（1-12）、星期（0-7，0和7都是周日）。常见示例：

- `0 * * * *` 每小时整点执行
- `0 9 * * 1-5` 每个工作日早上9点执行
- `0 0 1 * *` 每月第一天午夜执行
- `30 4 * * 0` 每周日早上4:30执行
- `*/15 * * * *` 每15分钟执行一次

高级表达式支持更多选项：`@hourly` 每小时；`@daily` 每天；`@weekly` 每周；`@monthly` 每月。Hermes Agent 还支持简化的时间表达式，如 `every 5 minutes`、`every day at 9am`。

### 2.3 任务执行上下文

定时任务在执行时需要了解上下文信息。内置变量提供动态信息：`${date}` 当前日期；`${time}` 当前时间；`${datetime}` 完整日期时间；`${week}` 当前星期；`${month}` 当前月份；`${year}` 当前年份；`${env.VAR_NAME}` 环境变量值。这些变量可以在任务定义中使用，使任务具有动态性。任务还可以访问上次执行的结果，通过 `${last_run_result}` 变量可以获取上次任务的成功或失败状态及输出。这对于依赖历史状态的自动化场景特别有用。

### 2.4 任务输出与通知

任务执行完成后可以处理输出和发送通知。输出处理选项包括：保存到文件（支持 Markdown、JSON、TXT 等格式）；追加到现有文件；发送邮件；保存到数据库；调用 Webhook。通知配置：

```yaml
notification:
  enabled: true
  channels:
    - type: email
      to: ["manager@company.com"]
      subject: "定时任务执行报告"
    - type: slack
      webhook: ${env.SLACK_WEBHOOK}
      channel: "#alerts"
    - type: webhook
      url: https://api.example.com/webhook
      method: POST
```

通知内容包括执行状态、输出摘要、错误信息（如果失败）和执行耗时。通知可以配置为仅在失败时发送，避免正常执行时的干扰。

## 三、工作流配置

### 3.1 工作流基础

工作流支持复杂的多步骤自动化。使用 YAML 定义工作流：

```yaml
workflows:
  code_review:
    name: 自动化代码审查
    description: 当有新提交时自动进行代码审查
    trigger:
      type: git_hook
      event: push
      repository: github.com/org/repo
    
    steps:
      - name: fetch_changes
        action: git_pull
        params:
          branch: main
      
      - name: analyze_code
        action: code_analysis
        params:
          tools: ["eslint", "mypy", "bandit"]
      
      - name: generate_review
        type: chat
        prompt: |
          请审查以下代码变更，总结发现的问题和建议。
          代码分析结果：
          ${step.analyze_code.output}
      
      - name: post_comment
        action: github_comment
        params:
          body: "${step.generate_review.output}"
```

这个工作流定义了完整的代码审查流程。

### 3.2 条件分支

工作流支持基于条件的分支执行：

```yaml
steps:
  - name: check_pr_size
    action: check_pull_request
    output_var: pr_info
  
  - name: large_pr_check
    type: condition
    condition: "${pr_info.lines_changed} > 500"
    on_true:
      - name: add_reviewers
        action: github_add_reviewers
        params:
          reviewers: ["senior-dev-1", "senior-dev-2"]
    on_false:
      - name: add_standard_reviewer
        action: github_add_reviewers
        params:
          reviewers: ["standard-reviewer"]
```

条件表达式使用变量和比较操作符，支持复杂的逻辑组合。

### 3.3 循环与迭代

工作流支持循环处理：

```yaml
steps:
  - name: get_files
    action: list_files
    params:
      path: ./src
      pattern: "*.py"
    output_var: python_files
  
  - name: process_files
    type: loop
    items: "${python_files}"
    loop_var: file
    steps:
      - name: analyze_file
        action: analyze_python
        params:
          file: "${file}"
      
      - name: log_result
        action: log
        params:
          message: "已分析 ${file}，发现 ${step.analyze_file.issues_count} 个问题"
```

循环变量在步骤中通过 `${loop.item}` 引用，每次迭代使用集合中的不同元素。

### 3.4 并行执行

某些步骤可以并行执行以提高效率：

```yaml
steps:
  - name: parallel_checks
    type: parallel
    max_workers: 4
    steps:
      - name: check_syntax
        action: python_syntax_check
      
      - name: check_types
        action: mypy_check
      
      - name: check_security
        action: bandit_check
      
      - name: check_format
        action: black_check
  
  - name: combine_results
    action: combine_reports
    params:
      reports: "${steps.parallel_checks.results}"
```

并行执行的步骤同时开始，最后汇总结果。注意并行步骤之间不应有依赖关系。

## 四、触发器与事件

### 4.1 触发器类型

触发器定义工作流何时执行。定时触发器基于时间计划执行，如上所述。Git 触发器响应 Git 事件：

```yaml
trigger:
  type: git
  events:
    - push
    - pull_request.opened
    - pull_request.merged
  branches:
    - main
    - develop
```

Webhook 触发器响应外部 HTTP 请求：

```yaml
trigger:
  type: webhook
  path: /webhook/github
  secret: ${env.WEBHOOK_SECRET}
  events:
    - github.push
    - github.pull_request
```

文件变化触发器响应文件系统变更：

```yaml
trigger:
  type: file_watch
  path: ./data
  patterns:
    - "*.csv"
    - "*.json"
  events:
    - created
    - modified
```

### 4.2 事件过滤

触发器可以配置过滤条件，只在满足条件时执行：

```yaml
trigger:
  type: git
  events: [push]
  filter: |
    ${event.commits.length} > 0 and 
    ${event.commits[0].author.email} != "bot@example.com"
```

过滤表达式使用触发器提供的事件数据，支持比较、逻辑和字符串操作。有效的过滤可以减少不必要的任务执行，提高效率。

### 4.3 事件数据访问

工作流可以访问触发事件的数据。Git 事件提供：提交信息、作者、变更文件列表、影响行数、分支名称等。Webhook 事件提供：请求体数据、HTTP 头信息、查询参数等。文件事件提供：文件路径、操作类型、文件大小等。事件数据通过 `${trigger.event.field}` 格式在工作流中引用。使用事件数据可以在工作流中做出智能决策，如根据提交的文件类型选择不同的处理方式。

### 4.4 触发器管理

管理定时任务和工作流触发器。查看所有触发器使用 `hermes triggers list` 命令，显示触发器名称、类型、目标工作流和状态。启用或禁用触发器使用 `hermes triggers enable <name>` 和 `hermes triggers disable <name>`。触发器执行历史使用 `hermes triggers history <name>` 查看，包括执行时间、状态和结果。手动触发工作流使用 `hermes workflow run <workflow_name> [--param key=value]` 命令，常用于测试或手动执行场景。

## 五、高级工作流特性

### 5.1 子工作流与复用

子工作流允许定义可复用的工作流片段。定义子工作流：

```yaml
workflows:
  _send_notification:
    name: 发送通知（子工作流）
    inputs:
      - name: message
        type: string
        required: true
      - name: channel
        type: string
        default: email
    steps:
      - name: format
        action: format_message
        params:
          message: "${inputs.message}"
      
      - name: send
        action: send_to_channel
        params:
          channel: "${inputs.channel}"
          content: "${steps.format.output}"
```

在主工作流中调用子工作流：

```yaml
steps:
  - name: notify_success
    action: workflow:_send_notification
    params:
      message: "代码审查完成，发现 ${issues_count} 个问题"
      channel: slack
  
  - name: notify_failure
    action: workflow:_send_notification
    condition: "${status} == 'failed'"
    params:
      message: "代码审查失败，请检查日志"
      channel: email
```

子工作流提高了工作流的模块化和复用性。

### 5.2 错误处理与重试

工作流需要完善的错误处理机制：

```yaml
steps:
  - name: api_call
    action: http_request
    params:
      url: https://api.example.com/data
    error_handling:
      on_failure: log_and_continue
      max_retries: 3
      retry_delay: 5
      retry_on:
        - timeout
        - 500
        - 502
        - 503
  
  - name: critical_step
    action: important_action
    error_handling:
      on_failure: stop_workflow
      notification:
        enabled: true
        message: "工作流执行失败：${error.message}"
```

重试策略可以设置最大重试次数、延迟时间和触发重试的错误条件。

### 5.3 超时与资源限制

防止工作流执行时间过长或消耗过多资源：

```yaml
workflows:
  long_running:
    timeout: 3600  # 1小时超时
    max_memory: 512MB
    max_cpu_percent: 50
    
    steps:
      # ...
```

单个步骤也可以设置超时：

```yaml
steps:
  - name: external_api
    action: call_api
    timeout: 300  # 5分钟超时
```

超时时可以选择停止工作流或跳过该步骤继续。

### 5.4 状态管理与持久化

工作流执行状态需要持久化以支持恢复：

```yaml
workflows:
  stateful_workflow:
    persist_state: true
    state_file: .workflow/state/{workflow_id}.json
    
    steps:
      - name: step1
        checkpoint: true  # 保存检查点
      
      - name: step2
        checkpoint: true
      
      # 如果工作流中断，可以从检查点恢复
```

检查点保存工作流执行到该点时的状态，中断后可以从该点恢复而非重新开始。

## 六、脚本与编程接口

### 6.1 Python 脚本任务

除了 YAML 配置，还可以用 Python 编写复杂的任务逻辑：

```python
# tasks/custom_task.py
from hermes.automation import Task

class CustomAnalysisTask(Task):
    name = "custom_analysis"
    description = "执行自定义数据分析"
    
    def execute(self, context):
        # 获取输入参数
        data_path = context.get_param("data_path")
        output_format = context.get_param("format", "json")
        
        # 执行分析
        results = self.run_analysis(data_path)
        
        # 处理结果
        formatted = self.format_output(results, output_format)
        
        # 保存输出
        output_path = context.get_param("output")
        self.save_output(output_path, formatted)
        
        return {
            "status": "success",
            "records_processed": len(results),
            "output": output_path
        }
    
    def run_analysis(self, data_path):
        # 分析逻辑
        pass
    
    def format_output(self, results, format):
        # 格式化逻辑
        pass
```

注册任务后可以在工作流中调用。

### 6.2 Webhook 与 API 触发

通过 API 触发工作流执行：

```python
from hermes.automation import WorkflowExecutor

executor = WorkflowExecutor()
result = executor.trigger(
    workflow_name="data_processing",
    params={
        "input_file": "/data/input.csv",
        "output_format": "json"
    },
    wait_for_completion=True,
    timeout=3600
)

print(f"工作流执行结果: {result.status}")
```

Webhook 端点暴露工作流供外部调用：

```yaml
server:
  webhooks:
    enabled: true
    port: 8080
    routes:
      - path: /trigger/backup
        workflow: daily_backup
        auth: api_key
```

### 6.3 事件发布与订阅

工作流可以发布事件供其他组件订阅：

```yaml
steps:
  - name: process_order
    action: process_order_data
    output_var: order_result
  
  - name: publish_event
    action: publish_event
    params:
      topic: order_completed
      payload: |
        {
          "order_id": "${order_result.order_id}",
          "customer": "${order_result.customer}",
          "total": "${order_result.total}"
        }
```

订阅这些事件：

```yaml
subscriptions:
  - topic: order_completed
    handler: notify_shipping_team
    workflow: prepare_shipment
```

事件驱动架构实现了工作流间的松耦合通信。

## 七、调度器与监控

### 7.1 调度器配置

调度器管理所有定时任务的执行：

```yaml
scheduler:
  enabled: true
  timezone: Asia/Shanghai
  max_concurrent_tasks: 10
  queue_size: 100
  
  backfill:
    enabled: true
    max_history: 7days
  
  clustering:
    enabled: false  # 分布式部署时启用
    nodes:
      - host1:8000
      - host2:8000
```

调度器支持集群模式，多个节点协同工作提高可用性和处理能力。

### 7.2 执行监控

监控任务和工作流执行状态：

```yaml
monitoring:
  metrics:
    - task_execution_count
    - task_success_rate
    - task_average_duration
    - workflow_execution_count
    - queue_depth
  
  export:
    prometheus:
      enabled: true
      port: 9090
  
  alerts:
    - name: task_failure
      condition: "task_success_rate < 0.9"
      action: notify
  
    - name: queue_overflow
      condition: "queue_depth > 80"
      action: scale_up
```

监控指标可以导出到 Prometheus、Grafana 等监控系统。

### 7.3 执行日志与追踪

完整的执行日志支持问题排查和优化：

```yaml
logging:
  level: INFO
  format: json
  outputs:
    - type: file
      path: /var/log/hermes/executions.log
      rotation: daily
      retention: 30days
    
    - type: database
      table: execution_logs
  
  tracing:
    enabled: true
    sample_rate: 1.0  # 100%采样
    backend: jaeger
```

执行追踪记录每个步骤的输入、输出、耗时和状态，支持分布式追踪。

### 7.4 性能优化

优化任务和工作流执行性能。任务合并将多个小任务合并为批次执行，减少调度开销。并行化启用步骤级并行执行，提高吞吐量。缓存对不变化的输出结果进行缓存，避免重复计算。资源池预热保持资源池处于热状态，减少冷启动延迟。增量处理对于处理大量数据的任务，使用增量处理而非全量处理。

## 八、最佳实践

### 8.1 工作流设计建议

设计良好工作流的建议包括：保持工作流简洁，每个工作流专注于一个目标；使用子工作流分解复杂逻辑，提高可维护性；添加充分的错误处理，确保工作流在异常情况下也能优雅处理；设置合理的超时时间，避免工作流卡住；记录详细的日志，便于问题排查；使用版本控制管理工作流定义，便于追踪变更；定期审查和优化工作流，适应需求变化。

### 8.2 自动化场景案例

常见自动化场景示例。每日站会准备：定时收集项目数据，生成站会报告并发送给团队。代码质量检查：Git 提交时自动运行 linter、formatter 和测试，生成检查报告。监控告警处理：收到告警时自动收集相关日志，进行初步分析，生成处理建议。数据备份与同步：定时备份数据库，同步到异地存储，验证备份完整性。客户反馈汇总：定期从多个渠道收集反馈，进行情感分析，生成汇总报告。

### 8.3 故障排查指南

工作流执行失败时的排查步骤：查看执行日志，从最后一步开始向前排查；检查触发器事件数据是否符合预期；验证输入参数和前置条件是否满足；检查依赖的服务或 API 是否可用；查看资源使用情况，是否触发了限制；使用手动触发复现问题；逐步执行每个步骤定位故障点。

## 九、总结

本篇我们全面学习了 Hermes Agent 的高级功能，重点是自动化任务和工作流。我们从自动化任务的价值和类型开始，理解了任务与工作流的区别。定时任务配置章节详细说明了 Cron 表达式、任务上下文和输出通知。工作流配置部分展示了基础定义、条件分支、循环和并行执行等核心特性。触发器与事件部分介绍了多种触发方式和事件过滤。高级特性章节包括子工作流复用、错误处理、资源限制和状态持久化。脚本与编程接口展示了如何用 Python 和 API 扩展自动化能力。调度器与监控部分确保自动化可靠运行。最后给出了最佳实践和故障排查指南。掌握这些内容后，你应该能够构建强大的自动化解决方案，将重复性工作交给 Hermes Agent 处理。下一篇我们将进入精通篇，学习最佳实践和故障排除。
