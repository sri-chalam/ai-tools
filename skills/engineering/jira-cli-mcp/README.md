# Jira AI Skill: CLI-First, MCP-Fallback, Zero Stored API Keys

An AI skill to interact with Atlassian Jira and convert issues to markdown files. Supports both CLI and MCP approaches.

## Summary

This skill helps AI agents interact with Jira for common tasks such as viewing issues, listing your issues, checking in-progress work, changing issue status, and assigning or unassigning issues. While it supports many Jira use cases, **one primary use is converting Jira issues into markdown files for automated workflows**. It uses a dual approach: CLI when available (for token efficiency), and MCP as a fallback (for non-engineering teams who may not have command-line tools installed). 

**Key Terms:**
- **CLI (Command Line Interface)** - An executable program that runs in the terminal, such as `git`, `gh`, or `acli`. It needs to be installed on your computer.
- **MCP (Model Context Protocol)** - A protocol that allows AI agents to interact with external services like Jira without needing local installations. It runs as a server that the AI connects to.

**Key Features:**
- Interact with Jira: view issues, list issues, check in-progress work, create issues, and more
- **Fetches Jira issues and converts them to a markdown file**
- Uses CLI first for lower token consumption, falls back to MCP if CLI is unavailable
- Works with the official Atlassian CLI for better security and long-term support
- Uses OAuth for authentication - no need to store API keys locally
- CLI path requires `acli`, `jq`, and `python3`; MCP path requires no local tools — see [Requirements](#requirements)

---

## Why This Skill?

### Background: MCP Servers and Token Consumption

In 2025, MCP (Model Context Protocol) servers gained quite a following in the AI community. They offer a convenient way for AI agents to interact with external services without requiring any local software to be installed.

Toward the end of 2025, however, concerns began to be raised about the number of tokens consumed by MCP servers. It was noticed that even when MCP tools are not actively being used in a session, the descriptions of all tools provided by each configured MCP server are pre-loaded by the model. Tokens are spent just by having an MCP server connected.

Some observed token costs for popular MCP servers:
- **Official GitHub MCP server**: approximately 17,700 tokens for 94 tools (some sources report up to ~55,000 tokens)
- **Atlassian Rovo MCP server**: approximately 10,000 tokens

As awareness of these numbers grew, CLI-based AI skills started to be recommended as a more token-efficient alternative.

### MCP vs CLI: Pros and Cons

Both approaches have their place, depending on the context — who is using the tool, and how heavily AI models are being used.

- **CLI-based AI skills** require the relevant command-line tools to be installed on the user's machine.
  - For engineering teams, this is generally not a barrier — developers are accustomed to tools like `git`, `gh`, or `acli`.
  - For non-engineering teams (such as product managers or business analysts), installing and maintaining CLI tools for each service can be a significant hurdle.
- **MCP servers** do not require any additional software to be installed locally, which is a meaningful advantage for non-engineering teams.
  - If AI models are not being used heavily — for example, if a team uses AI agents infrequently — token consumption may not be a pressing concern. In that case, using MCP is perfectly fine, even if it consumes more tokens per session.

A clear example is **GitHub**: it is primarily used by developers, who almost always have `git` installed. Using a GitHub MCP server that consumes 17,700 tokens is likely not the best choice for developers — a CLI-based approach would be more efficient. For GitHub, the choice is fairly straightforward: CLI is preferred.

**Atlassian Jira** is a different case. It is used by both engineering and non-engineering teams. For non-engineering teams, installing CLI tools along with dependencies like `jq` or Python can be difficult. This makes MCP more appealing for those users, even with its token cost.

### The Smart Zone and the Dumb Zone

Dex Horthy — an active AI Engineer conference speaker known for coining the term **Context Engineering** — also introduced the **smart zone** and **dumb zone** concept to describe how context size affects the model's ability to focus and retrieve relevant information.

Frontier AI models may advertise context windows of up to 1 million tokens. However, not all of that space performs equally well. When the context is kept compact, models tend to respond more accurately — this region is called the **smart zone**. As the context grows too large, the model's attention can become diluted and response quality may drop — this is the **dumb zone**.

It is generally suggested to stay within **120,000 tokens** to remain in the smart zone with current frontier models.

If both GitHub MCP and Atlassian MCP servers are configured, they together consume approximately **28,000 tokens** — about **25% of the smart zone budget** spent on just two MCP servers, before any actual task context, conversation history, or code is added.

For Atlassian Jira specifically, having an AI skill that supports both CLI (for engineering teams) and MCP as a fallback (for non-engineering teams) helps keep this token footprint in check while still being accessible to everyone.

### Built for AI Coding Workflows

When using AI coding agents to implement Jira issues, a common first step is to fetch the issue details and understand the requirements. This skill creates a markdown file from the Jira issue, which serves as the starting point for automated workflows.

> **Note:** Using prompts to interact with Jira for routine tasks (browsing, transitions, comments) can be slower than using the Jira web UI directly. The strongest use case for this skill is exporting issues to markdown — AI models work best when requirements are available in a local markdown file rather than fetched live during a workflow.

### Why Atlassian CLI (acli) Instead of jira-cli?

There is a popular third-party CLI called `jira-cli` that many developers use. However, this skill uses the official Atlassian CLI (`acli`) for the following reasons:

1. **OAuth-Based Security** - The Atlassian CLI supports OAuth-based authentication, so there is no need to save API keys or tokens on your local machine (such as in `.zshrc` or config files). This is more secure.

2. **Long-Term Support** - Since `acli` is the official CLI from Atlassian, it will receive long-term support and updates. The third-party `jira-cli` is maintained by a single person, which may not be sustainable over time.

---

## Why Not Use Existing Solutions?

This skill was created because existing options did not fully meet the needs of AI coding workflows.

### MCP Compression (mcp-compressor)

Atlassian's [mcp-compressor](https://github.com/atlassian-labs/mcp-compressor) is a proxy that wraps an MCP server and reduces token usage by 70–97% by fetching tool schemas on demand rather than pre-loading all of them. However, it requires `uvx` (a Python package runner) to be installed — which is still an additional dependency, particularly for non-engineering teams.

### Jira AI Skill (aitmpl.com)

[aitmpl.com](https://aitmpl.com/component/skill/ai-research/jira) offers a Jira AI skill, but it has several limitations that make it insufficient for AI coding workflows:

1. **Not a packaged skill** - It is a raw set of instructions, not a ready-to-use, progressively loaded AI skill.

2. **Uses an unofficial CLI** - It relies on `jira-cli` from a personal GitHub repository, which raises concerns about long-term support and security.

3. **No OAuth support** - Authentication depends on storing tokens in files like `.zshrc`, rather than using OAuth.

4. **No markdown export** - It does not support exporting Jira issues to markdown files for use in AI coding workflows.

5. **Heavy dependencies** - It does not distinguish between CLI and MCP paths, offering no lightweight fallback for non-engineering teams.

---

## Requirements

> **Note:** The installation commands in this document are written for **macOS**. Commands for other platforms may differ.

### CLI Path

To use the CLI path, the following tools need to be installed:

- **`acli`** (Atlassian CLI) — the primary tool for interacting with Jira from the terminal.
  - Install on macOS: `brew install atlassian/acli/acli`
  - [Other platforms](https://developer.atlassian.com/cloud/acli/guides/how-to-get-started/)
  - After installing, authenticate with: `acli jira auth login`
- **`jq`** — used for JSON field extraction from Jira API responses.
  - Install on macOS: `brew install jq`
- **`python3`** — used to parse Atlassian Document Format (ADF), the structure Jira uses for descriptions and comments.
  - Typically pre-installed on macOS.

### MCP Path

To use the MCP fallback path, the **Atlassian Rovo MCP server** needs to be configured in your AI agent. No additional software needs to be installed on your machine.

See [Adding the Atlassian MCP Server](#adding-the-atlassian-mcp-server) for setup instructions.

---

## Known Limitations

### Claude Code: acli / gh CLI Commands That Use Keychain OAuth Not Supported

Claude Code currently has a limitation where CLI commands that rely on OAuth tokens stored in the macOS Keychain do not work. This affects both `acli` and `gh` (GitHub CLI), as both use the macOS Keychain for authentication. The same limitation applies to the VS Code Claude plugin.

This means the CLI path of this skill is not available when using Claude Code or the VS Code Claude plugin on macOS.

A GitHub issue has been filed with Anthropic: [anthropics/claude-code #61895](https://github.com/anthropics/claude-code/issues/61895)

VS Code Copilot does not have this limitation — `acli` and other Keychain-backed CLIs work as expected.

---

## How to Use

AI skills evolve continuously. Copying files into your skills directory freezes them at a point in time. Instead, clone the skill Git repository and create a symbolic link to it — the skill stays current whenever you pull the latest changes.

**Step 1: Clone the repository**

```bash
git clone https://github.com/sri-chalam/ai-tools.git
```

**Step 2: Create a symbolic link**

```bash
ln -s /path/to/ai-tools/skills/engineering/jira-cli-mcp ~/.claude/skills/jira-cli-mcp
```

Replace `/path/to/ai-tools` with the actual path where you cloned the repository.

**Updating the skill**

When there are updates to the skill, simply pull the latest changes:

```bash
cd /path/to/ai-tools
git pull
```

Then restart your AI agent (e.g., restart VS Code) to use the updated skill.

---

## Adding the Atlassian MCP Server

The MCP fallback path of this skill requires the Atlassian Rovo MCP server to be configured in the AI agent. Below are setup instructions for Claude Code and VS Code Copilot.

### Claude Code

Add the Atlassian MCP server at the user level so it is available across all projects:

```bash
claude mcp add --transport sse --scope user atlassian https://mcp.atlassian.com/v1/sse
```

> **Important:** The Atlassian MCP server can also be added through Claude Desktop via **Customize → Connectors → search for "Atlassian Rovo MCP"**. However, MCP servers added through Claude Desktop are only available within the Claude Desktop app — they are **not** shared with Claude Code or the VS Code Claude extension. Use the `claude mcp add` command above for Claude Code.

### VS Code Copilot

Add the Atlassian MCP server to your VS Code user settings (`~/Library/Application Support/Code/User/settings.json` on macOS) so it is available across all workspaces:

```json
"mcp": {
  "servers": {
    "atlassian": {
      "type": "sse",
      "url": "https://mcp.atlassian.com/v1/sse"
    }
  }
}
```

Alternatively, add it to a `.vscode/mcp.json` file in a specific project to limit the scope to that workspace.

After saving, restart VS Code or reload the window (`Cmd+Shift+P` → **Developer: Reload Window**) for the MCP server to take effect.

> **Important:** The SSE endpoint `https://mcp.atlassian.com/v1/sse` (transport: `sse`) is deprecated by Atlassian and will stop working after **June 30, 2026**. The replacement endpoint is `https://mcp.atlassian.com/v1/mcp` (transport: `http`). However, organizations must approve the new URL before it can be used. Check with your Atlassian admin if the new endpoint is available.

---

## Pre-Approving acli Commands

### VS Code Copilot

Approving every `acli jira workitem` command invocation in VS Code Copilot is tedious. Auto-approval can be configured by adding the following entry to your VS Code `settings.json` (`~/Library/Application Support/Code/User/settings.json` on macOS):

```json
"chat.tools.terminal.autoApprove": {
  // Auto-approve all acli jira workitem commands
  "/^acli jira workitem\\b/": {
    "matchCommandLine": true,
    "approve": true
  }
}
```

This uses a regex pattern to match any terminal command starting with `acli jira workitem` and approves it automatically, skipping the confirmation prompt for every Jira operation.

---

## Example Skill Usage

> **Tip:** Using the explicit `/jira-cli-mcp` prefix is the recommended way to invoke this skill. If you prefer free-form prompts without the prefix, include the word **"Jira"** to avoid ambiguity — issue key patterns like `PROJ-123` are not unique to Jira and the model may not load the correct skill without that context.

### Export Issue to Markdown
```
/jira-cli-mcp PROJ-123 export to markdown
Export Jira PROJ-123 to markdown
```

### View Issue
```
/jira-cli-mcp PROJ-123
/jira-cli-mcp PROJ-123 show details
/jira-cli-mcp PROJ-123 fetch issue
/jira-cli-mcp PROJ-123 show description
Jira PROJ-123 show details
Jira PROJ-123 fetch issue
Jira PROJ-123 show description of the issue
```

### List My Issues
```
/jira-cli-mcp list my issues
/jira-cli-mcp what are my tickets
/jira-cli-mcp my open issues
List my Jira issues
List my open Jira issues
```

### My In-Progress Issues
```
/jira-cli-mcp in progress
/jira-cli-mcp what am I working on
/jira-cli-mcp my in-progress tasks
List my in progress Jira issues
```

### Active Sprint Issues
```
/jira-cli-mcp current sprint
/jira-cli-mcp show sprint items
List Jira issues of current sprint
```

### Create Issue

> **Note:** Every organization customizes Jira issue creation to suit their needs — required fields, epics, sprints, and other details vary significantly. It may not be advisable to create Jira issues through this generic skill. In a workflow that implements code from requirements, there may be a step to create multiple Jira issues from those requirements; however, issue creation should have its own organization-specific skill, customized to match the project's field requirements and workflows.

```
# Full natural-language example
create a new Jira issue under project PROJ, with summary: "my summary", description: "issue description", acceptance criteria: "list of criteria", take epic link from issue PROJ-123, status: Submitted, type: bug

# Or using the skill prefix:
/jira-cli-mcp create a bug in PROJ
/jira-cli-mcp new task in PROJ
/jira-cli-mcp create story in PROJ

```

### Transition Issue
```
/jira-cli-mcp PROJ-123 Change the status to In Progress
/jira-cli-mcp PROJ-123 move to Done
/jira-cli-mcp PROJ-123 transition to In Review
Change the status of Jira issue PROJ-123 to Ready for Deploy.
```

### Assign to Me
```
/jira-cli-mcp PROJ-123 assign issue to me
/jira-cli-mcp PROJ-123 assign to me
/jira-cli-mcp PROJ-123 take this issue
Assign the Jira issue PROJ-123 to me
```

### Unassign
```
/jira-cli-mcp PROJ-123 unassign issue
/jira-cli-mcp PROJ-123 unassign
/jira-cli-mcp PROJ-123 remove assignee
Unassign the Jira issue PROJ-123 from me
```

### Add Comment
```
/jira-cli-mcp PROJ-123 add a comment with the text "Test comment added using AI Skill".
Add a comment to the Jira issue PROJ-123 with the text "Test comment added using AI Skill". 
```

### Open in Browser
```
/jira-cli-mcp PROJ-123 open in a browser 
Open Jira issue PROJ-123 in a browser
```
---

## Continued Relevance

As AI skills become more common, vendors who previously released MCP servers are now creating CLI wrappers to reduce token usage. Atlassian may eventually release an official AI skill that wraps their CLI.

Even then, this skill offers the additional feature of converting Jira issues into markdown files, which remains useful for automated workflows.

---

## Credits

- Original idea: [aitmpl.com/component/skill/ai-research/jira](https://aitmpl.com/component/skill/ai-research/jira)
- [clawhub.ai/peetzweg/atlassian-cli](https://clawhub.ai/peetzweg/atlassian-cli) - Supports only CLI (not MCP) and does not support creating markdown files from issues.

---

## References

- [Improving Token Efficiency in GitHub Agentic Workflows](https://github.blog/ai-and-ml/github-copilot/improving-token-efficiency-in-github-agentic-workflows/) - GitHub's own engineering team confirms 10–15 KB of per-turn schema overhead per 40 tools, and recommends CLI (`gh`) over MCP for data-fetching.

- [MCP Compression: Preventing Tool Bloat in AI Agents](https://www.atlassian.com/blog/developer/mcp-compression-preventing-tool-bloat-in-ai-agents) - Atlassian Developer Blog
