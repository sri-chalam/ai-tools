# AI Tools

A collection of AI skills, organized by category. Most skills are model-agnostic and work with any AI agent; exceptions are noted in the table below.

Skills are structured following the Claude Code skill format — each skill lives under `skills/` and is loaded automatically (or on demand) when invoked.

---

## Skills

### Engineering

| Skill | Description | Works with | Status |
|-------|-------------|------------|--------|
| [jira-cli-mcp](skills/engineering/jira-cli-mcp/) | Interact with Jira and export issues to markdown. Uses CLI (token-efficient) with MCP fallback. | Any AI agent | ✅ Ready |
| [junit-guidelines](skills/engineering/junit-guidelines/) | JUnit 5 test guidelines for Java: FIRST principles, GWT structure, naming, mocking. | Any AI agent | ✅ Ready |

### Misc

| Skill | Description | Works with | Status |
|-------|-------------|------------|--------|
| [youtube-transcript](skills/misc/youtube-transcript/) | Fetch a YouTube transcript and produce a formatted Word document. | Claude Desktop | ✅ Ready |

---

## Repository Structure

```
skills/
├── engineering/
│   ├── jira-cli-mcp/       # Jira CLI + MCP skill
│   └── junit-guidelines/   # JUnit 5 test guidelines
└── misc/
    └── youtube-transcript/ # YouTube transcript → Word doc skill
```

---

## How to Use a Skill

Instead of copying skill files, use a symbolic link so updates are picked up with a simple `git pull`.

**Step 1: Clone this repository**

```bash
git clone https://github.com/sri-chalam/ai-tools.git
```

**Step 2: Link the skill you want**

```bash
# Example: Jira CLI MCP skill
ln -s /path/to/ai-tools/skills/engineering/jira-cli-mcp ~/.claude/skills/jira-cli-mcp

# Example: YouTube Transcript skill
ln -s /path/to/ai-tools/skills/misc/youtube-transcript ~/.claude/skills/youtube-transcript
```

**Step 3: Restart your AI agent** (e.g., reload VS Code) to pick up the new skill.

**Updating**

```bash
cd /path/to/ai-tools
git pull
```

Then restart your AI agent. No re-linking required.

See each skill's `README.md` for setup details and usage instructions.
