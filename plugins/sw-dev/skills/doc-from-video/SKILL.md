---
name: doc-from-video
description: Generates structured markdown documentation from a screen recording (.mov or .mp4). Extracts frames with ffmpeg, analyzes UI steps, selects the best screenshots, and produces a markdown doc with embedded screenshots. Use when the user provides a video walkthrough to document.
argument-hint: "[video-path] [project-name]"
allowed-tools: Bash Read Write Glob
context: fork
---

## doc-from-video

Generate structured markdown documentation from a screen recording.

### Phase 1: Setup

**1. Check ffmpeg is installed**

Run `ffmpeg -version`. If not found, detect the OS and instruct the user with the appropriate install command, then stop.

**2. Parse arguments**

- `$1` — path to the video file (`.mov` or `.mp4`)
- `$2` — name for this documentation (used for folder and doc naming)

If either is missing, ask the user before proceeding.

**3. Determine output location**

Check if a `docs/` folder exists in the current directory:
- **If yes** → output goes to `docs/$2/`
- **If no** → output goes to `$2/` in the current directory

Referred to as `<out>/` for the rest of this skill.

**4. Create folders**

```
<out>/
  images/       ← final screenshots (kept)

.docgen/        ← hidden working folder (deleted after)
  frames/       ← low-res analysis frames
  _tmp/         ← burst frames for screenshot selection
  temp-notes.md ← running notes during analysis
```

The video stays in its original location — do not copy it.

---

### Phase 2: Initial high-level scan

**5. Extract low-res analysis frames at 1fps (JPEG for token efficiency)**

```bash
ffmpeg -i "$1" -vf "fps=1,scale=960:-1" ".docgen/frames/frame_%04d.jpg"
```

Use `fps=2` only if the video is fast-paced or under 3 minutes.

**6. Sample every ~10th frame**

Read frames at roughly even intervals across the full set to build a rough section outline. Do not read every frame at this stage.

**7. Write `.docgen/temp-notes.md` — high-level outline**

```
## High-level outline

- 00:00–00:45 — <section description>
- 00:45–02:10 — <section description>
- ...
```

---

### Phase 3: Detailed section analysis

**8. For each section, read all frames in that time range from `.docgen/frames/`**

**9. For sections that need closer inspection, re-extract at 4fps (low-res)**

```bash
ffmpeg -i "$1" -vf "fps=4,scale=960:-1" -ss <start> -to <end> ".docgen/frames/detail_%04d.jpg"
```

**10. Append detailed notes per section to `.docgen/temp-notes.md`**

For each key moment, note:
- Timestamp (MM:SS)
- UI state — what is visible on screen
- Action performed — what the user is doing
- Key content — field names, values, labels, button text
- Screenshot candidate — flag the best timestamp for a screenshot

> **Long videos (> 10 min):** Process in ~2-minute partitions. Write notes per partition before moving on — do not try to hold the full video in context at once.

---

### Phase 4: Screenshot extraction (cursor-aware)

**11. For each flagged screenshot timestamp, extract a short burst at original resolution**

```bash
ffmpeg -i "$1" -vf "fps=5" -ss <t-2s> -to <t+2s> ".docgen/_tmp/burst_%04d.png"
```

**12. Read the burst frames and select the best one**

Pick the frame where:
- The cursor is not blocking key content (form fields, labels, buttons)
- No dropdown is mid-open
- No loading spinner is visible
- All relevant fields are visible and filled

**13. Copy the chosen frame to `<out>/images/`**

```bash
cp ".docgen/_tmp/burst_XXXX.png" "<out>/images/<NN>-<step-description>.png"
```

Naming: `01-open-form.png`, `02-fill-name-field.png`, etc.

**14. Delete `.docgen/_tmp/`**

---

### Phase 5: Generate the documentation

**15. Write `<out>/$2.md` from `.docgen/temp-notes.md`**

```markdown
# <Project Name>

## Overview
<what the walkthrough demonstrates>

## Prerequisites
<setup required before starting>

## Steps

### 1. <Step title>
<Written instruction>

![<Step title>](images/01-<step>.png)

### 2. <Step title>
...

<!-- TODO: <unrecorded section name> -->
```

Rules:
- Screenshot goes *after* the instruction it illustrates, not before
- Use markdown tables for form fields: `| Field | Value |`
- Add `<!-- TODO -->` placeholders for any steps not covered in the video
- Image paths are relative to `<out>/` (e.g. `images/01-step.png`)

**16. Clean up**

Delete `.docgen/` entirely. Final output is `<out>/images/` and `<out>/$2.md` only.
