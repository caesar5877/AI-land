OMC 最合理的使用方式是 Hybrid：框架安装在 User Scope，项目行为和项目记忆放在 Workspace Scope。

不要把所有内容都装成全局配置，也不要在每个项目中复制完整的 OMC Framework。

可以把它理解为 IntelliJ 或 VS Code：

VS Code / IntelliJ 本体             → 安装一次，属于用户
项目代码、项目配置、运行记录         → 每个 Workspace 独立维护

OMC 也应该这样分层。

⸻

一、先区分 OMC 的四类内容

内容	推荐 Scope	原因
OMC Plugin、内置 Agents、内置 Skills、Hooks Runtime	User-based	安装一次，所有项目复用
OMC 对 Claude Code 的编排规则	Workspace-based	不同项目的工作方式、风险和约束不同
当前任务的 Plan、State、Evidence、Handoff	Workspace-based	必须与代码库绑定，不能污染其他项目
你自己创建的 Skills	根据用途决定	通用 Skill 放 User Scope；项目专属 Skill 放 Workspace

OMC 官方文档也推荐使用 Project-Scoped Configuration。执行本地初始化后，它会在当前项目中创建 ./.claude/CLAUDE.md，配置只对当前项目生效，不会影响其他项目。 

⸻

二、推荐给你的安装方式

第一步：在 User Scope 安装 OMC Plugin

在 Claude Code 中执行一次：

/plugin marketplace add https://github.com/Yeachan-Heo/oh-my-claudecode

然后：

/plugin install oh-my-claudecode

Plugin 会集成到 Claude Code 的 Plugin System 中，并使用 Node.js Hooks。你不需要在每一个 Workspace 中重复安装 Plugin。 

⸻

第二步：在每一个需要使用 OMC 的 Workspace 中初始化

进入项目根目录：

cd ~/workspace/notification-platform
claude

然后执行：

/oh-my-claudecode:omc-setup --local

它会创建：

notification-platform/
└── .claude/
    └── CLAUDE.md

这意味着 OMC 的 Manager Agent、自动路由规则和项目约束只在当前项目中生效。 

以后换到另一个项目，例如：

cd ~/workspace/authentication-service
claude

再次执行：

/oh-my-claudecode:omc-setup --local

两个项目相互独立。

⸻

三、为什么不建议一开始使用 Global Configuration？

OMC 也支持全局配置：

/oh-my-claudecode:omc-setup

它会创建：

~/.claude/CLAUDE.md

并让 OMC 编排规则应用到所有 Claude Code Session。官方文档提醒：默认行为会覆盖现有的全局 ~/.claude/CLAUDE.md；也可以使用 Preserve Mode 保留原始文件。 

对于你的工作方式，我不建议一开始这么做。

你可能同时使用 Claude Code 处理：

公司 Java / Kafka 项目
个人 AI Agent 实验项目
简单脚本
前端项目
学习用途的 Sample Project

不同项目需要不同约束。

例如，在公司项目中，你可能希望：

- 不允许自动修改 Production 配置
- 修改 Kafka RBAC 前必须等待人工批准
- 优先比较 arsqa 和 cert 配置
- 不要将日志中的敏感信息写入文件
- Java 修改后必须运行 Maven 测试

而在个人 Demo 项目中，你可能允许：

- 使用 autopilot
- 自动修改代码
- 自动创建文件
- 自动运行测试
- 快速迭代，不需要每一步人工批准

使用 Workspace Scope 更安全，也更容易逐步调整。

⸻

四、OMC 的运行状态默认放在哪里？

OMC 默认将项目状态保存在当前 Worktree 的：

.omc/

典型结构可以理解为：

notification-platform/
├── .claude/
│   └── CLAUDE.md
│
├── .omc/
│   ├── specs/
│   ├── plans/
│   ├── state/
│   ├── notepads/
│   ├── artifacts/
│   ├── skills/
│   ├── project-memory.json
│   └── notepad.md
│
└── src/

.omc/ 中保存项目相关的计划、任务状态、Team 协作信息、调查结果、Notepad 和 Project Memory。官方文档将 .omc/ 定义为 OMC 的 Project-Local Root。 

其中常见内容包括：

文件或目录	作用
.omc/specs/<task>.md	需求、约束和验收标准
.omc/plans/<task>.md	已经审核的执行计划
.omc/state/	当前任务状态
.omc/state/team/	多 Agent Team 的调度状态
.omc/notepads/	执行过程中的持久化笔记
.omc/project-memory.json	项目长期知识
.omc/artifacts/	Advisor、Trace 或调查结果

这些信息应该与项目绑定，而不是所有项目共享。

⸻

五、你的场景中，推荐采用三层结构

第一层：User Scope，安装通用能力

~/.claude/
├── plugins/
├── agents/
├── skills/
├── hooks/
└── hud/

这里放：

OMC Plugin
OMC 自带的 Agents
OMC 自带的 Skills
通用 Hooks
你个人通用的 Skills

例如，你自己创建的这些 Skill 可以放在 User Scope：

standup-update-writer
pr-summary-writer
generic-java-code-review
general-git-troubleshooting
technical-spike-template

因为它们可以跨项目复用。

⸻

第二层：Workspace Scope，放项目规则

notification-platform/
└── .claude/
    ├── CLAUDE.md
    ├── skills/
    └── omc.jsonc

这里放项目专属规则，例如：

- 当前项目使用 Java 21 和 Spring Boot
- 使用 Confluent Cloud
- 多个 App UAT 环境映射到共享 Kafka SaaS UAT
- Topic 命名格式为 <environment>-<subdomain>-<topic>
- 排查日志时优先查看 Dynatrace，再查看 Splunk
- 部署和 RBAC 修改必须获得人工批准
- 禁止将 Token、客户数据和完整日志提交到 Git

OMC 支持项目级配置文件 .claude/omc.jsonc，也支持用户级配置文件 ~/.config/claude-omc/config.jsonc。 

⸻

第三层：Workspace Scope，放 AI State 和知识沉淀

notification-platform/
└── .omc/
    ├── plans/
    ├── state/
    ├── handoffs/
    ├── notepads/
    ├── artifacts/
    └── skills/

这里放：

当前排查进度
当前 Feature Plan
假设和证据
Agent 之间的交接记录
与当前代码库相关的 Runbook
项目专属 Skill

例如：

.omc/skills/kafka-saas-rbac-troubleshooting/
.omc/skills/arsqa-deployment-validation/
.omc/skills/webhook-replay-debugging/

OMC 官方将 .omc/skills/ 定义为项目本地 Skill 的标准目录，同时也可以读取 .claude/skills/ 和兼容目录 .agents/skills/。 

⸻

六、公司项目中，哪些内容不应该跨 Workspace 共享？

下面这些内容不适合放在全局用户目录：

特定项目的 Topic 名称
内部 Service URL
环境名称
日志内容
Trace ID
Jira Ticket
内部架构图
RBAC Mapping
Service Account
内部 Runbook
公司专属 Prompt

原因有两个：

1. Agent 在其他项目中可能错误引用旧项目的上下文。
2. 内部信息可能被意外带到个人项目或其他 Repository 中。

所以需要遵守一个清晰规则：

通用方法论 → User Scope
项目事实和内部知识 → Workspace Scope

例如：

Skill	Scope
如何做 Evidence-First Troubleshooting	User
如何排查当前项目的 Kafka SaaS Write Role	Workspace
如何 Review Java 并发代码	User
当前项目 Java Virtual Threads 的使用限制	Workspace
如何生成 Technical Spike Report	User
当前项目的 shared UAT Kafka Mapping 设计	Workspace

⸻

七、是否应该把 .omc/ 提交到 Git？

需要分类处理。

建议提交到 Git 的内容

.omc/skills/
.omc/specs/ 中经过审核的正式需求
.omc/plans/ 中值得团队共享的实施方案
docs/runbooks/
docs/adr/

这些文件是团队知识，可以进行 Code Review。

建议加入 .gitignore 的内容

.omc/state/
.omc/notepads/
.omc/artifacts/
.omc/logs/
.omc/sessions/

这些目录可能包含：

临时状态
Agent 调度信息
日志片段
本地路径
未经审核的推断
中间结果

在公司项目中，需要避免把敏感信息、完整日志和临时 Agent 状态提交到 Repository。

可以采用：

# OMC ephemeral state
.omc/state/
.omc/notepads/
.omc/artifacts/
.omc/logs/
.omc/sessions/
# Keep reviewed reusable knowledge
!.omc/skills/
!.omc/plans/
!.omc/specs/

实际提交前仍然应该人工检查文件内容。

⸻

八、使用 Git Worktree 时需要额外注意

你如果使用多个 Git Worktree，例如：

notification-platform-main/
notification-platform-feature-a/
notification-platform-bugfix/

OMC 默认将状态保存在每个 Worktree 自己的：

{worktree}/.omc/

删除 Worktree 后，其中的状态也会消失。 

如果希望 AI State 在 Worktree 被删除后仍然保留，可以设置：

export OMC_STATE_DIR="$HOME/.claude/omc"

OMC 会改为使用：

~/.claude/omc/{project-identifier}/

OMC 会根据 Git Remote URL 生成稳定的项目标识，因此状态依然按照项目隔离，不会将所有项目混合到一起。 

对于你之前关心的 Compact、Session Handoff 和任务恢复问题，这种配置比较合适。

建议加入：

# ~/.zshrc
export OMC_STATE_DIR="$HOME/.claude/omc"

这样可以同时满足：

状态持久化
跨 Worktree 恢复
不同项目隔离
避免将临时 State 提交到 Git

⸻

九、多 Repository Workspace 如何处理？

你的一个系统可能包含多个 Repository，例如：

notification-platform/
├── events-gateway/
├── events-processor/
├── webhook-service/
└── replay-service/

如果最外层目录本身不是 Git Repository，OMC 默认可能为每一个子 Repository 创建独立状态。

如果你希望它们共享一个工作空间状态，可以在父目录创建：

cd notification-platform
echo '{}' > .omc-workspace

.omc-workspace 会告诉 OMC：将父目录视为共享 Workspace Root。官方文档专门为 Multi-Repo Workspace 提供了这个机制。 

这样可以让多个相关服务共享：

一个 Technical Spike
一个跨服务 Trace 调查
一个 E2E Plan
一组 Handoff 文件

⸻

十、适合你的最终配置

对于你的公司项目，推荐这样配置：

User Scope
├── 安装 OMC Plugin 一次
├── 保存通用 Agents 和 Skills
└── 设置 OMC_STATE_DIR="$HOME/.claude/omc"
Workspace Scope
├── 使用 /oh-my-claudecode:omc-setup --local
├── 维护 .claude/CLAUDE.md
├── 保存项目特有规则
├── 保存项目专属 Skills
├── 持久化 Plan、Handoff 和 Project Memory
└── 将临时 State、Logs 和 Artifacts 加入 .gitignore

可以用一句话记忆：

OMC Runtime 属于你个人，
OMC Context 属于当前项目，
OMC State 默认属于当前 Workspace，
OMC Knowledge 根据复用范围决定存放位置。

对于你的情况，不要先启用 Global OMC Configuration。先安装一次 Plugin，然后在每一个真正需要使用 OMC 的代码库中执行：

/oh-my-claudecode:omc-setup --local

这样最安全，也最容易控制。