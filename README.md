# Postparty Outro Trimmer

A tiny single-page web app that removes the Postparty outro animation from the end of a clip, and (optionally) tags the file and/or adds a small Postparty watermark so the creators still get a little credit.

Everything runs **in your browser**. Your clip is never uploaded anywhere. The only network request the page makes is a one-time download of the video encoder (see [How it works](#how-it-works)), which your browser then caches.

**Live demo:** https://codymkw.github.io/postparty-trimmer

> Unofficial fan tool. Not affiliated with or endorsed by Postparty or Epic Games.

---

## What it does

1. **Drop in a clip** (MP4 recommended).
2. The page **scans the end of the clip** and finds the exact frame where the Postparty outro card starts.
3. It shows you the **last frame kept** and the **first frame removed** so you can check the cut. You can nudge the cut point if it's off.
4. Click **Trim & export**. You get a new MP4 with the outro gone.

Optional extras, both in the Export section:

| Option | Default | What it does |
| --- | --- | --- |
| "Clipped in Postparty" tag | On | Writes invisible metadata into the MP4 saying the clip was clipped in Postparty. Doesn't change the picture or sound. |
| Postparty logo watermark | Off | Burns a small, semi-transparent Postparty logo into the bottom-right corner. |
| Quality | Balanced | Re-encode quality (see [Quality presets](#quality-presets)). |

---

## Samples

The same clip, three ways. The files live in the [`videos/`](videos) folder.

| 1. Original | 2. Outro removed | 3. Outro removed + logo |
| :---: | :---: | :---: |
| <video src="videos/clip_with_outro.mp4" controls muted width="280"></video> | <video src="videos/clip_without_outro.mp4" controls muted width="280"></video> | <video src="videos/clip_with_logo_bottom_right.mp4" controls muted width="280"></video> |
| [`clip_with_outro.mp4`](videos/clip_with_outro.mp4) | [`clip_without_outro.mp4`](videos/clip_without_outro.mp4) | [`clip_with_logo_bottom_right.mp4`](videos/clip_with_logo_bottom_right.mp4) |
| The clip as exported from Postparty, with the outro animation at the end. | The outro is gone and the clip ends on the last frame of gameplay. | Same as #2, plus the optional semi-transparent Postparty logo in the bottom-right corner. |

If the players above don't show up (GitHub doesn't always render video tags for files in the repo), click the file names to open each video.

---

## The watermark (totally optional)

The watermark is **completely optional**. It's off by default, and you don't have to use it. The outro is gone either way.

But if you'd still like to give credit to Postparty (since the outro was their little advertisement), you can tick **Add the Postparty logo as a small watermark**, and a semi-transparent logo goes in the bottom-right corner of your clip. That way the creators still get a bit of exposure, lol.

Details:

- The logo is drawn **see-through** (the logo itself is transparent, not just its background), so you can see the video through it.
- **Opacity** dropdown: Subtle (35%), Normal (55%, default), Bold (80%).
- Size is **13% of the video width**, placed in the bottom-right with a margin of 2% of the video height.
- A faint dark shadow is added so the white logo stays readable on bright scenes.
- The logo is a white WebP with a transparent background, embedded in `index.html` as a base64 data URI (`LOGO_DATA_URI`). To swap it, replace that string.
- The Postparty logo belongs to Postparty. It's included here only so people can credit them.

Unlike the tag, the watermark is part of the picture, so it survives re-uploads to Discord, YouTube, etc.

---

## Using it

### Online
Use the live demo at https://codymkw.github.io/postparty-trimmer, or host `index.html` yourself anywhere that serves static files (GitHub Pages, Nekoweb, Cloudflare Pages, etc.). There is nothing to build and no server code.

To host it on GitHub Pages: put `index.html` in the repo root, then go to **Settings → Pages** and publish from the `main` branch (root folder).

### Locally
Serve the folder over HTTP instead of opening the file directly:

```bash
python3 -m http.server 8000
# then open http://localhost:8000/
```

No special headers are needed (no COOP/COEP), because the page uses the single-threaded encoder.

### Requirements
- A modern desktop browser. It was built and tested in Chromium (Chrome/Edge). Firefox and Safari should work but haven't been tested, and neither has mobile.
- The browser must be able to **play** the clip, because detection and the preview use the browser's own decoder. H.264 MP4 is the safe choice.
- An internet connection the first time, to download the encoder (about 31 MB). After that it's cached.

---

## Repo layout

```
.
├── index.html   # the entire app: HTML + CSS + JS, logo embedded
├── README.md
└── videos/      # sample clips shown in the README (not used by the app)
    ├── clip_with_outro.mp4
    ├── clip_without_outro.mp4
    └── clip_with_logo_bottom_right.mp4
```

---

## How it works

### 1. Finding the outro

The Postparty outro is a card with a flat, near-black, neutral background (about `RGB(18,18,18)` in the clip it was built from), and the gameplay before it never looks like that. The detector uses that:

1. Seek a hidden `<video>` to the last frame and draw it to a small 96×54 canvas.
2. Take four **8×5 corner patches** (one per corner) and average each.
3. If the last frame isn't **flat dark** (every corner has max channel ≤ 60 and channel spread ≤ 14), report "no outro found" and let you pick the cut manually.
4. Otherwise, walk **backwards in 0.25 s steps** while the corners still match the last frame (each channel within ±16).
5. When a step stops matching, **binary-search** between the last matching and first non-matching time (10 iterations, about 0.25 ms precision) to find the first outro frame.
6. The cut is placed **5 ms before** that frame so the last gameplay frame is kept and the first outro frame is dropped.

On the sample clip this found the outro at 29.450 s, which is the true first outro frame.

Tunable constants are at the top of the script and the detection section:

| Constant | Value | Meaning |
| --- | --- | --- |
| `SCAN_LOOKBACK_S` | `30` | How far back from the end to look for the outro. If the outro card runs longer than this, detection gives up and you set the cut by hand. |
| `SCAN_STEP_S` | `0.25` | Coarse step size while walking backwards. |
| `CUT_SAFETY_S` | `0.005` | How far before the first outro frame to cut. |
| `MARKER_KEY` | `postparty_clipped` | Metadata key written to, and searched for in, files. |
| `MARKER_COMMENT` | `Clipped in Postparty (outro removed)` | Human-readable Comment tag. |
| `CDN.core` | `@ffmpeg/core@0.12.10` on jsDelivr | Where the encoder is downloaded from. |

If the outro changes (different background colour or a new design), the thresholds in `flatDark()` and `matches()` are what to adjust. The manual slider always works as a fallback.

### 2. Cutting and encoding

The page uses the **ffmpeg.wasm core** (`@ffmpeg/core`), which is real ffmpeg compiled to WebAssembly. It is loaded through a small custom Web Worker created from a blob, rather than through `@ffmpeg/ffmpeg`, because that package's own worker can't be started from a CDN.

The command it runs looks like this (watermark and tag parts only present when enabled):

```bash
ffmpeg -i in.mp4 \
       -i wm.png \                                   # only with watermark
       -t <cut> \
       -filter_complex "[0:v:0][1:v]overlay=x=W-w-M:y=H-h-M:format=auto,format=yuv420p[v]" \   # only with watermark
       -map "[v]" -map 0:a? \                        # without watermark: -map 0:v:0 -map 0:a?
       -map_metadata 0 \
       -c:v libx264 -preset veryfast -crf <16|21|26> -pix_fmt yuv420p \
       -c:a copy \                                   # falls back to: -c:a aac -b:a 192k
       -movflags +faststart+use_metadata_tags \
       -metadata postparty_clipped=true \            # tag options, only if enabled
       -metadata postparty_trimmed_at=<cut> \
       -metadata postparty_original_duration=<duration> \
       -metadata "comment=Clipped in Postparty (outro removed)" \
       out.mp4
```

Flag notes:

- `-t <cut>`: keep only the first `<cut>` seconds.
- `-map 0:a?`: include audio if there is any.
- `-map_metadata 0`: keep the original file's metadata.
- `-c:a copy`: audio is copied untouched. If the source audio can't be put into an MP4, the page retries with AAC at 192 kbps.
- `-movflags +faststart`: puts the index at the start of the file so it plays/streams quickly.
- `-movflags use_metadata_tags`: lets ffmpeg write custom keys such as `postparty_clipped` (without it MP4 only keeps a few standard keys).
- `M` in the overlay is 2% of the video height.

**Why the video is re-encoded instead of stream-copied:** a lossless `-c copy` cut only works cleanly if the cut lands on a keyframe boundary. In the sample clip the encoder didn't put a keyframe at the outro boundary, and the first outro frame is stored *ahead of* the last gameplay frames. Copying left outro frames in the file. Re-encoding gives a frame-exact cut every time.

### Quality presets

| Preset | CRF | Notes |
| --- | --- | --- |
| Near-lossless | 16 | Biggest file. |
| Balanced (default) | 21 | On the sample clip, the 29 s output was about 10.4 MB, a bit smaller than the 11.3 MB, 34 s original. |
| Smaller file | 26 | Smallest file, visibly softer. |

Lower CRF means higher quality and a larger file.

### 3. The watermark image

The logo is decoded by the browser, drawn on a canvas at the chosen opacity (with a soft shadow), exported as a PNG, written into ffmpeg's in-memory filesystem as `wm.png`, and overlaid with ffmpeg's `overlay` filter. This avoids needing any font files inside ffmpeg.

---

## The "clipped in Postparty" tag

When enabled, these tags are written into the MP4:

| Key | Example | Meaning |
| --- | --- | --- |
| `postparty_clipped` | `true` | The marker. |
| `postparty_trimmed_at` | `29.445` | Where the clip was cut, in seconds. |
| `postparty_original_duration` | `34.441` | Length of the clip before trimming, in seconds. |
| `comment` | `Clipped in Postparty (outro removed)` | Shows up in places like Windows file Details. |

### Checking a file for the tag

```bash
ffprobe -v error -show_entries format_tags -of default=nw=1 clip-trimmed.mp4
```

You'll see `TAG:postparty_clipped=true` and the others. The page also shows the tags after export, and when you drop a file that already has the marker it tells you so (it looks for the text `postparty_clipped` in the first and last 4 MB of the file).

### What the tag is and isn't

- It **doesn't** change the picture or sound.
- It **doesn't survive** most platforms. Discord, YouTube, Twitter/X and others re-encode uploads and drop custom tags, so it only lasts if people share the raw file.
- It's **not proof** of anything. Anyone can write the same tag onto any video. (Making it verifiable would need something like a signed hash, which this project doesn't do.)

If you want credit that does survive re-uploads, use the watermark.

---

## Limitations and troubleshooting

- **Speed:** encoding is single-threaded WebAssembly, so it's slower than desktop ffmpeg. In headless Chromium, a ~29 s 896×504 clip took roughly 40 to 55 seconds. Longer or higher-resolution clips take proportionally longer.
- **Memory:** the whole clip is loaded into memory to process it. Clips of a few hundred MB or less work best.
- **"Your browser can't play this file":** detection and preview rely on the browser's decoder. Try an H.264 MP4 (HEVC/H.265 often won't play in browsers).
- **"Couldn't find an outro":** the last frame didn't look like the outro card, or the card runs longer than `SCAN_LOOKBACK_S`. Drag the slider, use "Set to player position", or adjust the thresholds above.
- **Cut is a frame off:** use the −0.1 s / +0.1 s buttons or type an exact time, and check the two preview frames.
- **Export fails or the encoder won't load:** check your connection (the encoder comes from jsDelivr), disable extensions that block blob workers, and look at the browser console. A failed load can be retried by clicking export again.
- **Rotated phone videos:** rotation metadata isn't handled specially. This tool is aimed at landscape game clips.

---

## Credits

- Video processing by [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm) (`@ffmpeg/core`), which bundles ffmpeg and x264 under their own licenses.
- The Postparty name and logo belong to Postparty. This project isn't affiliated with them, it just tries to keep some credit flowing their way.