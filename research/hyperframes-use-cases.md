# HyperFrames Use Cases

*A practical evaluation and use-case map · Prepared for James Cruce · May 31, 2026*

---

## Executive Summary

HyperFrames is HeyGen's open-source framework for turning HTML, CSS, and seekable animations into deterministic MP4 video, designed first and foremost to be driven by AI coding agents. You describe a video in plain language to Claude Code or Cursor, the agent writes an HTML composition, and the framework renders it to a reproducible video file — no timeline editor, no React, no proprietary format.

It is an unusually clean fit for the way you already work: you write HTML/CSS fluently, you live inside Claude Code and Cursor, you run a local-first AI stack on a Mac Studio that can absorb the render load, and your operating principle is "automate anything done more than twice." The framework's primary interface is an agent acting on natural-language intent, its output is deterministic and Git-versionable, and its dependencies (Node 22, FFmpeg, local TTS/Whisper/background-removal models) are things you already have or can run locally.

This document integrates three rounds of research — a direct read of the repository, a survey of how teams are using it in the wild, and a broad community/ecosystem scan — and maps the findings onto concrete use cases across your specific projects: ASTGL, Resist and Rise, the Income Investor Suite, Julie's reselling business, your desktop apps, and your local-AI/agent infrastructure.

**Bottom line:** This is worth a real pilot. The lowest-risk, highest-learning starting point is an automated release-walkthrough video for one of your repos — it mirrors a documented production pattern, exercises the full pipeline, and produces ASTGL content as a byproduct.

---

## What HyperFrames Is

HyperFrames treats every element in an HTML document as a video clip, with timing defined directly in markup via data attributes (`data-start`, `data-duration`, `data-track-index`, `data-volume`). A composition is a single `index.html` that plays as-is in the browser for preview and renders to MP4 (or MOV/WebM) on demand.

Key characteristics:

- **HTML-native.** Compositions are plain HTML files. No React requirement, no DSL, no build step.
- **Agent-first.** The CLI is non-interactive by default and built to be driven by scripts and agents. Installable "skills" teach agents the framework's specific patterns.
- **Deterministic.** The same input produces identical frames every time, which is what makes it safe for automated pipelines, CI, and regression testing.
- **Adapter-based animation.** A "Frame Adapter" pattern lets you bring GSAP, Anime.js, CSS keyframes, Lottie, Three.js, or the Web Animations API and have them seeked frame-accurately rather than played against the wall clock.
- **Open source.** Apache 2.0 — commercial use at any scale, no per-render fees, no seat caps. This is the explicit contrast with Remotion, which is source-available and charges above small-team thresholds.

It is published by HeyGen and was inspired by Remotion (both drive headless Chrome + FFmpeg). The core design difference is the authoring model: Remotion bets on React components; HyperFrames bets on plain HTML — partly on the reasoning that LLMs are saturated with HTML knowledge and only thinly trained on React + Remotion.

---

## How It Works

The render pipeline loads the HTML composition in headless Chrome (via Puppeteer), seeks the page to each frame's exact timestamp, captures the frame, and pipes the sequence into FFmpeg for encoding and audio mixing. Seeking — rather than letting the page play in real time — is what guarantees determinism.

The project is an ecosystem of packages rather than a monolith:

| Package | Role |
|---|---|
| `hyperframes` (CLI) | Create, preview, lint, and render compositions |
| `@hyperframes/core` | Types, parsers, generators, linter, runtime, frame adapters |
| `@hyperframes/engine` | Seekable page-to-video capture (Puppeteer + FFmpeg) |
| `@hyperframes/producer` | Full pipeline: capture + encode + audio mix |
| `@hyperframes/studio` | Browser-based composition editor UI |
| `@hyperframes/player` | Embeddable `<hyperframes-player>` web component |
| `@hyperframes/shader-transitions` | WebGL shader transitions |

Media preprocessing runs locally through dedicated skills: Kokoro for text-to-speech, Whisper for transcription/captions, and u2net for background removal. A catalog of 50+ installable blocks (social overlays, shader transitions, data charts, cinematic effects) is available via `npx hyperframes add <block>`. Rendering can run locally, in Docker, or distributed on AWS Lambda.

**Getting started (agent path):** `npx skills add heygen-com/hyperframes` installs the skills; in Claude Code they register as slash commands (`/hyperframes`, `/hyperframes-cli`, `/hyperframes-media`, plus adapter skills like `/gsap`, `/three`, `/lottie`). **Manual path:** `npx hyperframes init my-video` → `preview` → `render`.

---

## The Landscape: How Others Are Using It

### Production adopters

The repository's `ADOPTERS.md` lists named teams. The most instructive for a developer-operator is tldraw, which uses it inside their Git workflow rather than for marketing.

| Organization | How they use HyperFrames |
|---|---|
| **HeyGen** | Powers AI-generated video composition and rendering across their video product surface |
| **tldraw** | Automated pull-request walkthrough videos with GSAP-animated code diffs, narration, and captions |
| **TanStack** | Exploring short-form code demo videos and documentation |
| **OptinMonster** | Exploring marketing and product video content |

### Documented and community-discussed patterns

Across HeyGen's docs, help center, and independent write-ups, a consistent set of patterns recurs:

- **Avatar + HyperFrames compositing** — an AI-written script, a photorealistic HeyGen avatar speaking it, and HyperFrames rendering the dynamic HTML backgrounds and overlays behind the avatar.
- **Website / URL-to-video** — a structured seven-step pipeline (capture, design, script, storyboard, voiceover + timing, build, validate) that turns a live URL plus creative direction into a finished video.
- **Data / CSV / JSON to motion graphics** — animated bar-chart races and infographics rendered directly from data, including pointing the renderer at the live DOM elements feeding a website.
- **PDF / article / slides to video** — turning static documents and decks into narrated, animated presentations.
- **Social clips with synced captions** — vertical (9:16) hook videos with TTS narration and animated captions.
- **Effects-heavy launch reels** — combining Three.js, shaders, Lottie, and HTML-in-canvas in a single composition.
- **Prompt-to-video** — agents generating compositions from a topic or trend with no hand-authoring.
- **Agent-driven QA** — using an LLM to validate rendered output for correctness and accessibility.

### Integration and orchestration ecosystem

A clear meta-pattern emerges: HyperFrames is the deterministic *render/execution layer*, and the interesting work lives in the orchestration wrapped around it (data source → script → TTS → render → distribute).

| Integration | Workflow |
|---|---|
| **Claude Code / Cursor / Gemini CLI / Codex** | Agents generate, preview, and render videos from prompts via the skills |
| **n8n / Zapier** | Trigger rendering inside workflow-automation pipelines via API/HTTP |
| **MindStudio / Hermes Agent OS** | Orchestrated multi-step pipelines (e.g., weekly data → animated summary → auto-post to Slack) |
| **LangChain / CrewAI** | Agent frameworks use HyperFrames as a rendering backend |
| **Remy** | Compiles the agent + HyperFrames workflow into a full-stack app (auth, DB, UI) |
| **ElevenLabs** | TTS layer in a chained pipeline; audio duration drives clip length |
| **Docker / AWS Lambda / CI/CD** | Scalable, reproducible, distributed rendering environments |

One production technique worth stealing from a documented Claude Code + HyperFrames + ElevenLabs build: **generate and measure the audio first, then render video per segment to match**, because HyperFrames needs to know each clip's duration before it builds. FFmpeg then assembles and concatenates the segments.

### Downstream projects and learning resources

- **`hyperframes-student-kit`** — a community teaching kit of 12 finished motion-graphics projects built on HyperFrames + GSAP, structured to study and rebuild.
- **`hyperframes-launch-video`** — HeyGen's own launch video published as full open source, usable as a real-world reference composition.
- **Agent-Driven Video Studio** — a desktop studio combining Electron + the Claude Agent SDK + HyperFrames for agentic video editing.
- **hyperframes.dev** — a browser-based playground/studio to preview, iterate, and share projects without local setup.
- **Showcase + catalog** — an official showcase (launch videos, website-to-video product tours, effects-heavy reels) and the 50+ block catalog for remixing.
- **Active community discussion** across r/heygen, r/ClaudeCode, r/ClaudeAI, and r/LocalLLM, plus a Product Hunt presence for the broader HeyGen CLI dev platform.

---

## Use Cases Mapped to Your Projects

This is the actionable core — the landscape patterns above, applied to what you actually run.

### ASTGL (As The Geek Learns)

These hit your dual mandate: teach a skill *and* build platform credibility.

- **The meta-article.** "I built a Git-versioned, agent-driven video pipeline that renders MP4s from HTML — here's the architecture and what I learned." This is squarely your practitioner voice and captures the build-as-you-go content you like to harvest at decision points.
- **Animated technical diagrams.** Your MCS-vs-PVS, Horizon-to-Citrix migration, and PKI certificate-lifecycle explainers animate well as staged GSAP reveals — build the diagram once in HTML/CSS, animate the layers, render. These double as Substack embeds and standalone shorts.
- **Article-to-video companions.** Run finished ASTGL posts through the PDF/article-to-video path to produce explainer clips with Whisper-generated captions.
- **A teaching kit of your own.** The community student kit is a template: a small library of annotated ASTGL compositions (VCF 9.0 study topics, PowerCLI patterns) that teach motion-graphics-as-code while reinforcing your brand.
- **Consistent series branding** — one templated intro/outro bumper across every video, edited in a single HTML file.

### Resist and Rise

- **Data-driven explainers.** The `data-chart` catalog block plus bar-chart-race rendering fits accountability reporting — surveillance-spending growth, contract timelines, the Palantir/ICE material from your ebook — all of which travel further as video than as static charts.
- **9:16 social hooks from existing articles.** Turn a published investigation into a 30–45s vertical hook with TTS narration and synced captions, then push it through your existing Substack Notes distribution skill. On-screen source citations match the documented, accountability-first tone.
- **Agent-driven QA as an editorial gate.** The community "LLM validates output" pattern maps onto your fact-check discipline — a validation pass that checks on-screen claims and citations before publish.

### Income Investor Suite / The Income Method

- **Embedded animated explainers in the app** via `<hyperframes-player>` — a dividend-growth bar-chart race, sector-allocation animations, or a visual walkthrough of the Next Buy Recommender's "wait vs. buy" verdict logic, rendered from your actual tracker data.
- **The live-data chart pattern.** The "render animated charts directly from the data feeding your site" approach is a clean fit for recurring portfolio/income visualizations.
- **Book promo videos** for the Income Method / Smart Guide to Income Investing, plus short per-chapter teasers.

### Julie's reselling business (eBay / Poshmark)

This is the sleeper hit and the strongest "done more than twice" automation.

- **Templated, batch-rendered listing videos.** Build one HTML template with slots for product photo, title, price, and rotating beauty shots; feed it a row of listing data; render a clean, branded 9:16 video per item. The built-in u2net background removal cleans messy product photos automatically.
- **A simple internal video factory.** Wrap the render loop so Julie drops photos and a data row and gets finished videos back — the documented "compile the workflow into an app" pattern, scaled to a two-person business.
- **Distributed rendering when volume grows.** If batch sizes outgrow comfortable local render times, the AWS Lambda path scales the same compositions horizontally.

### Your apps and dev workflow

- **Automated release-walkthrough videos** — the tldraw pattern, applied to JDex, KlockThingy, substack-scheduler, and Income Investor Suite: animated code diff, narrated changelog, captions, generated on each tagged release and committed alongside the code.
- **CI-rendered demos.** Because rendering is deterministic and CLI-driven, wire it into the CI/CD pipelines you already run (pre-commit hooks, GitHub Actions) to regenerate demo or changelog videos automatically.
- **substack-scheduler as a video generator.** The "new post → extract a quote → render a branded video → queue to scheduler" pattern is a direct feature extension of an app you already own — turning a scheduler into a scheduler *that generates its own promo assets*.
- **Embeddable players** on app landing pages via the player web component.

### Local AI, ClaudeClaw, and MCP

- **A render agent inside ClaudeClaw Mission Control** — a dedicated video capability any other agent can hand a brief to, keeping the local media models (Kokoro/Whisper/u2net) on-machine.
- **Fits your local-first philosophy.** All media preprocessing runs locally on the Mac Studio; nothing requires a cloud call.
- **MCP research overlap.** The repo carries an `mcp` topic tag and ships agent plugins — worth a teardown both as a tool you use and as ASTGL content tied to your ongoing MCP work.
- **Orchestration overlap.** HyperFrames slots cleanly as a node into the n8n / Make / Zapier flows you've already been evaluating.

---

## Cross-Cutting Patterns and the Orchestration Insight

The single most useful lesson from how others use this tool: **HyperFrames is the render core, not the whole system.** Value compounds in the orchestration around it. Three reusable skeletons stand out:

1. **Scheduled data video.** A recurring job (Google Sheet / database / API) → script → TTS → render → auto-post. One skeleton serves a weekly Income Investor Suite income summary, Julie's weekly sales recap, *and* a Resist and Rise data roundup. Same structure, three data sources.
2. **Content-to-clip.** A published artifact (post, PDF, page) → extract the hook/quote → render a branded short → queue to distribution. Directly extends substack-scheduler.
3. **Build-event video.** A repo event (tagged release, merged PR) → diff/changelog → narrated walkthrough → commit/publish. The tldraw pattern, applied to your apps.

The "audio-first, measure duration, then render" technique threads through all three whenever narration is involved.

---

## Recommended Starting Points

Prioritized by the ratio of learning and payoff to effort and risk.

| Priority | Use case | Why start here | Effort |
|---|---|---|---|
| 1 | Release-walkthrough video for one repo | Proven pattern (tldraw); exercises the full pipeline; produces ASTGL content as a byproduct; you already have the repos + CI | Low–Medium |
| 2 | Julie's templated listing video (single item first) | Highest recurring-time payoff; tests background removal + templating; clear path to batch | Medium |
| 3 | One ASTGL animated technical diagram | Reuses skills you have (HTML/CSS/GSAP); immediate publishable asset | Low |
| 4 | Income Investor Suite dividend bar-chart race | Validates the data-to-motion path against real data | Medium |
| 5 | Scheduled "Monday data video" skeleton | Highest leverage once the basics are proven; reusable across all three brands | Medium–High |

A ten-minute warm-start test (`npx skills add heygen-com/hyperframes`, then point Claude Code at a single product photo or an existing post) will tell you more about real friction than any further evaluation.

---

## Caveats and Considerations

- **Pre-1.0 and fast-moving.** The project is at v0.6.x with a high release cadence — real velocity, but expect occasional breaking changes. Pin versions and update skills periodically.
- **Render is compute-heavy.** Headless Chrome + FFmpeg, frame by frame. Single videos are trivial on your Mac Studio; large batch jobs (Julie's full inventory) take meaningful wall-clock time — benchmark before scaling, and consider Lambda for volume.
- **Batch-at-extreme-scale is the known seam.** Coverage positions Remotion as stronger for spreadsheet-driven thousands-of-variants jobs; HyperFrames excels at individual high-quality videos and agent workflows. Batch works, but stress-test if you push into very high volumes.
- **Dependencies.** Node ≥ 22 and FFmpeg; a full repo clone pulls ~240 MB of Git LFS test baselines (skippable with `GIT_LFS_SKIP_SMUDGE=1`).
- **GSAP licensing.** GSAP became fully free (including commercial plugins) after the Webflow acquisition, so GSAP-based animation should be clear for Julie's commercial use — worth a quick confirmation if you lean on advanced plugins.
- **Vendor incentive.** HeyGen is a commercial avatar/TTS company, and a generous OSS tool is partly a funnel. Apache 2.0 protects you regardless; just expect deeper integrations to nudge toward their paid products.

---

## Appendix: Sources

Primary:

- HyperFrames repository (README, packages, skills, licensing): https://github.com/heygen-com/hyperframes
- Adopters list (HeyGen, tldraw, TanStack, OptinMonster): https://github.com/heygen-com/hyperframes/blob/main/ADOPTERS.md
- HeyGen Help Center — HyperFrames overview: https://help.heygen.com/en/articles/15001510-hyperframes-x-heygen
- Official docs and showcase: https://hyperframes.heygen.com/introduction · https://hyperframes.heygen.com/showcase

Analysis and community coverage:

- DEV Community — "How Code is Killing Traditional Video Editing" (avatar combo, live-data charts): https://dev.to/hugh1st/heygen-hyperframes-how-code-is-killing-traditional-video-editing-3f2h
- Agentpedia — HyperFrames guide (website-to-video pipeline, data-to-motion): https://agentpedia.codes/blog/heygen-hyperframes-guide
- MindStudio — "What Is HyperFrames" (scheduled/orchestrated pipelines): https://www.mindstudio.ai/blog/what-is-hyperframes-ai-video-rendering
- MindStudio — HyperFrames + ElevenLabs + Claude Code (audio-duration-first technique): https://www.mindstudio.ai/blog/ai-video-generation-workflow-hyperframes-elevenlabs
- MindStudio — AI video editing with Claude Code (chained skills, batch, app-wrapping): https://www.mindstudio.ai/blog/ai-video-editing-claude-code-hyperframes
- Medium (AI Engineering) — HyperFrames vs Remotion design philosophy: https://ai-engineering-trend.medium.com/heygens-hyperframes-the-open-source-framework-challenging-remotion-in-html-based-video-creation-c10437f0afca
- silenceper — open-source rendering framework overview (batch/data framing): https://silenceper.com/en/article/2026-05-02-hyperframes-html-video-rendering/
- Nidhin.dev — "Video as Code" deep dive: https://blog.nidhin.dev/video-as-code-a-deep-dive-into-heygen-s-hyperframes
- Downstream/community: `hyperframes-student-kit`, `hyperframes-launch-video`, and discussion across r/heygen, r/ClaudeCode, r/ClaudeAI, r/LocalLLM (compiled from supplied research).
