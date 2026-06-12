---
name: youtube-transcript-ingestion
description: Batch-grab transcripts from any YouTube channel and save them as readable markdown files with clean frontmatter. Guided 5-step flow -- channel, keyword + date filter, output folder, optional context-delta comparison, rate-limit strategy -- using yt-dlp for metadata and markitdown for transcripts. Optionally compares results against your existing notes to surface only what is genuinely new. Trigger phrases -- grab YouTube transcripts, ingest YouTube channel, yt-dlp transcripts, markitdown YouTube, research a creator's videos, build a corpus from a channel, filter content overload from a channel.
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
4. Fetches full transcripts for each matched video (markitdown by default; yt-dlp + browser cookies when rate-limited)
5. Saves each as a `.md` file with clean YAML frontmatter into a folder of the user's choice
6. Optionally compares transcripts against an existing notes/context folder to surface only what's genuinely new

---

## Pre-flight check

Before starting, verify the two required tools. Run this:

```bash
echo "yt-dlp:     $(which yt-dlp 2>/dev/null || echo NOT FOUND)"
echo "markitdown: $(which markitdown 2>/dev/null || echo NOT FOUND)"
# markitdown ships YouTube support as an OPTIONAL extra -- a bare install
# passes the check above but cannot fetch transcripts. Verify it for real:
python3 -c "import youtube_transcript_api" 2>/dev/null \
  && echo "youtube support: OK" \
  || echo "youtube support: MISSING (install the extra below)"
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

**If markitdown is missing (or has no YouTube support):**
```bash
# IMPORTANT: a bare "pip install markitdown" does NOT include YouTube support.
# You must install the youtube-transcription extra -- keep the quotes:
pip install 'markitdown[youtube-transcription]'
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

> "YouTube throttles transcript requests after roughly 15-20 in a row from the same IP (you'll see `HTTP 429` / 'Sign in to confirm you're not a bot'). Two ways to handle it:"
>
> **Option A -- Pacing** (recommended for a first, small run)
> A 5-second pause between requests. Reliable up to ~50 videos, no setup. Uses markitdown.
> Estimated time: roughly N × 7 seconds total.
>
> **Option B -- Browser cookies** (for large batches, or if you're already blocked)
> Reads your logged-in YouTube session straight from your browser via `yt-dlp --cookies-from-browser`. Bypasses the IP limit entirely -- no extension and no manual cookie export. Just tell me which browser you're signed into YouTube on.
>
> "Which would you like? (A / B) -- and if B, which browser are you logged into YouTube on? (chrome / safari / firefox / edge / brave)"

Two things to make clear to the user:
- If they are **already** rate-limited, Option A will not help -- every anonymous request still returns 429. Option B is the only path that works once a block is active.
- Option B uses **yt-dlp, not markitdown**, because the markitdown CLI has no cookie support. See **Understanding YouTube rate limits** below.

---

## Batch execution

Run after all 5 steps are confirmed. Fill in the variables at the top.

```python
import subprocess, re, os, time, json, glob, tempfile

# ---- configure these ----
DEST = "/path/to/your/output/folder"          # from Step 3
MARKITDOWN = "markitdown"                      # or full path if not in PATH
CHANNEL_NAME = "ChannelName"                   # human-readable label
VIDEOS = []                                    # confirmed list from Step 2: [(date_str, url, title), ...]

# Strategy (from Step 5):
USE_COOKIES = False     # Option A: False (markitdown + pacing)  |  Option B: True (yt-dlp + browser cookies)
BROWSER = "chrome"      # Option B only: chrome | safari | firefox | edge | brave
SLEEP_SECONDS = 5       # Option A: 5  |  Option B: 1 (cookies bypass the limit; a small pause still avoids bursts)
# -------------------------

def slugify(s):
    s = s.lower()
    s = re.sub(r'[^a-z0-9\s-]', '', s)
    s = re.sub(r'\s+', '-', s.strip())
    return re.sub(r'-+', '-', s)[:60].rstrip('-')

def fetch_markitdown(url):
    """Option A: markitdown CLI. Clean output, but no cookie support -- dies on an IP block."""
    r = subprocess.run([MARKITDOWN, url], capture_output=True, text=True, timeout=60)
    content = (r.stdout or "").strip()
    if "Could not retrieve a transcript" in content or "429" in (r.stderr or "") or len(content) < 500:
        raise ValueError("transcript unavailable or IP blocked (429)")
    return content

def fetch_ytdlp_cookies(url):
    """Option B: yt-dlp pulls the caption track directly, authenticated via your browser
    cookies. Survives an active IP block. Parses YouTube's json3 caption format."""
    vid = url.split("v=")[-1].split("&")[0]
    tmp = tempfile.mkdtemp()
    subprocess.run([
        "yt-dlp", "--cookies-from-browser", BROWSER, "--skip-download",
        "--ignore-no-formats-error",   # new yt-dlp aborts on missing video formats w/o a JS runtime; captions don't need them
        "--write-auto-subs", "--write-subs", "--sub-langs", "en.*", "--sub-format", "json3",
        "-o", os.path.join(tmp, "%(id)s.%(ext)s"), url,
    ], capture_output=True, text=True, timeout=120)
    files = sorted(glob.glob(os.path.join(tmp, f"{vid}*.json3")))
    if not files:
        raise ValueError("no caption track (video has none, or browser not logged into YouTube)")
    data = json.load(open(files[0]))
    parts = []
    for e in data.get("events", []):
        segs = e.get("segs")
        if not segs:
            continue
        t = "".join(s.get("utf8", "") for s in segs)
        if t.strip():
            parts.append(t.strip())
    text = re.sub(r"\s+", " ", " ".join(parts)).strip()
    if len(text) < 500:
        raise ValueError("transcript too short / empty")
    return text

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
    try:
        content = fetch_ytdlp_cookies(url) if USE_COOKIES else fetch_markitdown(url)

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
retry_note: "Failed during batch. If IP-blocked, re-run the batch with USE_COOKIES = True (Option B)."
---

*Transcript unavailable (IP block or no caption track). Re-fetch by re-running the batch in Option B / cookie mode.*
"""
        with open(filepath, 'w') as f:
            f.write(stub)
        failed.append((filename, str(e)))

    if SLEEP_SECONDS > 0:
        time.sleep(SLEEP_SECONDS)

print(f"\nDone: {len(ok)} saved  |  {len(failed)} failed  |  {len(skipped)} skipped")
if failed:
    print("\nFailed -- if these were IP-blocked, re-run with USE_COOKIES = True (Option B):")
    for fn, err in failed:
        print(f"  {fn}  ({err})")
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
| `Could not retrieve a transcript` / `HTTP 429` / `Sign in to confirm you're not a bot` | YouTube IP rate limit -- anonymous requests blocked | Switch to **Option B (browser cookies)**; it works even while blocked. Waiting it out is unreliable (a block can last a day+). |
| File saved but < 500 chars | Members-only or transcript-disabled video | Stub written with `transcript_available: false` |
| `yt-dlp: command not found` | Not installed or not in PATH | See Pre-flight check above |
| `markitdown: command not found` | Not installed or not in PATH | See Pre-flight check above |
| yt-dlp: `Requested format is not available` / `Only images are available` | New yt-dlp needs a JS runtime to resolve video formats | Harmless for transcripts -- the batch already passes `--ignore-no-formats-error`, and captions download regardless |
| All flat-playlist dates show "unknown" | yt-dlp limitation on this endpoint | Always use the Step 2c batch date-verify command -- never trust flat-playlist timestamps |

---

## Understanding YouTube rate limits

YouTube throttles **anonymous** transcript/caption requests from a single IP. Knowing how the block behaves saves a lot of confusion:

- **Trigger:** roughly 15-20 requests in a row with no authentication.
- **Symptom:** `HTTP 429: Too Many Requests` (often served as a `google.com/sorry` page) and, in yt-dlp, `Sign in to confirm you're not a bot`.
- **Stickiness:** the block does NOT clear in a few minutes. It commonly lasts hours, sometimes a day or more -- waiting is unreliable.
- **Key consequence:** once blocked, pacing (Option A) no longer helps; every anonymous request still returns 429. Only authentication gets through.

**How the two strategies map to this:**

| Strategy | Tool | Requests before block | Speed | Setup |
|---|---|---|---|---|
| No delay (don't) | markitdown | ~15-20 | fast | none |
| **Option A -- pacing** | markitdown | ~50+ | ~7s / video | none |
| **Option B -- browser cookies** | yt-dlp | effectively unlimited, and works even while already blocked | ~3s / video | none beyond being logged into YouTube in a browser |

**Why Option B uses yt-dlp, not markitdown:** the markitdown CLI has no cookie option, so it cannot authenticate. yt-dlp can read your browser session directly with `--cookies-from-browser <browser>` (no extension, no manual export) and pull the caption track even when the IP is blocked. That is why the cookie path is a separate code branch in the batch script above -- it fetches YouTube's `json3` caption file and parses it to plain text.

**On `--cookies-from-browser`:** yt-dlp reads the cookie database of the named browser (`chrome`, `safari`, `firefox`, `edge`, `brave`). On macOS it may request Keychain access the first time. If yt-dlp reports the cookie DB is locked, close the browser and retry.

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
