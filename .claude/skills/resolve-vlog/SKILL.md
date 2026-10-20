---
name: resolve-vlog
description: Turn raw vlog footage into a DaVinci Resolve-ready project — check and normalize the clips (variable frame rate, mixed formats), cut dead air with auto-editor, export an FCPXML timeline Resolve can open, make SRT captions that line up with the cut, and give Resolve import, finishing and render steps (optionally scripted through Resolve's Python API). Use when someone wants to edit a vlog, talking-head video or day-in-the-life recording in DaVinci Resolve, or asks for a "Resolve rough cut", "jump-cut vlog", or "vlog timeline for Resolve".
---

# Vlog → DaVinci Resolve

Do the mechanical part outside Resolve (inspect, normalize, rough-cut, captions)
and hand over a timeline the user only has to polish. Needs `ffmpeg`/`ffprobe`
and `auto-editor`. For auto-editor flag details, see the **auto-editor**,
**auto-editor-export** and **auto-editor-transcribe** skills.

Before you start, ask only for what you can't work out yourself:
- the target: horizontal YouTube (16:9) or vertical Shorts/Reels/TikTok (9:16)
- the pace: relaxed (keeps breaths) or tight (jump cuts)
- whether they want captions, and in which language
- free DaVinci Resolve or Studio (Studio has GPU H.265 encoding and AI subtitles)

## 1. Inspect the footage

```bash
for f in footage/*; do
  echo "== $f"
  ffprobe -v error -select_streams v:0 \
    -show_entries stream=codec_name,width,height,r_frame_rate,avg_frame_rate,pix_fmt \
    -show_entries format=duration -of default=nw=1 "$f"
  ffprobe -v error -select_streams a -show_entries stream=index,codec_name,channels \
    -of csv=p=0 "$f"
done
```

Things to look for:
- **Variable frame rate (VFR):** if `r_frame_rate` ≠ `avg_frame_rate`, the clip
  is probably VFR. That's common with phones and screen recorders. In Resolve,
  VFR causes audio drift and frames that slip after the cut, so normalize it (step 2).
- **Mixed frame rates or resolutions:** pick one timeline frame rate (usually
  the main camera's: 23.976/25/29.97/30) and make everything match it.
- **Several audio streams** (camera mic + lav/wireless): note the stream index of
  the clean mic. Use that stream to detect silence (step 3).
- **10-bit H.265 / HEVC from phones:** free Resolve on Linux can't decode it, and
  it plays back slowly everywhere. Transcode it.

## 2. Normalize (only when needed)

Convert to a constant-frame-rate edit codec. DNxHR HQ works in free Resolve on
every OS, and ProRes is fine on macOS:

```bash
mkdir -p normalized
ffmpeg -i in.mp4 -vf fps=30 -fps_mode cfr \
  -c:v dnxhd -profile:v dnxhr_hq -pix_fmt yuv422p \
  -c:a pcm_s16le normalized/in.mov
```

Set `fps=` to the timeline rate you chose. If the source is already CFR in a
codec Resolve can read, skip this step: re-encoding costs disk space and time.

**Several takes → one timeline:** concatenate them in shooting order first, so
auto-editor produces a single timeline. Stream copy only works when every file
has identical codecs and settings (usually true after normalizing):

```bash
ls normalized/*.mov | sort | sed "s/.*/file '&'/" > list.txt
ffmpeg -f concat -safe 0 -i list.txt -c copy vlog_full.mov
```

Alternatively, export one timeline per clip and lay them out in Resolve.

## 3. Rough cut with auto-editor

Always preview first. It prints what would be cut without rendering anything:

```bash
auto-editor vlog_full.mov --edit audio:threshold=0.04 --margin 0.2sec --preview
```

Pace presets for vlogs (adjust them after the preview):

| Feel | Flags |
|---|---|
| Relaxed (keeps breaths) | `--edit audio:-30dB --margin 0.35s,0.6sec` |
| Standard YouTube | `--edit audio:threshold=0.04 --margin 0.2s,0.4sec` |
| Tight jump cuts | `--edit audio:-22dB --margin 0.1s,0.2sec` |

- If background noise (traffic, wind, music) keeps everything "active", raise
  the threshold. If quiet speech is getting cut, lower it.
- To detect silence on the lav only, use `--edit audio:threshold=0.04,stream=1`.
- For B-roll sections with no talking that should stay, use
  `--edit "(or audio:0.04 motion:0.06)"`, or keep them by time with `--keep 12sec,40sec`.
- Remove a fixed intro or flubbed opening with `--cut 0,4sec`.

Export the timeline for Resolve. Keep the flags in a shell variable, because
step 4 has to reuse exactly the same ones:

```bash
CUT='--edit audio:threshold=0.04 --margin 0.2s,0.4sec'
auto-editor vlog_full.mov $CUT --export 'resolve:name="Vlog Rough Cut"' -o vlog_cut.fcpxml
```

If an older Resolve version rejects the FCPXML, try `--export resolve-fcp7`
(it writes `.xml`). The timeline points to the source media by absolute path,
so keep the media where it is, or relink it in Resolve.

## 4. Captions that match the cut

An SRT made from the source file has the wrong timings once the timeline is
cut. Instead, render an audio-only version of **the same cut** and transcribe
that. Its timing then matches the exported timeline:

```bash
auto-editor vlog_full.mov $CUT -o cut_audio.wav
auto-editor whisper cut_audio.wav ggml-medium.bin --format srt -o vlog_cut.srt
```

- Use a `.en` model for English-only vlogs. For other languages, use the
  multilingual model and set the language (see **auto-editor-transcribe**).
- For vertical videos, short caption lines read better. Resolve can re-split
  captions (Subtitle track → right-click → *Convert/Split* in recent versions),
  or you can post-process the SRT so no line is longer than about 32 characters.
- Resolve Studio users can skip this step and use *Timeline → Create Subtitles
  from Audio*.

## 5. In DaVinci Resolve

1. **New project.** In *File → Project Settings → Master Settings*, set the
   timeline frame rate to the one from step 2 **before** importing. It can't be
   changed once a timeline exists. Resolution: 3840×2160 or 1920×1080 for
   16:9; 1080×1920 for 9:16.
2. **File → Import → Timeline…** → `vlog_cut.fcpxml`. Leave
   "Automatically import source clips into media pool" checked. If clips show
   as offline, right-click them in the Media Pool → *Relink Selected Clips*.
3. **Captions:** import `vlog_cut.srt` into the Media Pool and drag it onto the
   timeline at 00:00. Resolve creates a subtitle track. Style it in the
   Inspector.
4. **Polish (Edit/Cut page):** go through every jump cut. Put B-roll over
   harsh ones, add a slow zoom (Inspector → Zoom keyframes, 100→110%), or a
   short cross-dissolve on the audio only.
5. **Audio (Fairlight):** in the dialogue track's Mixer, add *Dynamics* (light
   compression) and, if needed, *Voice Isolation* (Studio). Aim for about −14
   LUFS integrated for YouTube (*Fairlight → Loudness meter*). Put background
   music 18–24 dB under the voice, or duck it with sidechain compression.
6. **Color:** one node for exposure and white balance, then a look. If the
   footage is log (S-Log3, D-Log, Apple Log), add a Color Space Transform or the
   manufacturer's LUT first.
7. **Deliver:** see below.

### Render settings

| Target | Resolution | Codec | Notes |
|---|---|---|---|
| YouTube 16:9 | 3840×2160 or 1920×1080 | H.264 (H.265 on Studio), ~40–60 Mb/s for 4K, ~16 Mb/s for 1080p | Uploading 4K gets better YouTube compression, even when shot in 1080p |
| Shorts / Reels / TikTok | 1080×1920 | H.264, ~16 Mb/s | Keep under 60–90 s for Shorts; burn in captions (*Subtitle Settings → Burn into video*) |
| Archive master | timeline res | DNxHR HQ / ProRes 422 HQ | For re-exports later |

For **vertical from horizontal footage**, set the timeline to 1080×1920 and
reframe each shot with Inspector → Position (Studio users: *Smart Reframe*).
Or reframe with the **auto-editor-effects** skill before exporting.

## 6. Optional: drive Resolve with its scripting API

If Resolve runs on the same machine (scripting requires *Preferences → System
→ General → External scripting using: Local*), you can automate the import
and render. Paths for `DaVinciResolveScript` differ by OS. See *Help →
Documentation → Developer → Scripting/README.txt*.

```python
import DaVinciResolveScript as dvr

resolve = dvr.scriptapp("Resolve")
pm = resolve.GetProjectManager()
project = pm.CreateProject("Vlog") or pm.LoadProject("Vlog")
project.SetSetting("timelineFrameRate", "30")  # must happen before any timeline exists

pool = project.GetMediaPool()
timeline = pool.ImportTimelineFromFile(
    "/abs/path/vlog_cut.fcpxml",
    {"timelineName": "Vlog Rough Cut", "importSourceClips": True},
)
project.SetCurrentTimeline(timeline)

project.SetRenderSettings({
    "TargetDir": "/abs/path/renders",
    "CustomName": "vlog_final",
    "FormatWidth": 3840, "FormatHeight": 2160,
})
project.SetCurrentRenderFormatAndCodec("mp4", "H264")
project.AddRenderJob()
project.StartRendering()
```

Leave the creative passes (B-roll, color, music) to the user in the UI. Script
only the repetitive import and render steps.

## Checklist before handing over

- [ ] `--preview` output reviewed, and no words clipped at the start or end of cuts
- [ ] Same `$CUT` flags used for the FCPXML and the caption audio
- [ ] Timeline frame rate in Resolve matches the media
- [ ] Tell the user which files to open (`vlog_cut.fcpxml`, `vlog_cut.srt`) and
      that the source media must stay at its current path
