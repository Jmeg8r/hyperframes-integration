---
title: "I rebuilt a PKI lifecycle diagram as code-rendered video — here's what HyperFrames taught me"
subtitle: "Half a day, three real friction points, two MP4s. The workflow scales — and the lint catches are where the lessons live."
publication: ASTGL
draft: true
date: 2026-05-31
tags: [hyperframes, video-as-code, gsap, astgl-build, pki]
---

I have a PKI certificate lifecycle diagram I've drawn six different ways across six different jobs. Whiteboards, PowerPoint, Visio, Mermaid, draw.io, and at one point a hand-sketched napkin during a coffee meeting with our internal CA team. Same content every time. Same shape every time. Always static.

This weekend I rebuilt it as a 32-second animated video — and the file that produced it is a single HTML document.

The tool is [HyperFrames](https://github.com/heygen-com/hyperframes), HeyGen's open-source framework for turning HTML, CSS, and seekable GSAP animations into deterministic MP4 video. The pitch that hooked me: you write a video the same way you write a webpage, then an AI agent or a render command turns it into a frame-accurate clip. No timeline editor. No proprietary format. No React. Just HTML you preview in any browser.

I went in skeptical. I came out with two finished videos — one at 1920×1080 for blog embeds, one at 1080×1920 for Substack Notes and mobile feeds — and a clear sense of where this fits in my workflow.

## The setup took ten minutes

Installation is a single command:

```bash
npx skills add heygen-com/hyperframes
```

That registers fifteen skills with my agent: `/hyperframes` for the framework itself, plus adapters for GSAP, Three.js, Lottie, the Web Animations API, Tailwind, and a few migration helpers. It also installs the HyperFrames CLI globally for ad-hoc use.

The environment check passed clean on the first try:

```
hyperframes doctor

  ✓ Version          0.6.64 (latest)
  ✓ Node.js          v24.14.0 (darwin arm64)
  ✓ CPU              28 cores · Apple M3 Ultra @ 2400MHz
  ✓ Memory           256.0 GB total · 56.4 GB free
  ✓ FFmpeg           8.1.1
  ✓ FFprobe          8.1.1
  ✓ Chrome           cached
  ✓ Docker           running
  ◇ All checks passed
```

The "doctor" command alone is a tell. It checks Node, FFmpeg, Puppeteer's Chrome cache, Docker, and disk. The framework knows exactly what it depends on and inspects it before you ask. That's the kind of detail I read as a signal: someone thought about what an operator actually needs.

First smoke test was the bundled `warm-grain` example. Ten seconds of cream-paper textured intro, rendered in thirty seconds wall-clock. The output had stale GSAP warnings about a missing `#a-roll` target — a template artifact, harmless, but the first reminder that pre-1.0 software stays pre-1.0. Pin your versions and read the warnings.

## The real work: PKI Certificate Lifecycle in HTML

My plan was a circular diagram with seven phase nodes around the perimeter — Key Generation, CSR Submission, Validation, Issuance, Deployment, Active Use, Renewal — plus a Revocation branch breaking off the side. Title above. Outro below. About thirty seconds total, paced so each phase has time to read on screen.

The HyperFrames model is straightforward once you internalize it. The composition is one `index.html` with three things going on:

1. **HTML elements with timing attributes** — `data-start`, `data-duration`, `data-track-index` on every clip
2. **A GSAP timeline registered on `window.__timelines`** — paused, the framework drives playback
3. **A static "end-state" layout in CSS** — the most-visible frame, built first, before any animation

That third one is the rule I'd want printed and taped to my monitor.

### The "layout before animation" rule

The HyperFrames skill is explicit about this:

> Position every element where it should be at its most visible moment — the frame where it's fully entered, correctly placed, and not yet exiting. Write this as static HTML+CSS first. No GSAP yet.

The reason becomes obvious the moment you violate it. If you position elements at their *animated* start state — offscreen, scaled to zero, opacity zero — and tween them to where you think they'll land, you're guessing at the final layout. Overlaps stay invisible until you render. You catch them when you watch the MP4, and by then you've already burned the render time.

I built the static end state first. Seven phase nodes positioned with polar coordinates around a 720×720 diagram. Seven labels positioned outside the ring, each at a specific angle. A central hub with `PKI / Trust Chain` text. A Revocation badge off the lower left. Title and subtitle at the top, outro at the bottom. The whole composition rendered statically before I wrote a single tween.

Only then did I add the GSAP timeline:

```js
const tl = gsap.timeline({ paused: true });

tl.from("#title", { y: -40, opacity: 0, scale: 0.94, duration: 0.9, ease: "power3.out" }, 0.3);
tl.from("#subtitle", { y: -20, opacity: 0, duration: 0.7, ease: "power2.out" }, 0.7);

tl.from("#ring", { scale: 0.5, opacity: 0, duration: 1.1, ease: "expo.out" }, 2.0);
tl.from("#hub", { scale: 0.2, opacity: 0, duration: 0.9, ease: "back.out(1.7)" }, 2.4);

// 7 phase nodes + labels sequence at 2s intervals starting at 4s
tl.from("#node-1", { scale: 0, opacity: 0, duration: 0.5, ease: "back.out(2)" }, 4.0);
tl.from("#label-1", { opacity: 0, duration: 0.6, ease: "power2.out" }, 4.3);
// ... and so on through phase 7

window.__timelines["main"] = tl;
```

That's it. The framework does the rest — load the HTML in headless Chrome, seek to every frame, capture each one, pipe the sequence to FFmpeg, encode H.264, mux any audio, write the MP4.

## Where I learned the most: the three real lint catches

I ran `npx hyperframes lint` after my first draft. It came back with thirteen findings.

This was the most useful five minutes of the whole build.

### Catch #1: GSAP transform conflicts

I had positioned my phase labels with CSS transforms:

```css
#label-1 { left: 50%; top: 50%; transform: translate(-50%, -440px); }
```

Then I'd tried to animate them with `gsap.from("#label-1", { y: 20, opacity: 0 })`. The lint caught it instantly:

> GSAP will overwrite the full CSS transform, discarding any translateX(-50%) centering or CSS scale value.

This is the kind of bug you'd never catch by inspection. The CSS would look fine. The animation would *look* fine in preview because GSAP's "from" state would seem to work. But the final position would be wrong — GSAP would set its own transform value that wiped out my centering offset.

The fix: position elements with `left` and `top` plus negative margins for centering, leaving `transform` entirely owned by GSAP. It's a clean separation and once you internalize it, you don't go back.

### Catch #2: Track collisions

I had three text elements all on `data-track-index="1"` with overlapping time ranges. The lint flagged it:

> Track 1: clip ending at 32s overlaps with clip starting at 0s.

Tracks in HyperFrames aren't visual layers — those are still CSS `z-index`. Tracks are scheduling lanes. Two clips on the same track can't be visible at the same time. The fix is to put each concurrent clip on its own track index. Once I had title on track 1, subtitle on track 2, diagram on track 3, and outro on track 4, the collision was gone.

I'd never have caught this without the lint. The clips would have flickered or fought each other in the render and I'd have spent an hour bisecting which one.

### Catch #3: Font fallback warnings

My CSS had the usual web fallback chain:

```css
font-family: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
```

That's idiomatic on a website. It's wrong in a video composition. The renderer is headless Chrome, not a user's browser — it doesn't have access to the user's system fonts. The compiler tries to embed font files, sees `-apple-system` and `BlinkMacSystemFont`, and warns that it can't resolve them. Text would silently fall back to a generic sans-serif and the typography would be subtly wrong in the final MP4.

The fix is to use only the auto-resolved fonts and trust the compiler to embed them, or supply `.woff2` files in a `fonts/` directory and reference them explicitly. I dropped the fallbacks. Inter alone works fine.

After fixing all three, lint came back clean. Validate (which runs the composition in headless Chrome and audits WCAG contrast) reported sixty text elements passing WCAG AA. Inspect (which seeks through the timeline and looks for layout overflow at each sample) found zero issues across nine samples.

I rendered. Twenty-two seconds wall-clock for a thirty-two-second composition at 1920×1080, six parallel workers, 960 frames captured and encoded. The Mac Studio's M3 Ultra was running at about 1.5× real-time.

## The dual-resolution gotcha

The second deliverable was the same diagram rendered at 1080×1920 for Substack Notes. I assumed the framework would handle this with a flag.

It doesn't, quite. The `--resolution` flag exists, but it only scales — you can render a 1920×1080 composition at 3840×2160 (4K landscape) but not at 1080×1920 (portrait). The aspect ratio has to match the source. There's also a `--composition` flag for rendering specific files, but it only handles sub-compositions wrapped in `<template>` — not standalone alternates.

The practical workflow is: copy `index.html`, edit `data-width` and `data-height`, swap the file into the root temporarily, render, swap back. For my portrait version I also had to scale the 720×720 diagram down to fit horizontally — `transform: scale(0.68)` on the diagram container kept the layout proportional inside a narrower canvas.

This is the kind of thing I'd want to wrap in a small build script the second time I do it. For now it's manual. Eighteen seconds wall-clock for the portrait render — slightly faster than landscape because there are fewer total pixels.

Both videos are deterministic. Same input, same frames, every time. I rendered each twice as a sanity check; the MP4 hashes were identical.

## What this is good for

Three things became clear by the end of the build.

**One: this absolutely scales for technical explainers.** Every diagram I've ever drawn for a Citrix migration, an MCS-vs-PVS comparison, a Horizon-to-VDA upgrade, or a network segmentation rebuild can be done this way. The build cost is front-loaded — the first composition takes a few hours including learning. The second one I make from the same template will take twenty minutes. The Nth one approaches zero.

**Two: the agent integration is the actual point.** I built this one largely by hand to learn the framework. The next ones I'll describe to Claude Code in plain language and let it write the composition while I review. The HyperFrames skill is loaded — `/hyperframes`, `/gsap`, all the adapters — and the framework was designed from the start to be agent-driven. That's the leverage.

**Three: the lint is non-negotiable.** Every catch I described above would have been a wasted render cycle if I'd skipped the check step. On a thirty-second clip that's a minor inconvenience. On a five-minute walkthrough video that's twenty minutes of compute and you don't even know what's wrong yet. Run lint, validate, and inspect before every render. They're fast and they tell you what would have failed.

## What's next

This was the warm-start in a larger pilot. Next up I'm wiring HyperFrames into my existing `astgl-publish` Substack pipeline as a new media stage — the same way it already produces voiceover scripts, branded note graphics, and an article draft from a single session, it'll now produce a short promo video too. I'm running an A/B/C test on the three most recent ASTGL posts: three hook variants each, distributed via my existing Substack Notes scheduler, to see which framing converts best. That data will template every future post.

After that, the release-walkthrough pattern — using HyperFrames to generate animated changelog videos for my own apps on every tagged release, starting with substack-scheduler. A scheduler that schedules its own promo video on release. There's a joke in there about turtles all the way down.

Both are direct extensions of work I'm already doing. The framework slots into the existing pipelines instead of replacing them. That's the right shape for an integration.

For now though: two MP4s, half a day of work, three lint catches that taught me more than the documentation did. Worth it.

---

*HyperFrames is Apache 2.0 and lives at [github.com/heygen-com/hyperframes](https://github.com/heygen-com/hyperframes). The composition source for this PKI diagram is in [my hyperframes-integration repo](https://github.com/Jmeg8r/hyperframes-integration) — clone it, change the labels, render your own diagram.*

*Subscribe to [As The Geek Learns](https://astgl.substack.com) for more practical write-ups of AI-driven development from the inside.*
