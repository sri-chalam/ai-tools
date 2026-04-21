---
name: youtube-transcript
description: Downloads a YouTube transcript and formats it into a professional Word document.
argument-hint: [YouTube URL]
# Use the full YouTube URL (e.g., https://www.youtube.com/watch?v=...), not the shortened youtu.be link from "Share"
# When true, this skill must be invoked manually with /youtube-transcript (Claude won't auto-trigger it)
disable-model-invocation: true
---

# YouTube Transcript to Word Document

I want you to process the following YouTube video: $ARGUMENTS

### Your Task:
1. **Fetch Transcript:** Use the `youtube-transcript-api` Python library to retrieve the transcript. If that fails (e.g., due to restrictions), use browser access to navigate to the video and extract the transcript from the transcript panel (click "..." → "Show transcript").
2. **Organize Content:**
   - Generate a logical title based on the video content.
   - Insert descriptive headings throughout the transcript to break up the text into readable sections.
3. **Generate Word Document:** Create a `.docx` file using a Python script with the following specifications:
   - **Filename:** Use the format `Title-Words-Here-YT-Transcript.docx` (words separated by dashes, each word starting with uppercase).
   - **Font:** Helvetica Neue (fallback to Helvetica or Arial if unavailable).
   - **Font Size:** 12pt for body text.
   - **Structure:** Professional headings (bolded and slightly larger).
   - **Colors:** All titles and section headings should use green color.
   - **Highlights:** Identify important points in the content and make them bold with green color.
   - **Footer:** Include a footer with "Page X of Y" pagination on each page, in green color.
4. **Final Step:** Provide a link to the generated file or confirm the path where it was saved.

If you encounter a video without a transcript, please notify me immediately.
