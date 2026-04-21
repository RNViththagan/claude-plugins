---
name: doc-from-video
description: Generates structured markdown documentation from a screen recording (.mov or .mp4). Uses ffmpeg to extract frames, analyzes the video in phases — high-level scan then section-by-section — taking timestamped notes with full UI inventory on new views and action tracking within views. Selects cursor-aware screenshots via burst extraction. Outputs a markdown doc and images folder, respecting existing docs/ structure. Use when the user provides a screen recording to document.
argument-hint: "[video-path] [doc-name]"
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
- `$2` — doc name (used for the output folder and markdown filename)

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

**7. Begin `.docgen/temp-notes.md`**

Start writing notes immediately. Notes are the source of truth for the doc — be thorough.

Note-taking rules:

- **Stay task-focused**: only write detailed notes for parts of the video relevant to the documentation goal. For unrelated sections (e.g. navigation, loading, side interactions), write a single brief marker — enough to find it again if it turns out to be needed later.
- **When a view changes entirely** (new screen, dialog, page, panel) and it is relevant: mark it clearly and document *everything* visible — all fields, labels, buttons, values, layout, navigation state. This is the full inventory of that view.
- **Within the same view**: only note what changed — actions taken, fields filled, buttons clicked, responses shown.
- **Every entry has a timestamp** (MM:SS) so it can be traced back to the video.
- **Flag screenshot candidates** inline — mark the best timestamp in that section for a screenshot.

Structure:

```
## [00:00] <View name / screen title>
FULL INVENTORY: <describe every visible element — fields, labels, buttons, panels, existing content>
Screenshot candidate: 00:05

## [00:12] Action — <what happened>
<what changed, what was filled/clicked/selected>

## [00:28] <Unrelated section — brief marker>
SKIP: <one line describing what's happening — not relevant to task>

## [00:45] <New relevant view name>
FULL INVENTORY: <everything visible on this new screen>
Screenshot candidate: 00:48
...
```

---

### Phase 3: Detailed section analysis

**8. For each section identified, read all frames in that time range from `.docgen/frames/`**

**9. For sections that need closer inspection, re-extract at 4fps (low-res)**

```bash
ffmpeg -i "$1" -vf "fps=4,scale=960:-1" -ss <start> -to <end> ".docgen/frames/detail_%04d.jpg"
```

**10. Append detailed notes for each section to `.docgen/temp-notes.md`**

Follow the same note-taking rules from step 7 — full inventory on first view appearance, action tracking within a view.

> **Long videos (> 10 min):** Process in ~5-minute partitions. Write notes per partition before moving on — do not try to hold the full video in context at once.

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
```

Rules:
- Screenshot goes *after* the instruction it illustrates, not before
- Use markdown tables for form fields: `| Field | Value |`
- Image paths are relative to `<out>/` (e.g. `images/01-step.png`)

**16. Clean up**

Delete `.docgen/` entirely. Final output is `<out>/images/` and `<out>/$2.md` only.
