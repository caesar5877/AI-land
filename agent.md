你贴图里的“方式 B”非常适合改造成软件工程师的 AI 工作流。它的核心不是让一个 Agent 从头做到尾，而是把工作拆成不同角色：

图片中的角色	软件工程中的对应角色
研究员找资料，标注来源	收集代码、日志、配置、Jira、Confluence、历史 PR，并记录证据来源
分析师整理数据，推导过程	建立假设、排除错误方向、比较方案、分析风险和影响范围
写作者输出初稿，区分事实与推断	生成设计文档、排查结论、修复方案、PR Summary、ADR 或 Runbook
你只审核，不满意就返回上一步	你作为工程负责人做 Approval Gate：批准方案、批准改代码、批准部署

对于软件开发，还需要增加两个角色：Executor 负责真正修改代码或配置，Verifier 负责测试和验证结果。

⸻

一、先区分 Agent、Sub-Agent 和 Skill

这三个概念不要混在一起。

概念	定义	示例
Manager Agent	负责理解目标、选择工作流、分配任务、检查结果	判断当前问题是新功能、技术 Spike、部署故障还是代码 Bug
Sub-Agent	在独立上下文中扮演一个专业角色，只负责一个边界清晰的任务	Log Investigator、Solution Architect、Code Reviewer
Skill	可以重复调用的标准动作或操作手册	搜索 Jira、查询 Splunk、对比配置、运行单元测试、生成 ADR
Artifact	Agent 之间交接的文件，也是最终可以沉淀的知识	evidence.md、hypotheses.md、design.md、handoff.md

一个容易记忆的比喻是：

* Manager Agent 是项目经理。
* Sub-Agent 是不同岗位的工程师。
* Skill 是每个工程师工具箱里的标准操作。
* Artifact 是 Jira、设计文档、排查报告和交接记录。

不要让 Agent 只依赖聊天上下文。聊天可能被压缩，也可能在切换 Skill 后丢失信息。本地文件应该是 Agent 之间的接口和事实来源。

⸻

二、建立一个通用的 Software Engineering Agent Team

不需要每个场景都创建完全不同的一套 Agent。可以先建立一个固定团队，再根据任务类型选择其中一部分角色。

Agent 角色	主要职责	常用 Skills	输出文件
Manager / Orchestrator	分类任务、拆分阶段、安排 Agent、维护状态、设置审核点	task-classifier、task-planner、state-manager	task-state.yaml、plan.md
Context Scout	阅读项目结构、README、现有设计、相关模块	repo-navigator、code-search、dependency-map	context.md
Evidence Collector	搜集事实，不急于下结论	jira-search、confluence-search、pr-trace、log-query、config-diff	evidence.md
Analyst / Architect	根据证据建立假设，分析方案、风险和影响范围	hypothesis-builder、solution-comparator、impact-analysis	analysis.md、hypotheses.md
Implementer	修改代码、配置或测试	code-edit、refactor、test-generator	Code diff
Verifier	验证修复或功能是否真的成立	unit-test、integration-test、regression-test、log-verification	verification.md
Reviewer	反向审查，寻找遗漏、性能问题和并发风险	code-review、architecture-review、risk-review	review.md
Knowledge Curator	将一次性结果沉淀成可复用知识	adr-writer、runbook-writer、bugfix-note-writer	ADR、Runbook、Troubleshooting Note

总体流程可以理解为：

flowchart LR
    U[你提出目标] --> M[Manager Agent<br/>识别场景并拆分任务]
    M --> C[Context Scout<br/>读取项目上下文]
    M --> E[Evidence Collector<br/>搜集代码 日志 文档 PR]
    C --> A[Analyst / Architect<br/>建立假设或比较方案]
    E --> A
    A --> H{Human Approval Gate<br/>你审核}
    H -->|需要调整| E
    H -->|批准执行| I[Implementer<br/>修改代码或配置]
    I --> V[Verifier<br/>测试和验证]
    V --> R[Reviewer<br/>反向审查]
    R -->|发现问题| I
    R -->|通过| K[Knowledge Curator<br/>沉淀 ADR Runbook Bug Note]

⸻

三、使用统一的状态目录，不依赖 Chat Memory

建议在项目根目录中增加一个 AI 工作区：

.ai/
├── project-brief.md
├── task-state.yaml
├── plan.md
├── context.md
├── evidence.md
├── hypotheses.md
├── decision-log.md
├── verification.md
├── review.md
├── handoff.md
│
├── playbooks/
│   ├── feature-development.md
│   ├── technical-spike.md
│   ├── deployment-troubleshooting.md
│   └── code-bug-debugging.md
│
└── knowledge/
    ├── adr/
    ├── runbooks/
    ├── bug-fixes/
    └── reusable-patterns/

其中最重要的是 task-state.yaml。它是 Manager Agent 的任务看板：

task:
  title: "Investigate Kafka SaaS write failure in arsqa"
  type: "deployment-troubleshooting"
  status: "analyzing"
current_phase: "hypothesis-validation"
facts:
  - "events-gateway receives a write authorization error"
  - "the target topic exists in the shared UAT cluster"
open_questions:
  - "Does the identity pool have the producer role?"
  - "Is the arsqa topic name mapped correctly?"
hypotheses:
  - id: H1
    statement: "The application identity is missing the write RBAC role"
    confidence: high
    status: validating
next_actions:
  - "Compare RBAC assignments with the working cert environment"
  - "Verify the application principal used by arsqa"
artifacts:
  evidence: ".ai/evidence.md"
  handoff: ".ai/handoff.md"

每当完成一个阶段、切换 Agent、切换 Skill 或准备结束对话时，都更新：

.ai/task-state.yaml
.ai/handoff.md

这样即使上下文被 compact，新的 Agent 也可以先读取状态文件，再继续工作。

⸻

四、所有 Agent 都使用同一种证据格式

你贴图里有一个很重要的原则：区分事实和推断。

在软件工程场景里，建议要求所有 Agent 使用下面的格式：

## Facts
- [FACT] Error appeared at 10:42 AM in arsqa.
  Source: Splunk query / trace ID / deployment log
- [FACT] The same API works in cert with topic `c-cert-accounts-inbox`.
  Source: cert configuration file
## Inferences
- [INFERENCE] The failure is more likely related to RBAC than application code.
  Reason: The request reaches Kafka SaaS but is rejected during authorization.
## Recommendations
- [RECOMMENDATION] Compare the arsqa identity pool roles with cert before changing code.
## Unknowns
- [UNKNOWN] Whether the producer principal is using the expected service account.

这样可以避免 Agent 把猜测包装成结论。

⸻

五、四种典型场景应该使用不同 Playbook

场景 1：开发新需求或新功能

这个场景的重点是：先理解需求和影响范围，再改代码。

推荐 Agent 流程

阶段	负责 Agent	工作内容	输出
1. Requirement Intake	Manager Agent	读取 Jira、Acceptance Criteria、API Contract，明确业务目标	requirements.md
2. Repository Mapping	Context Scout	找到入口 API、Service、Repository、Kafka Topic、DB Table、下游依赖	context.md
3. Impact Analysis	Architect Agent	判断哪些模块会受影响，是否涉及兼容性、配置、监控和回滚	design.md
4. Human Review	你	检查设计方向，确认是否需要和架构师讨论	Approval
5. Implementation	Implementer	按 Milestone 修改代码，不一次性大改	Code diff
6. Verification	Test Agent	单元测试、集成测试、边界场景、回归测试	verification.md
7. Review	Reviewer	检查正确性、可读性、性能、并发安全、测试覆盖率	review.md
8. Sedimentation	Curator	生成 PR Summary；有重要决策时生成 ADR	PR Summary、ADR

一个适合你的 Milestone 拆分方式

以 Kafka Consumer 使用 Java Virtual Threads 为例，不应该直接让 Agent 修改所有消费者代码。更适合拆成：

Milestone 1: 找出哪些步骤是 blocking I/O
Milestone 2: 确认 Kafka consumer thread 和业务处理线程边界
Milestone 3: 建立 before / after 性能基线
Milestone 4: 只改一个处理链路做实验
Milestone 5: 验证吞吐量、延迟、线程数、错误率和消息顺序
Milestone 6: 再决定是否推广

⸻

场景 2：针对复杂问题做 Technical Spike，提出多个方案

这个场景不是让 Agent 立即执行，而是帮助你和经理、架构师做决策。

推荐 Agent Team

Agent	角色
Researcher	找官方文档、现有内部设计、历史 PR、同类服务做法
Option Designer A	给出方案 A，并写清楚适用条件
Option Designer B	给出方案 B，并写清楚适用条件
Devil’s Advocate	专门挑战每个方案，寻找失败场景和隐藏成本
Architect	综合比较，给出推荐方案
Writer	整理成可以拿去开会的 Spike Report

输出结构

# Technical Spike: Shared Confluent Cloud UAT Isolation Strategy
## 1. Context
多个应用 UAT 环境需要映射到一个共享的 Confluent Cloud UAT Cluster。
## 2. Confirmed Facts
- App environments: cert, qa, perf ...
- Kafka SaaS UAT cluster is shared.
- Logical isolation is required.
## 3. Options
### Option A: Environment prefix in topic names
Example: `c-cert-accounts-inbox`
### Option B: Separate Kafka clusters
...
## 4. Comparison
| Dimension | Option A | Option B |
|---|---|---|
| Isolation | Logical | Physical |
| Cost | Lower | Higher |
| Operational complexity | Lower | Higher |
## 5. Recommendation
Use Option A for non-production environments.
## 6. Risks
- Misconfigured producer may publish to the wrong environment topic.
- Consumer group naming must also be isolated.
## 7. Open Questions for Architect
- Do we need ACL isolation at the identity-pool level?
- How should replay tools select the target environment?

这里的关键是：让不同 Sub-Agent 独立提出意见，再由 Architect 汇总。
不要一开始就让一个 Agent 得出唯一答案，否则容易出现确认偏差。

⸻

场景 3：部署问题或环境问题排查

这个场景最需要纪律。不要让多个 Agent 同时修改配置，也不要让 Agent 看到一个错误就直接改代码。

推荐采用 Evidence-First Incident Workflow。

推荐流程

阶段	Agent	工作内容
1. Incident Intake	Incident Manager	明确环境、时间、错误、影响范围、最近变更
2. Evidence Collection	Log Investigator	查 Dynatrace、Splunk、pipeline log、deployment history
3. Historical Search	Knowledge Scout	用准确错误信息搜索 Jira、历史 Bitbucket PR、Confluence Primer、本地 Wiki、旧 AI State
4. Environment Comparison	Config Comparator	对比工作环境和故障环境，例如 cert vs arsqa
5. Hypothesis Ranking	Analyst	按证据强弱排序，不直接执行
6. Approval Gate	你	选择最小风险验证动作
7. Minimal Fix	Executor	只实施最小改动
8. Verification	Verifier	验证日志、Trace、健康检查和 E2E
9. Documentation	Curator	写 Runbook 或 Known Issue

建议让 Agent 按固定顺序排查

1. 问题是否可以稳定复现？
2. 哪个环境失败？哪个环境正常？
3. 最近发生过什么 deployment、config 或 permission 变化？
4. 请求是否到达应用？
5. 请求是否到达下游？
6. 是代码问题、配置问题、权限问题、网络问题还是依赖问题？
7. 历史上是否出现过同样的错误？
8. 最小验证动作是什么？
9. 最小修复动作是什么？
10. 修复后如何证明问题已经解决？

对你常见的 Kafka SaaS 场景，可以并行启动多个只读 Agent

Sub-Agent	负责检查
Kafka Config Agent	Topic 名称、Bootstrap Server、Consumer Group、环境映射
RBAC Agent	Identity Pool、Service Account、Write Role、Read Role
Deployment Agent	Pipeline、property file、secret injection、最近部署版本
Observability Agent	Dynatrace Trace、Splunk Error、时间窗口、关联 ID
Comparison Agent	将 arsqa 和正常工作的 cert 环境做 diff

但是只有一个 Executor Agent 可以修改配置。其他 Agent 只读，避免多个修改互相污染结果。

⸻

场景 4：程序 Bug 调试

这个场景需要逐步收窄范围，而不是让 Agent 立刻重写代码。

推荐流程

阶段	Agent	输出
1. Reproducer	明确输入、预期结果、实际结果、复现步骤	reproduction.md
2. Code Tracer	从入口沿调用链逐步追踪	trace.md
3. Hypothesis Builder	列出可能根因，并为每一个假设定义验证方法	hypotheses.md
4. Debugger	加断点、临时日志或最小测试，逐步排除假设	evidence.md
5. Patch Agent	只生成最小修复	Code diff
6. Test Agent	增加回归测试和边界测试	Tests
7. Reviewer	检查正确性、异常处理、性能和线程安全	review.md
8. Curator	记录根因和以后如何快速识别	bug-fix-note.md

对长代码或分多次发送的代码，需要维护 Live Model

不要每次收到新代码片段就重新开始分析。让 Agent 更新：

.ai/live-code-model.md

格式如下：

## Confirmed Structure
- `processEvent()` calls `validate()` before publishing to Kafka.
- The retry path invokes `sendToSafStore()`.
## Provisional Findings
- The duplicate message may be caused by retry logic.
- Thread safety of the shared map is not yet confirmed.
## Missing Context
- Implementation of `retryFailedEvents()`
- Kafka acknowledgment configuration
## Re-evaluation After Latest Chunk
- Previous hypothesis H1 remains possible.
- Hypothesis H2 is less likely because the shared object is request scoped.

每收到新的函数或类，Agent 都要重新评估整体调用链，而不是只分析最新的一段代码。

对于你的 Java 和 Kafka 项目，Reviewer 还应该强制检查：

- Kafka acknowledgment 和重试语义
- 是否可能重复消费或重复发送
- Thread safety
- Virtual Thread 下是否存在 blocking 或 pinning 风险
- Shared mutable state
- Timeout、backoff 和 replay
- Cassandra 与 Kafka 写入顺序
- At-least-once delivery 下的幂等性

⸻

六、不同复杂度使用不同模式，不要过度设计

不是每一个任务都需要八个 Agent。

任务复杂度	适合的模式	示例
简单	一个 Agent + 一个 Skill	修改一个配置项、修复拼写错误
中等	Manager + 2 至 3 个 Sub-Agent	小型功能、一般 Bug、单环境部署问题
复杂	Manager + 多个只读 Agent + Approval Gate + Executor + Reviewer	跨团队方案、Kafka SaaS 环境设计、生产或 UAT 故障
需要决策	Researcher + 多个 Option Agent + Devil’s Advocate + Architect	Technical Spike、架构讨论

一个重要原则是：

并行化适合用于“读取、研究、比较和审查”，不适合用于“多个 Agent 同时修改同一份代码或同一个环境”。

⸻

七、结果如何沉淀

并不是所有结果都要保存成永久文档。可以按照下面的规则判断：

结果类型	保存位置	什么时候保存
临时排查证据	.ai/evidence.md	当前任务期间
中间状态和下一步	.ai/task-state.yaml、.ai/handoff.md	每个阶段结束后
重要设计决策	knowledge/adr/	有多个方案、存在权衡、以后可能再次讨论
可重复部署排查流程	knowledge/runbooks/	类似问题可能再次发生
典型 Bug 根因	knowledge/bug-fixes/	根因隐蔽，未来可以通过关键词快速命中
可复用代码模式	knowledge/reusable-patterns/	适合多个服务或模块
一次性小修改	PR Summary 即可	不需要额外创建长文档

例如，你解决了 Kafka SaaS write failure 后，可以沉淀：

# Kafka SaaS Write Authorization Failure
## Symptom
`events-gateway` fails when publishing to the inbox topic.
## Root Cause
The application identity pool had no write role for the target topic.
## Fast Diagnosis
1. Check the exact topic name.
2. Identify the service account used by the environment.
3. Compare RBAC assignments with a working environment.
4. Confirm the producer role is attached.
## Fix
Grant the topic write role to the correct identity pool.
## Verification
Publish one test event and confirm successful consumption downstream.

下次出现类似问题时，Knowledge Scout 可以优先搜索这个文件，而不是从头分析。

⸻

八、建议先实现的最小版本

不要一开始就创建几十个 Skills。第一版只需要：

1. project-context-loader
2. code-and-config-search
3. troubleshooting-evidence-collector
4. hypothesis-and-solution-analyzer
5. test-and-verification-runner
6. code-reviewer
7. state-save-and-handoff
8. knowledge-curator

然后创建四个 Playbook：

feature-development
technical-spike
deployment-troubleshooting
code-bug-debugging

Manager Agent 的职责只是：

识别当前任务属于哪个 Playbook
→ 读取现有 AI State
→ 按 Playbook 调用 Skills 和 Sub-Agents
→ 要求每个 Agent 写入指定文件
→ 在关键步骤停下来等待你的审核
→ 最后生成沉淀文档

⸻

九、可以直接使用的 Manager Agent Prompt

You are the Software Engineering Orchestrator.
Your job is not to immediately modify code. Your job is to classify the task,
select the correct playbook, delegate bounded tasks to specialist agents,
maintain shared state files, enforce quality gates, and synthesize the final result.
Always start by reading:
1. .ai/project-brief.md
2. .ai/task-state.yaml
3. .ai/handoff.md
4. The relevant file under .ai/playbooks/
Classify the task into one of these modes:
- feature-development
- technical-spike
- deployment-troubleshooting
- code-bug-debugging
Use specialist agents for research, code navigation, log investigation,
solution analysis, implementation, verification, review, and documentation.
Rules:
1. Do not rely on chat memory. Use local files as the interface between agents.
2. Separate every important statement into FACT, INFERENCE, RECOMMENDATION, or UNKNOWN.
3. Record the source for every important fact: file path, log query, trace ID,
   Jira ticket, PR, documentation page, or test result.
4. Parallelize read-only research and review tasks.
5. Allow only one Executor Agent to modify code or configuration.
6. Do not make production or shared-environment changes without human approval.
7. Prefer the smallest safe change.
8. After every milestone, update:
   - .ai/task-state.yaml
   - .ai/handoff.md
9. Before declaring completion, require verification and independent review.
10. After completion, decide whether the result should be saved as:
    - ADR
    - Runbook
    - Bug-fix note
    - Reusable pattern
    - PR summary only

⸻

十、最适合你的最终工作模型

你可以把它记成六个词：

Context → Evidence → Analysis → Approval → Execution → Knowledge

中文就是：

先了解上下文
→ 再收集证据
→ 然后分析和提出方案
→ 由你审核
→ 再执行和验证
→ 最后沉淀成知识

你不是让 AI 替代软件工程师。你是在建立一个小型虚拟工程团队：AI 负责并行研究、整理证据、执行标准化动作和输出文档；你负责做方向判断、风险控制和最终决策。


