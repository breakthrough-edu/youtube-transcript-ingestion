---
name: youtube-transcript-ingestion
description: Batch-grab transcripts from any YouTube channel and save them as readable markdown files with clean frontmatter. Guided 5-step flow -- channel, keyword + date filter, output folder, optional context-delta comparison, rate-limit strategy -- using yt-dlp for metadata and markitdown for transcripts. Optionally compares results against your existing notes to surface only what is genuinely new. Trigger phrases: grab YouTube transcripts, ingest YouTube channel, yt-dlp transcripts, markitdown YouTube, research a creator's videos, build a corpus from a channel, filter content overload from a channel.
---

# YouTube Transcript Ingestion

Batch-grab transcripts from any YouTube channel and save them as readable `.md` files -- with a guided 5-step flow and two rate-limit strategies.

## When to load this skill

Load when the user wants to:
- Research what a specific creator has said about a topic
- Build a reading corpus from video content without watching every video
- Filter content overload from a channel ("what here is actually new for me?")
- Archive transcripts for future reference or AI-assisted analysis

Trigger phrases: "grab YouTube transcripts", "ingest YouTube channel", "yt-dlp transcripts", "markitdown YouTube".

---

## What this skill does

1. Extracts the full video list from any YouTube channel (no download, metadata only)
2. Filters by date range + title keywords
3. Validates actual upload dates (flat-playlist timestamps are unreliable -- always verify)
4. Fetches full transcripts for each matched video via markitdown
5. Saves each as a `.md` file with clean YAML frontmatter into a folder of the user's choice
6. Optionally compares transcripts against an existing notes/context folder to surface only what's genuinely new

---

## Pre-flight check

Before starting, verify the two required tools. Run this:

```bash
echo "yt-dlp: $(which yt-dlp 2>/dev/null || echo NOT FOUND)"
echo "markitdown: $(which markitdown 2>/dev/null || \
  find /Library/Frameworks/Python.framework -name 'markitdown' 2>/dev/null | head -1 || \
  echo NOT FOUND)"
```

**If yt-dlp is missing:**
```bash
# Option 1 (recommended):
brew install yt-dlp

# Option 2:
pip install yt-dlp

# Option 3 (npm):
npm install -g yt-dlp
```

**If markitdown is missing:**
```bash
pip install markitdown
# or, for the MCP server version:
pip install "markitdown-mcp==0.0.1a4" --no-deps && pip install "mcp>=1.0.0"
```

After install, find the binary path if it's not in PATH:
```bash
find ~ /Library/Frameworks -name "markitdown" 2>/dev/null | grep -v ".py$"
```

Store the full path -- you'll need it in the batch step.

---

## UX Flow

Run steps in order. Wait for user confirmation at each step before proceeding.

---

### Step 1 -- Channel

Ask:
> "Which YouTube channel? Paste the URL (e.g. `https://www.youtube.com/@ChannelName/videos`) or the @handle."

---

### Step 2 -- Keywords + Date Range

Ask:
> "What topics are you interested in? Give me keywords to match against video titles -- comma-separated (e.g. `Claude Code, AI agent, tutorial`)."
>
> "Date range? Default is last 6 months. You can type a number of months or a specific date (YYYY-MM-DD)."

**Run the extraction and filter:**

```bash
# Step 2a: dump full video list
yt-dlp --flat-playlist -j "<channel_url>" > /tmp/yt_videos.jsonl
echo "Total videos: $(wc -l < /tmp/yt_videos.jsonl)"
```

```python
# Step 2b: keyword filter
import json

KEYWORDS = ["keyword1", "keyword2"]  # fill from user input

urls = []
with open("/tmp/yt_videos.jsonl") as f:
    for line in f:
        d = json.loads(line.strip())
        title = d.get("title") or ""
        if any(kw.lower() in title.lower() for kw in KEYWORDS):
            urls.append(d.get("webpage_url", ""))

with open("/tmp/yt_filtered_urls.txt", "w") as f:
    for u in urls: f.write(u + "\n")

print(f"{len(urls)} keyword-matched videos")
```

```bash
# Step 2c: batch-verify actual upload dates (5 parallel)
# WARNING: flat-playlist timestamps are unreliable -- always run this step
cat /tmp/yt_filtered_urls.txt | xargs -I{} -P5 \
  yt-dlp --skip-download --print "%(upload_date)s\t%(webpage_url)s\t%(title)s" "{}" 2>/dev/null \
  | sort > /tmp/yt_dated.tsv
```

```python
# Step 2d: apply date cutoff
import datetime

# Calculate cutoff from user input (e.g. 6 months = subtract 180 days from today)
CUTOFF = datetime.datetime(2025, 12, 12)  # replace with calculated date

final = []
with open("/tmp/yt_dated.tsv") as f:
    for line in f:
        parts = line.strip().split("\t")
        if len(parts) < 3: continue
        date_str, url, title = parts[0], parts[1], parts[2]
        try:
            d = datetime.datetime.strptime(date_str, "%Y%m%d")
            if d >= CUTOFF:
                final.append((date_str, url, title))
        except: continue

final.sort(key=lambda x: x[0], reverse=True)
print(f"\n{len(final)} videos within date range:\n")
for date_str, url, title in final:
    fmt = f"{date_str[:4]}-{date_str[4:6]}-{date_str[6:]}"
    print(f"[{fmt}] {title}\n  {url}\n")
```

**Show the list to the user. Ask:**
> "Found N videos. Proceed with all, or are there any you want to remove?"

Wait for confirmation before Step 3.

---

### Step 3 -- Output folder

Ask:
> "Where should the files be saved? Paste an absolute path or a path relative to your home folder (e.g. `~/Documents/Research/ChannelName/`)."

```bash
mkdir -p "<user_path>"
```

---

### Step 4 -- Context folder (optional)

Ask:
> "Do you have a folder with notes about how you're currently using these tools or topics? If yes, I'll compare the transcripts against it after saving and highlight only what's genuinely new for you. (Optional -- press Enter to skip.)"

If the user provides a path: note it. Run the delta comparison after Step 5 completes (see **Delta Comparison** below).

---

### Step 5 -- Rate limit strategy

Explain, then ask:

> "YouTube rate-limits transcript requests after roughly 20 in a row from the same IP. Two options:"
>
> **Option A -- Safe mode** (recommended for first run)
> 5-second pause between each request. Reliable up to ~50 videos. No setup needed.
> Estimated time: approximately N × 7 seconds total.
>
> **Option B -- Cookie mode**
> Uses your logged-in YouTube session cookies. Fast, no rate limit, works for any batch size.
> Requires a one-time setup step. Type `setup cookies` for instructions.
>
> "Which would you like? (A / B)"

**Option B setup instructions** (show if requested):
1. Install the Chrome extension **"Get cookies.txt LOCALLY"**
2. Go to `youtube.com` while logged into your Google account
3. Click the extension → Export → save the file as `~/youtube-cookies.txt`
4. Pass the path to the batch script (see below)

---

## Batch execution

Run after all 5 steps are confirmed. Fill in the variables at the top.

```python
import subprocess, re, os, time

# ---- configure these ----
DEST = "/path/to/your/output/folder"          # from Step 3
MARKITDOWN = "markitdown"                      # or full path if not in PATH
CHANNEL_NAME = "ChannelName"                   # human-readable label
SLEEP_SECONDS = 5                              # Option A: 5; Option B: 0
COOKIES = None                                 # Option B: "/path/to/youtube-cookies.txt"
VIDEOS = []                                    # confirmed list from Step 2: [(date_str, url, title), ...]
# -------------------------

def slugify(s):
    import re
    s = s.lower()
    s = re.sub(r'[^a-z0-9\s-]', '', s)
    s = re.sub(r'\s+', '-', s.strip())
    return re.sub(r'-+', '-', s)[:60].rstrip('-')

os.makedirs(DEST, exist_ok=True)
ok, failed, skipped = [], [], []

for date_str, url, title in VIDEOS:
    date_fmt = f"{date_str[:4]}-{date_str[4:6]}-{date_str[6:]}"
    filename = f"{date_fmt}-{slugify(title)}.md"
    filepath = os.path.join(DEST, filename)

    if os.path.exists(filepath):
        print(f"SKIP (exists): {filename}")
        skipped.append(filename)
        continue

    print(f"Fetching: {title[:60]}...", flush=True)

    cmd = [MARKITDOWN, url]
    if COOKIES:
        cmd = [MARKITDOWN, "--cookies", os.path.expanduser(COOKIES), url]

    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=60)
        content = result.stdout.strip()

        # detect IP block or empty transcript
        if "Could not retrieve a transcript" in content or len(content) < 500:
            raise ValueError("transcript unavailable or IP blocked")

        frontmatter = f"""---
title: "{title}"
date: {date_fmt}
url: "{url}"
source: "{CHANNEL_NAME}"
type: video-transcript
transcript_available: true
---

"""
        with open(filepath, 'w') as f:
            f.write(frontmatter + content)

        print(f"  -> saved ({len(content):,} chars)")
        ok.append(filename)

    except Exception as e:
        print(f"  -> FAILED: {e}")
        stub = f"""---
title: "{title}"
date: {date_fmt}
url: "{url}"
source: "{CHANNEL_NAME}"
type: video-transcript
transcript_available: false
retry_note: "Re-run: {MARKITDOWN} \\"{url}\\""
---

*Transcript unavailable. Re-fetch:*
```
{MARKITDOWN} "{url}"
```
"""
        with open(filepath, 'w') as f:
            f.write(stub)
        failed.append((filename, str(e)))

    if SLEEP_SECONDS > 0:
        time.sleep(SLEEP_SECONDS)

print(f"\nDone: {len(ok)} saved  |  {len(failed)} failed  |  {len(skipped)} skipped")
if failed:
    print("\nFailed (retry after IP cooldown or switch to Option B):")
    for fn, err in failed:
        print(f"  {fn}")
```

---

## Delta comparison (Step 4 follow-up)

If the user provided a context folder in Step 4, run this after the batch completes.

1. Read a sample of files from the context folder (up to ~8,000 tokens total)
2. Read all newly saved transcripts
3. Write a single summary file `_delta-summary.md` in the output folder

```markdown
---
type: delta-summary
generated: <today's date>
context_folder: <path user provided>
transcripts_compared: N
---

# Delta Summary -- What's New For You

## High-delta (worth reading in full)
- [video title](filename.md) -- [one sentence: what's genuinely new vs your existing knowledge]

## Low-delta (you already cover this)
- [video title] -- [brief note on why it overlaps]

## Not checked (no transcript)
- [video title]
```

---

## Error handling reference

| Error | Cause | Fix |
|---|---|---|
| `Could not retrieve a transcript` | YouTube IP rate limit hit | Wait a few hours and retry, or switch to Option B (cookies) |
| File saved but < 500 chars | Members-only or transcript-disabled video | Stub written with `transcript_available: false` |
| `yt-dlp: command not found` | Not installed or not in PATH | See Pre-flight check above |
| `markitdown: command not found` | Not installed or not in PATH | See Pre-flight check above |
| All flat-playlist dates show "unknown" | yt-dlp limitation on this endpoint | Always use the Step 2c batch date-verify command -- never trust flat-playlist timestamps |

---

## Quick reference -- rate limit expectations

| Strategy | Requests before block | Speed | Setup needed |
|---|---|---|---|
| No delay (default) | ~15-20 | Fast | None |
| Option A: 5s sleep | ~50+ | ~7s per video | None |
| Option B: cookies | Effectively unlimited | ~3s per video | One-time Chrome export |

---

## Installation

**Recommended -- one command** (installs to all your agents and supports updates):

```bash
npx skills add breakthrough-edu/youtube-transcript-ingestion
```

Update later with `npx skills update youtube-transcript-ingestion`.

**Manual** (Claude Code only, no extra tooling):

```bash
mkdir -p ~/.claude/skills/youtube-transcript-ingestion
# copy this SKILL.md into that folder
cp SKILL.md ~/.claude/skills/youtube-transcript-ingestion/SKILL.md
```

Either way, invoke in any Claude Code session by saying "grab YouTube transcripts" -- or let the model auto-load it when your request matches.

**Requirements:** `yt-dlp` and `markitdown` (see Pre-flight check above for install commands).
