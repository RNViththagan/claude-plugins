---
name: doc-from-video
description: Generates structured markdown documentation from a screen recording (.mov or .mp4). Uses ffmpeg to extract frames, analyzes the video in phases — high-level scan then section-by-section — taking timestamped notes with full UI inventory on new views and action tracking within views. Presents a doc outline to the user for review before proceeding. Selects cursor-aware screenshots via burst extraction, cropping to the relevant UI region where it improves clarity. Outputs a markdown doc and images folder, respecting existing docs/ structure. Use when the user provides a screen recording to document.
allowed-tools: Bash Read Write Glob
---

### Phase 1: Setup

**1. Get the video path** — ask if not provided.

**2. Check ffmpeg** — run `ffmpeg -version`. If missing, give the install command for the user's OS and stop.

**3. Create working folders**

```
.docgen/        ← hidden working folder (kept until user confirms doc is final)
  frames/       ← low-res analysis frames
  _tmp/         ← burst frames for screenshot selection
  temp-notes.md ← running notes
```

The video stays in its original location — do not copy it.

---

### Phase 2: Initial high-level scan

**4. Extract low-res analysis frames at 1fps (JPEG for token efficiency)**

```bash
ffmpeg -i "<video-path>" -vf "fps=1,scale=960:-1" ".docgen/frames/frame_%04d.jpg"
```

Use `fps=2` only if the video is fast-paced or under 3 minutes.

**5. Sample every ~10th frame**

Read frames at roughly even intervals to understand what the video covers. From this, decide on a short descriptive doc name (e.g. `install-plugin`, `create-api-endpoint`) and determine the output location:

- If a `docs/` folder exists in the current directory → `docs/<doc-name>/`
- Otherwise → `<doc-name>/` in the current directory

Referred to as `<out>/` for the rest of this skill. Create `<out>/images/`.

**6. Begin `.docgen/temp-notes.md`** — notes are the source of truth for the doc.

Note-taking rules:

- **Stay task-focused** — only detail parts relevant to the documentation goal; for unrelated sections (e.g. navigation, loading, side interactions), one brief marker is enough
- **New view** (screen, dialog, page, panel): mark it clearly and document *everything* visible — all fields, labels, buttons, values, layout, navigation state
- **Same view**: note only what changed — actions taken, fields filled, buttons clicked, responses shown
- **Irrelevant section**: one `SKIP:` line with timestamp — enough to find it again if needed
- Every entry has a timestamp (MM:SS); flag screenshot candidates inline

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
```

---

### Phase 3: Detailed section analysis

**7. Read all frames** for each section from `.docgen/frames/`.

**8. Re-extract at 4fps** for sections needing closer inspection:

```bash
ffmpeg -i "<video-path>" -vf "fps=4,scale=960:-1" -ss <start> -to <end> ".docgen/frames/detail_%04d.jpg"
```

**9. Append detailed notes** per section — same rules as step 6.

> **Long videos (> 10 min):** Process in ~5-minute partitions. Write notes per partition before moving on — do not try to hold the full video in context at once.

---

### Phase 4: User review

**10. Present the outline before proceeding** — show the doc title, each step with its planned screenshot timestamp, and sections being skipped.

**Wait for user confirmation or corrections. Update notes with any changes before continuing.**

---

### Phase 5: Screenshot extraction

**11. Extract a 5fps burst at original resolution around each candidate timestamp**

```bash
ffmpeg -i "<video-path>" -vf "fps=5" -ss <t-2s> -to <t+2s> ".docgen/_tmp/burst_%04d.png"
```

**12. Read the burst frames and select the best one**

Pick the frame where:
- The cursor is not blocking key content (form fields, labels, buttons)
- No dropdown is mid-open
- No loading spinner is visible
- All relevant fields are visible and filled

**13. Save to `<out>/images/`** — crop to the relevant region when it improves clarity (dialog, panel, result), copy full-screen otherwise.

```bash
# cropped
ffmpeg -i ".docgen/_tmp/burst_XXXX.png" -vf "crop=<w>:<h>:<x>:<y>" "<out>/images/<NN>-<desc>.png"
# full screen
cp ".docgen/_tmp/burst_XXXX.png" "<out>/images/<NN>-<desc>.png"
```

**14. Delete `.docgen/_tmp/`** after all screenshots are saved.

---

### Phase 6: Generate the documentation

**15. Write `<out>/<doc-name>.md` from `.docgen/temp-notes.md`**

```markdown
# <Title>

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

---

### Phase 7: Clean up

**16. Tell the user the doc is ready** — share the output path and ask them to review. Keep `.docgen/` in place in case corrections are needed.

**17. Delete `.docgen/` only after the user confirms they are done.**
