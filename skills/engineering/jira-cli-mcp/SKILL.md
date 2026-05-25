---
name: jira-cli-mcp
description: Interact with Jira — view, list, create, transition, assign, comment, and export issues to markdown. Uses acli (Atlassian CLI) when available, falls back to Atlassian Rovo MCP.
argument-hint: "[ISSUE-KEY | natural language command]"
disable-model-invocation: false
---

# Jira Skill

The user has invoked this skill with: $ARGUMENTS

---

## Step 1: Detect Environment

Run the following to check if the Atlassian CLI is available:

```bash
which acli
```

- If `acli` is found → use the **CLI path** for all operations.
- If `acli` is not found → check whether the Rovo MCP server is connected.
  - If Rovo MCP is available → use the **MCP path** for all operations.
  - If Rovo MCP is also unavailable → stop and guide the user to install `acli`:

> Neither the Atlassian CLI (`acli`) nor the Rovo MCP server was found.
> To use this skill, install `acli`:
> - **Mac:** `brew install atlassian/acli/acli`
> - **Other platforms:** https://developer.atlassian.com/cloud/acli/guides/how-to-get-started/
>
> After installing, run `acli jira auth login` to authenticate, then retry.

---

## Step 2: Determine Intent

Parse `$ARGUMENTS` using natural language to identify what the user wants:

| Intent | Example inputs |
|---|---|
| View issue | `PROJ-123`, `show PROJ-123`, `fetch PROJ-123`, `PROJ-123 show details` |
| Export issue to markdown | `export PROJ-123`, `PROJ-123 export to markdown` |
| List my issues | `list my issues`, `what are my tickets`, `my open issues` |
| My in-progress issues | `in progress`, `what am I working on`, `my in-progress tasks` |
| Active sprint issues | `sprint`, `current sprint`, `show sprint items` |
| Create issue | `create a bug in PROJ`, `new task`, `create story` |
| Transition issue | `move PROJ-123 to Done`, `transition PROJ-123 to In Review` |
| Assign to me | `assign PROJ-123 to me`, `take PROJ-123` |
| Unassign | `unassign PROJ-123`, `remove assignee from PROJ-123` |
| Add comment | `comment on PROJ-123`, `add comment to PROJ-123` |
| Open in browser | `open PROJ-123`, `open PROJ-123 in browser` |

If intent is ambiguous or the argument is empty, ask the user what they would like to do before proceeding.

---

## CLI Path (acli available)

> **Important:** `acli` uses its own built-in interactive pager that **ignores** the `PAGER` environment variable. Setting `PAGER=cat` does not reliably suppress it. For any command that returns JSON output, redirect stdout to a temp file to bypass the pager:
> ```bash
> acli jira workitem view PROJ-123 --fields "*all" --json > /tmp/PROJ-123.json
> ```

> **JSON Parsing:** Use `jq` to extract simple fields (key, summary, status, etc.) — it is a single binary with no runtime dependency (`brew install jq`). For Jira's **Atlassian Document Format (ADF)** fields (description, comments), use Python as a fallback since ADF is deeply nested JSON that `jq` handles less cleanly. If neither tool is available, read the JSON file directly.

### View Issue

When the user provides an issue key or asks to view/show/fetch an issue:

**1. Fetch the issue:**
```bash
acli jira workitem view PROJ-123 --fields "*all" --json > /tmp/PROJ-123.json
```

**2. Parse the issue JSON:**

Use `jq` if available (preferred — lightweight, no runtime dependency):
```bash
jq -r '
  .key as $key | .fields |
  ($key + ": " + .summary),
  ("Type: " + .issuetype.name + " | Status: " + .status.name + " | Priority: " + (.priority.name // "")),
  ("Assignee: " + ((.assignee // {displayName:"Unassigned"}).displayName) + " | Reporter: " + .reporter.displayName),
  ("Sprint: " + ((((.customfield_10020 // []) | map(select(.state == "active")) | .[0]) // ((.customfield_10020 // []) | last) // {}) | .name // "")),
  ("Story Points: " + ((.customfield_10016 // .customfield_10028 // "") | tostring)),
  ("Labels: " + (.labels | join(", ")))
' /tmp/PROJ-123.json
```

For **description and comments** (Atlassian Document Format — deeply nested JSON), use Python:
```bash
python3 << 'EOF'
import json

def extract_text(node):
    if isinstance(node, dict):
        if node.get('type') == 'hardBreak':
            return '\n'
        return node.get('text', '') + ''.join(extract_text(c) for c in node.get('content', []))
    if isinstance(node, list):
        return ''.join(extract_text(n) for n in node)
    return ''

with open('/tmp/PROJ-123.json') as f:
    data = json.load(f)

fields = data['fields']
print('Description:')
print(extract_text(fields.get('description') or {}))

comments = (fields.get('comment') or {}).get('comments', [])
if comments:
    print('\nComments:')
    for c in comments:
        author = (c.get('author') or {}).get('displayName', '')
        print(f"  {author} — {c.get('created','')[:10]}")
        print(f"  {extract_text(c.get('body') or {})}")
EOF
```

**3. Display the issue details inline** in the chat using this structure:

```
PROJ-123: <Summary>

Type: <issue type> | Status: <status> | Priority: <priority>
Assignee: <assignee> | Reporter: <reporter>
Sprint: <sprint name> | Story Points: <story points>
Labels: <labels, comma-separated>

Description:
<description content>

Comments:
  <Author Name> — <date>
  <comment body>
```

If a field has no value, omit it. If there are no comments, omit that section.

---

### Export Issue to Markdown

When the user explicitly asks to export an issue to a markdown file:

**1. Ask the user for output paths:**
- "Where should I save the markdown file? (default: `./docs/requirements/PROJ-123-orig-requirements.md`)"
- "Where should I download attachments? (default: `./docs/requirements/screenshots`)"

Accept the user's input or proceed with defaults.

**2. Fetch the issue:**
```bash
acli jira workitem view PROJ-123 --fields "*all" --json > /tmp/PROJ-123.json
```

Parse fields using the same `jq` + Python approach described in [View Issue](#view-issue) above. Use the extracted values when generating the markdown in step 5.

**3. List attachments:**
```bash
acli jira workitem attachment-list --key PROJ-123 --json > /tmp/PROJ-123-attachments.json && cat /tmp/PROJ-123-attachments.json
```

**4. Download attachments:**

Create the attachments directory if it does not exist, then download each attachment using the download URLs from the attachment list. Use curl with Atlassian authentication headers obtained from `acli`:

```bash
mkdir -p ./screenshots
# For each attachment URL from the attachment-list JSON output:
curl -L -o "./screenshots/<filename>" "<attachment-download-url>"
```

If any attachment fails to download, note the failure and continue with the rest.

**5. Generate the markdown file and save it to the path chosen by the user in Step 1.** Use this structure:

```markdown
# PROJ-123: <Summary>

| Field | Value |
|---|---|
| **Type** | <issue type> |
| **Status** | <status> |
| **Priority** | <priority> |
| **Assignee** | <assignee display name> |
| **Reporter** | <reporter display name> |
| **Labels** | <labels, comma-separated> |
| **Sprint** | <sprint name> |
| **Story Points** | <story points> |

## Description

<description content>

## Attachments

- [filename.png](./screenshots/filename.png)

## Comments

### <Author Name> — <date>

<comment body>

---
```

If a field has no value, omit it from the table. If there are no attachments or comments, omit those sections.

---

### List My Issues

```bash
acli jira workitem search --jql "assignee = currentUser() AND statusCategory != Done ORDER BY updated DESC" --json > /tmp/my-issues.json && cat /tmp/my-issues.json
```

Display results as a formatted table in the response.

---

### My In-Progress Issues

```bash
acli jira workitem search --jql "assignee = currentUser() AND status = 'In Progress' ORDER BY updated DESC" --json > /tmp/my-inprogress.json && cat /tmp/my-inprogress.json
```

---

### Active Sprint Issues

```bash
acli jira workitem search --jql "sprint in openSprints() AND assignee = currentUser() ORDER BY updated DESC" --json > /tmp/sprint-issues.json && cat /tmp/sprint-issues.json
```

---

### Create Issue

**Gather from the user** (if not already in `$ARGUMENTS`):
- Project key
- Issue type (Bug, Story, Task, etc.)
- Summary
- Description (optional)
- Priority (optional)

**Show confirmation before executing:**

> **Creating issue:**
> - Project: PROJ
> - Type: Bug
> - Summary: "Login page crashes on submit"
>
> Proceed? (yes/no)

**On confirmation:**
```bash
acli jira workitem create --project "PROJ" --type "Bug" --summary "Login page crashes on submit"
```

---

### Transition Issue

**Show confirmation before executing:**

> **Transitioning** PROJ-123 → "In Review"
>
> Proceed? (yes/no)

**On confirmation:**
```bash
acli jira workitem transition --key "PROJ-123" --status "In Review" --yes
```

**On failure:** Report the exact error message from the command output and stop. Do not fetch the issue, check available transitions, or run any additional commands.

---

### Assign to Me

**Show confirmation before executing:**

> **Assigning** PROJ-123 to you.
>
> Proceed? (yes/no)

**On confirmation:**
```bash
acli jira workitem assign --key "PROJ-123" --assignee "@me" --yes
```

---

### Unassign

**Show confirmation before executing:**

> **Removing assignee** from PROJ-123.
>
> Proceed? (yes/no)

**On confirmation:**
```bash
acli jira workitem assign --key "PROJ-123" --remove-assignee --yes
```

---

### Add Comment

If the comment body is not provided in `$ARGUMENTS`, ask the user for it.

**Show confirmation before executing:**

> **Adding comment** to PROJ-123:
> "<comment text>"
>
> Proceed? (yes/no)

**On confirmation:**
```bash
acli jira workitem comment create --key "PROJ-123" --body "comment text here"
```

---

### Open in Browser

```bash
acli jira workitem view PROJ-123 --web
```

> Note: `--web` opens a browser and does not produce terminal output, so `PAGER=cat` is not needed here.

---

## MCP Path (Rovo MCP fallback)

Used when `acli` is not installed.

### Site Detection

Call `mcp__claude_ai_Atlassian_Rovo__getAccessibleAtlassianResources` to get the list of connected Atlassian sites.

- If one site → use it automatically.
- If multiple sites → ask the user which site to use before proceeding.

---

### View Issue

When the user provides an issue key or asks to view/show/fetch an issue:

**Fetch the issue:**
```
mcp__claude_ai_Atlassian_Rovo__getJiraIssue(issueKey: "PROJ-123")
```

**Display the issue details inline** in the chat using the same structure as the CLI path.

---

### Export Issue to Markdown

When the user explicitly asks to export an issue to a markdown file:

**1. Ask the user for the output path:**
- "Where should I save the markdown file? (default: `./docs/requirements/PROJ-123-orig-requirements.md`)"

Accept the user's input or proceed with the default.

**2. Fetch the issue:**
```
mcp__claude_ai_Atlassian_Rovo__getJiraIssue(issueKey: "PROJ-123")
```

**3. Generate the markdown file and save it to the path chosen by the user in Step 1.** Use the same structure as the CLI path.

**For the Attachments section**, list names only — download is not available via MCP:

```markdown
## Attachments

> Attachment download requires the Atlassian CLI (acli). Install it for full support.

- filename.png
- diagram.pdf
```

---

### List My Issues

```
mcp__claude_ai_Atlassian_Rovo__searchJiraIssuesUsingJql(
  jql: "assignee = currentUser() AND statusCategory != Done ORDER BY updated DESC"
)
```

---

### My In-Progress Issues

```
mcp__claude_ai_Atlassian_Rovo__searchJiraIssuesUsingJql(
  jql: "assignee = currentUser() AND status = 'In Progress' ORDER BY updated DESC"
)
```

---

### Active Sprint Issues

```
mcp__claude_ai_Atlassian_Rovo__searchJiraIssuesUsingJql(
  jql: "sprint in openSprints() AND assignee = currentUser() ORDER BY updated DESC"
)
```

---

### Create Issue

Gather the same fields as the CLI path. Show the same confirmation.

**On confirmation:**
```
mcp__claude_ai_Atlassian_Rovo__createJiraIssue(
  projectKey: "PROJ",
  issueType: "Bug",
  summary: "Login page crashes on submit"
)
```

---

### Transition Issue

**1. Fetch available transitions:**
```
mcp__claude_ai_Atlassian_Rovo__getTransitionsForJiraIssue(issueKey: "PROJ-123")
```

**2.** Match the user's requested status to a transition ID from the result.

**3.** Show confirmation, then on confirmation:
```
mcp__claude_ai_Atlassian_Rovo__transitionJiraIssue(
  issueKey: "PROJ-123",
  transitionId: "<matched-transition-id>"
)
```

---

### Assign to Me

**1.** Get the current user's account ID:
```
mcp__claude_ai_Atlassian_Rovo__atlassianUserInfo()
```

**2.** Show confirmation, then on confirmation:
```
mcp__claude_ai_Atlassian_Rovo__editJiraIssue(
  issueKey: "PROJ-123",
  fields: { assignee: { accountId: "<current-user-account-id>" } }
)
```

---

### Unassign

Show confirmation, then on confirmation:
```
mcp__claude_ai_Atlassian_Rovo__editJiraIssue(
  issueKey: "PROJ-123",
  fields: { assignee: null }
)
```

---

### Add Comment

If the comment body is not provided, ask the user for it. Show the same confirmation as the CLI path.

**On confirmation:**
```
mcp__claude_ai_Atlassian_Rovo__addCommentToJiraIssue(
  issueKey: "PROJ-123",
  body: "comment text here"
)
```

---

### Open in Browser

Provide the issue URL to the user:
```
https://<site-url>/browse/PROJ-123
```

Retrieve `<site-url>` from the site detection step.

