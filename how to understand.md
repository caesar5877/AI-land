这份 CLI.md 不是普通的命令手册。它更像是一份 AI Agent 执行任务时必须遵守的工作协议。

Flowtrace 的目标是：不要让 Agent 只在聊天窗口中连续输出一大段内容，而是把复杂任务拆成可追踪、可恢复、可检查、可重复执行的步骤。每一步的结果写入文件，并通过 Git 保存历史。官方将它描述为：把 Agent 的工作变成可以 follow、check 和 reuse 的 steps，而不是最终消失在聊天记录中的文本流。 

⸻

1. 用一个比喻理解 Flowtrace

你可以把 Flowtrace 理解成：

Jira Workflow + Git History + 文件化的 Agent State + 一个可视化 Dashboard

假设你让 AI 帮你排查一个 Kafka consumer lag 问题。

普通 AI Agent 的工作方式通常是：

读取日志
→ 分析代码
→ 猜测原因
→ 修改配置
→ 执行测试
→ 输出结论

这些过程都隐藏在聊天中。如果 conversation 被 compact，或者你开启新的 session，AI 很容易失去上下文。

Flowtrace 会把它变成明确的工作流：

collect_logs
    ↓
analyze_consumer_lag
    ↓
inspect_thread_pool
    ↓
propose_fix
    ↓
verify_fix
    ↓
deliver_report

每个步骤都有自己的文件夹、状态和产出物。例如：

runs/20260608-001/
├── state.json
├── collect_logs/
│   └── kafka-lag-summary.md
├── analyze_consumer_lag/
│   └── analysis.md
├── propose_fix/
│   └── fix-plan.md
└── verify_fix/
    └── benchmark-result.csv

如果你发现第二步分析错误，只需要重新运行第二步，以及依赖它的后续步骤。第一步收集日志的结果可以保留，不需要从头再来。

官方将这种特性称为 steerable：修正一个步骤，只重新执行依赖它的下游步骤，其余结果保留。 

⸻

2. 核心概念：Trace 和 Run 的区别

理解这两个概念后，整个 CLI 文档就比较容易读了。

Trace：静态工作流模板

trace.json 定义任务应该怎么做。

它类似于 Jenkins Pipeline、GitHub Actions Workflow 或 DAG：

{
  "id": "troubleshoot-kafka-lag",
  "title": "Troubleshoot Kafka Consumer Lag",
  "steps": {
    "collect_logs": {
      "name": "Collect logs",
      "does": "Collect Kafka lag metrics and application logs",
      "from_steps": [],
      "assets": ["lag-summary.md"]
    },
    "analyze_root_cause": {
      "name": "Analyze root cause",
      "does": "Identify the most likely cause",
      "from_steps": ["collect_logs"],
      "assets": ["root-cause.md"]
    }
  },
  "deliverable": {
    "description": "Final troubleshooting report",
    "assets": ["analyze_root_cause/root-cause.md"]
  }
}

这里的 from_steps 表示依赖关系。

Flowtrace 把步骤组织成 DAG，也就是有向无环图。每个 Trace 的根目录下都有一个 trace.json，它声明 steps、依赖关系和最终 deliverable。 

⸻

Run：某一次实际执行

Trace 是模板，Run 是一次具体执行。

例如，同一个 Kafka troubleshooting trace 可以执行多次：

runs/
├── 20260608-101500/
├── 20260609-093000/
└── 20260612-142000/

每一次 Run 都有自己的：

state.json
step outputs
replies
deliverable

创建新的 Run：

flowtrace run new --name "Investigate Kafka lag in arsqa"

文档规定，一个 Run 位于：

<trace_root>/runs/<run_id>/

并由 flowtrace run new 自动创建。 

⸻

3. 最重要的文件：state.json

state.json 是当前 Run 的状态中心。

文档明确称它为 single source of truth。每当 CLI 更新状态时，它都会原子化写入这个文件，并创建一次 Git commit。 

示例：

{
  "name": "Investigate Kafka lag in arsqa",
  "steps": {
    "collect_logs": {
      "status": {
        "kind": "done",
        "message": "Collected Splunk logs and Kafka lag metrics"
      },
      "assets": ["lag-summary.md"]
    },
    "analyze_root_cause": {
      "status": {
        "kind": "running",
        "message": "Checking consumer thread usage"
      },
      "assets": []
    }
  }
}

这意味着：即使聊天被 compact，新的 Agent 只需要读取 trace.json 和当前 Run 的 state.json，就可以恢复工作进度。

⸻

4. 一个 Step 有哪些状态？

每一个 Step 都有状态机：

idle
  ↓
running
  ↓
done

此外，还有两个异常状态：

blocked
error

完整状态如下：

状态	含义
idle	尚未开始
running	正在执行
done	已经完成
blocked	被外部依赖阻塞
error	执行失败

更新状态的命令：

flowtrace step <step_id> <status>

例如：

flowtrace step collect_logs running \
  --message "Collecting Kafka metrics from arsqa"

完成以后：

flowtrace step collect_logs done \
  --asset lag-summary.md \
  --message "Lag metrics collected"

blocked 和 error 必须提供 --message，因为 Agent 必须说明为什么无法继续。 

例如：

flowtrace step verify_fix blocked \
  --message "Waiting for partner team to deploy the producer"

这非常适合你的日常工作，因为你经常有类似情况：

Kafka SaaS resources 已经 onboard
但是 E2E test 等待 Huan 的 producer
Splunk ACM 等待 Bhaskar
component testing 等待 Rohit

这些都可以记录为 blocked，而不是消失在聊天记录中。

⸻

5. Asset：步骤真正产生的文件

Asset 是一个 Step 的正式输出文件。

例如：

mkdir -p "runs/$RUN/collect_logs"
echo "Kafka lag peaked at 50k messages" \
  > "runs/$RUN/collect_logs/lag-summary.md"
flowtrace step collect_logs done \
  --asset lag-summary.md

注意顺序：

先创建文件
→ 再通过 --asset 声明它

Flowtrace 不会替你创建文件。如果文件不存在，命令会失败。文档明确要求：资产必须在执行 CLI 命令时已经存在于磁盘上。 

在 Step 命令中：

--asset lag-summary.md

路径是相对于当前 Step 的目录。

但是在 deliverable 中：

flowtrace deliverable done \
  --asset "collect_logs/lag-summary.md"

路径是相对于整个 Run 的目录。

⸻

6. Reply：Agent 输出给人的结构化进度报告

Reply 是 Agent 在执行过程中写出的结构化总结。

最简单的 Reply：

{
  "headline": "Kafka lag analysis completed",
  "status": "complete"
}

提交方式：

flowtrace reply < reply.json

Reply 可以绑定到某个 Step：

{
  "headline": "Consumer lag is caused by slow webhook calls",
  "status": "complete",
  "checkpoint": {
    "step_id": "analyze_root_cause"
  },
  "findings": [
    {
      "title": "Blocking I/O",
      "detail": "Webhook calls occupy consumer threads for up to 3 seconds."
    }
  ],
  "suggestions": [
    "Run a Java 21 virtual-thread benchmark",
    "Compare throughput before and after the change"
  ]
}

Reply 的核心字段包括：

字段	用途
headline	一句话结论
status	partial、complete、blocked 或 error
checkpoint	这条 Reply 属于哪个 Step
support	支撑点
findings	结构化发现
suggestions	建议的下一步
evidence	证据文件
takeaway	总结

headline 和 status 是必需字段，其他字段是可选的。 

⸻

7. Evidence：让结论可以验证

Reply 中可以引用证据：

{
  "headline": "Virtual threads improved throughput",
  "status": "complete",
  "evidence": [
    {
      "type": "table",
      "title": "Before and after benchmark",
      "columns": ["Mode", "Events/sec", "P95 latency"],
      "rows": [
        ["Platform threads", "2200", "480 ms"],
        ["Virtual threads", "3900", "260 ms"]
      ]
    },
    {
      "type": "document",
      "path": "verify_fix/benchmark-result.md",
      "title": "Benchmark report"
    }
  ]
}

支持的 Evidence 类型包括：

figure
document
table
comparison
check
citation
appendix

对于包含文件路径的证据，Flowtrace 会验证文件是否真实存在。如果路径不存在，Reply 无法提交。 

这正是它和普通聊天的重要区别：

普通聊天：AI 说它已经验证过
Flowtrace：AI 必须留下文件、表格、截图或检查结果

⸻

8. 为什么每个 CLI 操作都会自动 Git commit？

Flowtrace 的设计理念是：

任何一次状态变化，都应该可以审计和回滚。

例如：

flowtrace step collect_logs running
flowtrace step collect_logs done --asset lag-summary.md
flowtrace reply < reply.json
flowtrace deliverable done --asset analyze_root_cause/root-cause.md

每执行一次写操作，Flowtrace 都会创建一个 Git commit。提交内容严格限制为它声明要修改的文件，例如：

state.json
正式 asset
reply 文件
reply 引用的 evidence 文件

没有声明的 scratch 文件不会被提交。官方说明 Git history 就是 audit trail，并且 Web UI 可以查看不同时间点的状态。 

例如：

git log --oneline

可能看到：

a91f... run/20260608: start
b13c... collect_logs: running
c71a... collect_logs: done
d81b... reply: collect_logs (0001)
e22f... deliverable: done

⸻

9. 如何重新执行一个步骤？

假设你的工作流是：

collect_logs
    ↓
analyze_root_cause
    ↓
propose_fix
    ↓
verify_fix

后来你发现 analyze_root_cause 的分析有问题。

你可以重新进入这个步骤：

flowtrace step analyze_root_cause running \
  --message "Re-evaluating thread usage after reviewing webhook latency"

查找所有受影响的后续步骤：

flowtrace show --downstream analyze_root_cause

输出可能是：

propose_fix
verify_fix

然后按照顺序重新执行它们。

Flowtrace 不会自动重新执行下游步骤。它只负责告诉 Agent 哪些 Step 可能已经过期。是否重新执行，由 Agent 自己负责。文档刻意没有设计自动 stale 标记。 

这是一项重要限制。

⸻

10. 最常用的命令可以分成四组

查看 Trace

flowtrace show
flowtrace show --fmt json
flowtrace show --fmt ascii
flowtrace show --fmt mermaid
flowtrace validate
flowtrace lint

用途：

查看 DAG
检查 trace.json 是否有效
查看 step_id

⸻

管理 Run

flowtrace run new --name "Kafka troubleshooting"
flowtrace run list
flowtrace run show
flowtrace run pause
flowtrace run resume
flowtrace run abort
flowtrace run rename "Kafka lag investigation"

注意：

flowtrace run abort

只是写入一个 advisory flag，并不是强制锁定。即使 Run 被标记为 aborted，后续写操作仍然可以成功。 

⸻

更新 Step 和 Deliverable

flowtrace step <step_id> running
flowtrace step <step_id> done --asset <file>
flowtrace step <step_id> blocked --message <reason>
flowtrace step <step_id> error --message <reason>
flowtrace deliverable done \
  --asset <step_id>/<file>

⸻

写入 Reply 和启动 Web UI

flowtrace reply < reply.json
flowtrace serve --open

Web UI 用于查看 DAG、每个节点的状态、输出文件以及 Git 历史。README 给出的基础启动方式是：

flowtrace serve

默认打开本地 Web 页面。 

⸻

11. 文档中的最小示例是什么意思？

官方示例只有一个 Step：

say_hello

执行逻辑：

RUN=$(flowtrace run new --name "first hello" | tail -1)
flowtrace step say_hello running \
  --message "drafting greeting"
mkdir -p "runs/$RUN/say_hello"
echo "hello world" > "runs/$RUN/say_hello/hello.txt"
flowtrace step say_hello done \
  --asset hello.txt \
  --message "one greeting emitted"
flowtrace deliverable done \
  --asset "say_hello/hello.txt" \
  --message "greeting delivered"

你可以把它理解为：

创建 Jira ticket
→ 将状态改为 In Progress
→ 创建正式产出文件
→ 将 ticket 改为 Done
→ 将文件标记为最终交付物

官方文档还通过 git log --oneline 展示：每一次 CLI 写操作都会对应一个 Git commit。 

⸻

12. 它和你之前讨论的 Agent State、handoff 文件有什么关系？

你之前在考虑：

如何让 Copilot 在 conversation compact 以后不丢失上下文？
如何自动保存 AI state？
如何让后续 skill 使用前一个 skill 的结果？

Flowtrace 与你的目标高度相关，但它解决的是其中一个特定层面。

你的问题	Flowtrace 是否解决？
保存任务进度	是
保存步骤产出物	是
保存步骤依赖关系	是
支持中断后恢复	是
防止 compact 后完全失忆	部分解决
自动执行所有步骤	否
自动调用 sub-agent	否
自动触发保存 hook	否
替代 MCP memory server	否
替代长期知识库	否

Flowtrace 更接近：

为 Agent 提供一个持久化、可审计的执行框架。

它不是 Agent 本身，也不是自动化调度器。真正执行工作的仍然是 Claude Code、Codex、Cursor，或者人工调用脚本。README 明确表示它与已有 Agent 配合使用，并列出了 Claude Code、Codex 和 Cursor。 

对于 GitHub Copilot，README 没有明确声称已经提供官方集成。因此，更稳妥的理解是：

可以尝试让 Copilot Agent 从 VS Code terminal 调用 flowtrace CLI
但不要假设已经存在 Copilot 专用 integration

⸻

13. Flowtrace 和 planning-with-files 的差异

你之前研究过 planning-with-files。两者方向相似，但层次不同。

能力	planning-with-files	Flowtrace
将计划写入文件	是	是
用文件保持上下文	是	是
明确定义 DAG 依赖	通常较弱	是
每一步独立输出 asset	可选	核心设计
自动保存 Git commit	通常没有	是
支持结构化 evidence	通常没有	是
支持 Web UI	通常没有	是
支持重跑下游步骤	通常需要人工判断	提供 --downstream 查询

可以这样理解：

planning-with-files
= 给 Agent 一本工作笔记
Flowtrace
= 给 Agent 一个轻量级的工作流引擎、文件系统和审计日志

⸻

14. 你最适合用它处理哪些任务？

对你的软件工程工作而言，Flowtrace 比较适合：

场景 A：复杂 Troubleshooting

收集 Dynatrace trace
→ 查询 Splunk
→ 检查 Kafka consumer lag
→ 对比配置
→ 定位 root cause
→ 修改代码
→ 验证
→ 输出 incident report

场景 B：新环境 Onboarding

创建 Resource ID
→ 创建 FID
→ 提交 C2C ticket
→ 创建 topics
→ 创建 identity pools
→ 创建 RBAC
→ 更新 property files
→ 部署
→ 执行 E2E test

场景 C：Java Virtual Thread Spike

识别 I/O-heavy workflow
→ 定义 baseline
→ 开发 prototype
→ 执行 benchmark
→ 分析 pinned-thread 风险
→ 输出建议

场景 D：AI Agent 研究任务

读取需求
→ 查询文档
→ 分析相关代码
→ 设计方案
→ 修改代码
→ 编写测试
→ 输出 handoff

它不太适合简单问题，例如：

解释一个 Java annotation
修复一行代码
生成一条 Slack message
查询一个 Git 命令

README 也明确说明：一次性的小问题直接聊天即可；当结果需要验证，或者任务未来会重复执行时，Flowtrace 才真正有价值。 

⸻

15. 阅读这份 CLI 文档时，优先看哪些章节？

不需要逐字阅读全部内容。建议按这个顺序理解：

1. Concepts
2. State machine
3. CLI commands
4. End-to-end worked example
5. Where the data lives
6. Reply payload schema
7. Path conventions
8. Error catalog

最核心的执行循环只有六步：

读取 trace.json
→ 创建 run
→ 将 step 标记为 running
→ 执行实际工作并生成文件
→ 将 step 标记为 done 并登记 asset
→ 最终将 deliverable 标记为 done

一句话总结：

Flowtrace 将 AI Agent 的一次长对话，变成一个基于 DAG、文件和 Git commit 的可恢复执行过程。