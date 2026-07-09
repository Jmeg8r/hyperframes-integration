---
name: hf-new-video
description: Scaffold a new HyperFrames composition in this project following every house convention — pinned package.json, palette declaration, track table, static end-state layout BEFORE animation, portrait sibling, and the quality gates. Use when starting any new video, explainer, diagram animation, or promo composition from a concept, article, or diagram.
---

# hf-new-video — new composition, the twenty-minute way

The first composition took half a day; the Nth should take twenty minutes.
This skill is the template that makes that true. It front-loads the two
decisions that change everything (content beats, visual direction) and makes
the mechanical rest deterministic.

Invoke `/hyperframes` (vendor skill) before writing any composition HTML —
this skill encodes the *project workflow*; the vendor skill encodes the
*framework semantics*. You need both.

## Step 1 — Brief (interview, one question at a time, only what's undecided)

Resolve these before any code; each answer changes the architecture:

1. **Content:** what's the source (article, diagram, concept)? What are the
   beats — the 5–9 things that must appear on screen, in order?
2. **Duration target:** promo (~20–35s) or walkthrough (longer)? If narrated,
   audio length will win (audio-first rule) — the target here just sizes the
   script's word budget.
3. **Primary aspect:** which of 16:9 / 9:16 is the design master? (Both get
   built; one drives layout.)
4. **Narrated?** If yes, the narration script gets written in Step 2 and
   timing follows the audio via `/hf-narrate`.
5. **Visual direction:** pick a palette category from
   `.agents/skills/hyperframes/house-style.md` (or derive OKLCH bg/fg/accent).
   If James hasn't specified and ≥2 directions are defensible, build 2–3
   static end-state variants and let him react — aesthetics are
   know-it-when-he-sees-it; never pick a subjective winner silently.

## Step 2 — Scaffold

```bash
mkdir -p compositions/<id> && cd compositions/<id>
```

`meta.json`:
```json
{ "id": "<id>", "name": "<id>", "createdAt": "<ISO timestamp>" }
```

`package.json` — copy from `compositions/pki-lifecycle/package.json` verbatim
(it pins `npx --yes hyperframes@0.6.64` in every script; the pin is a project
rule, never float the version).

`hyperframes.json` — copy from an existing composition (schema, registry, paths).

If narrated: write `narration.txt` now, following the script rules in
`/hf-narrate` Step 1 (standalone sign-off, no "astgl" as a word, spoken-form
numbers, ~2.5 words/sec budget). Generate and measure the audio *before*
building the timeline — cues get paced to the measured duration.

## Step 3 — Static end-state FIRST (the load-bearing rule)

Write `index.html` with **zero animation**: every element positioned where it
sits at its most-visible moment. This is the frame that must be correct;
animating a layout you haven't seen static hides overlaps until after a
burned render.

Requirements for this file:

- Head: pinned GSAP CDN (`gsap@3.14.2`), viewport meta matching the master
  aspect, universal reset, palette as CSS custom properties or documented
  hex values — bg/fg/accent declared once at the top, not invented per element.
- Root: `<div id="root" data-composition-id="main" data-start="0"
  data-duration="<target>" data-width="..." data-height="...">`.
- **Track table comment** before the first timed element, e.g.:
  ```html
  <!-- Tracks: 1=title 2=subtitle 3=diagram 4=outro 5=narration(audio) -->
  ```
  Every concurrently-visible clip gets its own track (tracks are scheduling
  lanes, not z-layers). Audio always on its own highest track.
- Every timed element: `class="clip"` + `data-start` + `data-duration` +
  `data-track-index`. All three or none.
- Positioning: `left`/`top` + negative margins for centering. `transform` in
  CSS only on containers GSAP will never touch (e.g. a static `scale()` to
  fit a diagram).
- Fonts: one embeddable face per role (`"Inter", sans-serif`). No
  `-apple-system` / `BlinkMacSystemFont` / system chains — headless Chrome
  has no system fonts.
- Background layer per house-style: 2–5 accent-tinted decoratives so scenes
  aren't empty during entrances (they get slow ambient motion in Step 4).
- Deterministic only: no `Date.now()`, `Math.random()`, or network fetches.

**Checkpoint:** `npm run check` on the static file, then preview it
(`npm run dev` with `run_in_background: true`) or `snapshot`. Layout problems
get fixed HERE. If visual direction was open in Step 1, this is the variant
James reacts to.

## Step 4 — Animate

Add the timeline in a `<script>` at the end of `<body>`:

```js
window.__timelines = window.__timelines || {};
const tl = gsap.timeline({ paused: true });   // paused — the framework seeks
// entrances: 0.3–0.6s, varied eases, combined transforms, overlapped starts
// ambient: slow breath/drift on decoratives
// cues paced to narration beats (+0.5s lead-pad offset) if narrated
window.__timelines["main"] = tl;
```

Use `.from()` tweens so the end state stays the CSS truth. Comment each cue
with the narration beat or content event it's paced to — those comments are
what make re-cueing possible later.

If narrated: wire the `<audio>` element and durations per `/hf-narrate` Step 5.

## Step 5 — Portrait sibling

Copy `index.html` → `index-portrait.html`, then change: viewport meta and
`data-width`/`data-height` (1080×1920), layout positions for the narrow
canvas, and typically a `transform: scale(~0.6–0.7)` on the main content
container rather than a re-layout. From now on the two files are parity-bound:
every content/timing change lands in both in the same session.

## Step 6 — Gate and hand off

```bash
npm run check     # zero errors, warnings each fixed or explained
```

Deliverables (`renders/`, both aspects, versioned names) are
`/hf-render-pack`'s job — invoke it rather than running render ad hoc.

## Definition of done

- [ ] Brief answered (or variants presented) before any HTML was written
- [ ] package.json/hyperframes.json copied with the 0.6.64 pin intact
- [ ] Static end-state existed and was checked/previewed before the first tween
- [ ] Track table comment matches actual track usage; audio on its own track
- [ ] Palette declared once; no lazy-default AI design tells without justification
- [ ] Timeline paused + registered; cues commented with their beats
- [ ] Portrait sibling built and parity-checked
- [ ] `npm run check` clean
- [ ] Friction points noted with `[ASTGL CONTENT]` — new-composition sessions
      are exactly the material James publishes
