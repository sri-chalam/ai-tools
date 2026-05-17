# YouTube Transcript Skill

An AI skill that downloads a YouTube video transcript and formats it into a professional, visually rich Word document. Works exclusively with **Claude Desktop**.

## Summary

This skill fetches the transcript from a YouTube video and produces a polished `.docx` file structured for easy reading — with a summary box, callout quotes, sectioned transcript, and consistent visual formatting.

**Key Features:**
- Fetches transcripts via `youtube-transcript-api` with automatic retries and user-agent rotation to handle 403 errors
- Falls back to headless Chrome via Selenium if the API is blocked
- Produces a professionally formatted Word document with green headings, callout boxes, and a "Summary at a Glance" section
- Strips timestamps from transcript body text
- Sections long transcript blocks by topic using Heading 3 labels
- Pulls standout quotes into shaded callout boxes
- Bolds key sentences (1–2 per section) for fast skimming
- Uses tables where appropriate for comparisons, steps, or definitions
- Includes a "Page X of Y" footer in brown color

**Key Terms:**
- **youtube-transcript-api** - Python library that fetches transcripts directly from YouTube's API without browser automation.
- **Selenium fallback** - Browser-based extraction using headless Chrome, used when the API is blocked or returns a 403.

---

## How to Use

Instead of copying the skill files to your Claude skills directory or Git repo, it is recommended to use a symbolic link. This way, when the skill is updated, you can simply pull the latest changes without copying files again.

**Step 1: Clone the repository**

```bash
git clone https://github.com/sri-chalam/ai-tools.git
```

**Step 2: Create a symbolic link**

```bash
ln -s /path/to/ai-tools/skills/misc/youtube-transcript ~/.claude/skills/youtube-transcript
```

Replace `/path/to/ai-tools` with the actual path where you cloned the repository.

**Step 3: Install dependencies**

```bash
pip install youtube-transcript-api python-docx

# Optional: browser fallback
pip install selenium webdriver-manager requests
```

A local Chrome installation is required for the Selenium fallback.

**Updating the skill**

When there are updates to the skill, simply pull the latest changes:

```bash
cd /path/to/ai-tools
git pull
```

Then restart Claude Desktop to use the updated skill.

**Invoking the skill**

```
/youtube-transcript https://www.youtube.com/watch?v=VIDEO_ID
```

Use the full YouTube URL. The shortened `youtu.be` link from the Share button is not supported.

---

## Why This Skill?

### Turning Video into a Scannable Reference

Watching a video to extract key information is time-consuming. This skill converts a transcript into a structured document you can skim in minutes — with a summary box at the top, callout quotes for memorable lines, and bolded key sentences throughout.

### Robust Against YouTube Blocking

YouTube occasionally blocks transcript API requests with 403 errors. The skill handles this with automatic retries, rotating user-agents, and a Selenium/Chrome fallback — so transient blocks do not silently produce empty documents.

### Strict Transcript Gate

The skill never substitutes web search results or external content for the actual transcript. If transcript extraction fails after all fallbacks, it stops and reports the failure clearly rather than generating a document from unrelated sources.

---

## Output Format

The generated `.docx` file follows this naming convention:

```
Title-Words-Here-YT-Transcript.docx
```

Document structure:
1. Title and video metadata
2. "Summary at a Glance" box (3–5 bullets, shaded background)
3. Organized body sections with Heading 2/3 labels
4. Verbatim transcript broken into topic-shifted subsections
5. Callout boxes for standout quotes
6. Tables for structured content (steps, comparisons, definitions)
7. Page X of Y footer
