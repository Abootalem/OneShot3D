---
name: OneShot3D
description: Use when someone asks to turn a video into a website, create a scroll-driven animated page from video frames, or build a premium scroll experience with canvas rendering. Also use for requests like "make my product video into a landing page," "Apple-style scroll video site," or "scrub video on scroll." Hardened by 11+ real production iterations (the ROKH cosmetics site) plus a v2 pass that adds a mandatory pre-flight video QC gate, network-adaptive delivery, and accessibility fallbacks — every MUST/FORBIDDEN rule below comes from a diagnosed defect or a sourced 2026 best-practice.
compatibility: Requires bash_tool with ffmpeg/ffprobe and Python+Playwright available, plus create_file/str_replace. Input is a video file path (MP4, MOV, etc.); output is a standalone HTML deliverable.
metadata:
  argument-hint: <video-file-path>
  version: 2.1 (production-hardened with continuous parallax crossfades, anti-cropping, and unified navigation — see improvements.txt)
  author: Abootaleb Moradi (ai-1.ir)
---

# Video → Premium Scroll-Driven Website (v2.1)

Turn a video file into a luxury, scroll-driven animated website where the video plays as a canvas/video background scrubbed by scroll position, with section content layered on top. Ships as a static deliverable (HTML + CSS + JS + frames/video) that runs without a server, ideally as a single standalone HTML file.

This skill is **opinionated and hardened**. Every "MUST" / "FORBIDDEN" / "CRITICAL" rule below is a scar from a real bug that cost 30+ minutes to diagnose on a production build, or a practice sourced from current (2026) web-performance research — see `improvements.txt` for the audit this fork was built from and `references/lessons-learned.md` for the original bug postmortems.

**What's new in v2.1 (The Production UX & Kinematics Pass)**:
- **Continuous Parallax Crossfades (Zero Instant Transitions)**: Solves user complaint *"scrolling causes instant transition between cards and not visually satisfactory"*. Replaces rigid binary opacity jumps with a 3-phase Smoothstep kinematic engine (glide-in +36px, stable hold, glide-out -32px) and 4–6% overlapping section ranges.
- **Zero Card Cropping (Card Height Budget)**: Solves user complaint *"all pop up menues cropped and not truely shown by scroll"*. Enforces a strict ≤ 420px height budget on floating cards, fixed-height product images (105px `object-fit: contain`), and compact typography so 100% of content is visible on laptop displays.
- **Lenis Inner-Scroll Protection**: Adds mandatory `data-lenis-prevent="true"` to scrollable cards so mouse wheel gestures inside cards are never hijacked by page scroll.
- **Zero Pseudo-Element Overflow**: Eliminates negative insets (`inset: -40px -32px`) that caused unwanted horizontal scrollbars at the bottom of cards.
- **Unified 3-Column Header & Hero Clearance**: Solves navbar flipping and logo collision bugs; provides `padding-top: calc(var(--header-h) + 36px)` clearance for hero titles.

**What's new in v2** (full rationale in `improvements.txt`):
- **Step 0: mandatory pre-flight video QC** — probe file size, codec, resolution, aspect ratio, and bitrate *before* any design work starts, and get an explicit weight decision from the user instead of discovering the file is too heavy after delivery.
- **Network-adaptive delivery option** — the site itself can pick the right bitrate tier at runtime via the Network Information API, instead of only offering multiple static HTML files for the user to choose between.
- **Accessibility motion fallback** — `prefers-reduced-motion` support is now a checklist item and a Definition-of-Done gate, not an afterthought.
- **Updated codec guidance** — reconciles the original ROKH-project finding ("AV1/VP9 didn't help") with 2026 encoder data (SVT-AV1 is now 3-8× slower than H.264, not 20-50×, and has real hardware-decode support) into a measure-don't-assume rule.
- **"3D" scoped honestly** — this skill produces a 2D canvas/video parallax experience (Apple-style), not WebGL 3D geometry. A separate, clearly-flagged Step 6b covers when/how to add true Three.js 3D, with the performance guardrails that make it safe.

## Required tools

- `bash_tool` — for `ffmpeg` (video/frame processing), `ffprobe` (verification), and Python/Playwright (QA screenshots + console checks)
- `create_file` / `str_replace` — for writing `index.html`, `css/style.css`, `js/app.js` (or the standalone single-file equivalent)
- `view` — for inspecting extracted frames/screenshots visually during QA
- `ask_user_input_v0` (when available in the host environment) — for the Step 0 weight-decision prompt; if unavailable, ask the question as plain conversational text and wait for a reply before proceeding
- Optionally `image_search` is NOT relevant here; all visual QA uses locally rendered screenshots, not web images

---

## Input

The user provides: a video file path (MP4, MOV, etc.) and optionally:
- A theme/brand name
- Desired text sections and where they appear
- Color scheme preferences
- Bilingual requirements (e.g. Persian/English with RTL/LTR)
- Any specific design direction

If the user doesn't specify these, ask briefly or use sensible creative defaults.


---

## Step 0: Pre-Flight Video QC & Weight Decision (MANDATORY, runs before Step 1)

**This is a hard gate, same status as Steps 1 and 2.** Never touch research, ffmpeg re-encoding, or design decisions until Step 0's probe has run and the user's weight decision is known. Skipping this step is how the original ROKH project shipped a 2.6MB page and found out it was "too heavy" only after delivery — see `references/lessons-learned.md` for that postmortem. Full commands, thresholds, and the decision-tree are in `references/qc-and-compression-research.md`; this section is the summary.

1. **Probe the uploaded file immediately** with a single `ffprobe` call — file size, container/codec, resolution, aspect ratio, duration, average bitrate, frame rate, and whether an audio track exists. This takes under a second and costs nothing to run even when the file turns out to be fine.
2. **Evaluate against thresholds** (full table in `references/qc-and-compression-research.md` §1):
   - File size: comfortable (< 15MB), moderate (15–50MB), large (> 50MB)
   - Aspect ratio: standard (9:16 to 21:9) vs extreme (taller than 9:16 or wider than 21:9 — flag, since these need deliberate `contain`-mode handling per Anti-pattern #2/#41)
   - Codec/container: already web-friendly (H.264/H.265/VP9/AV1 in MP4/MOV/WebM) vs needs transcoding (ProRes, DNxHD, unusual containers)
   - Bitrate-for-resolution: flag if the source is bitrate-bloated relative to its resolution (e.g. >12 Mbps at 1080p for a short clip) — re-encoding will shrink it with no visible loss
   - **Audio stream present:** flag if yes. This one isn't a threshold, it's an automatic action — see point 3a below.
3. **If the file is already comfortable and web-friendly, skip the question** — state in one line that QC passed (size, codec, aspect ratio) and proceed straight to Step 1. Don't interrupt the user with a decision that doesn't need making.
3a. **Strip any audio stream unconditionally, regardless of the weight decision below.** Every build in this skill mutes the `<video>` element (Anti-pattern #34), so an audio track is pure dead weight that downloads but is never heard — it's not a size/quality trade-off like the options in step 4, it's a free win. Add `-an` to every re-encode command that touches this file, in every tier. If the source has a substantial audio track (music, narration), mention the byte savings to the user in the Step 0 message as a heads-up, not as a choice.
4. **If the file is moderate or large, or has an unusual codec/aspect ratio, ask the user a single decision** before doing anything else (via `ask_user_input_v0` if available, otherwise as a plain question — either way, wait for the answer):
   - **Heavy / cinematic** — keep source quality, accept a larger deliverable. State the approximate resulting file size and that it won't be ideal for visitors on slow mobile connections.
   - **Robust / compressed (recommended default)** — apply the research-backed compression ladder from `references/qc-and-compression-research.md` (2-pass VBR H.264, short-GOP for scrub, resolution matched to display size) targeting the "recommended" tier in the Performance Budget table below.
   - **Network-adaptive (best of both, when Path A/video-scrub)** — build both a light and a heavy source and let the *visitor's* browser pick automatically via the Network Information API (falls back to the recommended tier on browsers without it, e.g. Safari). See `references/qc-and-compression-research.md` §3 for the snippet. This is the best default when the audience's network conditions are unknown, at the cost of slightly more build complexity.
   - **I'll re-upload a smaller file** — if the user would rather pre-compress on their end, pause here and wait for the new upload instead of proceeding.
5. **Record the decision** — it determines which Performance Budget tier to target in Step 4a/4b, whether to build the network-adaptive loader, and whether the multi-bitrate ladder is produced proactively now instead of reactively after a "this feels heavy" complaint later.

**Checkpoint before proceeding to Step 1:** you must be able to state the source file's size/codec/aspect-ratio and (if asked) the user's weight decision in one sentence. If you can't, Step 0 wasn't actually done.

---

## Premium Checklist (Non-Negotiable)

00. **Step 0 pre-flight QC is mandatory, not optional** — probe the source video and get a weight decision (heavy / robust / network-adaptive) before Step 1's research even starts. See the Step 0 section above.
0. **Step 1 research is mandatory, not optional** — never jump straight to frame extraction. Produce the reference synthesis (3-5 sites, point of view, borrowed tactics) before any design decision is made, even under time pressure.
0b. **Step 2 scrub-method choice is mandatory** — decide video-scrub vs frame-extraction before touching ffmpeg. The default is video-scrub (lighter, simpler); frame-extraction is only required for side-panel layouts where the subject must be offset off-center.
0c. **Standalone single-file HTML is the default delivery format** — only build multi-file if the user explicitly asks for it or says they're deploying. Multi-file relative paths (`css/style.css`, `js/app.js`) silently fail under `file://` and in chat preview.
0d. **Motion-reduced fallback is mandatory** — every scroll-jacked/parallax animation must have a `prefers-reduced-motion` fallback that keeps content visible and readable, per the new checklist item #24 below.
1. **Lenis smooth scroll** — native scroll feels "web page," Lenis feels "experience."
2. **4+ animation types** — never repeat the same entrance animation consecutively.
3. **Staggered reveals** — label → heading → body → CTA, never all at once.
4. **Solid glass panels, not bare glassmorphism** — text panels must have `background: rgba(…,≥0.88)` PLUS `backdrop-filter: blur(≥14px)`. Pure blur with low opacity = "amateur luxury."
5. **Direction variety** — sections enter from different directions (left, right, up, scale, clip).
6. **Dark overlay for stats** — 0.88-0.92 opacity, counters animate up, only time center text is OK.
7. **Counter animations** — all numbers count up from 0, never appear statically.
8. **Massive hero, but compact high-density cards** — hero can be bold (3.2–6.5rem), but floating section headings MUST be compact (`clamp(1.45rem, 2.2vw, 2.05rem)`) to guarantee the card height never exceeds 420px and crops on laptops.
9. **CTA persists** — `data-persist="true"` keeps final section visible, never disappears.
10. **Hero prominence + generous scroll** — hero gets 14%+ scroll range, 800vh+ total for 6 sections.
11. **Side-aligned text ONLY** — all text in outer 40-50% zones (`align-left`/`align-right`), never center. Exception: stats with full dark overlay.
12. **Circle-wipe hero reveal** — hero is standalone 100vh section, canvas reveals via `clip-path: circle()` as hero scrolls away.
13. **Frame speed 1.8-2.2** — product animation completes by ~55% scroll. Below 1.8 feels sluggish.
14. **Continuous Parallax Crossfade (Zero Dead Gaps)** — sections must overlap by 4–6% using a 3-phase Smoothstep kinematic curve (Phase 1 glide-in 28%, Phase 2 hold 44%, Phase 3 glide-out 28%). As card N ascends (-32px) and fades to 0, card N+1 ascends (+36px to 0) and fades to 1. Never jump opacity from 0 to 1; never leave empty dead gaps between sections.
15. **Section position uses `%` of scroll-container, NEVER `vh`** — `top: 50vh` inside an 800vh container becomes `top: 6.25%` — every section stacks at the top and is invisible. This was the most destructive bug in the entire ROKH project.
16. **Canvas bg color MUST be hardcoded identically in JS and CSS** — re-sampling at runtime causes 2-3% drift between canvas bg and panel bg → visible vertical seam.
17. **No pop-up menus of any kind** — no scroll-pop side nav, no hover dropdowns on header nav, no exit-intent popups, no cookie banners. The only overlay allowed is a tap-to-open mobile menu.
18. **No emojis anywhere** — scan with `grep -P "[\x{1F300}-\x{1FAFF}]"`. Use real, visually-vetted (via `view`) product images for e-commerce.
19. **No marquee / sliding text banner by default** — do not add a horizontal scrolling text element unless the user explicitly asks for one.
20. **No hero corner meta** — `top: calc(header-h + 24px); right: 5vw; font-size: 0.7rem` falls off narrow viewports and overlaps the language toggle. If a promo is needed, integrate into the hero-meta-row.
21. **0 console errors, 0 console warnings** — verified via Playwright `page.on("console")`.
22. **Review all 16+ screenshots yourself with `view`** — 0 issues allowed before delivery.
23. **`prefers-reduced-motion` fallback is required** — wrap Lenis smooth-scroll init, GSAP entrance stagger, and any scroll-jacking behavior in a check; reduced-motion visitors get instant scroll, content visible without animation, and (for Path A) an autoplaying-but-non-scrubbed or static-poster video instead of scroll-scrubbed seeking. Verified via Playwright's `page.emulate_media(reduced_motion="reduce")`. See `references/qc-and-compression-research.md` §4.
24. **Pre-flight QC (Step 0) was actually run** — you can state the source file's size, codec, and aspect ratio, and (if the file was moderate/large) the user's chosen weight tier.
25. **Zero card cropping (Card Height Budget)** — every floating section card MUST fit completely inside a 768px-tall laptop viewport. Total internal height ≤ 420px. Images must use fixed max-height (105px with `object-fit: contain`), never unconstrained 1:1 aspect ratios.
26. **Lenis inner-scroll protection** — any scrollable inner card container MUST declare `data-lenis-prevent="true"` to prevent Lenis from capturing wheel events and prematurely advancing the page scroll.
27. **Zero pseudo-element overflow** — no negative insets (`inset: -Xpx`) on pseudo-elements inside cards with `overflow: auto`. Always add `overflow-x: hidden;` to permanently eliminate horizontal scrollbars.
28. **Integrated 3-column header** — never inject isolated `position: fixed` return/portal buttons that overlap brand marks. Integrate portal chips inside the primary `.site-header` grid.
29. **Header clearance for hero** — `.hero-standalone` must have `padding-top: calc(var(--header-h) + 36px)` so large titles are never sliced by the fixed navbar.


---

## Workflow

Follow Step 0 (above) then these 10 steps in order. Step 0, and Steps 1 and 2, are mandatory decision points — never skip straight to ffmpeg. Full command-by-command detail (exact ffmpeg flags, complete HTML/CSS/JS code) for every step lives in `references/workflow-detail.md`; read that file before executing Step 3 onward. Video QC thresholds, compression-ladder recipes, the network-adaptive loader snippet, and the reduced-motion fallback pattern live in `references/qc-and-compression-research.md`.

1. **Research similar websites (MANDATORY).** Before any design decision, produce a short reference synthesis — 3-5 comparable premium sites, a point of view, and borrowed tactics. Never jump straight to frame extraction.
2. **Choose scrub method (MANDATORY).** Decide **video-scrub** (Path A — default, lighter, simpler, uses `<video>` + `object-fit`) vs **frame-extraction** (Path B — only when the layout needs the subject offset off-center with a specific pixel position, uses `<canvas>` + image sequence).
3. **Analyze the video** — duration, resolution, orientation, subject position, via `ffprobe`.
4. **Prepare the asset**: 4a (Path A) re-encode with short GOP (`-g 10 -keyint_min 10`) for smooth scrubbing; 4b (Path B) extract and compress frames to WebP.
5. **Asset polish** — visually check (via `view`) extracted frames/video for quality, crop/letterbox handling.
6. **Scaffold** the file structure. Default to a **standalone single-file HTML** deliverable (base64-inlined media) unless the user explicitly asks for multi-file or says they're deploying — multi-file relative paths silently fail under `file://` and in chat preview.
6b. **[Optional, opt-in only] True 3D (Three.js/WebGL).** This skill's default output is a 2D canvas/video parallax experience, not real 3D geometry — that distinction matters because "3D" is sometimes used loosely to mean "premium scroll feel." Only build an actual WebGL/Three.js scene if the user explicitly confirms they want real 3D depth or geometry (rotating product model, particle field, etc.), not just a nicer scroll. If confirmed: keep it to a single lightweight scene (low poly count, no heavy postprocessing), lazy-init it after the hero is interactive (don't block LCP on WebGL context creation), and re-run the full Performance Budget — interactive 3D is a documented Core Web Vitals risk (see `references/qc-and-compression-research.md` §5) and must not be added silently.
7. **Build `index.html`** — semantic section structure, hero, content sections, persistent CTA.
8. **Build `css/style.css`** — CSS variables (colors hardcoded identically to JS canvas bg), glass panels, typography, section positioning using `%` (never `vh`) inside the scroll container.
9. **Build `js/app.js`** — Lenis smooth scroll, ScrollTrigger-driven canvas/video scrubbing, staggered entrance animations, counters. This is the largest and most bug-prone step; see `references/workflow-detail.md` Step 9 in full before writing this file.
10. **QA loop** — Playwright screenshots at 16+ scroll positions, 0 console errors/warnings, you personally review every screenshot with `view` (0 issues allowed before delivery).

For copy-paste CSS variable blocks, JS CONFIG blocks, and recovery scripts for known failure modes, see `references/templates-and-scripts.md`.

---

## Animation Types Quick Reference

| Type | Initial State | Animate To | Duration | Ease |
|------|--------------|-----------|----------|------|
| `fade-up` | y:50, opacity:0 | y:0, opacity:1 | 0.9s | power3.out |
| `slide-left` | x:-80, opacity:0 | x:0, opacity:1 | 0.9s | power3.out |
| `slide-right` | x:80, opacity:0 | x:0, opacity:1 | 0.9s | power3.out |
| `scale-up` | scale:0.85, opacity:0 | scale:1, opacity:1 | 1.0s | power2.out |
| `rotate-in` | y:40, rotation:3, opacity:0 | y:0, rotation:0, opacity:1 | 0.9s | power3.out |
| `stagger-up` | y:60, opacity:0 | y:0, opacity:1 | 0.8s | power3.out |
| `clip-reveal` | clipPath:inset(100% 0 0 0) | clipPath:inset(0% 0 0 0) | 1.2s | power4.inOut |

All types use stagger (0.1-0.15s).

---

## Clip-Path Variations

- **Circle reveal**: `circle(0% at 50% 50%)` → `circle(75% at 50% 50%)` (default for hero)
- **Wipe from left**: `inset(0 100% 0 0)` → `inset(0 0% 0 0)`
- **Wipe from bottom**: `inset(100% 0 0 0)` → `inset(0% 0 0 0)`
- **Custom polygon**: `polygon(50% 0%, 50% 0%, 50% 100%, 50% 100%)` → `polygon(0% 0%, 100% 0%, 100% 100%, 0% 100%)`

Always include the `-webkit-clip-path` prefix for Safari compatibility.


---

## Anti-Patterns (DO NOT DO)

These are all real bugs. Each one cost 30+ minutes to diagnose on a production build.

| # | Anti-pattern | Why it breaks | Fix |
|---|--------------|---------------|-----|
| 1 | `top: Xvh` on absolutely positioned section inside scroll-container | Container is 800vh+ tall; `50vh` becomes `6.25%` → all sections stack at top | Use `top: X%` (percent of container) |
| 2 | `Math.max(vw/iw, vh/ih)` canvas scale (cover mode) on portrait video | Crops subject's head and feet | Use `Math.min` (contain mode) + side-offset |
| 3 | Subject drawn CENTERED in viewport | Section panel on left overlaps subject's left half | Offset subject to hero side |
| 4 | Re-sample bg color at runtime | Color drifts between canvas bg and panel bg → visible seam | Hardcode same hex in both files |
| 5 | Slow fade-out using `overshoot * 3` divisor | Sections remain visible at 50% scroll, overlapping next section | Fixed 4% fade range |
| 6 | Alternating `align-left`/`align-right` between sections | User perceives chaos as panels jump sides | All sections on SAME side (opposite hero) |
| 7 | Soft radial-gradient backdrop | Insufficient contrast for text — looks "amateur" | Solid dark glass card + blur(18px) |
| 8 | Floating scroll-pop nav at 24-64% scroll | Looks like WordPress widget, conceals video | Remove entirely |
| 9 | Hover dropdowns on header nav | Trigger on accidental hover, conceal video | Flat 7-link nav, no dropdowns |
| 10 | Emoji as product thumbnails | Cheap-looking, breaks on older Android | Real product images, visually vetted via `view` |
| 12 | Hero corner meta (`top-right: 5vw`, `font-size: 0.7rem`) | Falls off narrow viewports, overlaps lang toggle | Put promos in hero-meta-row instead |
| 13 | Poetry/philosophy content for e-commerce site | User said "design principles not followed" — site must be commercial | Product names, prices, Add to Bag, badges |
| 14 | Saffron/spice references on cosmetics site | Brand mismatch (saffron is food, not cosmetics) | Use rose, jasmine, neroli, oud |
| 15 | Frames extracted at native resolution (480×854) | Wastes 47% bytes — canvas upscales anyway | Downscale to 360×640 at extraction |
| 16 | `font-size: 0.7rem; letter-spacing: 0.18em` on tiny corner text | Illegible on mobile, "falls off" visually | Either integrate into hero-meta-row, or `font-size: 0.8rem+` |
| 17 | Re-extract frames at q=80 then "losslessly recompress" | Two passes for worse result than single q=65 pass | q=65 directly at extraction (for natural images) |
| 18 | Lossless WebP (`cwebp -lossless`) on natural video frames | Lossless is for synthetic content; produces 40-60KB/frame vs 14-18KB lossy | Use lossy q=65 for photographs/people |
| 19 | Skip visual vetting on product images | ~40% contain models' lips/hands | Always inspect with `view` against a "no person / clean bg" checklist |
| 20 | Numbered list as flex baseline alignment | Items look "floating, misaligned" | CSS grid `56px 1fr auto` |
| 21 | Outline CTA button with no hover state | Looks "cheap", not luxury | Gold→rose-gold gradient sweep + lift + glow |
| 22 | Cycling feature cards in a pinned section | Each card gets too little scroll time | Give each feature its own scroll-triggered section (8-10% range) |
| 23 | FRAME_SPEED > 1.5 (e.g. 2.0) | Frames deplete at 50% scroll — canvas freezes on last frame for entire second half of page. User said: "mid-scroll before reaching bottom, hero frames run out" | Use FRAME_SPEED = 1.0 (1:1 mapping). Frame 0 at 0% scroll, frame N at 100% scroll. |
| 24 | Hero < 14% scroll range | First impression needs breathing room | 14%+ for hero |
| 25 | Same animation for consecutive sections | Repetitive, boring | Never repeat the same entrance type back-to-back |
| 26 | Wide centered grids over canvas | Text overlaps video, looks cramped | Redesign as vertical lists in the 40% side zone |
| 27 | Scroll height < 800vh for 6 sections | Everything feels rushed | 800vh+ minimum |
| 28 | Cover mode with `IMAGE_SCALE = 0.85` (upstream skill's recommendation) | Crops subject on portrait video — "lady's head cut off" bug | Use contain mode (`Math.min`) with `IMAGE_SCALE = 0.95` + hardcoded bg color match |
| 29 | `.hero-standalone { justify-content: flex-end }` with RTL `dir="rtl"` | In RTL, `flex-end` = LEFT, same side as lady canvas (dx=0). Hero text overlaps lady during circle-wipe transition (0→7% scroll). User said: "initial text on LEFT (hero side)" | Use `justify-content: flex-start` — flex-start = LEFT in LTR, RIGHT in RTL. Hero text sits on OPPOSITE side of lady canvas in both directions. |
| 30 | Section `transform: translateY(-50%)` with 4 product cards (2×2 grid) + title + body + CTA | Section height exceeds viewport. With -50% centering, title scrolls off top of viewport. User said: "titles not readable". Visual review confirmed at 47% scroll: title above viewport, only product cards visible. | Use `translateY(-35%)` AND limit each section to 2 product cards (1×2 grid). Section height stays ≤ viewport, title always visible. |
| 31 | Removing HTML elements (marquee, dark-overlay, stats section) without updating JS setup functions | JS throws `Cannot read property 'style' of null` errors when trying to manipulate removed elements | Make setup functions null-safe: `if (!element) return;` at top of each setup function. Reference elements via `$("#id")` which returns null if missing. |
| 32 | `hero-meta-row` containing "Free Shipping / ارسال رایگان" text in hero | User said: "annoying Free Shipping text that falls on hero should not exist" | Remove the entire `hero-meta-row` div. Shipping info belongs in footer or product pages, not hero. |
| 33 | Path A: `<video>` without `playsinline` attribute | iOS Safari forces the video into fullscreen on first interaction, breaking the scroll-scrub experience entirely | Always add `playsinline webkit-playsinline` (the `webkit-` prefix is for iOS 8-9 legacy) |
| 34 | Path A: `<video>` without `muted` attribute | iOS autoplay policy blocks video loading until user gesture, so `loadedmetadata` never fires and loader hangs forever | Always add `muted`. Scrub videos have no audio track anyway. |
| 35 | Path A: setting `video.currentTime` directly on every ScrollTrigger `onUpdate` | ScrollTrigger fires 60+ times/sec during scroll; the browser queues seeks faster than it can decode them → visible jank, frame drops, fan spins up | Throttle via `requestAnimationFrame`: if a seek is pending, drop the new request. Add a 16ms delta threshold to skip redundant seeks. |
| 36 | Path A: video stays black until first scroll event | Decoder doesn't render the first frame until a seek or play is requested. After `loadedmetadata`, the video shows a black rectangle. | After `loadedmetadata` fires, immediately `video.currentTime = 0.001` to force decoder to render frame 0. |
| 37 | Path A: loader hangs on page reload (cache hit) | On second load, the video may already have `readyState >= 1` by the time JS attaches the `loadedmetadata` listener. Since the event already fired, the listener never triggers. | Check `video.readyState >= 1` BEFORE attaching the listener. If true, call `onLoadedMetadata()` directly. |
| 38 | Path A: video encoded with default `-g 250` (long GOP) | Seeking to a non-keyframe position requires decoding up to 250 frames forward, causing 200-500ms seek lag → scroll-scrub feels broken | Re-encode with `-g 10 -keyint_min 10` to force a keyframe every 10 frames. Seek time drops to <30ms. |
| 39 | Path A: `<video>` without `disablepictureinpicture disableremoteplayback` | Chrome shows PiP and Cast buttons on hover, which users click expecting them to do something useful — instead they hijack the video into a floating window or send it to TV | Always add both attributes. The scrub video is a background animation, not a playable video. |
| 40 | Path A: subject on wrong side after language toggle | The `<video>` element doesn't automatically move when `dir` changes — only flexbox/text reflows. The lady stays on her old side, overlapping the new hero text. | Call `updateVideoPosition()` in `applyLanguage()`: set `video.style.objectPosition = isRtl ? "left center" : "right center"`. |
| 41 | Path A: `object-fit: cover` on scrub video (for portrait video of people) | Cover mode crops the portrait video to fill the landscape viewport — cuts off the subject's head/feet, zooms 2.25x, and the subject disappears entirely at scroll positions where they've moved to the top/bottom of the video frame. User feedback: "by 50% scroll the lady is gone, only blurry fabric visible." | Use `object-fit: contain` to show the full video frame (matches Path B's `Math.min` canvas behavior). The empty letterbox space is filled by `video-wrap`'s `bg-dark`, which matches `--bg-dark` exactly so the seam is invisible. Use `object-position: left/right center` to position the fitted video on the opposite side from the hero text. Cover mode is only correct for landscape video with a centered-static subject. |
| 42 | Path A: large base64-inlined MP4 (>3MB before inlining) | base64 encoding adds 33% overhead, so a 3MB video becomes 4MB data URI → standalone HTML exceeds 5MB, slow to load on mobile | Target ≤2MB MP4 before inlining. Use `scale=720:-2 -crf 24` (or `-crf 26` if still too big). For >3MB videos, use external file reference and accept multi-file delivery. |
| 43 | Path A: `<video src="...">` set in HTML, then JS also sets `video.src` | Double-loading: the browser starts fetching from the HTML attribute, then JS reassigns and re-fetches. Doubles network cost and may cause race condition where `loadedmetadata` fires twice. | Either set `src` in HTML and skip JS assignment, OR leave HTML attribute empty and assign via JS only. For standalone bundles with base64, ALWAYS use JS-only assignment (the data URI is too large for HTML attribute parsing). |
| 44 | User says "page is too heavy, make it lighter" → just lower the CRF and rebuild | CRF mode gives constant quality but no size target — same CRF on different content produces wildly different sizes. You have no idea what the final file size will be until the encode finishes. | Use 2-pass VBR encoding with `-b:v <target>` instead. Pass 1 analyzes complexity, pass 2 distributes bits intelligently. Same size, better perceptual quality. See "Multi-bitrate delivery" section for the bitrate ladder. |
| 45 | Trusting a side-by-side comparison when evaluating compressed video | Visual review is overly picky in side-by-side mode — it flags ANY bitrate below source as "visible quality loss," even at 1200kbps vs 1290kbps source (only 7% smaller). This leads to delivering unnecessarily large files. | Test with SOLO assessment: view only the compressed frame and ask "is this acceptable for a luxury cosmetics website?" Users never see the original next to the compressed version. Soft-focus/fashion content gets "acceptable" ratings even at 300kbps. |
| 46 | Trying AV1 codec for scrub video | AV1 encoding is 5-10× slower than H.264, and at equivalent visual quality produces files of similar size (1.6-2.2MB at CRF 38-42 vs H.264's 1.5MB at CRF 24). Browser support is spotty — Safari < 16.4 has no AV1 support at all. | Stick with H.264. The codec's short-GOP support (`-g 10`) is also better for random access (scrubbing) than AV1's longer keyframe intervals. |
| 47 | Trying WebM/VP9 for scrub video | VP9 produced 3.2MB at CRF 32 — WORSE than H.264's 1.5MB at CRF 24 for the same content. VP9's complex intra-frame prediction (good for streaming) makes random access (scrubbing) slower than H.264's short-GOP. | Use H.264 with `-g 10 -keyint_min 10` for scrubbing. VP9 is optimized for sequential streaming, not random access. |
| 48 | Trying AVIF instead of WebP for Path B frames | AVIF was 18% smaller than WebP, but visual review flagged visible quality loss — hair strands and fabric texture became "softer and less defined." The savings aren't worth the quality regression. | WebP at q=65 is already optimal for natural video frames. AVIF's advantage is only meaningful for very small thumbnails, not full-size scrubbing frames. |
| 49 | Re-extracting Path B frames from the compressed H.264 video to save space | H.264 compression introduces high-frequency noise (DCT artifacts) that WebP can't compress as efficiently as the clean original. Net savings at same visual quality: only 9%. | Leave Path B frames extracted from the original source video. If Path B is too heavy, switch to Path A (video-scrub) entirely instead of trying to compress the frame sequence. |
| 50 | Skipping Step 0 and jumping to research/build on a video that turns out to be 80MB, 4K ProRes | Wastes an entire research+build cycle before discovering the source needs re-encoding, and the user never got a say in the size/quality trade-off | Run the Step 0 `ffprobe` gate immediately on upload, every time, even when the file "looks fine" — the check costs under a second |
| 51 | Citing row 46/47 above ("AV1/VP9 didn't help") as a permanent, universal conclusion | Those results are from one 10s portrait clip on one encoder build in 2025. 2026 encoder data shows SVT-AV1 is now 3-8× slower than H.264 (not 20-50×) with real hardware-decode support on current phones/Apple Silicon — the tradeoff has measurably shifted | Treat row 46/47 as "H.264 is the safe default for short-GOP scrub video," not "never test AV1 again." If the user explicitly wants the absolute smallest file and accepts an extra encode pass, test SVT-AV1 as an *additional* `<source>` with H.264 fallback and measure — don't assume either way |
| 52 | Adding GSAP ScrollTrigger pinning/scroll-jacking with no `prefers-reduced-motion` check | Scroll-jacked pages hijack the scroll thread; for motion-sensitive users this is a genuine accessibility problem (vestibular triggers), not just a preference | Wrap Lenis + ScrollTrigger init in a reduced-motion check; ship an equivalent static/instant-scroll experience for those users (see `references/qc-and-compression-research.md` §4) |
| 53 | Adding a Three.js/WebGL 3D scene because the user said "3D scrolling website" without confirming they meant literal 3D geometry vs. just a premium scroll feel | Interactive 3D ships a rendering engine to the browser and is a documented 2026 Core Web Vitals/LCP risk; building it unasked adds real perf cost for a request that usually just meant "make it feel premium like Apple's pages" | Ask/clarify before adding WebGL. Default output (canvas/video parallax) already reads as "3D-ish" to most users. Only build true Three.js 3D per Step 6b, opt-in, with its own perf budget |
| 50 | Reducing Path B frame count from 150 to 75 (halving fps) to save 50% size | Scroll feels noticeably choppier — 15fps scrubbing is the perceptual floor. Below that, users describe the experience as "slideshow" not "video." | Keep 150 frames minimum. If Path B is too heavy, switch to Path A which uses a single video file with temporal compression (much more efficient than independent frames). |
| 51 | Using `-tune ssim`, `-tune film`, `-tune fastdecode`, or `-tune animation` to improve compression | All tune presets produced LARGER files than default at the same CRF — `tune fastdecode` was 2× larger (3.06MB vs 1.57MB), `tune film` was 1.6× larger. None improved perceptual quality enough to justify the size increase. | Use default tune (no `-tune` flag). The default psychovisual settings are already optimal for general video content. Only use `-tune` for specific content types where you've measured the improvement. |
| 52 | Using `-preset veryslow` for better compression at same CRF | Slower presets give diminishing returns — `veryslow` was only 4% smaller than `slow` at CRF 28, but took 3× longer to encode. Not worth the time cost for iterative development. | Use `-preset slow` as the default. Only escalate to `slower`/`veryslow` for final production delivery when every KB matters and encode time is not a concern. |
| 53 | Delivering ONLY the 300kbps ultra-light version | User rejected it: "resolution dropped too much, useless." Aggressive compression is fine as a fallback for hostile network conditions, but should never be the primary deliverable. | Deliver THREE versions: `*-original.html` (full quality), `*.html` (recommended ~600kbps), `*-300k.html` (ultra-light). Let the user choose. The recommended version is the actual deliverable; the others are options. |
| 54 | Deleting the English-translated name when fixing a Persian spelling mistake | User had "ابوطالب مرادی" (wrong) and asked to fix to "ابوالطالب مرادی." Fixing only the Persian text removed the bilingual `data-en="Abootaleb Moradi"` attribute, breaking the language toggle. | When fixing text in a bilingual site, preserve BOTH `data-fa` and `data-en` attributes. Edit the value, don't remove the translation. Scan with `grep 'data-en='` before and after the edit to confirm count is unchanged. |
| 55 | Massive card typography + unconstrained 1:1 square product images | Card height reaches 850px+, exceeds 600px available laptop viewport → product cards, prices, and Add to Bag buttons chopped off at bottom ("pop up menues cropped and not truely shown by scroll") | Use High-Density Compact Layout: headings `clamp(1.45rem, 2.2vw, 2.05rem)`, images `height: 105px; object-fit: contain;`, card padding `22px 20px`, total height ≤ 420px. |
| 56 | Negative insets on `::before`/`::after` inside scrollable cards (`inset: -40px -32px`) | With `overflow: auto`, negative insets extend beyond container boundaries and spawn an ugly horizontal scrollbar | Remove negative insets. Style `.section-inner` directly with glass background and borders. Set `overflow-x: hidden;`. |
| 57 | Lenis smooth-scroll without `data-lenis-prevent="true"` on inner scrollable containers | Lenis hijacks all wheel events on window; scrolling over an inner container advances page scroll instead of scrolling the card, causing the card to immediately vanish | Add `data-lenis-prevent="true"` to all `.section-inner` elements with `overflow-y: auto`. |
| 58 | Binary `opacity = 1` jump on enter + non-overlapping section spans | Cards pop into existence abruptly like a switch, sit frozen for 75% of scroll time, then snap out into a blank screen before next card pops in ("instant transition between cards") | Use Continuous Parallax Crossfade Engine: 3-phase Smoothstep interpolation (glide-in from +36px, stable hold, glide-out to -32px), with 4-6% overlapping section ranges. |
| 59 | Standalone `position: fixed; top: 20px; right: 20px;` return button | Collides with brand logo in RTL and LTR, giving optical illusion of navbar jumping sides; hero title has no clearance for fixed navbar and gets sliced | Integrate portal chip into 3-column `.site-header`. Give hero `padding-top: calc(var(--header-h) + 36px);`. |
| 60 | Misinterpreting "too much empty space" by over-compressing scroll container (< 400vh) | Scroll distance becomes too short (~280vh), mouse wheel flings through sections in fractions of a second, depleting or rushing 3D animations | Maintain 440vh - 480vh for 1:1 video scrub mapping, while compacting intra-card spacing. |


---

## Common User Feedback → Fix Mapping

Real user feedback from the ROKH project, with the underlying bug and fix:

| User said | Actual bug | Fix location |
|-----------|-----------|--------------|
| "Lady's head cut off" | Cover mode on portrait video | `js/app.js drawFrame()` — use Math.min |
| "Content messed up and mixed (overlapping lady)" | Soft gradient backdrop insufficient contrast | `css/style.css .section-inner` — solid glass |
| "Saffron references on cosmetics site" | Brand mismatch in copy | `index.html` — replace with rose/jasmine |
| "Texts not readable, no proper styling" | Insufficient contrast + outline CTAs | `.section-inner` opacity 0.94, `.cta-button` luxury gradient |
| "Stray numbered list" | Flex baseline, no grid | `.section-list` CSS grid `56px 1fr auto` |
| "Design principles not followed, content is poems" | Poetry copy on e-commerce site | Rewrite all sections as product cards |
| "Pop-up menus concealing video" | scroll-pop nav + hover dropdowns | Remove scroll-pop, flatten nav |
| "Emojis everywhere" | Emoji used as product thumbnails | Replace with real, visually-vetted images |
| "Every product needs a real image" | Same as above | Same fix |
| "Texts unreadable during scroll" | Slow fade-out → sections overlap | 4% fixed fade range in `setupSectionAnimations()` |
| "Hero on one side, sections on the other, no overlap" | Alternating align-left/right + centered subject | Unified align-left + subject offset to hero side |
| "Panels still overlap with the lady halfway" | Centered subject canvas | `drawFrame()` dx offset based on RTL/LTR |
| "Too much empty space between hero and panels" | Fixed 56vw padding on all viewports | Responsive padding via media queries |
| "Hero side flip — Persian=LEFT, English=RIGHT" | (User preference) | `flex-start` in CSS (NOT flex-end — flex-end in RTL puts hero on LEFT, same side as lady) + canvas dx flipped in JS |
| "15% discount text broken, falling off hero" | hero-corner-tl positioned at top:5vw, font 0.7rem | Remove hero-corner-tl entirely |
| "Initial text on LEFT (hero side), back to stone age" | `justify-content: flex-end` in RTL puts hero text on LEFT, same side as lady canvas (dx=0) → overlap during circle-wipe transition | Change to `justify-content: flex-start` so hero text is on RIGHT in RTL, LEFT in LTR (opposite of lady) |
| "Titles not readable" | Section `translateY(-50%)` centering + 4 product cards (2×2 grid) makes section taller than viewport → title scrolls off top | Change to `translateY(-35%)` AND reduce to 2 product cards per section |
| "Stats section + annoying Free Shipping text on hero should not exist" | Stats section (6 stat cards) + hero-meta-row ("ارسال رایگان") + marquee ("ارسال رایگان / FREE SHIPPING") all present | Remove all three entirely. Make JS setup functions null-safe. |
| "Frames run out mid-scroll before reaching bottom" | `FRAME_SPEED = 2.0` → frames deplete at 50% scroll, canvas frozen for second half | Change to `FRAME_SPEED = 1.0` → frames map 1:1 to scroll (frame 0 at 0%, frame N at 100%) |
| "بر اساس فایل skill روش دیگه که فریم فریم نکنه از روش video scrub استفاده کنه مثل شرکت apple. یک فایل جدا درست کن" (use video scrub technique like Apple instead of frame-by-frame, make a separate file) | Two delivery formats requested: Path B (frame-based, canvas) AND Path A (video-scrub, `<video>`) | Build BOTH standalone HTMLs: `rokh-standalone.html` (Path B, 3.3MB, 150 frames) and `rokh-standalone-videoscrub.html` (Path A, 2.7MB, single MP4). Path A is 22% smaller and simpler, but only supports `object-position` for side-alignment (no asymmetric subject offset). |
| "فضای خالی بیین پنل ها و هیرو هنوز زیاده. پنلها رو عریض تر کن و به هیرو دست نزن" (too much empty space between panels and hero, make panels wider, don't touch the hero) | Panel padding too large (34vw) leaves dead space between panel edge and hero | Reduce padding progressively across breakpoints: 28vw default, 24vw at ≥1280px, 20vw at ≥1600px. Increase `max-width` of `.section-inner` accordingly. NEVER touch canvas/video CSS when adjusting panels — only panel padding. |
| "نام انگلیسی نباید حذف میشد. فضای مرده رو کمتر کن. دست به هیرو نزن" (English name shouldn't have been removed, reduce dead space, don't touch hero) | When fixing Persian spelling "ابوطالب مرادی" → "ابوالطالب مرادی", the bilingual `data-en="Abootaleb Moradi"` attribute was accidentally removed, breaking EN language toggle | Always preserve BOTH `data-fa` and `data-en` attributes when editing bilingual content. After the edit, run `grep 'data-en=' before == after` to confirm attribute count unchanged. |
| "حس می کنم صفحه سنگین شده. سرچ کن ببین چه راهکارهایی وجود داره. 2.6 مگابایت خیلی زیاده" (page feels heavy, search for solutions, 2.6MB is too much) | base64-inlined MP4 (1.57MB raw → 2.05MB base64) dominates standalone HTML size. CRF mode can't target specific file sizes. | Switch to 2-pass VBR encoding: `-b:v 600k -pass 1/2 -preset slow -g 10 -keyint_min 10 -pix_fmt yuv420p -movflags +faststart`. Result: 0.74MB MP4 → 1.52MB standalone (41% smaller). Solo visual assessment confirms "acceptable for luxury cosmetics" even at 300kbps. |
| "گزینه 1 که هیچ تفاوتی نمی کنه. اگه اینترنت کاربر کند باشه صفحه خیلی دیر بالا میاد" (Option 1 [external video file] makes no difference, on slow internet page loads very late) | Splitting HTML from MP4 doesn't help if total download is the same — browser still needs both files before scrubbing works. The real bottleneck is total bytes, not file count. | The correct solution is to reduce total bytes via 2-pass compression, not to split files. External video is only useful when video streaming (range requests) is supported, which doesn't apply to local standalone HTML. |
| "رزولوشنش خیلی کم شد. فایده نداره. یک دونه 1290k (اصلی) هم درست کن" (resolution dropped too much, useless, also make a 1290k original version) | 300kbps ultra-light version was too aggressive — Solo assessment looked "acceptable" but the user disagreed on actual viewing. Single-bitrate delivery is fragile. | Deliver THREE standalone HTML files at different bitrates: `*-original.html` (1290kbps source, 2.59MB), `*.html` (600kbps recommended, 1.52MB), `*-300k.html` (300kbps ultra-light, 1.04MB). Let user pick the right balance for their audience. |
| "داداش نمیشه مدرن تر درست کنی؟؟ خیلی رو اعصابه فضای خالی توی صفحات خیلی زیاده به خاطر نمی دونم چی" (can't you make it more modern? too much empty space on pages) | Floating card padding (56px 48px), huge headings (5rem), and oversized margins created excessive dead whitespace inside cards and wide gaps | Redesign with High-Density Bento Grid & compact luxury styling: padding 22px 20px, margins 8-14px, heading 1.8-2.1rem. Do NOT shrink scroll container below 440vh. |
| "the 3d showcases got broken. correct them. rokh got stucked. bmw the incoming menues and navbar texts got outside of the borders" | Global `$` selector broken when replaced with `$$`, and navbar had no top clearance for BMW title | Fixed selector `const $$ = (sel) => Array.from(document.querySelectorAll(sel));`, restored 1:1 scrub, added header clearance padding. |
| "for bmw showcase the navbar on farsi is left and in english went to right. also the navbar covered the text BMW PARTS PRO" | Standalone floating back button at `top: 20px; right: 20px;` collided with brand logo on RTL and LTR, creating the illusion of navbar jumping sides; hero title lacked `padding-top` clearance | Remove floating button; embed `.portal-chip` inside 3-column header. Add `padding-top: calc(var(--header-h) + 36px)` to hero container. |
| "on rokh all pop up menues cropped and not truely shown by scroll" | 850px card height exceeded 600px laptop viewport due to 1:1 square product images and 5rem headings; Lenis hijacked wheel events so user couldn't scroll card content | Apply High-Density Compact Layout (height ≤ 420px, images 105px contain). Add `data-lenis-prevent="true"` to `.section-inner`. Remove negative insets to fix horizontal scrollbar. |
| "in all 3d showcases the scrolling cause instant transition between cards and not visually satisfactory" | Binary opacity jump (0 to 1), non-overlapping section ranges with dead gaps, and 0.02 fast fade-out caused cards to abruptly pop in and pop out | Implement Continuous Parallax Crossfade Engine with 3-phase Smoothstep interpolation (+36px glide-in, hold, -32px glide-out) and 4-6% overlapping section ranges. |


---

## Performance Budget

| Metric | Target | Measurement |
|--------|--------|-------------|
| Initial HTML | ≤ 50KB | `du -sh index.html` |
| CSS | ≤ 60KB | `du -sh css/style.css` |
| JS | ≤ 30KB | `du -sh js/app.js` |
| Video file (Path A, original quality) | ≤ 2MB | `du -sh video/output.mp4` |
| Video file (Path A, 600kbps recommended) | ≤ 0.8MB | `du -sh video/output_compressed.mp4` |
| Video file (Path A, 300kbps ultra-light) | ≤ 0.4MB | `du -sh video/output_300k.mp4` |
| Frames total (Path B) | ≤ 3MB | `du -sh frames/` |
| Product images | ≤ 500KB | `du -sh img/products/` |
| Standalone HTML (original bitrate) | ≤ 2.7MB | `du -sh rokh-standalone-videoscrub-original.html` |
| Standalone HTML (600kbps recommended) | ≤ 1.6MB | `du -sh rokh-standalone-videoscrub.html` |
| Standalone HTML (300kbps ultra-light) | ≤ 1.1MB | `du -sh rokh-standalone-videoscrub-300k.html` |
| ZIP deliverable | ≤ 4MB | `du -sh <brand>-website.zip` |
| Time-to-first-frame | ≤ 1.5s | Playwright: `wait_for_selector("#loader.hidden")` |
| Time-to-interactive | ≤ 3s | All frames loaded + ScrollTrigger ready |
| Console errors | 0 | Playwright `page.on("console")` |
| Console warnings | 0 | Same as above |
| LCP (Largest Contentful Paint) | ≤ 2.5s | Playwright/Lighthouse trace on the hero |
| Reduced-motion variant works | Content visible, no scroll-jack | Playwright `page.emulate_media(reduced_motion="reduce")`, screenshot at 3 scroll depths |

**Multi-bitrate delivery guidance:** which tier(s) to build is now decided **upfront at Step 0**, not only reactively after a "too heavy" complaint. When the user picks **Robust/compressed**, build and deliver the recommended (600kbps-equivalent) tier directly. When they pick **Heavy/cinematic**, build the original-quality tier but still mention the lighter option exists. When they pick **Network-adaptive**, build both the light and recommended sources and wire up the runtime loader from `references/qc-and-compression-research.md` §3 instead of shipping three separate HTML files. Falling back to the old reactive flow (deliver one version, wait for a complaint, then build a lighter one) is still fine if Step 0 was skipped because the file passed QC cleanly — see "Multi-bitrate delivery" section under Step 4a for the bitrate ladder and ffmpeg commands.


---

## Content Rules (e-commerce)

When building an e-commerce site, follow the Charlotte Tilbury / Dior Beauty pattern — pure conversion copy, NO poetry.

| Section | MUST contain | MUST NOT contain |
|---------|--------------|------------------|
| Hero | Brand name, tagline (1 sentence), primary CTA, shipping threshold | Philosophy, etymology, year founded |
| Bestsellers | 2-4 product cards with name, price, badge, Add to Bag | "Selected by our atelier" prose |
| Fragrances | 4 product cards: name, for Her/Him/Unisex, ml size, price, Add to Bag | "Distilled over 80 days" poetry |
| Makeup | 4 product cards same structure | Color theory essays |
| Skincare | 4 product cards same structure | "Rituals of radiance" |
| Stats | E-commerce metrics: # products, # customers, rating, dispatch time | Abstract moisture percentages |
| Spotlight | Single hero product with image + metadata + CTA | Poetic descriptions |
| Nav | Flat 7-link: Home, Bestsellers, Fragrances, Makeup, Skincare, Spotlight, Contact | Hover dropdowns |

**Product card template:**

```html
<article class="product-card">
  <div class="product-badge" data-fa="پرفروش" data-en="BESTSELLER">پرفروش</div>
  <div class="product-image">
    <img src="img/products/lipstick-liquid.webp" alt="Velvet Matte Liquid Lip">
  </div>
  <h3 class="product-name">Velvet Matte Liquid Lip</h3>
  <p class="product-desc" data-fa="رژ لب مات مخملی" data-en="Matte velvet liquid lipstick">رژ لب مات مخملی</p>
  <div class="product-price">
    <span class="price-now">۸۹۰٬۰۰۰ ت</span>
    <span class="price-was">۱٬۱۰۰٬۰۰۰ ت</span>
  </div>
  <a href="#" class="cta-button cta-small" data-fa="افزودن به سبد" data-en="Add to Bag">افزودن به سبد</a>
</article>
```

**Badges**: BESTSELLER (gold), NEW (rose-gold), SALE (burgundy). One badge per card maximum.

**Prices**: always in Toman for Iranian brands, formatted with `toLocaleString("fa-IR")` for Persian digits.


---

## Deliverable Structure

```
/home/claude/<brand>-website/
├── index.html              (or single-file bundle with inlined CSS/JS)
├── favicon.svg
├── css/
│   └── style.css           (if multi-file)
├── js/
│   └── app.js              (if multi-file)
├── video/                  (Path A only)
│   └── output.mp4
├── frames/                 (Path B only)
│   ├── poster.webp
│   ├── frame_0001.webp
│   └── ... (150 frames)
└── img/
    └── products/
        ├── lipstick-liquid.webp
        └── ... (12-20 product images, e-commerce only)
```

Zip it, then copy the deliverable(s) to the outputs directory and present them to the user:

```bash
cd /home/claude
zip -rq <brand>-website.zip <brand>-website/
cp <brand>-website.zip /mnt/user-data/outputs/
# for a standalone single-file build, just copy the .html directly instead:
# cp <brand>-website/index.html /mnt/user-data/outputs/<brand>-website.html
```

Then call `present_files` on whatever landed in `/mnt/user-data/outputs/` — a file that's written but never presented has no file card and is unreachable by the user, especially on mobile.

---

## Troubleshooting

- **Frames not loading**: Must serve via HTTP, not `file://` (or use single-file bundle with base64 data URIs).
- **Choppy scrolling**: Increase `scrub` value, reduce frame count.
- **White flashes**: Ensure all frames loaded before hiding loader.
- **Blurry canvas**: Apply `devicePixelRatio` scaling to canvas dimensions.
- **Lenis conflicts**: Ensure `lenis.on("scroll", ScrollTrigger.update)` is connected.
- **Counters not animating**: Verify `data-value` attribute exists and snap settings match decimal places.
- **Memory issues on mobile**: Reduce frames to <150, resize to 1280px wide.
- **FFmpeg not found**: Install via `brew install ffmpeg` (macOS), `apt install ffmpeg` (Linux), or download from ffmpeg.org (Windows).
- **Section invisible / overlapping at top**: You used `top: Xvh` instead of `top: X%`. Switch to `%`.
- **Lady's head cropped**: You used cover mode (`Math.max`). Switch to contain mode (`Math.min`).
- **Visible vertical seam between canvas and panel**: bg colors don't match. Hardcode same hex in both `CANVAS_BG_FALLBACK` (JS) and `--bg-dark` (CSS).
- **Sections overlap during scroll**: fade-out range too slow. Use fixed 4% fade range.
- **15% discount text falling off hero corner**: Remove `.hero-corner-tl` entirely. Put promos in `.hero-meta-row`.
- **Subject doesn't move on language toggle**: Add `setTimeout(() => drawFrame(state.currentFrame), 60)` in `applyLanguage()`.
- **Popup menu cropped at bottom on laptops**: Section card height exceeds 600px. Apply High-Density Compact Layout: headings `clamp(1.45rem, 2.2vw, 2.05rem)`, product images `height: 105px; object-fit: contain;`, card padding `22px 20px`. Total card height MUST stay ≤ 420px.
- **Horizontal scrollbar at bottom of card**: A pseudo-element has negative insets (e.g. `inset: -40px -32px`). Remove negative insets and add `overflow-x: hidden;` to `.section-inner`.
- **Card content cannot be scrolled by mouse wheel**: Lenis smooth-scroll is hijacking wheel events. Add `data-lenis-prevent="true"` to the scrollable container.
- **Cards pop in or pop out instantly**: Binary opacity setting and non-overlapping section ranges. Implement the 3-phase Smoothstep Parallax Crossfade Engine and overlap adjacent section ranges by 4–6%.
- **Navbar covers hero title**: Hero container lacks clearance for fixed header. Add `padding-top: calc(var(--header-h) + 36px);` and adjust title font-size to `clamp(3.2rem, 7.5vw, 6.5rem)`.


---

## Final Definition of Done

The deliverable is complete when ALL of the following are true:

1. ✅ Site opens with no console errors or warnings in Chromium.
2. ✅ Loader hides within 3 seconds on a clean cache.
3. ✅ All frames load successfully (Path B) / video seeks smoothly (Path A).
4. ✅ Scroll-scrubbing the canvas/video feels smooth (60fps, no jank).
5. ✅ Hero text reveals cleanly on load (no flash of unstyled text).
6. ✅ Language toggle swaps all text AND moves the canvas subject to the opposite side (if bilingual).
7. ✅ Section panels never overlap the canvas subject at any viewport ≥ 900px wide.
8. ✅ Section fade transitions are clean — no two sections visible simultaneously.
9. ✅ All product images render and were visually vetted (no people, clean bg) — see `references/workflow-detail.md` §5.2.
10. ✅ You (via the `view` tool) reviewed all 16+ screenshots yourself and found 0 issues — see `references/workflow-detail.md` §10b.
11. ✅ Deliverable ≤ 4MB (or matches the tier the user explicitly accepted at Step 0's "Heavy/cinematic" choice).
12. ✅ Final standalone HTML (and any source files, if multi-file was requested) copied to `/mnt/user-data/outputs/`.
13. ✅ `present_files` called on the deliverable so it renders as a file card for the user — a file written but never presented is not visible to them.
14. ✅ Step 0 pre-flight QC was run and, if triggered, the user's weight decision (heavy / robust / network-adaptive) was honored in what got built.
15. ✅ `prefers-reduced-motion` fallback verified via Playwright — content is visible and readable with animation disabled, no scroll-jack.
16. ✅ If a Three.js/WebGL 3D scene was added (Step 6b), it was explicitly requested (not assumed from the word "3D" alone) and LCP still meets budget.


---

## Reference files

- `references/workflow-detail.md` — full step-by-step build instructions: exact `ffmpeg`/`ffprobe` commands, complete HTML/CSS/JS code for Steps 3–10 (Step 9, `js/app.js`, is the largest section). Read this before executing any step past the Step 1/2 decisions above.
- `references/templates-and-scripts.md` — copy-paste CSS variable blocks, JS CONFIG blocks (Path A and Path B), and recovery scripts for known failure modes.
- `references/lessons-learned.md` — three annotated production postmortems (Path B frame-based, Path A video-scrub, multi-bitrate compression) explaining *why* each MUST/FORBIDDEN rule above exists. Read when a bug feels familiar, or when a user pushes back on a rule and you need the original reasoning.
- `references/qc-and-compression-research.md` — **new in v2.** The Step 0 QC thresholds and `ffprobe` script, the full compression-ladder recipes (with 2026 codec benchmarks and citations), the network-adaptive loader snippet, the `prefers-reduced-motion` fallback pattern, and the Core Web Vitals case against unscoped "3D"/scroll-jacking. Read this before executing Step 0, and again before Step 4a/4b if the user picked "robust" or "network-adaptive."
- `improvements.txt` (skill root) — the audit this v2 fork was built from: what the original skill already did well, where it fell short of current best practice, what changed and why, and the sources consulted. Read this if you want the *reasoning* behind the v2 changes rather than just the instructions.
