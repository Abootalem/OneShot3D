# QC & Compression Research (v2 addition)

This file backs the Step 0 gate in `SKILL.md`. It has four parts: (1) the pre-flight QC thresholds and probe script, (2) compression-ladder recipes for the three Step 0 weight decisions, (3) the network-adaptive runtime loader, (4) the `prefers-reduced-motion` fallback pattern, and (5) the Core Web Vitals case for scoping "3D"/scroll-jacking carefully. Sources consulted are listed at the end of each part rather than inline — this is a working reference, not a report, so keep additions here terse and actionable.

---

## 1. Pre-flight QC: thresholds and probe script

Run this the moment a video file path is provided, before Step 1 research starts:

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,duration,r_frame_rate,codec_name,bit_rate,pix_fmt \
  -show_entries format=size,bit_rate,duration \
  -of json "$VIDEO_PATH"

# Human-readable size + aspect ratio
du -h "$VIDEO_PATH"
python3 -c "
w, h = WIDTH, HEIGHT   # fill in from ffprobe output above
from math import gcd
g = gcd(w, h)
print(f'{w}x{h}  aspect {w//g}:{h//g}  ({\"portrait\" if h > w else \"landscape\"})')
"
```

**Thresholds:**

| Signal | Comfortable — skip the question | Moderate/Large — ask Step 0 question |
|---|---|---|
| File size | < 15MB | 15–50MB = moderate, > 50MB = large |
| Codec/container | H.264, H.265, VP9, or AV1 in MP4/MOV/WebM | ProRes, DNxHD, MPEG-2, unusual/rare containers — needs transcode regardless of size |
| Aspect ratio | 9:16 to 21:9 | Taller than 9:16 or wider than 21:9 — flag for `object-fit: contain` handling per Anti-patterns #2/#41, independent of the size decision |
| Bitrate-for-resolution | Roughly ≤ 8 Mbps at 1080p-equivalent for a short (<30s) clip | Bitrate-bloated sources (e.g. >12 Mbps at 1080p) — re-encoding shrinks with no visible loss, mention this even if the user picks "heavy" |
| Duration | Any — duration doesn't gate the question by itself, but a long clip (>60s) used for full-scroll scrubbing multiplies every downstream size number, so call it out if it appears alongside a moderate/large file size |

If every signal lands in the "comfortable" column, state the numbers in one line and move on — don't manufacture a decision the user doesn't need to make. This mirrors the general web-performance guidance to treat a **performance budget as a product requirement decided up front**, not a cleanup pass after the fact — the same principle Step 0 applies to a single video file instead of a whole site. (Source: youcom.agency, "How to Make Your Website Faster: Practical Performance Best Practices for 2026".)

**The Step 0 question** (verbatim structure to use, adapt numbers to the actual probe result):

> This video is [X]MB, [codec] at [WxH] ([aspect ratio])[, plus an audio track that the site will never play — I'll strip that either way]. That's [comfortable/moderate/large] for a scroll-scrub site. Do you want:
> 1. **Heavy/cinematic** — keep it close to source quality (~[Y]MB final page, best on wifi/fast mobile)
> 2. **Robust/compressed** (recommended) — I compress it to the recommended tier (~[Z]MB final page), works well on slower connections too
> 3. **Network-adaptive** — I build both and the site picks automatically based on the visitor's connection
> 4. You re-upload a smaller/pre-compressed file yourself

---

## 1b. Strip audio unconditionally — free, before any tier decision

This applies **regardless of which Step 0 tier the user picks**, including "heavy/cinematic." It's not a size/quality trade-off like the recipes in §2 below — it's a pure win with zero visible or audible cost, and it should happen automatically, not be offered as a choice.

Every scrub-video build in this skill sets `muted` on the `<video>` element (Anti-pattern #34 — iOS autoplay policy requires it, and a scroll-scrubbed background video was never meant to have audible sound in the first place). That means any audio stream in the source file is dead weight: it downloads, it gets base64-inlined if standalone, and it is never once heard. The original skill's Path A ffmpeg commands in `references/workflow-detail.md` already strip it (`-an`), which is correct — the miss in earlier drafts of this v2 file was not carrying that same `-an` flag into the new tier recipes below, and not calling it out as its own step. Both are fixed now.

Action: as part of the Step 0 probe, note whether an audio stream exists (`ffprobe ... -show_streams` includes an `audio` stream entry if so). If it does, every re-encode command in §2 below — heavy, robust, or either network-adaptive tier — includes `-an`. This is not conditional on the user's tier choice. On a source with a substantial audio track (e.g. music or narration baked into an export), this alone can be a meaningful size cut before any video-quality trade-off is even considered, and it's worth mentioning to the user as a free win in the Step 0 message ("your file also has an audio track that the site will never play — I'll strip that regardless of which option you pick").



These extend, not replace, the existing "Multi-bitrate delivery" ffmpeg commands in `references/workflow-detail.md` Step 4a. Use whichever recipe matches the Step 0 answer.

### Heavy / cinematic
Re-encode only to fix codec/container/GOP issues, not to shrink aggressively. For Path A (video-scrub): `-c:v libx264 -crf 20 -preset slow -g 10 -keyint_min 10 -pix_fmt yuv420p -movflags +faststart -an`. CRF 18–20 is the documented sweet spot for high-fidelity 1080p web delivery. (Source: superrendersfarm.com, "H.264 vs H.265: Video Encoding Guide"; mux.com, "How to compress video files while maintaining quality with ffmpeg".)

### Robust / compressed (recommended default)
Use 2-pass VBR, not CRF, when you need a predictable target size — CRF gives constant quality with an unpredictable size, 2-pass VBR gives a predictable size with the encoder distributing bits intelligently across the two passes. This was already the hard-won lesson from the ROKH multi-bitrate case study (`references/lessons-learned.md`) and it's consistent with current guidance. (Source: vibbit.ai, "FFmpeg CRF Examples: H.264, H.265, VP9 & AV1 (2026)".)

```bash
ffmpeg -i input.mp4 -c:v libx264 -b:v 600k -pass 1 -preset slow -g 10 -keyint_min 10 -pix_fmt yuv420p -f mp4 -an /dev/null -y
ffmpeg -i input.mp4 -c:v libx264 -b:v 600k -pass 2 -preset slow -g 10 -keyint_min 10 -pix_fmt yuv420p -movflags +faststart -an output_compressed.mp4
```

A useful rule of thumb from current guidance: CRF 23–25 H.264 typically gets a 50–70% size reduction before quality loss becomes visible on typical displays — use that as a sanity check on the 2-pass target, not as the encoding mode itself. (Source: mux.com, "How to compress video files while maintaining quality with ffmpeg".)

### Network-adaptive
Build **two** encodes — a "light" tier (≈300kbps, same recipe as above with `-b:v 300k`) and the "recommended" tier (600kbps) — and let the runtime loader in §3 pick. Skip building a third "heavy" tier for this path; if the user wanted maximum fidelity they'd have picked "heavy" instead of "network-adaptive."

### Codec choice — measure, don't assume
The original ROKH case study concluded AV1/VP9 weren't worth it for scrub video: AV1 encoded 5–10× slower than H.264 with similar output size, and VP9's intra-frame prediction made scrubbing (random-access seeking) slower than H.264's short-GOP setup (`references/lessons-learned.md`, and Anti-patterns #46/#47). **That conclusion still holds as the safe default** — H.264 with `-g 10 -keyint_min 10` remains the right choice for scrub/random-access video specifically.

But the underlying encoder landscape has moved since that test was run: SVT-AV1 is now a practical encoder, roughly 3–8× slower than H.264 (not 20–50× like the older libaom-av1 encoder), and hardware AV1 decoding is now common on flagship phones and Apple Silicon — though older devices still need an H.264 fallback. (Source: ffhub.io, "FFmpeg Video Compression: Best Practices Guide"; red5.net, "AV1 vs H.264 in 2026".) Current industry survey data still shows H.264 far more deployed than AV1 in production (84% vs 17%), though a large share of teams plan to add AV1 in 2026. (Source: red5.net, "AV1 vs H.264 in 2026", citing NETINT's 2026 State of Video Encoding survey.)

**Rule:** keep H.264 short-GOP as the scrub source by default. Only test SVT-AV1 as an *additional* `<source>` (with H.264 fallback) if the user explicitly wants the smallest possible file and accepts one extra encode pass — and actually measure the result on the specific clip rather than assuming either the old "AV1 doesn't help" or the new "AV1 is now fine" conclusion. Both are true only sometimes; the content and length of the clip changes the answer.

For Path B (frame extraction), the existing WebP q=65 guidance stands — Anti-pattern #48 already tested AVIF and found it visibly softened hair/fabric detail at a marginal size win. No new evidence changes that for this skill's typical natural-image (people/product) frame content.

---

## 3. Network-adaptive runtime loader (Path A only)

Only build this when the user picks "network-adaptive" at Step 0. It lets the deliverable pick the light or recommended video source based on the *visitor's* measured connection, using the Network Information API. Support is Chromium-based only (no Safari/Firefox) — always default to the "recommended" tier when the API is absent so non-Chromium visitors still get a reasonable experience rather than the heaviest one. (Source: developer.mozilla.org, "Network Information API"; web.dev, "Adaptive loading: improving web performance on slow devices".)

```javascript
function pickVideoTier() {
  const conn = navigator.connection || navigator.webkitConnection || navigator.mozConnection;
  if (!conn) return "recommended"; // Safari/Firefox: no API, use the safe default
  if (conn.saveData) return "light"; // user explicitly asked to save data
  if (["slow-2g", "2g", "3g"].includes(conn.effectiveType)) return "light";
  return "recommended";
}

const tier = pickVideoTier();
video.src = tier === "light" ? "video/output_light.mp4" : "video/output_compressed.mp4";

// Optional: react to a live connection change (e.g. wifi -> cellular) before the video
// has started loading. Don't swap mid-playback/mid-scroll — only before first load.
if (navigator.connection && !video.readyState) {
  navigator.connection.addEventListener("change", () => {
    if (!video.currentSrc || video.readyState === 0) {
      video.src = pickVideoTier() === "light" ? "video/output_light.mp4" : "video/output_compressed.mp4";
    }
  });
}
```

For a standalone single-file HTML (the default delivery format), base64-inline **both** tiers and swap the `src` via a data-URI variable rather than a file path — the loader logic is identical, only the two constants change from paths to `data:video/mp4;base64,...` strings. This roughly doubles the media payload in the HTML itself (both tiers are embedded), so mention that trade-off to the user: network-adaptive means the file you download is bigger, but what any individual visitor actually has to *play* is right-sized for them.

---

## 4. `prefers-reduced-motion` fallback

Required for every build (checklist item #24, DoD item #15), not just when the user asks. This is a low-effort, high-impact accessibility fix with excellent modern browser support, and scroll-jacked/parallax pages are a specifically documented trigger for users with vestibular disorders. (Source: css-tricks.com, "prefers-reduced-motion"; blog.openreplay.com, "Using prefers-reduced-motion for Accessible Animation"; blog.pope.tech, "Designing accessible animation and movement on your website".)

**CSS — belt-and-suspenders layer** (catches anything the JS check misses):

```css
@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto !important; }
  .section-panel, .hero-text, .marquee-text { animation: none !important; transition: none !important; }
}
```

**JS — the layer that actually matters for this skill**, since Lenis and ScrollTrigger are JS-driven, not CSS animations:

```javascript
const prefersReducedMotion = window.matchMedia("(prefers-reduced-motion: reduce)").matches;

if (!prefersReducedMotion) {
  // existing Lenis init + ScrollTrigger scroll-scrub + stagger entrance code goes here, unchanged
} else {
  // reduced-motion path: no Lenis (native scroll), no scroll-scrubbing.
  // Path A: play the video normally (autoplay+muted+loop) instead of seeking it on scroll,
  //         OR show the poster frame only — either is acceptable, pick whichever reads better for the brand.
  // Path B: show a single representative frame (e.g. the midpoint) as a static background instead of a canvas sequence.
  // All section content must still be visible without needing scroll-triggered opacity/transform —
  // set panels to their "revealed" end-state by default (opacity:1, no transform) instead of hiding them
  // and relying on GSAP to reveal them, since GSAP itself is skipped in this branch.
}
```

Use a small duration (e.g. `0.01ms`) rather than `0` if any code still listens for a `transitionend`/`animationend` event in the reduced-motion branch — a true zero-duration transition can skip firing that event in some browsers. Only relevant if you kept any CSS transitions active in the reduced-motion path. (Source: blog.openreplay.com, "Using prefers-reduced-motion for Accessible Animation".)

Verify with Playwright: `page.emulate_media(reduced_motion="reduce")`, then screenshot at 0%, 40%, 80% scroll and confirm every section's text is visible and legible without needing to trigger any animation first.

---

## 5. Why "3D" and scroll-jacking need scoping, not blanket enthusiasm

Current 2026 web-performance commentary specifically calls out interactive 3D, oversized variable type, and scroll-jacking animation as trend-driven Core Web Vitals risks: dropping an unoptimized WebGL hero or autoplay video into the LCP slot pushes Largest Contentful Paint past its 2.5s target, and scroll-jacking runs on the main thread — the same thread the browser needs to respond to a tap — so it degrades responsiveness (INP) on lower-end devices even when it looks great in a demo reel. The same source cites a well-known finding that a one-second delay can cut conversions by roughly 7%, and a real-world case (Rakuten) where cleaning up LCP produced a 53% lift in revenue per visitor. (Source: jacobtyler.com, "2026 Web Design Trends Are a Core Web Vitals Problem".)

This is the direct justification for two v2 decisions:
1. **Don't add a true WebGL/Three.js 3D scene by default.** This skill's existing canvas/video parallax output already reads as "3D-ish" (depth via layered panels + subject offset + scroll-driven motion) without shipping a rendering engine. Only build actual Three.js geometry when the user confirms, opt-in, via Step 6b — and re-check the Performance Budget afterward, specifically LCP, since WebGL context creation is exactly the kind of main-thread cost the source above warns about.
2. **Scroll-jacking (this skill's core technique) needs the `prefers-reduced-motion` fallback from §4 as a non-negotiable, not a nice-to-have.** The skill's entire premise is a scroll-jacked/pinned canvas-scrub experience, so the accessibility and performance mitigation has to be built in by default rather than bolted on when someone complains.

---

## Sources consulted (v2 research pass)

- superrendersfarm.com — "H.264 vs H.265: Video Encoding Guide for 3D Artists" (2026)
- red5.net — "AV1 vs H.264 in 2026: Quality, Speed, Cost, Compatibility"
- transloadit.com — "How to compress video with FFmpeg" (2026)
- mux.com — "How to compress video files while maintaining quality with ffmpeg"
- vibbit.ai — "FFmpeg CRF Examples: H.264, H.265, VP9 & AV1 (2026)"
- ffmpeglab.com — "Video Compression with FFmpeg – Complete Guide to Reducing File Size" (2026)
- ffhub.io — "FFmpeg Video Compression: Best Practices Guide"
- ezwebtools.net — "Video Compression Complete Guide: From Principles to FFmpeg Practical Use" (2026)
- jacobtyler.com — "2026 Web Design Trends Are a Core Web Vitals Problem" (2026)
- involvedigital.com — "Core Web Vitals & Website Performance Guide 2026"
- youcom.agency — "How to Make Your Website Faster: Practical Performance Best Practices for 2026"
- smashingmagazine.com — "Respecting Users' Motion Preferences"
- css-tricks.com — "prefers-reduced-motion"
- blog.pope.tech — "Designing accessible animation and movement on your website" (2025)
- blog.openreplay.com — "Using prefers-reduced-motion for Accessible Animation" (2026)
- web.dev — "Adaptive loading: improving web performance on slow devices"; "Adaptive serving based on network quality"
- developer.mozilla.org — "Network Information API"
- addyosmani.com — "Adaptive Serving using JavaScript and the Network Information API"
