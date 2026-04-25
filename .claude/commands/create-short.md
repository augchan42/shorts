Create a YouTube Short from a panel/episode clip.

## Input

The user provides:
- Which source clip to cut from (e.g. "Aug's panel clip", "Alex's clip")
- The segment to extract — either by timestamp range (e.g. "2:16–3:02") or by topic description (e.g. "the capability vs functionality part")

If the user gives a topic description instead of timestamps, read the relevant SRT in `docs/transcripts/` to find the matching segment. Keep it under 60 seconds.

## Setup

This project gets edited on both **macOS** and **WSL/Linux**. The cut/crop/upload steps work identically on both. The burn-in step (#7) needs a libass-enabled `ffmpeg` — different stories per platform:

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

## Workflow

### 1. Identify the segment
- Read the source SRT to confirm exact in/out timecodes
- Verify the segment is ≤ 60 seconds (YouTube Shorts limit)

### 2. Locate the source video
- Check `~/Downloads/` for the source mp4 (e.g. `snowball_panel_aug.mp4`)
- If not found, download from YouTube using `yt-dlp`

### 3. Determine the speaker and framing
This is the critical step — decide the vertical crop BEFORE cutting.

- Read the SRT to identify who is speaking during the segment (check the transcript context — is it Aug, Alex, Tanya, a guest?)
- Extract a frame from the MIDDLE of the segment: `ffmpeg -y -ss {MID_TIMECODE} -i {source} -frames:v 1 /tmp/short_frame.png`
- Read the frame image to see the stage layout and where each person is sitting
- Extract a second frame from a different point in the segment to confirm the camera is static (no panning)
- Determine the 9:16 crop rectangle (608x1080 from 1920x1080):
  - Calculate x-offset to center on the speaker identified from the transcript
  - If multiple people are speaking (dialogue), try to frame both — or center between them
  - If the camera pans or cuts during the segment, note this — a static crop may not work
- Show the user a cropped test frame before proceeding if there's any ambiguity

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
- Take the matching blocks from the source SRT
- Zero the timestamps (subtract the IN timecode from all timestamps)
- Write to `docs/transcripts/short_{slug}.srt`

### 7. Burn in subtitles (English)
YouTube Shorts auto-display the caption track unreliably on mobile, so bake subs into the pixels. Keep the un-burned `{slug}.mp4` around so you can produce zh-Hant / zh-Hans burns later from the same source.

**macOS:**
```bash
~/.local/bin/ffmpeg -y -i docs/shorts/{slug}.mp4 \
  -vf "subtitles=docs/transcripts/short_{slug}.srt:force_style='FontName=Arial,FontSize=11,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,BorderStyle=1,Outline=2,Shadow=0,Alignment=2,MarginV=60'" \
  -c:v libx264 -preset medium -crf 20 -c:a copy \
  docs/shorts/{slug}-en.mp4
```

**WSL/Linux:** same command, just use `ffmpeg` instead of `~/.local/bin/ffmpeg`.

**Style choices** (don't change without reason):
- `FontSize=11` — libass scales relative to its internal PlayResY (288), so `11` renders at ~7-8% of frame height on 1080. Larger values overflow.
- `Bold=1`, `Outline=2` — readable on any backdrop, no drop-shadow noise needed.
- `Alignment=2`, `MarginV=60` — anchors text to bottom-center; sub block lands in the lower-third (~y=75-85%). Below speaker face/torso, above YouTube's mobile UI band (bottom ~15-20%). If a viewer reports the bottom line getting clipped, bump MarginV to 80-100.
- `-c:a copy` — preserve speech quality; no audio re-encode.

**Verify:** extract frames at 3-5 different timestamps with `ffmpeg -ss N -i ... -frames:v 1 /tmp/p{N}.png` and inspect. On macOS, `open docs/shorts/{slug}-en.mp4` for a real-time preview.

### 8. Write the YouTube description
- Write to `docs/yt-descriptions/short-{slug}.txt`
- Format:
  ```
  "{Key quote from the segment}"

  {Speaker Name} on {topic — one sentence}.

  From the Hong Kong AI Podcast x {Event} panel ({date}).

  Full panel: https://www.youtube.com/watch?v={PANEL_VIDEO_ID}
  Full clip: https://www.youtube.com/watch?v={CLIP_VIDEO_ID}
  Website: https://hongkongaipodcast.com

  #HongKongAI #AIPodcast #AI #{TopicTag} #Shorts
  ```

### 9. Upload to YouTube
Upload the burned-in version, NOT the un-burned source:
```bash
pnpm yt upload docs/shorts/{slug}-en.mp4 \
  --title "{Title} #Shorts" \
  --description-file docs/yt-descriptions/short-{slug}.txt \
  --tags "Hong Kong AI,AI Podcast,{speaker},{topic tags},Shorts" \
  --privacy unlisted
```

### 10. Add to Clips playlist
```bash
pnpm yt add-to-playlist PLDb1wIgsBAZ0JnypBYQnOWevr-QJPilyq {VIDEO_ID}
```

### 11. Save artifacts to repo
- Original cropped mp4 (no subs): `docs/shorts/{slug}.mp4`
- Burned-in English mp4: `docs/shorts/{slug}-en.mp4`
- SRT: `docs/transcripts/short_{slug}.srt` (already from step 6)
- Description: `docs/yt-descriptions/short-{slug}.txt` (already from step 8)
- Cut metadata: `docs/shorts/{slug}.md`:
  ```
  source: {source file} ({youtube ID})
  in: {MM:SS.mmm}
  out: {MM:SS.mmm}
  duration: {N.N}s
  crop: center {W}x{H} from 1920x1080 (x offset {X})
  burned_in: docs/shorts/{slug}-en.mp4
  youtube: {VIDEO_ID} (unlisted)
  playlist: Clips (PLDb1wIgsBAZ0JnypBYQnOWevr-QJPilyq)
  srt: docs/transcripts/short_{slug}.srt
  ```

### 12. Commit and push
Stage all new files (both mp4s, SRT, description, metadata) and commit.

## Title format
Keep titles punchy, under 70 chars before `#Shorts`. Use the core insight as the title, not a generic label.

## Naming convention
- slug: kebab-case topic summary (e.g. `capability-not-functionality`)
- mp4 (no subs): `docs/shorts/{slug}.mp4`
- mp4 (burned-in EN): `docs/shorts/{slug}-en.mp4`
- srt: `docs/transcripts/short_{slug}.srt`
- description: `docs/yt-descriptions/short-{slug}.txt`
- metadata: `docs/shorts/{slug}.md`