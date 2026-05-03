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

### Prerequisites:
- Python 3.7+
- Install required dependencies: `pip install youtube-transcript-api python-docx`
- Optional fallback dependencies: `pip install selenium webdriver-manager`
- Optional video availability check: `pip install requests`
- Optional browser fallback requires a local Chrome installation

### Implementation:
This skill uses the `scripts/youtube_transcript_to_docx.py` script which implements the following functionality:

**Enhanced 403 Error Handling:**
- **Retry Logic**: Automatically retries API calls up to 3 times with exponential backoff
- **User-Agent Rotation**: Uses different Chrome browser user-agents to bypass simple blocking
- **Video Availability Check**: Pre-validates video accessibility before extraction attempts
- **Specific Error Handling**: Distinguishes between 403 (retry), 404 (unavailable), and other errors
- **Browser Fallback Improvements**: Enhanced headless Chrome setup with anti-detection measures

### Your Task:
1. **Fetch Transcript:** Use the `youtube-transcript-api` Python library to retrieve the transcript. If that fails (e.g., due to restrictions or a 403 response), optionally fall back to browser-based transcript extraction using Selenium and Chrome. If Selenium is unavailable, surface the failure clearly rather than silently dropping the transcript.

> Strict transcript gate: after each extraction attempt, explicitly check that you have actual transcript text. If not, do not proceed to document generation. Try the next fallback, or stop with a clear failure. Never use web search results or external internet content as a substitute for the transcript.

2. **Organize Content:**
   - Generate a logical title based on the video content.
   - Insert descriptive headings throughout the transcript to break up the text into readable sections.
   - Use only the transcript text from the video and do not include information from other internet sources, other videos, or unrelated topic research.
   - Use tables where appropriate to organize key points, steps, comparisons, definitions, or summaries.
   - Use icons and visual markers to improve readability, such as bulbs for insights, memory markers for key takeaways, checkmarks for actions, stars for highlights, and arrows for flows.
   - Make the document visually appealing and easy to scan with polished formatting, consistent spacing, and clear section structure.
   - Apply visual formatting techniques like shaded table headers, bold callout lines, boxed summary sections, and consistent spacing so the document looks professional and readable.
   - Define custom paragraph styles for speakers (bold, colored), timestamps (italic, smaller font), and transcript text (standard body font with line spacing).
   - Use a professional color palette: dark green (e.g., #006400) for headings, beige (#F5F5DC) for table headers, red (#B22222) for highlights, ensuring accessibility with good contrast ratios (aim for 4.5:1 text-to-background).
   - Add visual separators like horizontal rules, section breaks, or blockquotes for logical divisions in the transcript.
3. **Generate Word Document:** Create a `.docx` file using a Python script with the following specifications:
   - **Filename:** Use the format `Title-Words-Here-YT-Transcript.docx` (words separated by dashes, each word starting with uppercase).
   - **Font:** Helvetica Neue (fallback to Helvetica or Arial if unavailable).
   - **Font Size:** 12pt for body text.
   - **Structure:** Professional headings (bolded and slightly larger).
   - **Colors:** All titles and section headings should use green color.
   - **Highlights:** Identify important points in the content and make them bold with brown color.
   - **Footer:** Include a footer with "Page X of Y" pagination on each page, in brown color.
4. **Final Step:** Provide a link to the generated file or confirm the path where it was saved.

### Usage:
Run the script with: `python scripts/youtube_transcript_to_docx.py "https://www.youtube.com/watch?v=VIDEO_ID"`

If you encounter a video without a transcript, please notify me immediately.
