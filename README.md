# claude-xhs-skill

A Claude Code skill that fetches and digests **Xiaohongshu (小红书 / RedNote)** posts — no login required.

## What it does

- Extracts post title, author, and body text
- Downloads all images to `research/media/<title>/`
- Extracts video stream URLs (H.264 / H.265) for video posts
- Works with share links, direct URLs, and XHS share text

## How it works

XHS embeds full note data in `window.__INITIAL_STATE__` in the page HTML, accessible via a browser-like User-Agent without authentication. The same technique used by the [Obsidian XHS Importer](https://github.com/bnchiang96/xiaohongshu-importer) plugin.

## Install

### As a Claude Code plugin

```
/plugin install https://github.com/oldmercy/claude-xhs-skill
```

### Manual (global skill)

Copy `skills/digest-xhs/SKILL.md` to `~/.claude/skills/digest-xhs/SKILL.md`.

## Usage

Paste any XHS share link or share text into your Claude Code session and type:

```
/digest-xhs https://www.xiaohongshu.com/discovery/item/<id>?xsec_token=...
```

Or with share text:

```
/digest-xhs 【2016年经济学诺奖解释了agent为什么犯错 - ody | 小红书】... https://www.xiaohongshu.com/...
```

## Output

```
## Post Title

**Author**: ody
**Source**: https://www.xiaohongshu.com/...
**Type**: image (3 photos)
**Tags**: #AI #大模型 #Agent

---

Post body text without hashtags...

---

**Media**
- Images (3): downloaded to research/media/Post-Title/ → 01.jpg, 02.jpg, 03.jpg
- Video: N/A
```

## Supported URL formats

- `https://www.xiaohongshu.com/discovery/item/<id>`
- `https://www.xiaohongshu.com/explore/<id>`
- `https://xhslink.com/<code>`
- Share text containing any of the above

## Limitations

- Text and images only for public share links; private posts return incomplete data
- CDN image URLs may expire — download immediately after fetching
- Video download (saving .mp4 locally) is not yet implemented; video URL is provided for manual download

## Platform

- **Windows**: PowerShell (`Invoke-WebRequest`)
- **Mac/Linux**: Bash (`curl` + `python3`)

## License

MIT
