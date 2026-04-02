---
name: ai-video-analyzer
description: Analyze videos and audio using Gemini 3.1 Pro — frame-by-frame visual analysis, transcription with timestamps and emotions, scene breakdown, ad review, caption/subtitle generation, audio understanding, and content insights. Supports local files (File API), inline small files, YouTube URLs, and audio files. Use when the user wants to analyze a video, transcribe speech, understand video/audio content, extract scenes, review ads, generate captions/SRT, get ffmpeg commands, or needs to see/understand media content before processing it.
allowed-tools: Read, Write, Edit, Bash, Glob
---

# AI Video Analyzer (Gemini 3.1 Pro)

Analyze any video or audio with AI — get transcriptions, scene breakdowns, content analysis, captions, and actionable insights.

## Setup

1. Set your API key:
   ```bash
   # Windows (PowerShell)
   $env:GEMINI_API_KEY="your-api-key-here"

   # macOS / Linux
   export GEMINI_API_KEY=your-api-key-here
   ```

2. Install the SDK:
   ```bash
   npm install @google/genai
   ```

## Video Input Methods

| Method | Max Size | When to Use |
|--------|----------|------------|
| **File API upload** | 20GB (paid) / 2GB (free) | Files > 20MB, videos > 1 min. **Recommended for most cases.** |
| **Inline data** | < 100MB | Small files, short clips < 1 min |
| **YouTube URL** | N/A | Public YouTube videos — no download needed |

**Rule: Always use File API for files > 20MB or videos > 1 minute.**

## Audio Input Methods

| Method | Max Size | Formats |
|--------|----------|---------|
| **File API upload** | Same as video | WAV, MP3, AIFF, AAC, OGG, FLAC |
| **Inline data** | < 20MB total request | Same formats |

**Audio specs:** 32 tokens/second, max 9.5 hours, auto-downsamples to 16Kbps mono.

---

## Quick Start — Analyze a Local Video File

```javascript
import { GoogleGenAI, createUserContent, createPartFromUri } from "@google/genai";

const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

// Step 1: Upload video
const uploadedFile = await ai.files.upload({
  file: "path/to/video.mp4",
  config: { mimeType: "video/mp4" },
});

// Step 2: Wait for processing (REQUIRED — video needs time to process)
let file = await ai.files.get({ name: uploadedFile.name });
while (file.state === "PROCESSING") {
  console.log("Processing video...");
  await new Promise((r) => setTimeout(r, 5000)); // Wait 5 seconds
  file = await ai.files.get({ name: uploadedFile.name });
}
if (file.state === "FAILED") throw new Error("Video processing failed");

// Step 3: Analyze
const response = await ai.models.generateContent({
  model: "gemini-3.1-pro-preview",
  contents: createUserContent([
    createPartFromUri(file.uri, file.mimeType),
    "Analyze this video in detail: scene-by-scene breakdown with timestamps, full transcription, and summary.",
  ]),
});
console.log(response.text);

// Step 4: Cleanup (free up quota)
await ai.files.delete({ name: file.name });
```

## Analyze a YouTube Video (No Download Needed)

```javascript
const response = await ai.models.generateContent({
  model: "gemini-3.1-pro-preview",
  contents: [
    { fileData: { fileUri: "https://www.youtube.com/watch?v=VIDEO_ID" } },
    { text: "Analyze this video: scene breakdown, transcription, and key insights." },
  ],
});
console.log(response.text);
```

**YouTube limits:** Free tier = max 8h/day. Only public videos. Gemini 2.5+ supports up to 10 videos per request.

## Analyze Small Videos Inline (< 100MB)

```javascript
import * as fs from "node:fs";

const base64Video = fs.readFileSync("short-clip.mp4", { encoding: "base64" });

const response = await ai.models.generateContent({
  model: "gemini-3.1-pro-preview",
  contents: [
    { inlineData: { mimeType: "video/mp4", data: base64Video } },
    { text: "What happens in this video? Describe each scene with timestamps." },
  ],
});
```

## Analyze Audio Files

```javascript
// Upload audio
const audioFile = await ai.files.upload({
  file: "path/to/audio.mp3",
  config: { mimeType: "audio/mp3" },
});

// Wait for processing
let file = await ai.files.get({ name: audioFile.name });
while (file.state === "PROCESSING") {
  await new Promise((r) => setTimeout(r, 5000));
  file = await ai.files.get({ name: audioFile.name });
}

const response = await ai.models.generateContent({
  model: "gemini-3.1-pro-preview",
  contents: createUserContent([
    createPartFromUri(file.uri, file.mimeType),
    "Transcribe this audio with timestamps. Detect language and speaker emotions.",
  ]),
});

// Cleanup
await ai.files.delete({ name: file.name });
```

**Inline audio (< 20MB):**
```javascript
const base64Audio = fs.readFileSync("clip.mp3", { encoding: "base64" });

const response = await ai.models.generateContent({
  model: "gemini-3.1-pro-preview",
  contents: [
    { inlineData: { mimeType: "audio/mp3", data: base64Audio } },
    { text: "Transcribe and summarize this audio." },
  ],
});
```

---

## Advanced Features

### Video Clipping — Analyze Specific Segments

```javascript
const contents = [{
  role: 'user',
  parts: [
    {
      fileData: { fileUri: file.uri, mimeType: 'video/mp4' },
      videoMetadata: {
        startOffset: '30s',   // Start at 0:30
        endOffset: '90s',     // End at 1:30
      },
    },
    { text: 'Analyze only this segment.' },
  ],
}];
```

### Custom Frame Rate (FPS)

Default is 1 FPS. Adjust based on content:

```javascript
// Fast-action (sports, quick cuts) — more frames = more detail
videoMetadata: { fps: 5 }

// Long lectures, static content — fewer frames = save tokens
videoMetadata: { fps: 0.5 }
```

### Timestamp References

Always use `MM:SS` format:
```
"What exactly happens at 01:23? Describe the visual and audio."
```

### Structured JSON Output

Force JSON response for programmatic use:
```javascript
const response = await ai.models.generateContent({
  model: "gemini-3.1-pro-preview",
  contents: [...],
  config: {
    responseMimeType: "application/json",
    responseSchema: {
      type: "OBJECT",
      properties: {
        summary: { type: "STRING" },
        segments: {
          type: "ARRAY",
          items: {
            type: "OBJECT",
            properties: {
              timestamp: { type: "STRING" },
              content: { type: "STRING" },
              language: { type: "STRING" },
              emotion: { type: "STRING", enum: ["happy", "sad", "angry", "neutral"] },
            },
            required: ["timestamp", "content"],
          },
        },
      },
      required: ["summary", "segments"],
    },
  },
});
const parsed = JSON.parse(response.text);
```

---

## Prompting Frameworks

### 1. Full Video Analysis
```
Analyze this video completely:
1. Scene-by-scene breakdown with timestamps (MM:SS)
2. Full transcription of all speech/dialogue
3. Description of visual elements, transitions, and effects
4. Background music/sound effects description
5. Overall summary and key takeaways
```

### 2. Ad/Marketing Video Review
```
Review this marketing video as a senior ad strategist:
1. Hook effectiveness (first 3 seconds) — does it stop the scroll?
2. Message clarity — is the value proposition clear?
3. Call-to-action analysis — is it compelling?
4. Pacing and retention — where might viewers drop off?
5. Visual quality and brand consistency
6. Specific improvement recommendations with timestamps
```

### 3. Transcription with Emotions & Speakers
```
Transcribe this video/audio with:
- Timestamps in MM:SS format
- Speaker identification (Speaker 1, Speaker 2, etc.)
- Emotion detection per segment (happy, sad, angry, neutral)
- Non-verbal audio cues [applause], [music], [silence]
- Language detection per segment
- Output as JSON array
```

### 4. Caption/Subtitle Generation (SRT)
```
Generate SRT subtitles for this video:
- Short captions (max 2 lines, ~42 chars per line)
- Synced to speech with accurate timestamps
- Format: standard SRT with sequence numbers
```

### 5. Video Edit Guidance (ffmpeg)
```
I need to edit this video with ffmpeg. Analyze it and provide:
1. Exact timestamps of scene changes
2. Which segments to keep vs cut
3. Ready-to-run ffmpeg commands to:
   - Trim to the best segments
   - Remove dead air/silence
   - Extract a 15-second highlight clip
   - Add fade transitions between cuts
```

### 6. AI-Generated Video Quality Check
```
Review this AI-generated video for quality:
1. Visual artifacts or glitches (with timestamps)
2. Unnatural movements or physics
3. Face/hand consistency issues
4. Audio sync problems
5. Overall quality score (1-10) and specific fixes needed
```

### 7. Competitor Content Analysis
```
Analyze this competitor's video content:
1. Hook technique and opening strategy
2. Content structure (intro → body → CTA)
3. Engagement techniques used
4. What makes it effective (or not)
5. Replicable formula for my own content
```

### 8. Audio Transcription with Translation
```
Process this audio and generate a detailed transcription:
1. Timestamps in MM:SS format
2. Detect the primary language of each segment
3. Provide English translation if not in English
4. Detect speaker emotion (happy, sad, angry, neutral)
5. Brief summary at the beginning
```

---

## Token Calculation

### Video
| Duration | Default Res (~300 tok/s) | Low Res (~100 tok/s) |
|----------|--------------------------|----------------------|
| 30 seconds | 9,000 | 3,000 |
| 1 minute | 18,000 | 6,000 |
| 10 minutes | 180,000 | 60,000 |
| 1 hour | 1,080,000 | 360,000 |

### Audio Only
- **32 tokens per second** (1,920 per minute)
- Max 9.5 hours per prompt

**Save tokens on long videos:**
```javascript
config: { mediaResolution: 'low' }
```

---

## Supported Formats

**Video:** mp4, mpeg, mov, avi, x-flv, mpg, webm, wmv, 3gpp
**Audio:** wav, mp3, aiff, aac, ogg, flac

## Best Practices

1. **One video per request** for best quality
2. **Text prompt AFTER the video** in contents array
3. **File API for anything > 20MB** — inline will fail
4. **Always poll file.state** until "ACTIVE" before analyzing
5. **Delete uploaded files** after analysis to save quota
6. **Use clipping** for long videos — analyze segments, not the full thing
7. **Increase FPS** for fast-action (sports, quick transitions)
8. **Decrease FPS** for lectures, interviews, static content
9. **Use context caching** for repeated queries on the same video
10. **MM:SS timestamps** — always use this format

## Common Use Cases

1. **Ad review** — Hook, message, CTA effectiveness analysis
2. **Transcription** — Speech-to-text with timestamps, speakers, emotions
3. **Caption generation** — SRT subtitles for social media
4. **Content analysis** — Break down viral videos to learn what works
5. **Video editing prep** — Get timestamps and ffmpeg commands
6. **AI video QA** — Check Seedance/Veo outputs for artifacts
7. **Competitor analysis** — Analyze competitor ads and strategy
8. **Meeting summaries** — Transcribe and summarize recordings
9. **Audio analysis** — Podcast transcription, music analysis, sound identification
10. **Accessibility** — Audio descriptions for visual content
