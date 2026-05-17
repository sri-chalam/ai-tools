# YouTube Transcript to Word Document Scripts

This directory contains the implementation scripts for the YouTube transcript to Word document conversion skill.

## Files

- `youtube_transcript_to_docx.py` - Main script that processes YouTube videos and generates formatted Word documents
- `requirements.txt` - Python dependencies required for the scripts

## Setup

1. Install Python dependencies:
   ```bash
   pip install -r requirements.txt
   ```

2. For browser fallback functionality, ensure Google Chrome browser is installed on your system.

## Usage

Run the main script with a YouTube URL:

```bash
python youtube_transcript_to_docx.py "https://www.youtube.com/watch?v=VIDEO_ID"
```

### Options

- `-o, --output DIR` - Output directory for the generated document (default: current directory)

### Example

```bash
python youtube_transcript_to_docx.py "https://www.youtube.com/watch?v=dQw4w9WgXcQ" -o ./documents
```

This will create a file like `Never-Gonna-Give-You-Up-YT-Transcript.docx` in the specified output directory.

## Features

- **Dual extraction methods**: Uses YouTube API first, falls back to browser automation
- **Enhanced 403 error handling**: Retry logic with Chrome user-agent rotation and exponential backoff
- **Video availability validation**: Checks video accessibility before extraction attempts
- **Smart title generation**: Creates meaningful titles from transcript content
- **Professional formatting**: Applies consistent styling, colors, and layout
- **Timestamp preservation**: Includes timestamps for each transcript segment
- **Section organization**: Automatically identifies and creates section headings

## Dependencies

- `youtube-transcript-api`: For YouTube transcript extraction
- `python-docx`: For Word document generation
- `selenium`: For browser automation fallback
- `webdriver-manager`: For automatic Chrome driver management

## Troubleshooting

- If transcript extraction fails, ensure the video has captions available
- For browser fallback, Chrome must be installed and accessible
- Check that all Python dependencies are properly installed