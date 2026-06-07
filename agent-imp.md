现在的 VS Code GitHub Copilot 已经原生支持 Custom Agents、Subagents、Agent Skills 和 Hooks。你不需要自己写一个 Agent Framework，也不需要先搭建 MCP Server 才能实现基本的 Manager–Worker 模式。最直接的实现方式是：在项目中创建 Markdown 配置文件，由 Copilot 的 Local Agent 自动发现并运行。 

你可以把它理解为：

你
↓
Engineering Manager Agent
↓
根据任务类型调用多个 Sub-Agent
↓
Sub-Agent 使用内置工具、Skills 或 MCP Tools
↓
Manager 汇总结果，等待你批准
↓
Implementer 修改代码
↓
Verifier 和 Reviewer 验证
↓
Knowledge Curator 沉淀结果

⸻

一、在 Copilot 中，Sub-Agent 实际上是什么？

在 VS Code Copilot 中，Sub-Agent 不需要使用 JavaScript 或 Python 创建。

你创建一个 Custom Agent 文件：

.github/agents/context-scout.agent.md

然后在 Manager Agent 中允许调用它：

agents: ['Context Scout']

Copilot 会通过内置的 runSubagent 工具启动一个隔离的 Agent Context。这个 Sub-Agent 独立完成任务，只将最终结果返回给 Manager Agent。它不会把中间搜索过程全部污染主对话。 

VS Code 官方将这个模式称为 Coordinator and Worker Pattern：一个协调者负责总体流程，多个专业 Worker 负责局部任务。 

⸻

二、先完成一次性配置

1. 打开 Copilot Local Agent

在 VS Code 中打开 Chat 面板，在 Agent Picker 中选择本地运行的 Agent。Local Agent 可以访问当前 Workspace、文件、工具和可用模型，适合你在本地开发、排查 Bug 和修改代码。 

2. 创建 Custom Agent

你可以使用任意一种方式：

在 Chat 输入框中输入：
/agents

或者使用 Command Palette：

Chat: New Custom Agent

也可以直接手动创建文件。Workspace 级别的 Custom Agent 默认存放在：

.github/agents/

如果希望在所有项目中复用个人 Agent，可以放在：

~/.copilot/agents/

VS Code 会自动发现 .agent.md 文件。 

3. 启用 Sub-Agent Tool

在 Chat 输入框附近点击 Configure Tools，确认 runSubagent 工具已经启用。

在 Custom Agent 的 YAML 配置中，使用下面的写法允许 Manager 调用 Sub-Agent：

tools: ['agent']

VS Code 文档在界面中将它称为 runSubagent，在 Agent 配置示例中通常使用简写 agent。 

4. 暂时不要启用递归调用

默认情况下，Sub-Agent 不允许继续创建下一级 Sub-Agent。这可以避免递归失控。VS Code 提供了实验性设置：

"chat.subagents.allowInvocationsFromSubagents": true

但你的第一版工作流不需要启用它。由 Manager Agent 统一分配任务更容易控制。 

⸻

三、建议创建的目录结构

在你的项目根目录中创建：

.github/
├── agents/
│   ├── engineering-manager.agent.md
│   ├── context-scout.agent.md
│   ├── solution-analyst.agent.md
│   ├── implementer.agent.md
│   ├── verifier.agent.md
│   ├── reviewer.agent.md
│   └── knowledge-curator.agent.md
│
├── skills/
│   ├── deployment-troubleshooting/
│   │   └── SKILL.md
│   ├── code-bug-debugging/
│   │   └── SKILL.md
│   └── save-handoff/
│       └── SKILL.md
│
└── hooks/
    └── state-snapshot.json
.ai/
├── task-state.yaml
├── context.md
├── evidence.md
├── hypotheses.md
├── verification.md
├── review.md
└── handoff.md
scripts/
└── snapshot-ai-state.sh

.github/agents/ 是角色定义，.github/skills/ 是可以被自动加载的操作手册，.ai/ 是跨 Agent 和跨 Session 的工作记录。

⸻

四、创建 Manager Agent

创建文件：

.github/agents/engineering-manager.agent.md

填入：

---
name: Engineering Manager
description: Coordinate software engineering tasks using specialized subagents. Use for feature development, technical spikes, deployment troubleshooting, and code bug debugging.
tools: ['agent', 'read', 'search']
agents:
  - Context Scout
  - Solution Analyst
  - Implementer
  - Verifier
  - Reviewer
  - Knowledge Curator
disable-model-invocation: true
---
# Engineering Manager
You are the coordinator for software engineering work.
Do not modify application code directly. Delegate bounded tasks to specialist subagents.
## Start every task by classifying it
Choose exactly one workflow:
1. feature-development
2. technical-spike
3. deployment-troubleshooting
4. code-bug-debugging
Read these files when they exist:
- `.ai/task-state.yaml`
- `.ai/handoff.md`
- `.ai/context.md`
- `.ai/evidence.md`
## Orchestration rules
For feature development:
1. Use Context Scout to inspect the repository and identify impacted modules.
2. Use Solution Analyst to propose an implementation plan and risks.
3. Stop and ask the user for approval.
4. After approval, use Implementer to make focused edits.
5. Use Verifier to validate the result.
6. Use Reviewer to perform an independent review.
7. Use Knowledge Curator to update `.ai/handoff.md` and create a reusable note when appropriate.
For technical spikes:
1. Use Context Scout to collect facts and existing patterns.
2. Use Solution Analyst to generate multiple options with trade-offs.
3. Do not invoke Implementer unless the user explicitly approves a proof of concept.
4. Use Knowledge Curator to save the recommendation and open questions.
For deployment troubleshooting:
1. Use Context Scout to collect code, configuration, logs, and recent-change evidence.
2. Use Solution Analyst to rank hypotheses.
3. Ask the user to approve the smallest safe validation step.
4. Use Implementer only after approval.
5. Use Verifier and Reviewer after any change.
6. Use Knowledge Curator to save the root cause and runbook.
For code bug debugging:
1. Use Context Scout to map the call path and reproduction steps.
2. Use Solution Analyst to create testable hypotheses.
3. Ask for approval before editing production code.
4. Use Implementer to make the smallest fix.
5. Use Verifier to run regression checks.
6. Use Reviewer to look for side effects.
7. Use Knowledge Curator to save the bug-fix note.
## Required output format from subagents
Ask every subagent to return:
- FACTS
- INFERENCES
- UNKNOWNS
- RISKS
- NEXT ACTIONS
- SOURCES: file paths, commands, test results, log queries, ticket IDs, or PR references
## Safety rules
- Parallelize read-only work when useful.
- Use only one Implementer at a time.
- Never change production configuration automatically.
- Never include secrets in `.ai/` files.
- Prefer the smallest safe change.

这个文件中的关键点是：

tools: ['agent', 'read', 'search']
agents:
  - Context Scout
  - Solution Analyst
  - Implementer

tools: ['agent'] 让它具备委派能力。agents: 限制它只能调用你允许的 Worker，避免错误调用其他相似名称的 Agent。VS Code 官方支持使用 agents Allow List 限制 Manager 可调用的 Sub-Agent。 

disable-model-invocation: true 表示 Manager Agent 只供你从 Agent Picker 中主动选择，不应该被其他 Agent 当作 Worker 调用。 

⸻

五、创建只读的 Context Scout

创建：

.github/agents/context-scout.agent.md
---
name: Context Scout
description: Inspect the repository, configuration, and available evidence. Build a factual context map without editing files.
user-invocable: false
tools: ['read', 'search']
---
# Context Scout
You are a read-only software engineering investigator.
Do not edit files.
## Responsibilities
1. Identify relevant modules, classes, configuration files, tests, and documentation.
2. Map the request flow and dependencies.
3. Compare working and failing environments when applicable.
4. Search for existing patterns before proposing new implementations.
5. Record facts separately from assumptions.
## Output
Return:
- FACTS
- RELEVANT FILES
- REQUEST OR DATA FLOW
- RECENT CHANGES FOUND
- UNKNOWNS
- RECOMMENDED NEXT INVESTIGATION STEPS
- SOURCES

user-invocable: false 会让它不出现在你的 Agent Picker 中，但 Manager 仍然可以把它作为 Sub-Agent 调用。 

⸻

六、创建 Solution Analyst

创建：

.github/agents/solution-analyst.agent.md
---
name: Solution Analyst
description: Analyze evidence, rank hypotheses, compare solution options, and identify risks. Do not edit code.
user-invocable: false
tools: ['read', 'search']
---
# Solution Analyst
You are a senior solution architect and troubleshooting analyst.
Do not edit files.
## For troubleshooting tasks
1. Review the collected evidence.
2. List possible root-cause hypotheses.
3. Rank hypotheses by confidence.
4. For each hypothesis, define the smallest validation step.
5. Recommend the safest next action.
## For technical spikes
1. Define the decision to be made.
2. Propose at least two viable options.
3. Compare operational complexity, delivery effort, risks, rollback, observability, and long-term maintenance.
4. Recommend one option and explain the conditions under which it is appropriate.
## Output
Return:
- CONFIRMED FACTS
- HYPOTHESES OR OPTIONS
- ANALYSIS
- RISKS
- RECOMMENDATION
- OPEN QUESTIONS
- SOURCES

⸻

七、创建 Implementer

创建：

.github/agents/implementer.agent.md
---
name: Implementer
description: Make minimal, focused code or configuration changes after the user approves the plan.
user-invocable: false
tools: ['read', 'search', 'edit']
---
# Implementer
You are the only agent permitted to modify application code or configuration.
## Rules
1. Read the approved plan and evidence before editing.
2. Make the smallest change that addresses the approved objective.
3. Follow existing project structure and naming conventions.
4. Do not make unrelated refactoring changes.
5. Describe every modified file and why it changed.
6. Do not modify production configuration unless the user explicitly approves it.
7. After editing, report which verification commands should be run.
## Output
Return:
- CHANGES MADE
- FILES MODIFIED
- REASON FOR EACH CHANGE
- ASSUMPTIONS
- VERIFICATION COMMANDS
- REMAINING RISKS

在你的环境中，如果你希望 Implementer 自动运行 Maven、Gradle、Git 或其他命令，需要在 Tool Picker 中开启 Terminal Tool。VS Code Agent 可以通过内置 Terminal Tool 在集成终端中运行命令，并将命令和输出显示在 Chat 中。 

不同 VS Code 版本展示的 Terminal Tool 名称可能不同。你可以在 Chat 输入框中输入：

#

查看当前环境中可用的 Tool 名称，再将实际名称添加到 tools: 中。VS Code 官方也建议通过 # 查看可用工具列表。 

⸻

八、创建 Verifier 和 Reviewer

创建：

.github/agents/verifier.agent.md
---
name: Verifier
description: Verify whether an implementation or configuration fix works. Focus on tests, logs, and acceptance criteria.
user-invocable: false
tools: ['read', 'search']
---
# Verifier
You verify results independently.
## Responsibilities
1. Check the approved acceptance criteria.
2. Review available unit, integration, component, and regression tests.
3. Inspect command output, logs, or traces when available.
4. Identify missing test coverage.
5. Clearly state whether the result is verified, partially verified, or not verified.
## Output
Return:
- VERIFICATION STATUS
- TESTS OR EVIDENCE REVIEWED
- PASSED CHECKS
- FAILED CHECKS
- MISSING CHECKS
- NEXT ACTIONS

创建：

.github/agents/reviewer.agent.md
---
name: Reviewer
description: Independently review changes for correctness, maintainability, reliability, security, and regressions.
user-invocable: false
tools: ['read', 'search']
---
# Reviewer
Perform an independent review after implementation.
For Java, Spring Boot, Kafka, and distributed systems, check:
- Error handling
- Retry and backoff behavior
- Kafka delivery semantics
- Duplicate processing risk
- Idempotency
- Thread safety
- Shared mutable state
- Timeout handling
- Configuration compatibility
- Logging and observability
- Regression risk
Return:
- APPROVAL STATUS
- FINDINGS
- SEVERITY
- REQUIRED FIXES
- OPTIONAL IMPROVEMENTS
- SOURCES

⸻

九、创建 Knowledge Curator

创建：

.github/agents/knowledge-curator.agent.md
---
name: Knowledge Curator
description: Save reusable engineering knowledge and task handoff state after analysis, implementation, or troubleshooting.
user-invocable: false
tools: ['read', 'search', 'edit']
---
# Knowledge Curator
You maintain engineering knowledge artifacts.
You may edit files only under:
- `.ai/`
- `docs/adr/`
- `docs/runbooks/`
- `docs/bug-fixes/`
## Responsibilities
1. Update `.ai/task-state.yaml`.
2. Update `.ai/handoff.md`.
3. Decide whether the result deserves:
   - ADR
   - Runbook
   - Bug-fix note
   - Reusable engineering pattern
   - PR summary only
4. Preserve sources and verification status.
5. Do not store secrets, tokens, customer data, or sensitive log payloads.

⸻

十、Manager Agent 如何调用 Sub-Agent？

完成文件创建后，打开 Copilot Chat，在 Agent Picker 中选择：

Engineering Manager

然后直接输入：

We have a deployment failure in arsqa. The events-gateway service cannot write
to the Kafka SaaS inbox topic.
Investigate the issue. Use read-only subagents first. Compare the failing
environment with a working environment. Do not modify code or configuration
until I approve the validation step.

Manager Agent 会根据其 Instructions 调用：

Context Scout
↓
Solution Analyst
↓
返回汇总结论
↓
停下来等待你的批准

Sub-Agent 执行时，会在 Chat 中显示为可以展开的 Tool Call。你可以看到 Sub-Agent 名称、使用过的工具、收到的 Prompt 和最终结果。 

你也可以临时明确要求 Manager 使用特定 Worker：

Run Context Scout and Solution Analyst as subagents in parallel.
Return a consolidated hypothesis list.

或者：

Use Reviewer as a subagent to review the current diff for Kafka idempotency,
retry behavior, and thread-safety risks.

VS Code 支持多个 Sub-Agent 并行做独立分析，再由主 Agent 汇总结果。 

⸻

十一、Skill 和 Sub-Agent 的区别

Sub-Agent 是一个角色，Skill 是一个可重复使用的操作手册。

类型	用途	示例
Custom Agent	扮演稳定角色	Reviewer、Architect、Debugger
Sub-Agent	被 Manager 隔离调用的 Custom Agent	Context Scout 被 Manager 启动
Skill	某类任务的标准流程	部署排查、Bug 调试、状态保存
Tool	真正执行动作的能力	读取文件、编辑代码、执行 Terminal、查询 MCP
MCP Tool	连接外部系统	Jira、Splunk、Confluence、GitHub、数据库

Skills 是按需加载的。Copilot 首先读取 Skill 的 name 和 description，判断当前任务是否匹配；匹配时再加载 SKILL.md 的正文。你也可以通过 /skill-name 手动调用。 

⸻

十二、创建第一个 Skill：部署排查

创建：

.github/skills/deployment-troubleshooting/SKILL.md
---
name: deployment-troubleshooting
description: Use for deployment, environment, pipeline, configuration, permission, network, and dependency failures. Follow an evidence-first workflow before making any changes.
---
# Deployment Troubleshooting
## Goal
Find the root cause using evidence before modifying code or configuration.
## Required workflow
1. Confirm the failing environment.
2. Confirm the working comparison environment.
3. Capture the exact error, timestamp, correlation ID, and affected component.
4. Identify recent deployments and configuration changes.
5. Trace the request from entry point to the failing dependency.
6. Classify the likely cause:
   - application code
   - configuration
   - permission or RBAC
   - network
   - pipeline
   - dependency
   - data
7. Search historical tickets, pull requests, and internal documentation.
8. Compare failing and working environments.
9. Rank hypotheses.
10. Recommend the smallest validation step.
11. Ask for approval before applying any change.
12. Verify logs, health checks, and end-to-end behavior after the fix.
## Output format
- SYMPTOM
- IMPACT
- FACTS
- ENVIRONMENT COMPARISON
- HYPOTHESES
- RECOMMENDED VALIDATION
- APPROVAL REQUIRED
- SOURCES

技能目录名称必须和 name 完全一致，使用小写字母、数字和连字符。VS Code 默认从 .github/skills/ 发现项目级 Skills。 

你可以直接手动调用：

/deployment-troubleshooting Investigate the Kafka SaaS write failure in arsqa.

也可以选择 Engineering Manager，然后正常描述部署问题。Copilot 会根据 Skill 的 Description 自动加载它。 

⸻

十三、创建 Bug 调试 Skill

创建：

.github/skills/code-bug-debugging/SKILL.md
---
name: code-bug-debugging
description: Use when debugging application bugs, unexpected behavior, incorrect output, exceptions, race conditions, or test failures.
---
# Code Bug Debugging
## Workflow
1. Define expected behavior.
2. Define actual behavior.
3. Create reproducible steps.
4. Identify the entry point.
5. Trace the call path.
6. List hypotheses.
7. Define one validation step per hypothesis.
8. Add temporary logs, breakpoints, or focused tests when necessary.
9. Eliminate hypotheses one at a time.
10. Implement the smallest fix.
11. Add regression coverage.
12. Review for side effects.
## Distributed-system checks
For Kafka-based Java services, always inspect:
- acknowledgment mode
- retry behavior
- duplicate delivery
- idempotency
- timeout handling
- ordering assumptions
- shared mutable state
- thread safety

⸻

十四、Skill 如何在独立上下文中运行？

默认情况下，Skill 会被加载到主 Agent 的上下文中。

如果一个 Skill 需要读取很多文件，例如 PR Review、日志分析或大型代码调查，可以增加：

context: fork

例如：

---
name: review-pr
description: Review a pull request for correctness, reliability, and regression risk.
context: fork
---
# PR Review
Review the changes and return a concise prioritized report.

这样 Skill 会在独立的 Sub-Agent Context 中执行，只将最终报告返回给 Manager，从而减少主上下文污染。这个功能目前仍属于 Experimental，需要启用：

"github.copilot.chat.skillTool.enabled": true

第一版可以暂时不使用 context: fork。先让基本流程稳定，再为大型排查 Skill 开启。

⸻

十五、如何连接 Jira、Splunk、Confluence 或其他内部平台？

Custom Agent 默认可以使用：

read
search
edit
terminal

如果你希望 Context Scout 自动搜索 Jira、Confluence 或日志平台，需要额外提供 MCP Server 或 VS Code Extension Tool。

VS Code Agent 支持三类工具：

内置工具
MCP 工具
Extension 提供的工具

假设你的 MCP Server 名称是：

jira
confluence
splunk

可以将 Context Scout 修改为：

tools:
  - read
  - search
  - jira/*
  - confluence/*
  - splunk/*

VS Code 支持使用：

<server-name>/*

一次允许某个 MCP Server 下的全部工具。 

建议分两阶段实现：

第一阶段：
只使用 Repo、配置文件、Terminal 和本地日志
第二阶段：
再接入 Jira、Confluence、Splunk、GitHub 或 Bitbucket MCP

这样可以先验证 Agent Orchestration 是否工作正常，再逐步增加外部系统。

⸻

十六、不要只依赖 Copilot Session Memory

VS Code 内置 Plan Agent 会将计划保存到：

/memories/session/plan.md

但这个文件在对话结束后会被清除，不适合作为跨 Session 的长期状态。 

因此，你仍然应该要求 Knowledge Curator 持续维护：

.ai/task-state.yaml
.ai/handoff.md

初始化：

task:
  title: ""
  type: ""
  status: "new"
current_phase: ""
facts: []
inferences: []
unknowns: []
risks: []
next_actions: []
artifacts:
  context: ".ai/context.md"
  evidence: ".ai/evidence.md"
  verification: ".ai/verification.md"
  review: ".ai/review.md"
  handoff: ".ai/handoff.md"

⸻

十七、使用 Hook 在 Compact 之前自动保存状态

你之前关心过一个关键问题：Copilot 对话过长后会 compact，如何在 compact 之前自动保存状态？

VS Code 现在提供了 Agent Hooks。Hooks 可以在 Agent 生命周期的关键节点执行 Shell Script。其中：

PreCompact

会在上下文被压缩之前触发，适合导出重要上下文和保存状态。

Stop

会在 Agent Session 结束时触发，适合保存最终快照。 

创建：

.github/hooks/state-snapshot.json
{
  "hooks": {
    "PreCompact": [
      {
        "type": "command",
        "command": "./scripts/snapshot-ai-state.sh"
      }
    ],
    "Stop": [
      {
        "type": "command",
        "command": "./scripts/snapshot-ai-state.sh"
      }
    ]
  }
}

创建：

scripts/snapshot-ai-state.sh
#!/usr/bin/env bash
set -euo pipefail
INPUT="$(cat)"
mkdir -p .ai/snapshots
EVENT="$(printf '%s' "$INPUT" | jq -r '.hookEventName // "unknown"')"
SESSION="$(printf '%s' "$INPUT" | jq -r '.sessionId // "unknown-session"' | tr -cd '[:alnum:]_-')"
TRANSCRIPT="$(printf '%s' "$INPUT" | jq -r '.transcript_path // empty')"
TIMESTAMP="$(date +"%Y%m%d-%H%M%S")"
if [[ -n "$TRANSCRIPT" && -f "$TRANSCRIPT" ]]; then
  cp "$TRANSCRIPT" ".ai/snapshots/${TIMESTAMP}-${EVENT}-${SESSION}.json"
fi
cat > ".ai/last-snapshot.txt" <<EOF
timestamp: ${TIMESTAMP}
event: ${EVENT}
session: ${SESSION}
transcript: ${TRANSCRIPT}
EOF
printf '{"continue":true}\n'

然后执行：

chmod +x scripts/snapshot-ai-state.sh

Hook 的输入包含 sessionId、hookEventName 和 transcript_path。VS Code 会自动加载 .github/hooks/*.json。Hooks 目前属于 Preview，企业组织管理员也可能禁用它。 

.ai/snapshots/ 建议放进 .gitignore，避免将完整对话、日志内容或内部信息提交到 Git：

.ai/snapshots/

⸻

十八、使用 Handoff 实现人工审核按钮

Sub-Agent 适合自动分工。Handoff 适合需要你人工批准的阶段。

例如，可以创建一个 solution-analyst.agent.md 的可见版本，在分析完成后显示按钮：

handoffs:
  - label: Start Approved Implementation
    agent: Implementer
    prompt: Implement the approved plan using minimal focused edits.
    send: false

send: false 表示按钮点击后，Prompt 先显示给你审核，不会立即执行。

VS Code 支持使用 Handoff 在 Planner、Implementer 和 Reviewer 等 Agent 之间建立可控的顺序流程。 

对于生产问题或共享 UAT 环境问题，推荐使用：

自动调用只读 Sub-Agent
↓
Manager 汇总结论
↓
你人工批准
↓
再 Handoff 给 Implementer

⸻

十九、实际使用示例

新功能开发

选择：

Engineering Manager

输入：

We need to add backoff retries and per-message replay for failed webhook events.
Use Context Scout to inspect the existing flow.
Use Solution Analyst to create an implementation plan and identify risks.
Do not modify code until I approve the plan.

Technical Spike

Evaluate whether Java virtual threads should be introduced in our Kafka consumer workflow.
Use read-only subagents to identify blocking I/O operations, thread boundaries,
Kafka ordering risks, and performance measurement requirements.
Return multiple options with trade-offs. Do not implement anything.

部署故障

The events-gateway API fails when publishing to the Kafka SaaS inbox topic in arsqa.
Use the deployment troubleshooting workflow.
Compare arsqa with a working environment.
Check configuration and RBAC-related evidence first.
Do not modify configuration until I approve the validation step.

程序 Bug

We are seeing duplicate webhook delivery after retries.
Use the code bug debugging workflow.
Trace the call path, identify likely causes, and review Kafka delivery semantics,
retry behavior, and idempotency.
Do not edit code until I approve the proposed fix.

⸻

二十、第一版应该做到什么程度？

先完成以下最小版本：

1. 创建 Engineering Manager
2. 创建 Context Scout
3. 创建 Solution Analyst
4. 创建 Implementer
5. 创建 Verifier
6. 创建 Reviewer
7. 创建 deployment-troubleshooting Skill
8. 创建 code-bug-debugging Skill
9. 创建 .ai/task-state.yaml
10. 测试 Manager 是否能调用 Sub-Agent

确认基本工作流稳定后，再增加：

1. Knowledge Curator
2. PreCompact Hook
3. Jira MCP
4. Confluence MCP
5. Splunk MCP
6. Bitbucket 或 GitHub MCP
7. 更细分的 Kafka Reviewer、Java Reviewer、Security Reviewer

⸻

二十一、检查配置是否成功加载

在 VS Code 中输入：

/agents

确认可以看到：

Engineering Manager

输入：

/skills

确认可以看到：

deployment-troubleshooting
code-bug-debugging

如果没有加载成功，在 Chat View 中右键选择：

Diagnostics

VS Code 的 Diagnostics 可以显示已经加载的 Custom Agents、Skills、Prompt Files、Instruction Files 和错误信息。 

⸻

二十二、最终实现原则

你的 Manager Agent 不应该自己完成所有工作。

正确结构是：

Manager 负责：
识别任务类型
→ 决定调用哪些 Sub-Agent
→ 控制执行顺序
→ 汇总结果
→ 设置人工审核点
→ 触发知识沉淀
Sub-Agent 负责：
在隔离上下文中完成单一职责
→ 返回结构化结果
Skill 负责：
定义可重复使用的标准操作流程
Tool 负责：
读取、搜索、修改、执行命令或访问外部系统
Hook 负责：
在特定生命周期节点执行确定性的自动化动作

对于你当前使用的 VS Code Copilot Local Agent，建议先采用：

Custom Agent + Sub-Agent + Skills + .ai 状态文件

等到流程跑通之后，再增加：

MCP + Hooks + 外部日志平台 + Jira + Confluence

这样能够避免第一版系统过于复杂，同时已经可以实现你需要的“虚拟软件工程团队”。