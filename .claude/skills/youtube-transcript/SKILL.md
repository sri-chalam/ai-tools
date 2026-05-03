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
   - **Strip timestamps:** Remove all timestamp markers (e.g., `[17:00]`, `[0:00]`, `00:00`) from the transcript text before formatting. Timestamps should not appear in the body of the document.
   - Generate a logical title based on the video content.
   - **Add a "Summary at a Glance" box at the very top:** Immediately after the title and metadata block, insert a visually distinct summary box containing 3–5 bullet points. Each bullet must be a single concise sentence capturing one essential idea from the video. A reader skimming only this box should understand the core argument and key takeaways in under 30 seconds — without reading anything else in the document. Use a shaded background (beige or light green), a bold green heading "Summary at a Glance", and a subtle border to visually separate it from the body content below. Do not use more than 5 bullets — if the video has more takeaways, pick only the most important ones.
   - Use only the transcript text from the video and do not include information from other internet sources, other videos, or unrelated topic research.
   - Use tables where appropriate to organize key points, steps, comparisons, definitions, or summaries.
   - Use icons and visual markers to improve readability, such as bulbs for insights, memory markers for key takeaways, checkmarks for actions, stars for highlights, and arrows for flows.
   - **Section all verbatim transcript content:** Any block of raw transcript text — regardless of what the section is called — must never be left as a single unbroken wall of text. Analyse it for natural topic shifts: a change in argument, a new concept being introduced, or a transition phrase from the speaker (e.g. "Now let me…", "The third point is…", "So back to the question…"). Insert a Heading 3 label above each shift, aiming for one heading every 3–5 paragraphs. Headings must reflect the speaker's actual argument at that point — not generic labels like "Part 1" or "Section 2". Apply the same green color and bold style used for all other section headings in the document.
   - **Bold key sentences in the transcript:** As you section the verbatim transcript, identify sentences that carry the core argument, a surprising claim, or a critical distinction. Bold the entire sentence — not just a word or phrase within it. Aim for 1–2 bolded sentences per section, no more. Over-bolding defeats the purpose — if everything is emphasised, nothing is. Do not bold transitional or contextual sentences; only bold a sentence a reader would regret skipping.
   - **Pull standout quotes into callout boxes:** As you read through the transcript, identify 2–4 lines where the speaker delivers an insight, analogy, or memorable phrase that captures a bigger idea sharply. Do not bury these in the body text. Instead, lift them out into a visually distinct callout box placed immediately after the paragraph they came from. The box should use a shaded background (light green or beige), a left border accent in dark green, and the quote in italic text with a slightly larger font size than body text. A good candidate for a callout is any sentence a reader might screenshot or share — a strong opinion, a vivid analogy, or a counterintuitive claim. Generic summaries do not qualify.
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
   - **CRITICAL** **Highlights:** Identify important points in the content and make them bold with brown color.
   - **Footer:** Include a footer with "Page X of Y" pagination on each page, in brown color.
4. **Final Step:** Provide a link to the generated file or confirm the path where it was saved.

### Usage:
Run the script with: `python scripts/youtube_transcript_to_docx.py "https://www.youtube.com/watch?v=VIDEO_ID"`

If you encounter a video without a transcript, please notify me immediately.
