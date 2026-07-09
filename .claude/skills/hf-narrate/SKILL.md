---
name: hf-narrate
description: Generate or revise narration for a HyperFrames composition the ASTGL way — script rules (TTS traps, standalone sign-off), local Kokoro TTS with am_michael, ffmpeg silence padding, ffprobe duration measurement, and re-cueing the composition to the audio. Use whenever narration/voiceover is created, edited, re-recorded, or when audio sounds clipped or mis-synced.
---

# hf-narrate — audio-first narration pipeline

Narration drives timing in this project, never the reverse. The pipeline is:
script → TTS → pad → measure → wire → re-cue → check. Every step below exists
because skipping it produced a real defect in a shipped render.

Work from inside the composition directory (`compositions/<id>/`). The CLI
treats a missing input path as literal text to speak — running from the wrong
cwd produces audio that says "narration dot t x t."

## Step 0 — Preflight (cold environments only)

```bash
python3 -c "import kokoro_onnx, soundfile" 2>/dev/null || pip3 install kokoro-onnx soundfile
```

`hyperframes tts` shells out to Python; these aren't bundled with the npm
package. First-ever run also downloads the Kokoro model (~few hundred MB) —
tell James before incurring that the first time.

## Step 1 — Write or revise `narration.txt`

Script rules (each one is a James correction — do not relax them):

1. **One beat per visual event.** Each sentence/paragraph should map to a
   timeline cue (a node appearing, a section changing). Note the intended cue
   next to each beat while drafting; you'll need the map in Step 5.
2. **Standalone sign-off.** The final line is its own complete sentence,
   never attachable to the previous one. Good: "Another walkthrough from As
   The Geek Learns." Bad: ending the content sentence and appending "As The
   Geek Learns" — it reads as part of the preceding sentence.
3. **No TTS traps:**
   - Never a pronounceable "astgl" — Kokoro says "asccul." Use `A-S-T-G-L`
     (letters read individually) or drop the URL from speech.
   - Write numbers, versions, and acronyms as they should be *spoken*
     ("two hundred fifty-six gigabytes", "C S R", "O C S P" — check each
     acronym: is it spoken as letters or as a word?).
   - URLs: "dot" spelled out ("substack dot com") only if the URL must be
     spoken at all; prefer naming the publication instead.
4. **Length sanity:** Kokoro speaks ≈ 2.4–2.6 words/second at speed 1.0.
   Target duration ÷ 2.5 ≈ word budget. State the estimate before generating.

## Step 2 — Generate

```bash
npx --yes hyperframes@0.6.64 tts narration.txt -o narration-raw.wav -v am_michael
```

Voice is `am_michael`, speed 1.0 (James chose it after listening; slowdowns
were rejected — do not change voice or speed without producing samples for
him to play; give him the file *paths*, don't describe the audio).

## Step 3 — Pad (mandatory, both ends)

Kokoro clips the first syllable and the render can clip the last one. Pad:

```bash
ffmpeg -y -i narration-raw.wav -af "adelay=500|500,apad=pad_dur=0.7" narration.wav
rm narration-raw.wav
```

- 500ms lead silence — absorbs start-of-audio clipping.
- 700ms tail — protects the final word (the word "Learns" was clipped in
  three consecutive takes before this rule existed).

## Step 4 — Measure, never assume

```bash
ffprobe -v error -show_entries format=duration -of csv=p=0 narration.wav
```

Sanity-check: duration should ≈ (word count ÷ 2.5) + 1.2s of padding. If it's
wildly short (~1–2s), the CLI probably spoke a filename — Step 0/cwd problem.

If possible, also verify the ends are intact (extract and inspect, or ask
James to listen — and say explicitly if you couldn't listen yourself):

```bash
ffmpeg -v error -y -i narration.wav -ss 0 -t 1.5 /tmp/head.wav -sseof -2 /tmp/tail.wav
```

## Step 5 — Wire and re-cue the composition

In `index.html` AND `index-portrait.html` (parity rule):

1. `<audio>` element: `class="clip"`, `data-start="0"`,
   `data-duration="<measured>"`, own `data-track-index` (highest track, audio
   never shares a lane), `src="narration.wav"`, `data-volume="1.0"`.
2. Root `data-duration` = measured audio duration + ~1.5s visual hold.
3. **Re-pace every timeline cue** against the new audio using the beat map
   from Step 1. Remember: the 500ms pad shifts every cue by +0.5s relative to
   an unpadded take. If only the script's tail changed, only tail cues move —
   but verify, don't assume.

## Step 6 — Gate

```bash
npm run check
```

Zero errors before any render. Then hand off to `/hf-render-pack` for
deliverables — this skill's job ends at a clean check, not at an MP4.

## Definition of done

- [ ] Script passes all four rules in Step 1
- [ ] `narration.wav` padded both ends; raw file removed
- [ ] ffprobe duration recorded in your summary and wired into both HTML files
- [ ] All cues re-paced; landscape and portrait updated together
- [ ] `npm run check` clean
- [ ] Ends verified intact (or explicitly flagged for James to listen)
