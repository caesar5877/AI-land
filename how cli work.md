Flowtrace 和 Agent 的关系可以理解为：

Agent 是工程师，Flowtrace 是 Jira Board + 文件柜 + Git 审计日志。

Flowtrace 不会自己分析代码，也不会自己生成文档。它也不是 MCP Server。
真正执行任务的仍然是 Claude Code、Codex、Cursor，或者具有终端权限的 Copilot Agent。

Agent 通过普通 shell 命令调用 Flowtrace CLI，并使用普通文件读写工具将结果保存到本地文件系统。Flowtrace CLI 负责更新 state.json、验证正式输出文件是否存在，并自动创建 Git commit。官方设计明确说明：每个步骤之间通过文件传递数据，而不是通过聊天上下文或函数参数传递数据。 

⸻

1. 整体架构

你
│
│  “调查 Kafka consumer lag，并保存每一步结果”
▼
Coding Agent
│
│  ① 读取 trace.json 和 STEP.md
│  ② 在 terminal 中执行 flowtrace CLI
│  ③ 分析代码、日志、配置
│  ④ 使用文件工具写入 Markdown、JSON、CSV
▼
Flowtrace CLI
│
│  ① 更新 state.json
│  ② 校验 asset 是否存在
│  ③ 保存 structured reply
│  ④ 自动 git commit
▼
Trace Repository
├── trace.json
├── steps/
└── runs/<run_id>/
    ├── state.json
    ├── replies/
    └── <step_id>/
        ├── scratch files
        └── official assets

Agent 和 Flowtrace 之间没有特殊 API。Agent 调用 CLI 的方式与你在 terminal 里手工执行命令完全相同。

⸻

2. Agent 为什么可以调用 Flowtrace CLI？

Coding Agent 通常具有三类本地工具：

read     读取文件
edit     修改或创建文件
terminal 执行 shell 命令

例如，VS Code 中的 Copilot Agent 可以读取文件、编辑代码、运行命令，并在命令失败时根据输出继续修正。VS Code 也允许你控制当前 Agent 可使用哪些 tools。 

因此，Agent 可以在 terminal 工具中执行：

flowtrace run new --name "Investigate Kafka lag"
flowtrace step collect_logs running
mkdir -p runs/$RUN/collect_logs
echo "..." > runs/$RUN/collect_logs/lag-summary.md
flowtrace step collect_logs done --asset lag-summary.md

Flowtrace README 官方列出的 Agent 是 Claude Code、Codex 和 Cursor。Copilot 没有被 Flowtrace README 明确列为官方集成对象。 

但是，从机制上判断，VS Code Copilot Agent 也可以使用 Flowtrace，因为它具备 terminal、文件读取和文件修改能力。这是基于两套官方文档做出的合理推断，而不是 Flowtrace 官方已经提供了 Copilot 专用插件。

⸻

3. Agent 保存的不是“脑内状态”，而是显式文件

这是理解 Flowtrace 最重要的一点。

Flowtrace 不会自动读取 Agent 的隐藏 reasoning，也不会把聊天历史完整保存下来。它要求 Agent 将真正有价值的信息显式写成文件。

例如，Agent 排查 Kafka consumer lag 时，可能将结果写入：

runs/20260608-190001/
├── state.json
├── collect_logs/
│   └── lag-summary.md
├── analyze_root_cause/
│   └── root-cause-analysis.md
├── propose_fix/
│   └── implementation-plan.md
└── verify_fix/
    ├── benchmark.csv
    └── verification-report.md

每一个下游步骤读取前一步生成的文件：

collect_logs/lag-summary.md
          ↓
analyze_root_cause/root-cause-analysis.md
          ↓
propose_fix/implementation-plan.md
          ↓
verify_fix/verification-report.md

Flowtrace README 明确写到：步骤之间通过文件传递数据；每个步骤写出结果，下游步骤读取它。 

这就是它解决 context compaction 问题的方式：

聊天上下文可能丢失
但是文件不会丢失
Git commit 也不会丢失

⸻

4. Flowtrace 文件系统如何组织？

一个 Trace 是一个独立 Git repository：

<trace_root>/
├── .git/
├── trace.json
├── scripts/
├── resources/
├── steps/
│   ├── collect_logs/
│   │   └── STEP.md
│   ├── analyze_root_cause/
│   │   └── STEP.md
│   └── verify_fix/
│       └── STEP.md
└── runs/
    └── <run_id>/
        ├── state.json
        ├── replies/
        │   ├── 0001.json
        │   └── 0002.json
        ├── collect_logs/
        ├── analyze_root_cause/
        └── verify_fix/

官方文档区分了四类文件。 

文件	用途	谁创建
trace.json	静态 DAG 工作流定义	你或 Agent
steps/<id>/STEP.md	每个步骤的执行说明	你或 Agent
runs/<run_id>/state.json	当前 Run 的状态	Flowtrace CLI
runs/<run_id>/<step_id>/*	本次执行的实际输出	Agent

⸻

5. trace.json 是 Agent 的任务地图

例如：

{
  "id": "kafka-lag-troubleshooting",
  "title": "Troubleshoot Kafka Consumer Lag",
  "version": "0.1.0",
  "steps": {
    "collect_logs": {
      "name": "Collect logs",
      "does": "Collect lag metrics and application logs",
      "from_steps": [],
      "assets": ["lag-summary.md"]
    },
    "analyze_root_cause": {
      "name": "Analyze root cause",
      "does": "Identify the likely bottleneck",
      "from_steps": ["collect_logs"],
      "assets": ["root-cause-analysis.md"]
    },
    "verify_fix": {
      "name": "Verify the fix",
      "does": "Run tests and compare metrics",
      "from_steps": ["analyze_root_cause"],
      "assets": ["verification-report.md"]
    }
  },
  "deliverable": {
    "description": "Final troubleshooting report",
    "assets": ["verify_fix/verification-report.md"]
  }
}

Agent 可以执行：

flowtrace show --fmt json

找到所有 steps。

也可以执行：

flowtrace show --fmt mermaid

查看 DAG。

CLI 文档说明，flowtrace show --fmt json | jq .steps 是 Agent 发现 step IDs 的标准方式。 

⸻

6. STEP.md 是每个节点的操作手册

trace.json 只告诉 Agent：

有哪些步骤
步骤之间如何依赖
每一步应该生成什么 asset

每一步具体怎么做，写在：

steps/<step_id>/STEP.md

例如：

---
name: analyze_root_cause
description: Analyze Kafka consumer lag root cause
reads:
  - collect_logs/lag-summary.md
writes:
  - root-cause-analysis.md
---
# Analyze Root Cause
Read the lag summary and inspect the consumer implementation.
Check:
1. Whether a slow webhook call blocks the Kafka consumer thread.
2. Whether consumer concurrency matches partition count.
3. Whether retries cause message processing backlog.
4. Whether database or Cassandra latency is elevated.
Write a concise analysis to `root-cause-analysis.md`.

官方 make-trace skill 说明：每个 Step 应该拥有一个 STEP.md contract；其中 reads 和 writes 描述输入输出，正文说明具体执行方法。 

⸻

7. 一次真实执行过程

假设 Agent 已经进入 Trace repo：

cd ~/traces/notification-platform/kafka-lag-troubleshooting

Flowtrace CLI 会从当前目录向上寻找 trace.json，因此 Agent 必须在 Trace repo 内部执行命令。 

第一步：Agent 读取工作流

flowtrace validate
flowtrace show --fmt json

然后创建一次 Run：

RUN=$(flowtrace run new \
  --name "Investigate arsqa Kafka consumer lag" | tail -1)

CLI 创建：

runs/<run_id>/
└── state.json

⸻

第二步：Agent 将 Step 标记为 running

flowtrace step collect_logs running \
  --message "Collecting Kafka lag metrics and application logs" \
  --run "$RUN"

Flowtrace 更新：

{
  "steps": {
    "collect_logs": {
      "status": {
        "kind": "running",
        "message": "Collecting Kafka lag metrics and application logs"
      },
      "assets": []
    }
  }
}

⸻

第三步：Agent 执行真正的工作

Agent 使用 terminal、日志查询脚本、代码搜索工具或人工提供的数据完成分析，然后主动创建目录并写入文件：

mkdir -p "runs/$RUN/collect_logs"
cat > "runs/$RUN/collect_logs/lag-summary.md" <<'EOF'
# Kafka Lag Summary
## Environment
arsqa
## Findings
- Consumer lag peaked at 52,000 messages.
- Webhook latency increased to 2.8 seconds at the same time.
- Consumer concurrency is 4.
- Topic partition count is 12.
EOF

注意：

Flowtrace CLI 不会创建 Step 文件夹，也不会替 Agent 创建 asset 文件。

官方 CLI 文档明确要求：Agent 必须先在磁盘上创建文件，再使用 --asset 登记它。文件不存在时，CLI 会失败。 

⸻

第四步：Agent 登记正式输出

flowtrace step collect_logs done \
  --asset lag-summary.md \
  --message "Lag metrics collected" \
  --run "$RUN"

这时 Flowtrace 会：

验证 lag-summary.md 是否存在
→ 更新 state.json
→ 将 asset 登记为正式输出
→ git add state.json 和 lag-summary.md
→ git commit

Flowtrace 每次写操作只提交它明确声明的文件，不会顺便提交其他 scratch 文件。 

⸻

第五步：下游 Step 读取前一步文件

Agent 开始分析 root cause：

flowtrace step analyze_root_cause running \
  --message "Analyzing consumer bottleneck" \
  --run "$RUN"

然后读取：

cat "runs/$RUN/collect_logs/lag-summary.md"

并写出：

mkdir -p "runs/$RUN/analyze_root_cause"
cat > "runs/$RUN/analyze_root_cause/root-cause-analysis.md" <<'EOF'
# Root Cause Analysis
The most likely cause is blocking webhook I/O combined with insufficient
consumer concurrency. The topic has 12 partitions but only 4 consumer workers.
EOF

登记正式结果：

flowtrace step analyze_root_cause done \
  --asset root-cause-analysis.md \
  --run "$RUN"

⸻

第六步：Agent 写 structured reply

Asset 是正式文件。

Reply 是给 UI 和人阅读的结构化总结。

例如：

cat > /tmp/reply.json <<EOF
{
  "headline": "Kafka lag is likely caused by slow webhook I/O and limited consumer concurrency",
  "status": "complete",
  "checkpoint": {
    "step_id": "analyze_root_cause"
  },
  "findings": [
    {
      "title": "Webhook latency increased",
      "detail": "Latency rose to 2.8 seconds during the lag spike."
    },
    {
      "title": "Consumer concurrency is below partition count",
      "detail": "The topic has 12 partitions but only 4 consumer workers."
    }
  ],
  "evidence": [
    {
      "type": "document",
      "path": "collect_logs/lag-summary.md",
      "title": "Kafka lag summary"
    },
    {
      "type": "document",
      "path": "analyze_root_cause/root-cause-analysis.md",
      "title": "Root cause analysis"
    }
  ]
}
EOF
flowtrace reply --run "$RUN" < /tmp/reply.json

Flowtrace 把它保存为：

runs/<run_id>/replies/0001.json

Reply 中引用的 evidence 文件也必须已经真实存在，否则提交失败。 

⸻

第七步：关闭 deliverable

最后：

flowtrace deliverable done \
  --asset "verify_fix/verification-report.md" \
  --message "Kafka lag troubleshooting completed" \
  --run "$RUN"

然后检查：

flowtrace run show --run "$RUN"
git log --oneline

官方 CLI 文档中提供了相同生命周期的最小示例。 

⸻

8. state.json 如何帮助 Agent 恢复工作？

假设 Copilot conversation 被 compact，或者你第二天重新开启 session。

新的 Agent 不需要重新阅读整个聊天记录。

它只需要进入 Trace repo：

cd ~/traces/notification-platform/kafka-lag-troubleshooting

执行：

flowtrace run list
flowtrace run show
cat trace.json

就可以看到：

{
  "steps": {
    "collect_logs": {
      "status": {
        "kind": "done"
      },
      "assets": [
        "lag-summary.md"
      ]
    },
    "analyze_root_cause": {
      "status": {
        "kind": "done"
      },
      "assets": [
        "root-cause-analysis.md"
      ]
    },
    "verify_fix": {
      "status": {
        "kind": "blocked",
        "message": "Waiting for partner QA producer deployment"
      },
      "assets": []
    }
  }
}

新的 Agent 立即知道：

日志收集完成
Root cause 分析完成
验证尚未完成
当前 blocker 是 partner QA producer deployment

官方 CLI 文档把 state.json 定义为 Run Status 的 single source of truth，并说明 AI 每个 session 只需要读取 CLI 协议和当前 Trace 的 trace.json；其余信息可以通过 binary 自解释命令按需查询。 

⸻

9. Scratch 文件和正式 Asset 的区别

Agent 在执行过程中可能产生很多中间文件：

runs/<run_id>/analyze_root_cause/
├── grep-output.tmp
├── raw-log.txt
├── notes.md
└── root-cause-analysis.md

只有通过 --asset 登记的文件才是正式结果：

flowtrace step analyze_root_cause done \
  --asset root-cause-analysis.md

结果：

文件	Git commit	显示为正式输出
grep-output.tmp	否	否
raw-log.txt	否	否
notes.md	否	否
root-cause-analysis.md	是	是

官方说明：未登记的文件属于 scratch，不会被提交。 

这个设计很适合 Troubleshooting，因为 Agent 可以自由产生临时日志，不会污染长期状态。

⸻

10. 如何与你的 VS Code Copilot Agent 集成？

对于你现在使用的 Copilot VS Code 工作流，推荐采用两层设计：

Copilot Custom Agent
        ↓
Flowtrace Agent Skill
        ↓
flowtrace CLI
        ↓
Trace Repository

⸻

层级一：创建通用 Flowtrace Skill

VS Code 目前支持 Agent Skills。Skill 是一个包含 SKILL.md、脚本和资源的目录，可以放在 workspace 或 user profile 中；个人 Skill 可以放在：

~/.copilot/skills/

项目专属 Skill 可以放在：

.github/skills/

VS Code 会在相关任务中按需加载 Skill。 

Flowtrace 仓库已经提供：

skills/make-trace/SKILL.md

它的作用是将 runbook、现有 skill、聊天记录或文字描述转换为可执行 Trace。 

你可以复制它：

mkdir -p ~/.copilot/skills
cp -R ~/src/Flowtrace/skills/make-trace \
      ~/.copilot/skills/

⸻

层级二：增加一个 run-flowtrace Skill

官方仓库目前主要提供 make-trace。你还可以为自己增加一个很薄的执行 Skill：

~/.copilot/skills/run-flowtrace/
└── SKILL.md

内容可以是：

---
name: run-flowtrace
description: Execute an existing Flowtrace trace and persist each step result.
---
# Run Flowtrace
When asked to execute a reusable or multi-step task:
1. Locate the relevant trace repository.
2. Change the terminal working directory to the trace root.
3. Read `trace.json`.
4. Run `flowtrace validate`.
5. Inspect the latest state with `flowtrace run show`.
6. Start a new run only when the user requests a clean execution.
7. Before executing a step, mark it `running`.
8. Read the step's `steps/<step_id>/STEP.md`.
9. Create the step folder under `runs/<run_id>/<step_id>/`.
10. Save meaningful results as files.
11. Mark the step `done` with one or more declared `--asset` files.
12. Write a structured reply with evidence.
13. If blocked, use `flowtrace step <id> blocked --message "..."`.
14. Before finishing, run `flowtrace run show`.
15. Never rely only on chat history when a file should be persisted.

这不是 Flowtrace 官方现成文件，而是根据官方 CLI 生命周期为你的 Copilot 工作流增加的一层适配。

⸻

层级三：创建一个 Copilot Custom Agent

VS Code 支持 workspace-level 和 user-level custom agents：

Workspace:
.github/agents/
User:
~/.copilot/agents/

Custom Agent 可以定义行为、tools、handoffs 和模型。 

你可以创建：

~/.copilot/agents/flowtrace-executor.agent.md

内容：

---
name: Flowtrace Executor
description: Execute complex engineering work as durable Flowtrace steps.
---
You are a software engineering execution agent.
For reusable, multi-step, troubleshooting, spike, onboarding, or architecture tasks:
1. Use Flowtrace as the durable task state.
2. Do not rely only on chat context.
3. Read the trace DAG and current run state before making changes.
4. Use terminal commands to call the `flowtrace` CLI.
5. Write meaningful intermediate results into the run step folder.
6. Register official outputs as assets.
7. Record blockers explicitly.
8. Before stopping, save the current state and inspect the resulting run.
9. Use Git history as the audit trail.

在 VS Code Chat 中选择该 Agent，并确保启用了 terminal、read 和 edit tools。

⸻

11. 你实际可以怎样向 Copilot 下达任务？

第一次创建 Trace：

Use the make-trace skill.
Create a reusable Flowtrace for Kafka SaaS onboarding.
The steps should include resource ID creation, FID creation, C2C ticket,
topic creation, identity pools, RBAC, application configuration,
deployment, and E2E verification.
Store the trace under:
~/traces/notification-platform/kafka-saas-onboarding

执行已有 Trace：

Use the run-flowtrace skill.
Continue the latest run under:
~/traces/notification-platform/kafka-saas-onboarding
Read trace.json and the current state before doing any work.
Continue from the first unfinished step.
Persist each meaningful result as an asset.
Do not redo completed steps unless their upstream dependencies changed.

从 compaction 后恢复：

Resume the latest Flowtrace run under:
~/traces/notification-platform/kafka-saas-onboarding
Inspect the run state, replies, and existing assets first.
Summarize the completed steps, blockers, and the next actionable step.
Then continue execution.

⸻

12. Flowtrace 是否会自动保存一切？

不会。

它没有自动监听 Agent，也不会自动判断什么时候应该保存文件。

Flowtrace 提供的是：

CLI
文件目录规范
状态机
Git commit
Web UI

但是 Agent 必须按照协议主动执行：

标记 step running
→ 执行任务
→ 创建文件
→ 登记 asset
→ 写 reply
→ 标记 deliverable done

因此，要让 Agent 稳定使用 Flowtrace，你至少需要下面三种方式之一：

方式	自动化程度	适合场景
每次在 prompt 中要求使用 Flowtrace	低	初期试验
安装 run-flowtrace Agent Skill	中	推荐
创建专门的 Flowtrace Custom Agent	高	经常处理复杂任务

⸻

13. 本地 Agent 和 Cloud Agent 的差别

这一点对你很重要。

本地 VS Code Agent

Agent terminal
→ 你的本地电脑
→ ~/traces/
→ 本地 Git repo

结果直接保存在你的文件系统中。
这最适合你希望解决的：

Conversation compact 后恢复状态
不同 session 之间继续工作
保存 troubleshooting 和 onboarding 进度

Cloud Agent

Cloud Agent
→ 远程临时环境
→ 远程 checkout
→ 远程 Flowtrace repo

Flowtrace 仍然可以生成 Git commits，但是生成的数据只存在于远程执行环境，除非你将 Trace repo 推送到远程仓库、通过 PR 保存，或者在任务完成后同步回来。

因此，对于你的个人开发工作流：

优先使用本地 VS Code Agent 或本地 Copilot CLI 驱动 Flowtrace。

⸻

14. 最适合你的最终组合

你的目标是：

自动保存 AI state
Conversation compact 后可以恢复
多个 Agent 或 Skill 复用前一个步骤的结果
复杂任务可以继续执行而不是重新分析

推荐结构：

~/.copilot/
├── agents/
│   └── flowtrace-executor.agent.md
└── skills/
    ├── make-trace/
    ├── run-flowtrace/
    └── handoff-state/
~/traces/
└── notification-platform/
    ├── kafka-saas-onboarding/
    ├── partner-qa-deployment/
    ├── kafka-troubleshooting/
    └── java-virtual-thread-spike/

职责划分：

组件	职责
make-trace	将已有流程转成 DAG
run-flowtrace	要求 Agent 逐步执行 CLI 生命周期
handoff-state	在 session 结束前写简短交接摘要
Flowtrace state.json	保存精确运行状态
Flowtrace assets	保存真实分析结果
Flowtrace Git history	保存每次状态变化
Flowtrace Web UI	查看 DAG 和执行历史

一句话总结：

Agent 通过 terminal 像普通开发者一样运行 Flowtrace CLI；Agent 通过文件工具将分析结果写入 Run 目录；Flowtrace 将这些文件登记为 assets，更新 state.json，并使用 Git commit 形成可恢复、可审计的长期状态。