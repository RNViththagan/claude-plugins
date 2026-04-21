# sw-dev

Software development productivity and documentation tools for Claude Code.

## Install

```
/plugin install sw-dev@RNViththagan
```

## Skills

### doc-from-video

Generate structured markdown documentation from a screen recording.

```
/sw-dev:doc-from-video <video-path> <doc-name>
```

**What it does:**
- Extracts frames from the video using ffmpeg and analyzes them in phases
- Takes detailed timestamped notes — full UI inventory on new views, action tracking within views
- Selects cursor-aware screenshots via burst extraction
- Outputs a markdown doc and `images/` folder
- Respects existing `docs/` folder structure if present
- Cleans up all intermediate work when done

**Requirements:** `ffmpeg` must be installed. If not found, the skill will detect your OS and provide the install command.

**Output:**
- If `docs/` exists in current dir → `docs/<doc-name>/`
- Otherwise → `<doc-name>/` in current dir

```
<doc-name>/
  <doc-name>.md
  images/
    01-<step>.png
    02-<step>.png
    ...
```
