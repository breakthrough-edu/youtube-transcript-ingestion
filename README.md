# YouTube Transcript Ingestion

A Claude Code skill that batch-grabs transcripts from any YouTube channel and saves them as clean, readable markdown files -- so you can research a creator's body of work, build a corpus, or filter content overload without watching every video.

## Install

```bash
npx skills add breakthrough-edu/youtube-transcript-ingestion
```

That's it. Update later with:

```bash
npx skills update youtube-transcript-ingestion
```

> Manual install (Claude Code only): copy `SKILL.md` into `~/.claude/skills/youtube-transcript-ingestion/`.

## Requirements

Two command-line tools:

```bash
brew install yt-dlp                              # video metadata (no download)
pip install 'markitdown[youtube-transcription]'  # transcript extraction (keep the quotes)
```

> Note: a bare `pip install markitdown` will *not* fetch YouTube transcripts -- the `[youtube-transcription]` extra is required.

The skill runs a pre-flight check and tells you exactly what to install if anything is missing.

## How to use

In any Claude Code session, just say:

> grab YouTube transcripts

The skill then walks you through five steps:

1. **Channel** -- paste a channel URL or `@handle`
2. **Filter** -- keywords to match in titles + a date range
3. **Output folder** -- where the `.md` files land
4. **Context folder** *(optional)* -- point it at your existing notes, and it surfaces only what's genuinely new to you
5. **Rate-limit strategy** -- safe mode (sleep) or cookie mode (fast)

It confirms the matched video list with you before fetching anything, then saves one markdown file per video with clean YAML frontmatter (title, date, URL, source).

## Why it exists

Learning from a fast-moving channel usually means drowning in hours of video. This pulls the text, filters to the topics you care about, and -- if you give it your notes -- tells you which videos are actually new signal versus stuff you already know.

## License

MIT -- see [LICENSE](LICENSE). Use it, fork it, share it.
