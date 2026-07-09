---
name: hf-render-pack
description: Produce the full deliverable set for a HyperFrames composition — gated by npm run check, both aspect ratios (with the safe portrait swap dance), versioned filenames that never overwrite, and post-render verification via snapshots. Use whenever rendering final MP4s, re-rendering after changes, or producing the landscape+portrait pair.
---

# hf-render-pack — checked, versioned, dual-aspect deliverables

Every finished video ships as a pair: 1920×1080 (blog embed) and 1080×1920
(Substack Notes / mobile feeds). Rendering is minutes of compute — treat it
like a deploy: everything cheap runs first, nothing gets overwritten, and the
output is verified before it's called done.

Work from inside `compositions/<id>/`.

## Step 1 — Gate (no exceptions)

```bash
npm run check        # lint + validate + inspect — must be CLEAN
```

If check isn't clean after a genuine fix attempt: STOP and report the
findings. Never render over a dirty check. Also confirm portrait parity —
`index.html` and `index-portrait.html` must reflect the same content/timing
state (`diff` them; the differences should be exactly viewport dimensions,
scale factors, and layout positions).

## Step 2 — Pre-render visual spot-check (cheap)

```bash
npx --yes hyperframes@0.6.64 snapshot
```

Inspect the key frames: title readable, no text overlapping the title block
(this exact bug shipped twice), outro intact. A snapshot costs seconds; a
render costs minutes.

## Step 3 — Choose output names (never overwrite)

Convention: `renders/<id>-<W>x<H>[-narrated][-vN].mp4`

- `-narrated` if the composition has a narration audio track.
- Version: look at what exists. First cut has no suffix; any subsequent cut
  of the same variant gets `-v2`, `-v3`, … Existing MP4s are immutable —
  James compares takes. If `renders/` doesn't exist, create it (older
  compositions have flat MP4s in the project root; leave those where they
  are, put new output in `renders/`).

```bash
mkdir -p renders && ls -1 renders/ *.mp4 2>/dev/null   # decide -vN from this
```

## Step 4 — Render landscape

```bash
npm run render -- -o renders/<id>-1920x1080[-narrated][-vN].mp4
```

## Step 5 — Render portrait (the swap dance, done safely)

`--resolution` only scales — it cannot change aspect ratio. The portrait file
must temporarily *become* the root composition. Make the swap crash-safe:

```bash
cp index.html index-landscape.bak.html \
  && cp index-portrait.html index.html \
  && npm run render -- -o renders/<id>-1080x1920[-narrated][-vN].mp4 ; \
mv index-landscape.bak.html index.html
```

The trailing `mv` restores the landscape master even if the render fails —
never leave the swap in place. Verify afterwards:

```bash
grep -o 'data-width="[0-9]*"' index.html    # must be 1920 again
```

## Step 6 — Verify the output (rendering ≠ done)

For each MP4:

```bash
ffprobe -v error -show_entries format=duration -of csv=p=0 <file>   # matches root data-duration?
ffprobe -v error -select_streams a -show_entries stream=codec_name -of csv=p=0 <file>  # audio present if -narrated
```

Then extract spot frames and *look at them* (start, one mid-phase, outro):

```bash
ffmpeg -v error -y -ss 1 -i <file> -frames:v 1 /tmp/f-start.png \
  -ss <mid> -i <file> -frames:v 1 /tmp/f-mid.png \
  -sseof -2 -i <file> -frames:v 1 /tmp/f-end.png
```

Checking: no text overlap, title/outro readable, portrait layout not clipped
at the edges. If anything looks off, fix the composition and bump `-vN` — do
not re-render into the same filename.

## Step 7 — Report

Summarize with the concrete numbers James tracks:

- Files produced (paths + sizes)
- Wall-clock render time per file and effective ×-real-time (baseline on the
  Mac Studio: ~1.5× landscape, ~1.7× portrait, 6 workers — flag deviations)
- Check/inspect stats (elements validated, samples inspected)
- Anything you noticed for `[ASTGL CONTENT]`

`npm run publish` (public URL upload) is NEVER part of this skill — it
requires James's explicit go-ahead each time.

## Definition of done

- [ ] `npm run check` was clean immediately before rendering
- [ ] Both aspect ratios rendered with convention-correct, non-clobbering names
- [ ] Landscape master restored and verified after the portrait swap
- [ ] Durations and audio streams ffprobe-verified
- [ ] Spot frames extracted and visually checked
- [ ] Render stats reported
