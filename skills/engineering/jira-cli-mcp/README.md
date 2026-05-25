# Jira CLI MCP Skill

An AI skill to interact with Atlassian Jira and convert issues to markdown files. Supports both CLI and MCP approaches.

## Summary

This skill helps AI agents interact with Jira for common tasks such as viewing issues, listing your issues, checking in-progress work, and creating issues. While it supports many Jira use cases, **one primary use is converting Jira issues into markdown files for automated workflows**. It uses a dual approach: CLI when available (for token efficiency), and MCP as a fallback (for non-engineering teams who may not have command-line tools installed). 

**Key Features:**
- Interact with Jira: view issues, list issues, check in-progress work, create issues, and more
- **Fetches Jira issues and converts them to markdown file**
- Uses CLI first for lower token consumption, falls back to MCP if CLI is unavailable
- Works with the official Atlassian CLI for better security and long-term support
- Uses OAuth for authentication - no need to store API keys locally
- No dependency on Python, Node.js, or other runtimes

**Key Terms:**
- **CLI (Command Line Interface)** - An executable program that runs in the terminal to interact with Jira in the cloud. It needs to be installed on your computer.
- **MCP (Model Context Protocol)** - A protocol that allows AI agents to interact with external services like Jira without needing local installations. It runs as a server that the AI connects to.

---

## How to Use

Instead of copying the skill files to your Claude skills directory or Git repo, it is recommended using a symbolic link. This way, when the skill is updated, you can simply pull the latest changes without copying files again.

**Step 1: Clone the repository**

```bash
git clone https://github.com/sri-chalam/ai-tools.git
```

**Step 2: Create a symbolic link**

```bash
ln -s /path/to/ai-tools/skills/jira-cli-mcp ~/.claude/skills/jira-cli-mcp
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

## Known Limitations

### Claude Code: acli / gh CLI commands that use Keychain OAuth Not Supported

Claude Code currently has a limitation where CLI commands that rely on OAuth tokens stored in the macOS Keychain do not work. This affects both `acli` and `gh` (GitHub CLI), as both use the macOS Keychain for authentication. The same limitation applies to the VS Code Claude plugin.

This means the CLI path of this skill is not available when using Claude Code or the VS Code Claude plugin on macOS.

A GitHub issue has been filed with Anthropic: [github.com/anthropics/claude-code/issues/61895](https://github.com/anthropics/claude-code/issues/61895)

VS Code Copilot does not have this limitation — `acli` and other Keychain-backed CLIs work as expected.

---

## Why This Skill?

### The Problem with MCP Token Consumption

MCP servers tend to consume more tokens during AI interactions. For engineering teams comfortable with command-line tools, using a CLI is more efficient. However, non-engineering teams like product managers may prefer MCP over installing CLI tools on their laptops.

This skill bridges both needs by trying CLI first, then falling back to MCP if CLI is not found. If neither is available, it guides the user to install the CLI.

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

### Inspiration and Improvements

The idea for this skill comes from [aitmpl.com/component/skill/ai-research/jira](https://aitmpl.com/component/skill/ai-research/jira). The following improvements have been made:

1. **Packaged as an AI Skill** - The source is a set of instructions, not a ready-to-use and progressively loaded AI skill.

2. **Uses Official Atlassian CLI** - The source uses `jira-cli` from a personal GitHub repository. This skill uses the official Atlassian CLI, which offers better security and long-term support.

3. **OAuth Authentication** - The official CLI supports OAuth-based authentication instead of storing tokens in files like `.zshrc`.

4. **Markdown Export for Workflows** - The primary focus is creating markdown files from Jira issues for use in AI coding workflows.

5. **No Runtime Dependencies** - Works without Python, Node.js, or other runtimes.

---

## Looking Ahead

As AI skills become more common, vendors who previously released MCP servers are now creating CLI wrappers to reduce token usage. Atlassian may eventually release an official AI skill that wraps their CLI.

Even then, this skill offers the additional feature of converting Jira issues into markdown files, which remains useful for automated workflows.

---

## Example Skill Usage

> **Tip:** Using the explicit `/jira-cli-mcp` prefix is the most reliable way to invoke this skill. If you prefer free-form prompts without the prefix, include the word **"Jira"** to avoid ambiguity — issue key patterns like `PROJ-123` are not unique to Jira and the model may not load the correct skill without that context.

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
```
/jira-cli-mcp create a bug in PROJ
/jira-cli-mcp new task in PROJ
/jira-cli-mcp create story in PROJ
```

### Transition Issue
```
/jira-cli-mcp PROJ-123 Change the status to In Progress
/jira-cli-mcp PROJ-123 move to Done
/jira-cli-mcp PROJ-123 transition to In Review
Change the status Jira issue PROJ-123 to Ready for Deploy.
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
/jira-cli-mcp PROJ-123 open in a 1browser
Open Jira issue PROJ-123 in a browser.
```

---

## Credits

- Original idea: [aitmpl.com/component/skill/ai-research/jira](https://aitmpl.com/component/skill/ai-research/jira)
- [clawhub.ai/peetzweg/atlassian-cli](https://clawhub.ai/peetzweg/atlassian-cli) - Supports only CLI (not MCP) and does not support creating markdown files from issues.
