可以把 OMC（oh-my-claudecode） 理解成：给 Claude Code 安装了一套“虚拟软件研发部门操作系统”。

它不是单独的一个 Skill，也不是几个简单 Prompt 的集合。它将 Manager、专业 Sub-Agent、Skills、任务状态、交接文档、自动化 Hooks、验证循环 打包成了一个可直接运行的多 Agent 编排层。项目自己的描述也是：OMC 是 Claude Code 的 multi-agent orchestration layer，负责协调专业 Agent、工具和 Skills。 

它和我们前面讨论的架构非常相似。更准确地说：我们前面设计的是一套适合你工作方式的轻量级工程架构；OMC 是一个已经产品化、自动化程度更高的通用实现。

⸻

一、先用一个容易理解的比喻

没有 OMC 时，你使用 Claude Code 或 Copilot，就像雇佣了一位能力很强的全栈工程师：

你提出问题
↓
同一个 AI 查代码、想方案、改代码、写测试、审查自己的修改
↓
输出结果

问题是：同一个 Agent 同时扮演研究员、架构师、开发人员和审核人员，容易出现上下文混乱、自我确认偏差以及中间状态丢失。

安装 OMC 后，更像拥有一个完整的小型研发团队：

你：产品负责人 / 最终审批人
↓
Lead Orchestrator：工程经理
↓
按任务阶段分配工作
├── explore：搜索代码和上下文
├── analyst：澄清需求
├── planner：拆解任务
├── architect：分析系统设计
├── tracer：调查根因
├── debugger：定位 Bug
├── executor：修改代码
├── test-engineer：编写测试
├── verifier：验证结果
├── security-reviewer：安全审查
└── code-reviewer：独立代码审查
↓
将状态、证据、交接记录写入文件
↓
验证失败后进入修复循环

OMC 的 Agent Catalog 中确实定义了 explore、analyst、planner、architect、debugger、executor、verifier、tracer、security-reviewer、code-reviewer、test-engineer 等角色。 

⸻

二、OMC 的核心架构

OMC 可以拆成六层。

第 1 层：Lead Orchestrator，也就是 Manager Agent

OMC 会向 Claude Code 注入一套编排规则。它要求主 Agent：

* 简单问题直接处理；
* 多文件修改、调试、规划、调研和验证任务交给专业 Agent；
* 选择合适的 Agent；
* 根据任务难度选择不同等级的模型；
* 验证通过后才能宣布完成；
* 写代码和审核代码必须分开进行，不能让同一个上下文自我批准。 

所以，OMC 的 Manager 并不是单独的一个 engineering-manager.md 文件。它更接近：

全局编排规则
+ Team Skill
+ 状态管理工具
+ Hooks
+ 专业 Agent Catalog

我们前面设计的 Engineering Manager Agent，可以看成 OMC Lead Orchestrator 的轻量版本。

⸻

第 2 层：专业 Sub-Agent

OMC 将不同职责分配给不同 Agent，而不是让所有 Agent 都是“万能程序员”。

例如：

OMC Agent	角色定位	适合你的工作场景
explore	搜索代码、配置和上下文	找调用链、找类似实现
analyst	澄清模糊需求和边界	新需求分析、Technical Spike
planner	将目标拆成任务图	开发新功能
architect	系统设计、复杂边界分析	和经理、架构师讨论方案
tracer	证据驱动的根因调查	部署异常、复杂生产问题
debugger	定位编译错误、回归和 Bug	Java 程序 Bug
executor	修改代码	实现已经批准的方案
test-engineer	增加测试	单元测试、回归测试
verifier	独立验证结果	检查是否真正完成
code-reviewer	独立审查	检查质量和副作用
security-reviewer	安全审查	Authentication、Token、加密相关修改

OMC 的 team Skill 会根据不同阶段自动选择适合的 Agent，而不是始终启动一组 executor。例如，规划阶段使用 explore 和 planner，验证阶段必须使用 verifier，安全敏感变更还会增加 security-reviewer。 

⸻

第 3 层：阶段化流水线

OMC 推荐的 Team 模式不是“启动几个 Agent 并行干活”这么简单，而是按照固定流水线运行：

team-plan
↓
team-prd
↓
team-exec
↓
team-verify
↓
team-fix
↓
验证失败则循环

OMC 文档明确将 Team 定义为推荐的标准编排方式，并使用：

team-plan → team-prd → team-exec → team-verify → team-fix (loop)

作为核心流水线。 

每一个阶段都有进入和退出条件：

阶段	目的	对应 Agent
team-plan	搜索代码、拆解任务、建立任务图	explore、planner、可选 architect
team-prd	澄清范围、验收条件和边界	analyst、可选 critic
team-exec	实现具体任务	executor、debugger、designer、test-engineer
team-verify	验证测试、审查代码、检查风险	verifier、code-reviewer、security-reviewer
team-fix	修复验证阶段发现的问题	executor、debugger

⸻

第 4 层：状态管理和交接文档

这一点和我们前面讨论的 .ai/task-state.yaml、.ai/handoff.md 几乎完全一致。

OMC 不依赖聊天记录作为唯一记忆，而是把状态写入 .omc/：

.omc/
├── specs/
├── plans/
├── state/
├── handoffs/
├── notepads/
├── artifacts/
├── skills/
├── project-memory.json
└── notepad.md

OMC 的参考文档明确规定：

* .omc/specs/ 或 .omc/plans/ 保存目标、约束和批准后的计划；
* .omc/state/ 保存机器可读的流程状态；
* .omc/state/team/ 保存 Team 的任务、状态、消息和调度元数据；
* .omc/notepads/ 保存执行过程中的持久化笔记；
* .omc/project-memory.json 保存可复用的项目知识。 

更重要的是，OMC 的 Team Skill 要求：每次从一个阶段进入下一个阶段之前，都必须生成 Handoff 文件。

.omc/handoffs/team-plan.md
.omc/handoffs/team-prd.md
.omc/handoffs/team-exec.md

交接文件只保留重要信息：

## Handoff: team-plan → team-exec
- Decided: 已批准的技术方向
- Rejected: 被排除的方案以及原因
- Risks: 下一阶段需要关注的风险
- Files: 关键文件
- Remaining: 尚未完成的任务

这样即使上下文被 compact、Agent 重启或任务暂停，后续 Agent 仍然能够恢复工作。 

这正是你之前一直关心的问题：如何防止 Copilot compact 以后丢失 AI State。

⸻

第 5 层：Skills

OMC 的 Skill 不仅是一个 Prompt，而是标准化工作流。

例如：

Skill	用途
team	启动阶段化多 Agent 流水线
autopilot	端到端自主开发
ralph	持续执行，直到验证完成
ultrawork	最大化并行执行
ultraqa	循环执行 QA 检查
trace	用证据驱动方式分析根因
deep-interview	在编码之前澄清模糊需求
skillify	将本次经验提炼为可复用 Skill

OMC 将不同模式区分开：Team 适合有共享任务列表的多 Agent 协作；Autopilot 适合端到端功能开发；Ralph 适合必须完成的任务；Pipeline 适合严格顺序执行的任务。 

⸻

第 6 层：Hooks

OMC 通过 Hooks 将一些关键动作变成确定性行为，而不是仅仅依赖 AI “记得做”。

它注册了覆盖多个生命周期事件的 Hook，例如：

SessionStart
UserPromptSubmit
PreToolUse
PostToolUse
SubagentStart
SubagentStop
PreCompact
Stop
SessionEnd

OMC 会在不同节点执行 Skill 注入、Project Memory 更新、工具调用校验、Sub-Agent 跟踪、Compact 前保存状态、Stop 时检查是否真的可以结束等操作。 

这一点非常关键：

Prompt：希望 Agent 这样做
Hook：保证某个时刻一定执行脚本

⸻

三、OMC 和我们前面设计的架构如何对应？

两套架构的核心思想高度一致。

我们前面设计的组件	OMC 中的对应组件
Engineering Manager Agent	Lead Orchestrator + team Skill
Context Scout	explore、document-specialist
Solution Analyst	analyst、planner、architect、critic
Implementer	executor、debugger
Verifier	verifier、qa-tester、test-engineer
Reviewer	code-reviewer、security-reviewer
Knowledge Curator	skillify、Project Memory、Notepad
.ai/task-state.yaml	.omc/state/
.ai/handoff.md	.omc/handoffs/<stage>.md
Playbooks	OMC Skills
PreCompact Hook	OMC PreCompact Hooks
Approval Gate	OMC Verify/Fix Gate，加上你自己的人工审核点
Jira、Splunk、Confluence MCP	OMC 外部工具或 MCP 接入点

可以把两套方案总结为：

我们前面的方案：
为你的工作场景量身定制的最小可用架构
OMC：
已经实现了大量通用能力的完整 Claude Code 编排框架

⸻

四、最相似的部分：OMC 的 Trace Skill

对于你经常遇到的部署问题、Kafka SaaS 权限问题、Pipeline 问题和程序 Bug，OMC 中最值得借鉴的是 /trace。

它要求将问题拆成：

Observation
↓
Competing Hypotheses
↓
Evidence For
↓
Evidence Against / Gaps
↓
Current Best Explanation
↓
Critical Unknown
↓
Discriminating Probe

也就是：

观察到的事实
↓
多个相互竞争的根因假设
↓
支持每个假设的证据
↓
反对每个假设的证据
↓
当前最可能的解释
↓
目前最关键的未知信息
↓
成本最低、区分度最高的下一步验证动作

OMC 还要求对复杂问题默认建立多个 Tracer Lane，每一个 Sub-Agent 负责一个不同假设，而不是让多个 Agent 重复调查同一个方向；最后再让领先假设和第二名进行一次反驳。 

例如，你遇到：

events-gateway 在 arsqa 无法写入 Kafka SaaS inbox topic

可以分成三个 Lane：

Lane 1：RBAC 权限缺失
Lane 2：Topic name 环境映射错误
Lane 3：Service account 或 identity pool 绑定错误

每个 Agent 独立找证据，最后由 Manager 汇总：

最可能根因：RBAC 缺少 write role
关键未知信息：arsqa 实际使用哪个 service principal
下一步验证：将 arsqa identity pool 和 cert identity pool 的 RBAC 做 diff

这比“看到报错以后直接修改配置”可靠得多。

⸻

五、最值得你借鉴的部分：Skillify

OMC 的 skillify 很接近我们前面定义的 Knowledge Curator。

它要求从已经完成的任务中提取：

* 输入；
* 有序步骤；
* 成功条件；
* 限制和常见陷阱；
* 验证证据；
* Skill 应该保存在哪里。 

它还设置了质量门槛：

这个知识能否通过 Google 搜索 5 分钟得到？
是否和当前代码库、项目或内部流程有关？
是否确实经过了有价值的调试、设计或运维工作？

只有满足这些条件，才值得沉淀成 Skill。 

例如：

如何写一个普通 Java try-catch

不需要保存为 Skill。

但是：

arsqa 环境中 events-gateway 写入 Kafka SaaS topic 失败时，
如何确认 identity pool、RBAC role、topic prefix 和 service account 映射

非常适合保存成内部 Skill。

⸻

六、OMC 和我们架构的主要区别

区别 1：OMC 更像一个 Runtime，不只是 Prompt

我们前面设计的第一版架构主要依赖：

Custom Agents
+ Skills
+ 状态文件
+ 少量 Hooks

OMC 则进一步实现了：

任务拆解
任务依赖关系
Team 创建
Worker 分配
Worker 消息
Worker 状态跟踪
阶段切换
断点恢复
验证失败循环
模型路由
Compact 前保存状态

例如，OMC 的 Team Skill 会创建 Team、拆分任务、分配 Owner、启动 Worker、轮询进度、处理卡住的 Agent，并在验证失败后进入 team-fix。 

⸻

区别 2：OMC 自动化程度更高

我们前面的架构强调：

Context
→ Evidence
→ Analysis
→ Human Approval
→ Execution
→ Verification
→ Knowledge

OMC 默认更偏向：

Plan
→ PRD
→ Execute
→ Verify
→ Fix Loop
→ Complete

OMC 还提供 autopilot、ralph 和 ultrawork 等高自动化模式。 

对于个人开源项目，这种自动化很方便。

但是对于你的企业环境，尤其是：

共享 UAT Kafka Cluster
RBAC 权限
Pipeline 配置
Cassandra
生产或类生产环境

建议继续保留人工审核点：

只读调查可以自动化
代码修改需要审核
环境配置修改必须审核
部署和权限变更必须审核

⸻

区别 3：OMC 是为 Claude Code 设计的，不是为 Copilot 设计的

OMC 官方文档明确说明，它通过 Claude Code Plugin 系统安装，并使用 Node.js Hooks。它要求安装 Claude Code。 

你现在主要使用的是 VS Code 中的 GitHub Copilot。VS Code 本身也支持 Custom Agents、Agent Skills、MCP、Hooks 和 Sub-Agent 委派，因此可以实现类似架构。 

但不要假设 OMC 可以原封不动安装到 Copilot 中。

更准确的理解是：

OMC 是 Claude Code 生态中的成熟实现
↓
你可以借鉴它的设计
↓
在 Copilot VS Code 中创建自己的轻量版本

VS Code 可以读取部分 Claude 风格 Hook 配置，但官方文档明确提醒：Claude Code 和 VS Code 的工具名称、参数命名和 Matcher 行为存在差异，因此 Hook 脚本需要适配，不能直接复制。 

⸻

七、你应该直接使用 OMC，还是参考 OMC 自己搭建？

方案 A：继续使用 Copilot，在 VS Code 中实现轻量版 OMC

这更适合你当前的工作环境。

建议保留：

.github/
├── agents/
│   ├── engineering-manager.agent.md
│   ├── context-scout.agent.md
│   ├── solution-analyst.agent.md
│   ├── tracer.agent.md
│   ├── implementer.agent.md
│   ├── verifier.agent.md
│   ├── reviewer.agent.md
│   └── knowledge-curator.agent.md
│
├── skills/
│   ├── feature-development/
│   ├── technical-spike/
│   ├── deployment-troubleshooting/
│   ├── code-bug-debugging/
│   ├── trace-investigation/
│   └── skillify/
│
└── hooks/
    ├── state-snapshot.json
    └── approval-guard.json
.ai/
├── state/
├── handoffs/
├── evidence/
└── knowledge/

第一阶段不需要照搬 OMC 的全部复杂能力。先复制它最有价值的五个模式：

1. 阶段化流水线
2. 专业角色分工
3. 持久化 State 和 Handoff
4. Trace 证据链
5. Skillify 知识沉淀

VS Code 官方已经支持 Custom Agent、Skills、MCP 和 Hooks 的组合方式；Custom Agent 可以限制工具，也可以委派给其他 Agent。 

⸻

方案 B：在个人项目中试用 OMC

你可以在独立的个人项目或 Sandbox 中试用 OMC，观察它如何：

拆解任务
调度 Agent
生成 Handoff
保存 State
执行 Verify/Fix Loop
提取 Skills

然后挑选适合企业环境的模式移植到 Copilot。

不要直接在公司主项目中启用未经审核的第三方 Hook。OMC 会注册多个生命周期 Hook，而 Hooks 可以执行脚本。VS Code 官方也提醒：Hooks 以 VS Code 的权限运行，启用来自不可信来源的 Hook 前必须审查脚本、限制权限，并避免硬编码凭据。 

⸻

八、针对你的四类工作场景，如何借鉴 OMC？

你的工作场景	最适合借鉴的 OMC 模式
开发新需求、新功能	team-plan → team-prd → team-exec → team-verify → team-fix
对复杂问题提出可行方案	analyst + planner + architect + critic
部署、环境、Kafka、Pipeline 故障	/trace：多个根因 Lane + 证据对比
Java 程序 Bug 调试	tracer → debugger → executor → verifier → code-reviewer
经验沉淀	/skillify
防止 compact 丢失上下文	.ai/state/ + .ai/handoffs/ + PreCompact Hook

对于你的场景，最优先实现的并不是 autopilot，而是：

trace-investigation
↓
human approval
↓
minimal implementation
↓
independent verification
↓
skillify

因为你的很多任务涉及跨服务依赖、共享测试环境、Kafka SaaS、权限和部署配置。调查是否准确，比自动修改代码更重要。

⸻

九、一个需要注意的小问题

OMC 项目迭代速度很快。当前仓库的不同文档中，专业 Agent 数量存在不一致：参考文档写的是 29 个，英文 README 写的是带 Tier Variant 的 19 个专业 Agent，中文 README 中则写了 32 个。 

这不影响理解它的核心架构，但意味着：

不要过度关注 Agent 的准确数量
不要一次性照搬全部 Agent
优先复制稳定的设计模式

⸻

十、最终理解

OMC 和我们前面讨论的方案，本质上使用了同一个思想：

一个 Manager 不负责亲自做所有事情
↓
根据阶段调用专业 Sub-Agent
↓
Sub-Agent 使用 Skills 和 Tools 完成边界清晰的任务
↓
通过文件交接状态和证据
↓
由独立 Agent 验证和审查
↓
将有价值的经验提炼为新的 Skill

不同之处是：

我们的方案：
更轻量、更适合 Copilot、更容易按企业规则控制风险
OMC：
更完整、更自动化、更适合 Claude Code，已经实现大量编排基础设施

对于你当前的 Copilot VS Code 工作流，最合理的路线是：把 OMC 当成参考架构，先实现一套最小可用的 Copilot 版 OMC，而不是直接复制整个项目。