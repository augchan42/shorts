Create a YouTube Short from a panel/episode clip.

Paths in this skill are written relative to the **repo root** (the checked-out copy of this `shorts` workflow). Where commands `cd` away (e.g. transcribe steps run from `~/projects/transcriber`), absolute paths are required — substitute `{REPO}` for your repo root (e.g. `/Users/you/projects/shorts`).

## Input

The user provides:
- The source video (path to a local mp4, or a YouTube URL to download)
- The segment to extract — either by timestamp range (e.g. "2:16–3:02") or by topic description (e.g. "the part about capability vs functionality")
- Optionally: speaker name, context (event/panel name + date), tags, and the Full Panel / Full Clip / Website URLs that should appear in the description

If the user gives a topic description instead of timestamps, read the relevant SRT in `docs/transcripts/` to find the matching segment. Keep it under 60 seconds.

## Setup

The cut/crop/upload steps work identically on **macOS** and **WSL/Linux**. The burn-in step (#7) needs a libass-enabled `ffmpeg` — different stories per platform:

### macOS (one-time)

The default Homebrew `ffmpeg` does NOT include `libass`, so the `subtitles` filter is unavailable and burn-in fails with a cryptic "No option name near …" error. Install a libass-enabled static build to `~/.local/bin/ffmpeg`:

```bash
mkdir -p ~/.local/bin
curl -L https://evermeet.cx/ffmpeg/getrelease/ffmpeg/zip -o /tmp/ff.zip
unzip -o /tmp/ff.zip -d ~/.local/bin/
chmod +x ~/.local/bin/ffmpeg
~/.local/bin/ffmpeg -version | grep -E "libass|libfreetype"   # verify both appear
```

Use `~/.local/bin/ffmpeg` for burn-in only; system `ffmpeg` is fine for cut/crop.

We tried the `homebrew-ffmpeg/ffmpeg/ffmpeg` tap as the "proper" path — it fails at configure with `aom not found in pkg-config` on Tahoe / Xcode 26.3 (Tier 2 build issues). The static binary sidesteps all of that.

### WSL/Linux

System `ffmpeg` (apt or Linuxbrew) ships with `libass` baked in. Just use `ffmpeg` — no static-binary dance.

### Python env for Whisper (one-time, both platforms)

The transcription steps depend on the `transcriber` repo (https://github.com/augchan42/transcriber) checked out at `~/projects/transcriber` with a venv. If it's not there:

```bash
mkdir -p ~/projects && cd ~/projects
git clone https://github.com/augchan42/transcriber.git
cd transcriber
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt    # installs faster-whisper, yt-dlp, requests
```

**macOS only** — also install mlx-whisper for Apple Silicon GPU inference:
```bash
cd ~/projects/transcriber && source venv/bin/activate
pip install mlx-whisper
```

Verify both work:
```bash
cd ~/projects/transcriber && source venv/bin/activate
python3 transcribe.py --help    # faster-whisper path
mlx_whisper --help              # macOS only — MLX path
```

Models download lazily from HuggingFace on first run and cache under `~/.cache/huggingface/`. The base English model (~140MB) downloads in seconds; large multilingual models (~3GB) take a couple of minutes.

## Prerequisites: transcribe the source video (if no SRT exists)

Before starting, check whether a source SRT already exists in `docs/transcripts/`. If it does, skip to the Workflow. If not, transcribe the source video first.

### Get the source video

If you have a local mp4, use it directly. If the video is on YouTube (including unlisted), download it first:

```bash
yt-dlp -f "bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]" \
  "https://www.youtube.com/watch?v={VIDEO_ID}" \
  -o "~/Downloads/{slug_source}.mp4"
```

For unlisted videos, yt-dlp needs cookies. If it fails with a bot/auth error:
```bash
yt-dlp --cookies-from-browser chrome \
  -f "bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]" \
  "https://www.youtube.com/watch?v={VIDEO_ID}" \
  -o "~/Downloads/{slug_source}.mp4"
```

### Transcribe

#### macOS (Apple Silicon) — mlx-whisper

Uses the GPU via MLX. Much faster than CPU. One-time install:
```bash
cd ~/projects/transcriber && source venv/bin/activate && pip install mlx-whisper
```

**English source:**
```bash
cd ~/projects/transcriber && source venv/bin/activate && mlx_whisper \
  ~/Downloads/{slug_source}.mp4 \
  --model mlx-community/whisper-base-mlx \
  --language en \
  --output-format srt \
  --output-dir {REPO}/docs/transcripts \
  --condition-on-previous-text False
```

**Cantonese/mixed source:**
```bash
cd ~/projects/transcriber && source venv/bin/activate && mlx_whisper \
  ~/Downloads/{slug_source}.mp4 \
  --model mlx-community/whisper-large-v3-mlx \
  --language yue \
  --output-format srt \
  --output-dir {REPO}/docs/transcripts \
  --condition-on-previous-text False
```

For other languages, swap `--language` (ISO 639-1, e.g. `zh`, `ja`, `es`) and pick a matching model from `mlx-community/`.

Models are downloaded from HuggingFace on first use and cached locally.

#### WSL/Linux (CUDA) — faster-whisper

**English source:**
```bash
cd ~/projects/transcriber && source venv/bin/activate && python3 transcribe.py \
  ~/Downloads/{slug_source}.mp4 \
  -l en -m base --vad-filter --no-condition-prev \
  -o {REPO}/docs/transcripts
```

**Other languages** — point `-m` at the model dir or HuggingFace repo and pass the language code:
```bash
cd ~/projects/transcriber && source venv/bin/activate && python3 transcribe.py \
  ~/Downloads/{slug_source}.mp4 \
  -m {model_path_or_hf_id} \
  -l {iso_code} --device cuda --compute-type float16 \
  --vad-filter --no-condition-prev \
  -o {REPO}/docs/transcripts
```

Output: `docs/transcripts/{slug_source}.srt` and `.txt`.

### Quick review pass

Whisper mangles proper nouns. Before using the SRT to find segments, fix at minimum:

- Speaker names (keep a name-mapping note alongside the SRT if you cut multiple shorts from the same source)
- Company/product names that appear in the segment you're cutting
- Strip any gibberish from the first SRT block (intro music hallucination)

Fix both `.srt` and `.txt` in lockstep. Full review isn't required before cutting a Short — just clean enough to identify the right segment and produce readable subs.

---

## Workflow

### 1. Identify the segment
- Read the source SRT to confirm exact in/out timecodes
- Verify the segment is ≤ 60 seconds (YouTube Shorts limit)

### 2. Locate the source video
- Use the path the user provided, or check `~/Downloads/` for the source mp4
- If not found, download from YouTube using `yt-dlp` (see Prerequisites)

### 3. Determine the speaker and framing
This is the critical step — decide the vertical crop BEFORE cutting.

- Read the SRT to identify who is speaking during the segment
- Extract a frame from the MIDDLE of the segment: `ffmpeg -y -ss {MID_TIMECODE} -i {source} -frames:v 1 /tmp/short_frame.png`
- Read the frame image to see the stage layout and where each person is sitting
- Extract a second frame from a different point in the segment to confirm the camera is static (no panning)
- Determine the 9:16 crop rectangle (608x1080 from 1920x1080):
  - Calculate x-offset to center on the speaker identified from the transcript
  - If multiple people are speaking (dialogue), try to frame both — or center between them
  - If the camera pans or cuts during the segment, note this — a static crop may not work
- Show the user a cropped test frame before proceeding if there's any ambiguity

If the source isn't 1920x1080, scale the crop dimensions to match its 9:16 ratio (height × 9/16 = crop width).

### 4. Cut and crop
```bash
ffmpeg -y -ss {IN} -to {OUT} -i {source} \
  -vf "crop=608:1080:{X_OFFSET}:0" \
  -c:v libx264 -preset medium -crf 20 \
  -c:a aac -b:a 128k \
  ~/Downloads/{slug}_vertical.mp4
```

### 5. Verify the result
- Extract a frame from the cropped video and visually check it
- Confirm duration is ≤ 60s
- Move/copy the file into the repo: `docs/shorts/{slug}.mp4`

### 6. Create the SRT
**Do NOT zero the source SRT timestamps** — offset-from-source has drift and goes out of sync by the end of the clip. Instead, run Whisper on the cut clip itself to get accurate timings, then overlay the clean text:

**macOS:**
```bash
cd ~/projects/transcriber && source venv/bin/activate && mlx_whisper \
  {REPO}/docs/shorts/{slug}.mp4 \
  --model mlx-community/whisper-base-mlx --language en \
  --output-format srt --output-dir /tmp/short_retranscribe \
  --condition-on-previous-text False
```

**WSL/Linux:**
```bash
cd ~/projects/transcriber && source venv/bin/activate && python3 transcribe.py \
  {REPO}/docs/shorts/{slug}.mp4 \
  -l en -m base -o /tmp/short_retranscribe --format both
```

Then write `docs/transcripts/short_{slug}.srt` using **Whisper's timestamps** but **the clean text** from the source SRT (Whisper's text will have garbled proper nouns and filler words — always swap in the cleaned version). Align text blocks to Whisper's segment boundaries; merge or split segments as needed to fit the clean text naturally.

### 7. Burn in subtitles
YouTube Shorts auto-display the caption track unreliably on mobile, so bake subs into the pixels. Keep the un-burned `{slug}.mp4` around so you can produce other-language burns later from the same source.

**macOS:**
```bash
~/.local/bin/ffmpeg -y -i docs/shorts/{slug}.mp4 \
  -vf "subtitles=docs/transcripts/short_{slug}.srt:force_style='FontName=Arial,FontSize=11,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,BorderStyle=1,Outline=2,Shadow=0,Alignment=2,MarginV=60'" \
  -c:v libx264 -preset medium -crf 20 -c:a copy \
  docs/shorts/{slug}-en.mp4
```

**WSL/Linux:** same command, just use `ffmpeg` instead of `~/.local/bin/ffmpeg`.

The output suffix (`-en`) reflects the burned-in language — change it (`-zh-Hant`, `-ja`, etc.) when burning a translated SRT.

**Style choices** (don't change without reason):
- `FontSize=11` — libass scales relative to its internal PlayResY (288), so `11` renders at ~7-8% of frame height on 1080. Larger values overflow.
- `Bold=1`, `Outline=2` — readable on any backdrop, no drop-shadow noise needed.
- `Alignment=2`, `MarginV=60` — anchors text to bottom-center; sub block lands in the lower-third (~y=75-85%). Below speaker face/torso, above YouTube's mobile UI band (bottom ~15-20%). If a viewer reports the bottom line getting clipped, bump MarginV to 80-100.
- `-c:a copy` — preserve speech quality; no audio re-encode.

**Verify:** extract frames at 3-5 different timestamps with `ffmpeg -ss N -i ... -frames:v 1 /tmp/p{N}.png` and inspect. On macOS, `open docs/shorts/{slug}-{lang}.mp4` for a real-time preview.

### 8. Write the YouTube description
- Write to `docs/yt-descriptions/short-{slug}.txt`
- Suggested format (adapt to your show — keep the quote on top, links toward the bottom, hashtags last):
  ```
  "{Key quote from the segment}"

  {Speaker Name} on {topic — one sentence}.

  {Optional: where this is from, e.g. "From the {Show Name} x {Event} panel ({date})."}

  {Optional: Full panel: https://www.youtube.com/watch?v={PANEL_VIDEO_ID}}
  {Optional: Full clip: https://www.youtube.com/watch?v={CLIP_VIDEO_ID}}
  {Optional: Website: https://your-site.example}

  #{TopicTag} #Shorts
  ```

### 9. Upload to YouTube
Upload the burned-in version (`{slug}-{lang}.mp4`), NOT the un-burned source. This skill is upload-tool-agnostic — use whichever CLI/SDK you have. The metadata to set:

- **File:** `docs/shorts/{slug}-{lang}.mp4`
- **Title:** `{Title} #Shorts` (under 70 chars before `#Shorts`)
- **Description:** contents of `docs/yt-descriptions/short-{slug}.txt`
- **Tags:** comma-separated topical tags + `Shorts`
- **Privacy:** `unlisted` (or `public` once you've verified it on YouTube)

Common options:
- YouTube Studio web UI (manual)
- `youtube-upload` (PyPI) — `youtube-upload --title=... --description-file=... --privacy=unlisted docs/shorts/{slug}-en.mp4`
- A custom CLI in your consuming project (e.g. one that wraps the YouTube Data API v3)

After upload, capture the resulting `VIDEO_ID` for steps 10 and 11.

### 10. Add to a playlist (optional)
If you maintain a "Clips" / "Shorts" playlist, add the new video to it. Use whatever your upload tool supports (most YouTube CLIs have an `add-to-playlist` or `playlists.insert` operation). Record the playlist ID in the metadata file (step 11) so future shorts can be added to the same one without re-looking-up.

### 11. Save artifacts to repo
- Original cropped mp4 (no subs): `docs/shorts/{slug}.mp4`
- Burned-in mp4: `docs/shorts/{slug}-{lang}.mp4`
- SRT: `docs/transcripts/short_{slug}.srt` (already from step 6)
- Description: `docs/yt-descriptions/short-{slug}.txt` (already from step 8)
- Cut metadata: `docs/shorts/{slug}.md`:
  ```
  source: {source file} ({youtube ID if applicable})
  in: {MM:SS.mmm}
  out: {MM:SS.mmm}
  duration: {N.N}s
  crop: center {W}x{H} from {SOURCE_W}x{SOURCE_H} (x offset {X})
  burned_in: docs/shorts/{slug}-{lang}.mp4
  youtube: {VIDEO_ID} ({privacy})
  playlist: {PLAYLIST_NAME} ({PLAYLIST_ID})    # optional
  srt: docs/transcripts/short_{slug}.srt
  ```

### 12. Commit and push
Stage all new files (both mp4s, SRT, description, metadata) and commit.

## Title format
Keep titles punchy, under 70 chars before `#Shorts`. Use the core insight as the title, not a generic label.

## Naming convention
- slug: kebab-case topic summary (e.g. `capability-not-functionality`)
- mp4 (no subs): `docs/shorts/{slug}.mp4`
- mp4 (burned-in, language-suffixed): `docs/shorts/{slug}-{lang}.mp4` (e.g. `-en`, `-zh-Hant`, `-ja`)
- srt: `docs/transcripts/short_{slug}.srt`
- description: `docs/yt-descriptions/short-{slug}.txt`
- metadata: `docs/shorts/{slug}.md`
