Since you are using the default Local Agent in VS Code, my earlier explanation about the separate Claude Agent integration does not apply.

The likely issue is a VS Code Copilot Chat Local Agent bug or configuration conflict affecting Claude models, not a problem with your prompt and not a general Claude outage.

Why the same Claude model can behave differently

In VS Code Local Agent mode, the model does not receive your message directly. The VS Code coding harness builds a larger request containing:

* your prompt
* system instructions
* workspace information
* conversation history
* enabled tools and their JSON schemas
* custom instructions
* prior tool results

It then runs a tool-calling loop around the selected model.  

The IntelliJ Copilot plugin uses a different client integration. Therefore, selecting the same Claude model in IntelliJ and VS Code does not guarantee that the model receives an identical request.

The fact that:

* Claude works in IntelliJ
* GPT-5.4 works in VS Code Local Agent
* Claude fails only in VS Code Local Agent

strongly suggests that VS Code is sending Claude a malformed, incomplete, or confusing request. A reply such as “What can I do for you?” often indicates that the effective user prompt was lost, truncated, or overshadowed by another instruction.

Claude Sonnet and Opus models are officially supported in GitHub Copilot, including Agent mode, so this is not expected behavior.  

Most likely causes

1. A stale or corrupted chat session

A previous agent session may contain a malformed tool result, an outdated system prompt, or excessive context. GPT-5.4 may recover from it while Claude does not.

Start a completely new chat instead of continuing the existing session.

2. Too many MCP or extension-provided tools

VS Code Local Agent can expose built-in tools, MCP tools, and tools contributed by extensions. The enabled set can change per request. VS Code recommends selecting only the tools relevant to the task.  

Claude may be more sensitive than GPT-5.4 to a large or malformed tool schema.

3. A custom instruction or skill conflict

A workspace instruction file, custom agent, prompt file, or skill may contain instructions that interfere with the Local Agent prompt. Since you previously added AI-state rules, handoff skills, and compaction rules for Copilot, this is especially worth checking.

4. A VS Code or Copilot Chat extension regression

VS Code Local Agent depends heavily on the coding harness. A model can work normally in another IDE while failing in one version of the VS Code extension. GitHub recommends keeping the Copilot extensions current because older clients may not communicate correctly with Copilot services.  

There have previously been VS Code Copilot issues where Claude entered loops or returned repeated generic behavior in Agent mode, although I did not find a current report matching your exact greeting response.  

Run this isolation test

Use a fresh chat for every row.

Test	Agent	Model	Workspace	Tools
A	Ask	Claude Sonnet 4.6	Current project	Default
B	Agent	Claude Sonnet 4.6	Current project	Default
C	Agent	Claude Sonnet 4.6	Empty VS Code window	Disable MCP and extension tools
D	Agent	GPT-5.4	Current project	Default

Send the same simple prompt:

Reply with exactly: TEST

Then try:

List the files in the current workspace. Do not modify anything.

Interpret the results:

Result	Likely cause
Ask works, but Agent fails	Tool exposure or Local Agent harness issue
Empty window works, but your project fails	Workspace instructions, skills, MCP tools, or project context
Claude fails everywhere in VS Code, but GPT-5.4 works	VS Code Claude-model adapter issue
Claude fails intermittently	Extension regression, rate limiting, or a stale session

Recommended fix sequence

1. Update VS Code and both GitHub Copilot and GitHub Copilot Chat extensions.
2. Run Developer: Reload Window.
3. Create a new Local Agent chat.
4. Open the Configure Tools picker and temporarily disable MCP servers and extension tools.
5. Test Claude in an empty VS Code window.
6. Temporarily rename workspace customization folders such as:

.github/instructions
.github/agents
.github/prompts
.github/skills

Also temporarily disable any user-level Copilot instruction files.

7. Add tools and instruction files back in small groups until the issue reappears.

Capture logs when Claude still fails

In VS Code:

1. Open the Command Palette.
2. Run Developer: Set Log Level.
3. Select GitHub Copilot Chat.
4. Set the level to Trace.
5. Reproduce the issue.
6. Open View → Output and select GitHub Copilot.
7. Run Developer: Chat Diagnostics.

GitHub documents these steps for collecting VS Code Copilot logs and connectivity diagnostics.  

Search the logs for:

claude
tool
schema
invalid_request
messages
context
truncated
rate_limit

Practical workaround

Continue using GPT-5.4 in VS Code Local Agent for repository-level coding. It is officially supported and appears to work correctly in your environment.  

For Claude, use IntelliJ temporarily or use Ask mode in VS Code when your isolation test shows that only Agent mode is affected. You can also select Auto in VS Code; its task-optimized routing considers model health and availability when choosing a model.  