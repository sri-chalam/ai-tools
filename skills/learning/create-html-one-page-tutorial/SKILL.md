---
name: create-html-one-page-tutorial
description: >
  Researches a topic across web articles, X.com posts, Medium articles, and YouTube videos, then
  builds (or progressively deepens) a single self-contained HTML tutorial page with chapter quizzes.
  Invoke manually with /create-html-one-page-tutorial <topic>.
argument-hint: "[topic]"
# When true, this skill must be invoked manually with /create-html-one-page-tutorial (Claude won't auto-trigger it)
disable-model-invocation: true
version: "0.1.0"
tags:
  - learning
  - tutorial
  - html
maintainers:
  - name: Sri Chalam
category:
  - learning
license: UNLICENSED
---

# One-Page HTML Tutorial Generator

Build a single-page HTML tutorial on: $ARGUMENTS

## Step 1 — Research

Search and read from a spread of sources before writing a word of content:
- General web articles (search engines, official docs, reputable blogs)
- X.com / Twitter posts and threads
- Medium articles
- YouTube videos — pull the transcript/description when available rather than guessing at content from the title alone

Cross-check claims across at least 2–3 sources before stating them as fact. Prefer recent, authoritative sources; note where sources disagree instead of silently picking one.

Research is done when you have enough material to write every planned chapter without guessing.

## Step 2 — Find or create the output file

Filename: `tutorials/<topic-slug>.html` (kebab-case slug of the topic), relative to the current working directory.

- **File doesn't exist** → Initial mode (Step 3).
- **File exists** → Elaboration mode (Step 4): read the existing HTML fully before touching it.

## Step 3 — Initial mode: introduce concisely

Break the topic into 4–8 logical chapters. For each chapter:
- A short, simple-English explanation (2–4 short paragraphs) — no unexplained jargon, plain sentences, define any term the first time it's used
- Mark the intro explanation with `data-level="intro"` so later sessions know what they're deepening
- End the chapter with a 3–5 question multiple-choice quiz (see Step 5)

Keep the first pass shallow — a fast, accurate overview a beginner can finish in one sitting. Initial mode is done when every chapter has both an intro block and a quiz.

## Step 4 — Elaboration mode: deepen progressively

Each subsequent invocation deepens the *existing* chapters rather than replacing them:

- Keep every `data-level="intro"` block intact and unedited — it's the anchor a returning reader recognizes.
- Immediately below it, add or extend a `data-level="deep-dive-N"` block (increment N each session) — one new layer per session, not everything at once. Each layer must add at least one concrete example, edge case, or common misconception; a restated or reworded sentence doesn't count as a new layer.
- Extend that chapter's quiz with 1–3 new questions that probe the newly added depth. Never delete existing questions.
- Only add a brand-new chapter if the topic's scope has genuinely grown (e.g. the user asks for it) — don't pad.
- If a claim from an earlier session turns out to be wrong or outdated per new research, correct it in place and note the correction rather than leaving stale text.

## Step 5 — Quiz mechanics

Each chapter's quiz is inline HTML/JS, no server, no page reload:
- Multiple choice, one correct answer per question
- Clicking an answer immediately reveals correct (green) / incorrect (red) styling plus a one-line explanation
- Track score per chapter in the DOM (e.g. "3/4 correct") — no need to persist across page loads
- Below each chapter's quiz, add a collapsed "Answer key" section (`<details>`/`<summary>`, closed by default) listing every question in that quiz with its correct answer and the one-line explanation. A chapter's quiz isn't done until its answer key lists all of that chapter's questions — none skipped, none from other chapters.

## Step 6 — Visual design

Applies when creating the file (Step 3). Elaboration mode reuses the existing design unchanged unless it's broken.

Single self-contained `.html` file — all CSS and JS inline, no external requests, must render correctly by double-clicking and opening in a browser.

Palette (use consistently, check contrast ≥ 4.5:1 for text):
- **Green** `#2F5233` (or similar deep green) — headings, correct-answer states, primary accents
- **Beige** `#F5F0E1` — page/card backgrounds, quiz option backgrounds
- **Red** `#B22222` (firebrick) — incorrect-answer states, sparing emphasis/callouts
- **Brown** `#6B4226` — secondary text, borders, footer, chapter numbers

Modern look and feel:
- System font stack (no external font requests) with clear type hierarchy
- Card-based chapter layout, rounded corners, subtle shadows
- Sticky chapter navigation / table of contents for jumping between chapters
- Responsive — usable on mobile widths without horizontal scroll
- Smooth-scroll between chapters

## Step 7 — Sources

Add a compact "Sources" section at the very bottom of the page linking out to the articles, posts, and videos used in research. Keep it out of the main teaching flow — a footer-style list, not inline citations breaking up the lesson.

## Step 8 — Finish

Save the file and report its path back to the user. Tell them to open it in a browser to view it — describe the actual chapters, quizzes, and structure you generated, not a generic claim that it "looks good."
