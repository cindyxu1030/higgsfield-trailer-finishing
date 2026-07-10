# Trailer finishing — command recipes

Copy-paste mechanics for each stage. POSIX shell + ffmpeg; paths and clip names are examples — swap in the real ones.

---

## Stage 0 — Fetch clips generated earlier (optional)

Generation itself belongs to the `higgsfield-generate` skill. To pull down clips from previous jobs:

```bash
higgsfield generate list --json                 # find the job ids
higgsfield generate get <job_id> --json         # → result URL in the job object
curl -sS -o clip-01-opening.mp4 "<result_url>"
```

Name downloads descriptively (`clip-NN-slug.mp4`) — the edit order should be readable from the filenames.

---

## Stage 1 — Stitch

Set the frame once — everything downstream inherits it:

```bash
W=1920; H=1080   # 16:9 landscape · use W=1080; H=1920 for 9:16 vertical
```

Normalize each clip identically, then concat (prevents SAR/fps mismatch failures). `-an` strips any clip audio. When Stage 2 (grade) follows, keep intermediates near-lossless (`-crf 10`) — the master gets its final CRF at the grade encode. Skipping the grade? Use `-crf 16` here instead:

```bash
for f in clip1.mp4 clip2.mp4 clip3.mp4; do
  ffmpeg -hide_banner -loglevel error -y -i "$f" \
    -vf "scale=$W:$H:force_original_aspect_ratio=decrease,pad=$W:$H:(ow-iw)/2:(oh-ih)/2,setsar=1,fps=30" \
    -c:v libx264 -crf 10 -preset slow -pix_fmt yuv420p -an "norm_$f"
done
printf "file 'norm_clip1.mp4'\nfile 'norm_clip2.mp4'\nfile 'norm_clip3.mp4'\n" > list.txt
ffmpeg -hide_banner -y -f concat -safe 0 -i list.txt -c copy stitched.mp4
```

Trim a beat during normalization (no extra encode) — add `-ss <in> … -t <len>` to that clip's normalize command:

```bash
ffmpeg -hide_banner -loglevel error -y -ss 0.4 -i clip2.mp4 -t 2 \
  -vf "scale=$W:$H:force_original_aspect_ratio=decrease,pad=$W:$H:(ow-iw)/2:(oh-ih)/2,setsar=1,fps=30" \
  -c:v libx264 -crf 10 -preset slow -pix_fmt yuv420p -an norm_clip2.mp4
```

Cross-dissolves (0.5s) instead of hard cuts — run on the NORMALIZED clips, re-encode at intermediate quality (never ffmpeg defaults). `xfade` offsets are **cumulative**: `offset_k = d1+…+dk − k×0.5` (durations of all prior clips minus all prior fades):

```bash
# two clips
ffmpeg -y -i norm_clip1.mp4 -i norm_clip2.mp4 -filter_complex \
 "[0:v][1:v]xfade=transition=fade:duration=0.5:offset=<d1-0.5>,format=yuv420p" \
 -c:v libx264 -crf 10 -preset slow stitched.mp4
# three clips — chain pairwise with cumulative offsets
ffmpeg -y -i norm_clip1.mp4 -i norm_clip2.mp4 -i norm_clip3.mp4 -filter_complex \
 "[0:v][1:v]xfade=transition=fade:duration=0.5:offset=<d1-0.5>[v01]; \
  [v01][2:v]xfade=transition=fade:duration=0.5:offset=<d1+d2-1.0>,format=yuv420p" \
 -c:v libx264 -crf 10 -preset slow stitched.mp4
```

Fade-to-black ending: don't spend an extra encode — append `,fade=t=out:st=<dur-0.6>:d=0.6` to the Stage-2 full-render filter (not per-clip, or every clip fades out). If you're skipping the grade, run one final encode with just the fade as the filter.

---

## Stage 2 — Grade (optional)

The orange-teal default chain — single source of truth (SKILL.md only describes it; tune it here). `curves` does the split-tone (red up in highlights = amber; blue up in shadows / down in highlights = teal); `noise=…:allf=t` is *temporal* grain:

```bash
GRADE="eq=contrast=1.12:saturation=1.14:gamma=0.97:brightness=-0.01,curves=r='0/0 0.5/0.53 1/1':g='0/0.015 0.5/0.5 1/0.985':b='0/0.06 0.5/0.47 1/0.88',vignette=PI/4.5,unsharp=5:5:0.5:5:5:0.0,noise=alls=5:allf=t"
```

Test on stills first — 3 frames spread across DIFFERENT shots, not one:

```bash
for T in 1 6 12; do   # pick times that land in different scenes of the cut
  ffmpeg -y -ss $T -i stitched.mp4 -frames:v 1 "f$T.png"
  ffmpeg -y -i "f$T.png" -vf "$GRADE" "f${T}_graded.png"
done
# LOOK at all three, tune, repeat
```

Full render once the look is dialed:

```bash
ffmpeg -hide_banner -loglevel error -y -i stitched.mp4 -vf "$GRADE" \
  -c:v libx264 -crf 16 -preset slow -pix_fmt yuv420p -tune film -an graded.mp4
```

Tuning: warmer footage → lower the red-highlight lift (`0.53`→`0.51`) so it doesn't clip; too dark → raise `gamma` toward 1.0 / `brightness` toward 0; more grit → `noise=alls=8`, `vignette=PI/4`.

Other looks: **horror/dread** → `saturation=0.85`, drop the red-highlight lift, push shadows green-cyan (raise the green shadow point instead of blue), heavier vignette. **Soft/brand** → keep just the `eq` contrast lift; drop curves, vignette, and noise entirely.

---

## Stage 3 — Add music matched to the cut (Sonilo)

**Make a lightweight proxy.** Sonilo only needs to see the motion and timing; a small copy uploads faster and matches identically. `scale=1280:720` for landscape, `scale=720:1280` for 9:16 vertical:

```bash
ffmpeg -hide_banner -loglevel error -y -i graded.mp4 \
  -vf "scale=1280:720" -c:v libx264 -crf 30 -preset veryfast -pix_fmt yuv420p -an proxy.mp4
```

**Call Sonilo** (MCP tool; needs the user's `SONILO_API_KEY` in the server registration — see SKILL.md Prerequisites). The returned file is matched to the video's length:

```
video_to_music({
  video_path: "path/to/proxy.mp4",
  prompt: "warm hopeful cinematic, building to an uplifting swell"   // optional single style hint for the whole track
})
// → an .m4a soundtrack matched to the video's length
```

Platforms can use the Sonilo REST API instead: async job + webhook → `preview_url`, `final_audio_url`, and a per-track `license_id` (an auditable license record). Full docs at [sonilo.com](https://sonilo.com).

**Want an alternate take?** Call `video_to_music` again — each run is a fresh interpretation. Deliver two and let the user pick.

**In-catalog fallback** (no Sonilo key; a fixed-length bed generated from the prompt, not from the video — it won't follow the edit):

```bash
higgsfield generate create sonilo_music --prompt "cinematic instrumental, slow build" --duration 15 --wait
higgsfield generate create seed_audio --prompt "cinematic rain ambience with distant thunder" --wait
```

---

## Stage 4 — Mux onto the master

Confirm the soundtrack duration matches the cut, then lay it over the FULL-RESOLUTION master with a short tail fade, video untouched (`-c:v copy` keeps the grade byte-identical):

```bash
MUS="soundtrack.m4a"
DUR=$(ffprobe -v error -show_entries format=duration -of default=nk=1:nw=1 "$MUS")   # check ≈ video duration
FADE=$(awk "BEGIN{printf \"%.2f\", $DUR-0.5}")
ffmpeg -hide_banner -loglevel error -y -i graded.mp4 -i "$MUS" \
  -filter_complex "[1:a]afade=t=out:st=${FADE}:d=0.5[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 320k -shortest trailer_final_v1.mp4
```

If the soundtrack runs slightly long (e.g. the in-catalog fallback), `-shortest` plus the tail fade handles it; if it's off by much more than ~0.5s, regenerate or trim the audio first.

---

## Delivery — compress under a size cap

Single-pass CRF, keep the soundtrack stream as-is:

```bash
ffmpeg -hide_banner -loglevel error -y -i trailer_final_v1.mp4 \
  -c:v libx264 -crf 24 -preset slow -pix_fmt yuv420p -c:a copy trailer_final_v1_web.mp4
du -h trailer_final_v1_web.mp4
```

Lower CRF = higher quality + bigger file; CRF 24 is a good first try for a short 1080p cut. Grained footage has a bitrate floor and compresses non-linearly — bitrate-targeted (ABR/two-pass) encodes can't force it under an arbitrary cap. Hit caps with CRF: encode once, measure, raise CRF ~2, repeat.
