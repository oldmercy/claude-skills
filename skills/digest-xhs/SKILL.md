---
name: digest-xhs
description: Fetch and digest a Xiaohongshu (小红书 / RedNote) post from a URL or share text. Extracts title, author, body text, downloads images, and provides video URLs. No login required. Invoke when the user shares a xiaohongshu.com link or XHS share text containing a URL.
tools: PowerShell, Bash, Write
---

# Digest XHS (小红书) Post

Fetch a Xiaohongshu post without authentication and extract all content: text, images, and video.

## How it works

XHS embeds full note data in `window.__INITIAL_STATE__` in the page HTML, accessible without login via a browser-like User-Agent. Images and videos are served from XHS CDN and downloadable directly.

---

## Step 1: Extract URL

From user input, extract the canonical XHS URL. Input may be:
- Direct URL: `https://www.xiaohongshu.com/discovery/item/<id>?...`
- Explore URL: `https://www.xiaohongshu.com/explore/<id>`
- Short link: `https://xhslink.com/<code>`
- Share text: `【title - author | 小红书】... https://www.xiaohongshu.com/...`

Normalize `explore/` paths to `discovery/item/` if needed. Keep all query params (especially `xsec_token`) — they are required.

---

## Step 2: Fetch and Parse

### Windows (PowerShell)

```powershell
$url = "<EXTRACTED_URL>"
$headers = @{
    "User-Agent"      = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
    "Accept-Language" = "zh-CN,zh;q=0.9,en;q=0.8"
    "Referer"         = "https://www.xiaohongshu.com/"
}
$html = (Invoke-WebRequest -Uri $url -Headers $headers -UseBasicParsing).Content

# Parse window.__INITIAL_STATE__ — replace JS `undefined` with JSON null before parsing
$html -match 'window\.__INITIAL_STATE__=([\s\S]+?)</script>' | Out-Null
$state  = ($Matches[1] -replace '\bundefined\b', 'null') | ConvertFrom-Json
$noteId = ($state.note.noteDetailMap.PSObject.Properties.Name)[0]
$note   = $state.note.noteDetailMap.$noteId.note

# Title — strip " - 小红书" suffix
$title = if ($html -match '<title>([^<]+)</title>') {
    $Matches[1] -replace '\s*-\s*小红书.*', ''
} else { $note.title }

# Body — use div extraction first (more reliable than note.desc from JSON)
$body = ''
if ($html -match '<div id="detail-desc"[^>]*>([\s\S]*?)</div>') {
    $body = $Matches[1] -replace '<[^>]+>', '' -replace '<!--[\s\S]*?-->', '' -replace '\s+', ' '
    $body = $body.Trim()
}
if (-not $body -and $note.desc) { $body = $note.desc }

# Separate clean text from hashtags
$cleanBody = ($body -replace '#\S+', '' -replace '\s+', ' ').Trim()
$tags      = [regex]::Matches($body, '#(\S+)') | ForEach-Object { "#$($_.Groups[1].Value)" }

# Author and type
$author = $note.user.nickname
$type   = $note.type   # "normal" (image post) or "video"

# Image URLs — note.imageList[].urlDefault is often empty string in server-rendered HTML;
# use regex on raw JSON to capture urlDefault values from infoList and other structures,
# then decode Unicode-escaped slashes (/ → /)
$imageUrls = @(
    [regex]::Matches($rawJson, '"urlDefault"\s*:\s*"([^"]+)"') |
        ForEach-Object { $_.Groups[1].Value -replace '\\u002F', '/' } |
        Where-Object { $_ -match '^https?://' }
)

# Video URL — prefer H.264, fall back to H.265
$videoUrl = $null
if ($type -eq 'video' -and $note.video) {
    $streams = $note.video.media.stream
    if ($streams.h264 -and $streams.h264.Count -gt 0) {
        $videoUrl = $streams.h264[0].masterUrl
    } elseif ($streams.h265 -and $streams.h265.Count -gt 0) {
        $videoUrl = $streams.h265[0].masterUrl
    }
}
```

### Mac / Linux (Bash fallback)

```bash
URL="<EXTRACTED_URL>"
HTML=$(curl -s -L \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 Chrome/124.0.0.0 Safari/537.36" \
  -H "Accept-Language: zh-CN,zh;q=0.9" \
  -H "Referer: https://www.xiaohongshu.com/" \
  "$URL")

# Extract body text from detail-desc div
BODY=$(echo "$HTML" | python3 -c "
import sys, re
html = sys.stdin.read()
m = re.search(r'<div id=\"detail-desc\"[^>]*>([\s\S]*?)</div>', html)
if m:
    text = re.sub(r'<[^>]+>', '', m.group(1))
    text = re.sub(r'<!--[\s\S]*?-->', '', text)
    print(text.strip())
")
echo "$BODY"
```

---

## Step 3: Download Images

Save images to `research/media/<safe-title>/` under the current working directory.
Name files `01.jpg`, `02.jpg`, etc. Create the directory if it does not exist.

```powershell
$safeTitle = ($title -replace '[\\/:*?"<>|]', '-').Trim()
$mediaDir  = Join-Path (Get-Location) "research\media\$safeTitle"
New-Item -ItemType Directory -Force -Path $mediaDir | Out-Null

$localPaths = @()
for ($i = 0; $i -lt $imageUrls.Count; $i++) {
    $filename = '{0:D2}.jpg' -f ($i + 1)
    $dest     = Join-Path $mediaDir $filename
    try {
        Invoke-WebRequest -Uri $imageUrls[$i] -OutFile $dest -UseBasicParsing
        $localPaths += "research/media/$safeTitle/$filename"
        Write-Output "  Downloaded: $filename"
    } catch {
        Write-Output "  Failed [$filename]: $_"
    }
}
```

If imageUrls is empty and the post is not a video, note that the note may be text-only or
the images may require session authentication (less common for public share URLs).

---

## Step 4: Output Format

After fetching and downloading, output a clean markdown digest:

```
## <title>

**Author**: <author>
**Source**: <url>
**Type**: image (<N> photos) | video | text
**Tags**: <#tag1> <#tag2> ...

---

<clean body text — no hashtags>

---

**Media**
- Images (<N>): downloaded to `research/media/<title>/` → 01.jpg, 02.jpg, ...
- Video: <videoUrl | N/A>
```

If images failed to download, list the CDN URLs directly so the user can access them manually.

---

## Notes

- `xsec_token` in the URL is required for newer share links — do not strip it.
- CDN image URLs may expire after several hours; download immediately.
- Video notes (`type === "video"`) have no imageList; only extract videoUrl.
- This skill fetches publicly shared notes only. Private or login-gated notes will return incomplete data.
