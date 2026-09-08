# Workflow — Full Step-by-Step Detail

Full instructions for building the site: exact ffmpeg commands, HTML/CSS/JS code, and the QA process. Read this after deciding the scrub method in SKILL.md Step 2.

**Contents** (search for these headings, in order):
Step 1: Research Similar Websites · Step 2: Choose Scrub Method · Step 3: Analyze the Video ·
Step 4a: Prepare Video Asset (Path A) · Step 4b: Extract Frames (Path B) · Step 5: Asset Polish ·
Step 6: Scaffold · Step 7: Build index.html · Step 8: Build css/style.css ·
Step 9: Build js/app.js (largest section — canvas render loop, ScrollTrigger, config) ·
Step 10: QA Loop (Playwright + view tool)

---

### Step 1: Research Similar Websites (MANDATORY — do this before designing anything)

This step is a hard gate, not a nice-to-have. Do not touch ffprobe/ffmpeg or write any HTML/CSS until Step 1's deliverable (the synthesis in point 4) exists.

1. **Identify the category.** From the video content, theme/brand name, and any design direction the user gave, infer the product/industry category (e.g. "SaaS analytics dashboard," "DTC skincare," "fitness app," "architecture studio," "luxury cosmetics e-commerce").
2. **Search for top examples.** Use web search (and `web_fetch` on promising results) for queries like:
   - `"<category>" award winning website`
   - `"<category>" scroll animation site`
   - `best "<category>" website design 2026`
   - `<competitor/brand name if known> website`
   - Sites like Awwwards, Godly, Land-book, or Siteinspire are good hunting grounds — search `site:awwwards.com <category>` or similar.
3. **Pull 3-5 reference sites** and note for each:
   - Overall layout/structure (hero style, section flow, navigation pattern)
   - Color palette and typography feel
   - Standout interaction/animation moments worth adapting (not copying)
   - What makes it feel premium vs. generic
4. **Synthesize a point of view** before Step 6 (Scaffold): 1-2 sentences on which direction this project should lean toward, and 2-3 concrete tactics borrowed from the references (e.g. "lean into oversized serif type like site Y," "Charlotte Tilbury e-commerce pattern: product names + prices + Add to Bag, NO poetry"). Do not default to a horizontal marquee — that element is opt-in only now (see §9g).
5. **Never copy verbatim.** Reference sites inform layout patterns, palette moods, and animation ideas — not exact copy, exact assets, or reproduced code. Treat this like moodboarding, not cloning.

If the user already gave explicit design direction (specific colors, fonts, a named site to emulate), do a lighter-touch version of this step — confirm/validate against 1-2 references rather than a full search — and move on quickly. Even in this shortened path, state the point of view (point 4) explicitly before Step 6.

**Checkpoint before proceeding:** you must be able to state, in 1-2 sentences, the design direction and 2-3 borrowed tactics before starting Step 3. If you can't, the research wasn't done.

### Step 2: Choose Scrub Method (video-scrub default, frame-extraction for side-panel layouts)

**Know this before extracting a single frame:** exploding a video into independently-compressed stills throws away the video codec's temporal compression (it only stores what *changes* between frames). A frame sequence will almost always be many times larger than the source video, because each frame is compressed alone instead of relative to its neighbors. This is an architecture choice, not a compression-tuning problem.

#### Two valid paths — choose based on layout needs

**Path A — Video-scrub (default, lighter, simpler):**
- A single compressed `<video>` file, paused, with `currentTime` bound to scroll position via `requestAnimationFrame`/ScrollTrigger.
- Stays close to the source file's size (~1-2MB typically).
- Simpler to build — no preloader, no canvas renderer, no per-frame bg sampling.
- **Constraint**: the video plays as a **full-bleed background**. You cannot offset the subject off-center because the video fills the viewport.
- **Use when**: the layout is full-bleed video + overlay text (Apple product page style).

**Path B — Frame extraction (heavier, but supports side-panel layouts):**
- 150 WebP frames extracted at 15fps, drawn on a `<canvas>` via `drawImage()`.
- Canvas lets you position the frame at any `(dx, dy)` — so you can offset the subject to one side, leaving room for section panels on the other side.
- **Constraint**: heavier payload (~2-3MB frames vs ~1-2MB video), more complex preloader.
- **Use when**: the layout has hero on one side, section panels on the other (luxury e-commerce pattern with product cards).

**Decision rule:**
- If the design needs section panels overlapping the video area → **Path B** (frame extraction).
- If the design is full-bleed video with text overlay → **Path A** (video-scrub).
- If unsure → start with Path A; switch to Path B only if the layout requires subject offset.

#### Path A — Video-scrub workflow

```bash
# Re-encode with short GOP for responsive seeking
ffmpeg -i input.mp4 \
  -vf "scale=<WIDTH>:-1" \
  -c:v libx264 -preset slow -crf 24-26 \
  -g 10-15 -keyint_min 10-15 \
  -pix_fmt yuv420p \
  -movflags +faststart \
  -an \
  output.mp4
```

Then bind scroll progress to `video.currentTime` (paused `<video>` element, muted, `playsinline`) instead of drawing canvas frames — same `FRAME_SPEED`-accelerated progress math as Path B, just applied to `video.currentTime` instead of a frame index.

Sanity-check the result against the source: `du -h` both files; the scrub-ready encode should land at or near the source size, not many times larger.

Skip Steps 4b, 5.1, 5.3, 9b, 9c — no frame extraction, no per-frame bg sampling, no preloader/canvas renderer. Go straight from Step 3 (analyze) to Step 6 (scaffold) with the video file as the only media asset.

#### Path B — Frame extraction workflow

Continue to Step 3 (probe) → Step 4b (extract) → Step 5 (compress + bg sample) → Step 6 (scaffold with canvas renderer in Step 9c).

### Step 3: Analyze the Video

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,duration,r_frame_rate,nb_frames,codec_name \
  -of csv=p=0 "<VIDEO_PATH>"
```

Capture into variables:
- `WIDTH`, `HEIGHT` → portrait (H > W) or landscape (W > H)?
- `FPS` → typically 30
- `DURATION_SEC`
- `TOTAL_FRAMES` = `FPS * DURATION_SEC`

**Decision rule for portrait vs landscape** (Path B only — Path A handles this automatically via CSS `object-fit`):
- Portrait video (e.g. 480×854) → `contain` canvas mode (see Step 9c), subject sits on one side of viewport.
- Landscape video → `cover` canvas mode, full-bleed background.
- Square video → `contain` mode with vignette.

### Step 4a: Prepare Video Asset (Path A — video-scrub only)

Already covered in Step 2 (Path A workflow). Skip to Step 5.2 for product image fetching.

#### Hardened ffmpeg command (post-ROKH Path A build)

```bash
# Re-encode with short GOP for responsive seeking
ffmpeg -i input.mp4 \
  -vf "scale=720:-2:flags=lanczos" \
  -c:v libx264 -preset slow -crf 24 \
  -profile:v high -level 4.0 \
  -g 10 -keyint_min 10 \
  -pix_fmt yuv420p \
  -movflags +faststart \
  -an \
  -y \
  video/output.mp4
```

**Why each flag (hardened by the ROKH Path A build):**
- `scale=720:-2`: 720px wide is the sweet spot for portrait scrubbing video. 480px looks soft on retina, 1080px doubles the bytes for invisible gain. Use `-2` (not `-1`) to ensure even dimensions (libx264 requirement).
- `-crf 24`: 22-26 range. Below 22 = 2× larger file for invisible quality gain. Above 26 = visible banding on skin tones.
- `-preset slow`: ~30% smaller than `-preset fast` for the same CRF. Worth the encode time.
- `-profile:v high -level 4.0`: best seeking performance. Baseline profile is for legacy iOS only — modern browsers all support High.
- **`-g 10 -keyint_min 10` (CRITICAL)**: forces a keyframe every 10 frames. Without this, ffmpeg defaults to `-g 250` and seeking to a non-keyframe position requires decoding up to 250 frames forward — causing visible lag/stutter during scroll-scrubbing. With `-g 10`, every seek lands within 10 frames of a keyframe, so seek time is <30ms.
- `-pix_fmt yuv420p`: required for browser compatibility. `yuv444p` produces smaller files but won't play in Safari.
- **`-movflags +faststart` (CRITICAL)**: moves the `moov` atom to the start of the file so the browser can begin playback/seeking immediately without downloading the entire file. Without this, base64 data URIs still work (file is local) but remote video streaming fails.
- `-an`: strip audio. Scrub video has no audio track, and including one triggers iOS autoplay policies.

#### Verify the output

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,duration,nb_frames,codec_name,profile \
  -of default=noprint_wrappers=1 video/output.mp4

du -h video/output.mp4   # target: ≤2MB
```

**Keyframe check (sanity):**
```bash
ffprobe -v error -select_streams v:0 \
  -show_entries frame=pict_type -of csv | head -30 | grep -c I
# Should show ~3 I-frames in first 30 frames (every 10th frame = keyframe)
```

If the file is >2MB, re-encode with `-crf 26` or `scale=640:-2`. If scrubbing feels laggy in browser test, your `-g` is too large — re-encode with `-g 5`.

#### Path A size budget

| Asset | Size | Notes |
|-------|------|-------|
| `video/output.mp4` | ~1.5-2MB | 720px wide, 10s, CRF 24 |
| Product images | ~25KB × 12 | Same as Path B |
| Standalone HTML (inlined) | ~2.5-3MB | Video as base64 adds ~33% overhead |
| **Total standalone** | **≤3MB** | vs Path B's ~3.3MB |

The base64 inline expansion is the trade-off: a 1.6MB MP4 becomes a 2.1MB data URI. For very large videos (>3MB), consider external file reference instead of inline — but standalone delivery requires inline for `file://` compatibility.

#### Multi-bitrate delivery (when user asks "page is too heavy, make it lighter")

When the user flags the standalone HTML as too heavy and asks for compression research, **do NOT just lower the CRF**. CRF mode gives constant quality but no size target. The correct tool is **2-pass VBR encoding** at a target bitrate — this is what streaming services actually use.

**Bitrate ladder (tested on ROKH 10s portrait video, verified by visual review):**

| Bitrate | MP4 size | Standalone HTML | Savings vs original | Verdict (solo view) |
|---------|----------|-----------------|---------------------|-------------------------|
| 1290kbps (source) | 1.57 MB | 2.59 MB | — | reference |
| 1200kbps 2-pass | 1.51 MB | 2.52 MB | 3% | "subtle loss" side-by-side |
| 1000kbps 2-pass | 1.27 MB | 2.16 MB | 17% | "acceptable for luxury" solo |
| 600kbps 2-pass | 0.74 MB | 1.52 MB | 41% | "acceptable for luxury" solo |
| 400kbps 2-pass | 0.51 MB | 1.08 MB | 58% | "acceptable for luxury" solo |
| 300kbps 2-pass | 0.39 MB | 1.04 MB | 60% | "acceptable for luxury" solo |

**Critical insight from testing**: side-by-side comparison makes a visual reviewer very picky — it will flag any bitrate below source as "visible loss." But **solo assessment** (showing only the compressed frame, not the original) gets "acceptable" ratings even at 300kbps. The soft-focus, high-contrast aesthetic of cosmetics/fashion video naturally hides compression artifacts. **Trust solo assessment over side-by-side** — users never see the original next to the compressed version.

**2-pass encode command (use this, NOT CRF, when targeting a specific size):**

```bash
# Pass 1 — analysis (output to /dev/null, just writes stats)
ffmpeg -y -i input.mp4 \
  -c:v libx264 -b:v 600k -pass 1 -preset slow \
  -g 10 -keyint_min 10 -pix_fmt yuv420p \
  -an -f mp4 /dev/null

# Pass 2 — actual encode with faststart
ffmpeg -y -i input.mp4 \
  -c:v libx264 -b:v 600k -pass 2 -preset slow \
  -g 10 -keyint_min 10 -pix_fmt yuv420p \
  -movflags +faststart \
  -an \
  output_600k.mp4

# Cleanup
rm -f ffmpeg2pass-*.log*
```

**Why 2-pass beats CRF for size-targeted compression:**
- CRF: constant quality, variable bitrate. No control over final size — same CRF on different content produces wildly different sizes.
- 2-pass: target bitrate, variable quality. Pass 1 analyzes complexity per-frame, pass 2 distributes bits intelligently (more bits to complex frames, fewer to simple ones). Same size, better perceptual quality than 1-pass VBR.

**Recommended bitrates by content type:**

| Content type | Min acceptable | Recommended | Notes |
|--------------|----------------|-------------|-------|
| Soft-focus cosmetics/fashion | 300kbps | 600kbps | Soft aesthetic hides artifacts |
| Sharp product (watch, jewelry) | 800kbps | 1200kbps | Hard edges show artifacts |
| Talking head / interview | 500kbps | 800kbps | Face is forgiving, background not |
| Landscape / nature | 1000kbps | 1500kbps | Foliage/water very artifact-prone |
| Animated / motion graphics | 400kbps | 700kbps | Flat colors compress extremely well |

**Deliver multiple bitrates as separate standalone HTML files:**
- `rokh-standalone-videoscrub-original.html` (source bitrate, full quality)
- `rokh-standalone-videoscrub.html` (600kbps, recommended balance)
- `rokh-standalone-videoscrub-300k.html` (300kbps, ultra-light for slow internet)

The user may reject the ultra-light version ("resolution dropped too much, useless") — that's fine, the recommended 600kbps version is the actual deliverable, the 300k is a fallback for hostile network conditions.

**What NOT to try (tested, all failed):**
- **AV1 (libsvtav1, libaom-av1)**: even at CRF 38-42, files were 1.6-2.2MB — no better than H.264 at same visual quality, 5-10× slower to encode, spotty browser support (Safari < 16.4 has none).
- **WebM/VP9**: 3.2MB at CRF 32 — **worse** than H.4 at same quality. VP9's complex intra-frame prediction makes random access (scrubbing) slower than H.264's short-GOP.
- **AVIF for Path B frames**: 18% smaller than WebP but visual review flagged visible quality loss (hair/fabric detail blurred). WebP at q=65 is already optimal.
- **Re-extracting Path B frames from the compressed video**: the H.264 compression artifacts add noise that doesn't compress well as WebP — net savings only 9% at same visual quality.
- **Reducing frame count from 150 to 75**: 50% size savings but scroll feels noticeably choppier. Not worth it.

### Step 4b: Extract Frames (Path B — frame extraction only)

**Frame count budget: 150 frames** for a 10-second video at 15fps. This is the sweet spot:
- 150 frames × ~18KB each = ~2.7MB total — fine for mobile.
- 15fps feels smooth when scrubbed; 30fps is overkill and doubles payload.
- For a 20-second video, extract at 8fps (160 frames) — keep total ≤200 frames.

```bash
mkdir -p <site>/frames
ffmpeg -i "<VIDEO_PATH>" \
  -vf "fps=15,scale=360:-1:flags=lanczos" \
  -frames:v 150 \
  -c:v libwebp \
  -quality 65 \
  -compression_level 4 \
  -preset drawing \
  -an \
  -y \
  <site>/frames/frame_%04d.webp
```

**Why these flags (hardened by experiment — these are NOT generic recommendations):**
- `scale=360:-1` for portrait: downscales to 360px wide. Canvas upscales anyway, so visual loss is invisible. **Original resolution is wasted bytes** — never extract at native res. The ROKH project went from 3.9MB to 2.05MB (47% reduction) by switching from native 480×854 to 360×640, with zero visible difference.
- `-quality 65` (NOT 80): below 60 you get banding on skin tones, above 75 you waste 40% bytes for no visible gain. The ROKH project tested q=80 vs q=65 — 47% byte increase for invisible quality gain. **Do not extract at q=80 and then "losslessly recompress"** — that's two passes for the same result q=65 gives in one pass.
- `-compression_level 4`: good CPU/size tradeoff. Level 6 is 2× slower for ~3% smaller files.
- `-preset drawing`: optimized for natural images (people, products), not line art.

**Counter-claim to the "lossless recompression" approach:** the upstream skill recommended extracting at q=80 and then running `cwebp -lossless -z 9` on each frame. This is wrong for natural video frames for two reasons:
1. Lossless WebP on natural images is **larger** than lossy WebP at q=65 — lossless is designed for synthetic/line-art content, not photographs. Lossless encoding of a 360×640 portrait frame typically produces 40-60KB; lossy q=65 produces 14-18KB for visually identical results.
2. The "two-pass" approach (extract at q=80, then lossless-recompress) wastes CPU and produces worse results than a single q=65 lossy extraction.

**Use lossless WebP only when:** the source is synthetic content (screenshots, UI mockups, line art, text-heavy frames) where every pixel must be preserved. For video of people/products/scenes, lossy q=65 is correct.

**Generate a poster frame:**

```bash
ffmpeg -i "<VIDEO_PATH>" -frames:v 1 -vf "scale=720:-1" -c:v libwebp -quality 80 \
  <site>/frames/poster.webp
```

This poster is shown as a low-quality placeholder while the main frames load.

**Verify:**

```bash
ls <site>/frames/frame_*.webp | wc -l   # must be exactly 150
du -sh <site>/frames                     # target: ≤3MB
du -sh <site>/frames/poster.webp         # target: ≤60KB
```

If frames total > 3MB, re-extract with `-quality 55` or `scale=320:-1`.

### Step 5: Asset Polish

#### 5.1 Background color sampling (Path B only)

Sample the dominant background color from the corners of frame 1. This color fills the canvas behind the subject when in `contain` mode, so it MUST match the section panel bg or you get a visible seam.

**HARDENED APPROACH** (post-mortem: auto-sampling at runtime caused a 2-3% color mismatch between canvas bg and panel bg, visible as a vertical seam):

```python
# /home/claude/scripts/sample_bg.py
from PIL import Image
import numpy as np

img = np.asarray(Image.open("frames/frame_0001.webp").convert("RGB"))
h, w, _ = img.shape
corners = [
    img[0:20, 0:20],         # top-left
    img[0:20, w-20:w],       # top-right
    img[h-20:h, 0:20],       # bottom-left
    img[h-20:h, w-20:w],     # bottom-right
]
avg = np.concatenate(corners).mean(axis=0).astype(int)
hex_color = "#{:02x}{:02x}{:02x}".format(*avg)
print(hex_color)
```

Then **hardcode** that hex into BOTH:
- `js/app.js` → `const CANVAS_BG_FALLBACK = "#1a1b1e";`
- `css/style.css` → `--bg-dark: #1a1b1e;` and `.section-inner::before { background: rgb(26,27,30); }`

**CRITICAL**: The CSS panel background and the JS canvas fill MUST be the same RGB value. If they differ by even 2-3 units, the seam is visible. **Do NOT re-sample at runtime** — that introduces drift.

#### 5.2 Product image fetching (e-commerce sites only)

For e-commerce builds, NEVER use emoji as product thumbnails — they look cheap and break on older Android. Fetch real product images.

**Pipeline:**
1. Use the `image_search` tool to query each product (e.g. "luxury lipstick product photo on white background").
2. Download the top 3 candidates per product with `bash_tool` (`curl`/`wget`), or fetch via `web_fetch` if `image_search` doesn't give a direct file.
3. Compress each to 700×700 WebP `q=78` (target ≤30KB each) via `ffmpeg` or Pillow.
4. **Vet each candidate visually yourself** with the `view` tool — no external "VLM" call is needed here, since you (Claude) are already vision-capable. Open each compressed candidate and check: does it contain a person, face, or hand? Is the background clean (white/solid)? Judge directly against those two questions.
5. Keep only images with no person/hand/face AND a clean background. Re-search if all 3 candidates fail.

**Bug to prevent**: if you skip this visual vetting step, ~40% of stock cosmetics photos will contain a model's lips or hand holding the product — totally unsuitable for a luxury product card. Don't skip straight from search results to placement without opening and looking at each image first.

#### 5.3 Image compression budget (Path B)

| Asset type        | Count | Size each | Total | Format |
|-------------------|-------|-----------|-------|--------|
| Video frames      | 150   | ~18KB     | ≤3MB  | WebP q=65 |
| Poster            | 1     | ~50KB     | 50KB  | WebP q=80 |
| Product images    | 12-20 | ~25KB     | ≤500KB| WebP q=78, 700×700 |
| Favicon           | 1     | <2KB      | 2KB   | SVG |
| **Total**         |       |           | **≤4MB** | |

If deliverable ZIP exceeds 4MB, the number one offender is always frames. Re-extract at lower quality.

### Step 6: Scaffold

**Standalone single-file HTML is the default, without asking.** Only use the multi-file structure if the user explicitly asks for it or explicitly says they're deploying/hosting the project.

Multi-file relative paths (`css/style.css`, `js/app.js`, `video/...`) will silently fail to load when opened outside an HTTP server — e.g. in a chat preview, as a downloaded artifact, or via `file://`. The standalone bundle avoids this entirely: inline the CSS in a `<style>` tag, the JS in a `<script>` tag, and embed video/frame assets as base64 data URIs in one `.html` file.

**Multi-file structure** (only when explicitly requested):

```
project-root/
  index.html
  css/style.css
  js/app.js
  frames/frame_0001.webp ...   (Path B)
  video/output.mp4             (Path A)
  img/products/*.webp          (e-commerce)
  favicon.svg
```

No bundler. Vanilla HTML/CSS/JS + CDN libraries.

### Step 7: Build index.html

Required structure (in this order):

```html
<!-- 1. Loader: #loader > .loader-brand, #loader-bar, #loader-percent -->
<!-- 2. Fixed header: .site-header > nav with logo + flat 7-link nav + lang toggle -->
<!-- 3. Hero: .hero-standalone (100vh, solid bg, word-split heading) -->
<!--    Contains: .section-label, .hero-heading (words in spans), .hero-tagline -->
<!--    Hero CTA row + hero-meta-row (NOT hero-corner-tl!) -->
<!--    Scroll indicator with arrow -->
<!-- 4. Canvas: .canvas-wrap > canvas#canvas (fixed, full viewport) -->
<!--    (Path A: replace with <video class="bg-video" muted playsinline>) -->
<!-- 5. Dark overlay: #dark-overlay (fixed, full viewport, pointer-events:none) -->
<!-- 6. (Marquee removed from default build — see note below §9g. Do NOT add one unless the user explicitly asks for a sliding text banner.) -->
<!-- 7. Scroll container: #scroll-container (800vh+) -->
<!--    Content sections with data-enter, data-leave, data-animation -->
<!--    Stats section with .stat-number[data-value][data-decimals] -->
<!--    CTA section with data-persist="true" -->
<!-- 8. Footer -->
```

**Content section example (Path B, side-aligned):**

```html
<section class="scroll-section section-content align-left"
         data-enter="22" data-leave="38" data-animation="slide-left">
  <div class="section-inner">
    <span class="section-label">002 / Feature</span>
    <h2 class="section-heading">Feature Headline</h2>
    <p class="section-body">Description text here.</p>
  </div>
</section>
```

**Stats section example:**

```html
<section class="scroll-section section-stats"
         data-enter="54" data-leave="72" data-animation="stagger-up">
  <div class="stats-grid">
    <div class="stat">
      <span class="stat-number" data-value="24" data-decimals="0">0</span>
      <span class="stat-suffix">hrs</span>
      <span class="stat-label">Cold retention</span>
    </div>
  </div>
</section>
```

**Bilingual data attributes** (when applicable — e.g. Persian/English):

```html
<h2 data-fa="پرفروش‌ها" data-en="Bestsellers">پرفروش‌ها</h2>
```

**CDN scripts** (end of body, this order):

```html
<script src="https://cdn.jsdelivr.net/npm/lenis@1.1.13/dist/lenis.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/ScrollTrigger.min.js"></script>
<script src="js/app.js"></script>
```

For standalone single-file: inline these as `<script>` blocks instead of CDN.

**Persian font**: `Vazirmatn` (Google Fonts). **Latin serif heading**: `Cormorant Garamond`. **Latin sans body**: `Manrope`. Never use system fonts for Persian — they render ugly and break letter-spacing.

### Step 8: Build css/style.css

/* --- High-Density Compact Card Layout (v2.1 Anti-Cropping Standard) --- */
.scroll-section {
  position: absolute;
  width: 100%;
  left: 0;
  pointer-events: none;
  z-index: 20;
}

.section-inner {
  pointer-events: auto;
  max-width: 480px;
  max-height: 420px;            /* STRICT budget: guarantees fit on 768p laptops */
  padding: 22px 20px;           /* Compact padding */
  border-radius: 18px;
  background: rgba(22, 22, 26, 0.92);
  backdrop-filter: blur(18px);
  -webkit-backdrop-filter: blur(18px);
  border: 1px solid rgba(255, 255, 255, 0.12);
  box-shadow: 0 16px 40px rgba(0, 0, 0, 0.5);
  overflow-y: auto;            /* Allow scrolling if content slightly overflows */
  overflow-x: hidden;          /* CRITICAL: prevents ghost horizontal scrollbars */
}

/* Headings within cards MUST use compact clamp */
.section-heading {
  font-family: var(--font-display);
  font-size: clamp(1.45rem, 2.2vw, 2.05rem);
  line-height: 1.25;
  margin-bottom: 12px;
}

/* Product images within cards MUST have fixed height */
.product-image img,
.product-card img {
  height: 105px !important;
  width: auto !important;
  object-fit: contain !important;
  margin: 0 auto;
}

/* Unified 3-Column Header */
.site-header {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: var(--header-h);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 4vw;
  background: rgba(18, 18, 20, 0.75);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

/* Hero container MUST have clearance for fixed header */
.hero-standalone {
  position: relative;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  padding-top: calc(var(--header-h) + 36px);  /* CRITICAL: header clearance */
  padding-bottom: 40px;
  z-index: 15;
}


Use the **frontend-design** skill for creative, distinctive styling. Key technical patterns:

```css
:root {
  --bg-light: #f5f3f0;
  --bg-dark: #1a1b1e;          /* MUST match CANVAS_BG_FALLBACK in JS */
  --text-on-light: #1a1a1a;
  --text-on-dark: #f0ede8;
  --gold: #c9a55c;             /* luxury accent */
  --rose-gold: #b76e79;
  --burgundy: #6b2c2c;

  --font-display: "Cormorant Garamond", "Vazirmatn", serif;
  --font-body: "Manrope", "Vazirmatn", sans-serif;

  --ease-lux: cubic-bezier(0.22, 1, 0.36, 1);
  --header-h: 80px;
}

/* Side-aligned text zones — product occupies center or opposite side */
.align-left  { padding-left: 5vw;  padding-right: 55vw; }
.align-right { padding-left: 55vw; padding-right: 5vw;  }
.align-left .section-inner,
.align-right .section-inner { max-width: 40vw; }

/* Responsive padding (CRITICAL — fixed vw breaks narrow viewports) */
@media (min-width: 1280px) {
  .align-left  { padding-right: 50vw; }
  .align-right { padding-left:  50vw; }
  .align-left .section-inner,
  .align-right .section-inner { max-width: 52vw; }
}
@media (min-width: 1600px) {
  .align-left  { padding-right: 46vw; }
  .align-right { padding-left:  46vw; }
  .align-left .section-inner,
  .align-right .section-inner { max-width: 58vw; }
}
```

**Solid glass panel (CRITICAL — soft gradients = "amateur luxury"):**

```css
.section-inner {
  background: linear-gradient(135deg,
    rgba(20,18,22,0.94) 0%,
    rgba(28,24,30,0.92) 100%);
  backdrop-filter: blur(18px) saturate(140%);
  -webkit-backdrop-filter: blur(18px) saturate(140%);
  border: 1px solid rgba(201,165,92,0.22);
  border-radius: 4px;
  box-shadow: 0 24px 60px -12px rgba(0,0,0,0.6);
  padding: 56px 48px;
  isolation: isolate;        /* stacking context for ::before */
  overflow: hidden;          /* prevent text overflow beyond panel */
}

/* Gold accent line at top-left corner (editorial detail) */
.section-inner::after {
  content: "";
  position: absolute;
  top: 24px; left: 24px;
  width: 64px; height: 2px;
  background: linear-gradient(90deg, var(--gold), transparent);
}
```

**Body text contrast**: use `rgba(245,230,211,0.92)` (warm cream) — pure white is too harsh, pure grey is too muddy. Add `text-shadow: 0 1px 2px rgba(0,0,0,0.4)` for extra pop.

**Hero-first layout**: Hero is standalone 100vh with solid bg. Canvas starts hidden, reveals via circle-wipe as hero scrolls away.

**Scroll sections**: `position: absolute` within scroll container, positioned at midpoint of enter/leave range via `top: X%` (NOT `vh`!), `transform: translateY(-35%)` (NOT `-50%` — `-50%` centers the section vertically, causing the title to scroll off the top of the viewport when section height exceeds 100vh; `-35%` shifts the section down so the title stays visible).

**Mobile (<768px)**: Collapse side alignment to centered text with dark backdrop overlays. Reduce scroll height to ~550vh. Hide hero corner meta.

**Text contrast**: Never use `#999` for important text on light backgrounds. Use `#666` minimum for body, `var(--text-on-dark)` for headings.

**Header nav**: flat 7-link max, NO hover dropdowns (they trigger on accidental hover and conceal the video).

**CTA button (luxury, not outline):**

```css
.cta-button {
  display: inline-flex;
  align-items: center;
  gap: 12px;
  padding: 18px 38px;
  font-family: var(--font-body);
  font-weight: 600;
  font-size: 0.8rem;
  letter-spacing: 0.28em;
  text-transform: uppercase;
  color: var(--text-on-dark);
  background: linear-gradient(135deg, rgba(201,165,92,0.08), rgba(201,165,92,0.02));
  border: 1px solid rgba(201,165,92,0.5);
  border-radius: 2px;       /* sharp corners = editorial */
  cursor: pointer;
  transition: all 0.4s var(--ease-lux);
  position: relative;
  overflow: hidden;
}
.cta-button::before {
  content: "";
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, var(--gold) 0%, var(--rose-gold) 100%);
  opacity: 0;
  transition: opacity 0.4s var(--ease-lux);
  z-index: -1;
}
.cta-button:hover {
  color: var(--bg-dark);
  border-color: var(--rose-gold);
  transform: translateY(-2px);
  box-shadow: 0 12px 40px -8px rgba(201,165,92,0.5);
  letter-spacing: 0.32em;
}
.cta-button:hover::before { opacity: 1; }
```

**Marquee (OPT-IN ONLY — not part of the default build, use only if explicitly requested; no emojis, no diamonds):**

```css
.marquee-text {
  font-family: var(--font-display);
  font-size: 10vw;
  font-weight: 300;
  letter-spacing: 0.05em;
  white-space: nowrap;
  display: flex;
  align-items: center;
}
.marquee-text span { padding: 0 2vw; }
.marquee-text .marquee-dot {
  color: var(--gold);
  font-size: 0.6em;
  opacity: 0.7;
}
```

**`◆` diamond character is FORBIDDEN** — renders as missing-glyph box on some Android. Use `/` slash separator instead.

**Hero corner meta: DO NOT ADD.** If a promo is needed, integrate into the hero-meta-row at the bottom of the hero content block — it has proper flex layout and respects RTL. (This was the bug: `top: calc(header-h + 24px); right: 5vw; font-size: 0.7rem` fell off narrow viewports and overlapped the language toggle. User said: "متن تخفیف 15 درصد بدرد نخوره و افتاده رو هیرو. کلا حذفش کن.")

### Step 9: Build js/app.js

#### 9a. Lenis Smooth Scroll (MANDATORY)

```js
const lenis = new Lenis({
  duration: 1.2,
  easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
  smoothWheel: true,
  smoothTouch: false
});
lenis.on("scroll", ScrollTrigger.update);
gsap.ticker.add((time) => lenis.raf(time * 1000));
gsap.ticker.lagSmoothing(0);
```

#### 9b. Frame Preloader (Path B only)

Two-phase loading: load first 10 frames immediately (fast first paint), then load remaining frames in background. Show progress bar during load. Hide loader only after all frames are ready.

```js
function loadFrames() {
  state.frames = new Array(FRAME_COUNT).fill(null);

  const loadFrame = (idx) => new Promise((resolve) => {
    const img = new Image();
    img.onload = () => {
      state.frames[idx] = img;
      state.framesLoaded++;
      updateLoader();
      if (!state.bgSampled && idx === 0) {
        state.bgColor = CANVAS_BG_FALLBACK;  // hardcoded, NOT re-sampled
        state.bgSampled = true;
        drawFrame(0);
        state.currentFrame = 0;
      }
      resolve(img);
    };
    img.onerror = () => { state.framesLoaded++; updateLoader(); resolve(null); };
    img.src = FRAME_URL_BASE + String(idx + 1).padStart(4, "0") + ".webp";
  });

  // Phase 1: first 10 frames
  Promise.all(Array.from({ length: 10 }, (_, i) => loadFrame(i))).then(() => {
    // Phase 2: remaining
    return Promise.all(Array.from({ length: FRAME_COUNT - 10 }, (_, i) => loadFrame(i + 10)));
  }).then(() => {
    setTimeout(() => loader.classList.add("hidden"), 300);
    if (typeof ScrollTrigger !== "undefined") ScrollTrigger.refresh();
  });
}
```

#### 9c. Canvas Renderer — `contain` mode + side-offset (CRITICAL)

This is the most bug-prone file in the entire project. Here is the hardened, post-all-fixes version with annotations.

```js
const IMAGE_SCALE = 0.95;  // 0.92-0.97 sweet spot for contain mode
const CANVAS_BG_FALLBACK = "#1a1b1e";  // MUST match --bg-dark in CSS

function drawFrame(index) {
  const img = state.frames[index];
  if (!img) return;

  const vw = window.innerWidth;
  const vh = window.innerHeight;
  const iw = img.naturalWidth;
  const ih = img.naturalHeight;

  // CONTAIN mode (NOT cover): scale to fit ENTIRE image inside viewport.
  // Math.min ensures neither dimension overflows → no cropping.
  // Cover mode (Math.max) crops the subject's head or feet on mismatched
  // aspect ratios — DO NOT USE for portrait video on landscape viewport.
  const scale = Math.min(vw / iw, vh / ih) * IMAGE_SCALE;
  const dw = iw * scale;
  const dh = ih * scale;

  // SIDE OFFSET: position the subject on the SAME side as the hero text.
  // Hero side is set in CSS via flex justify-content.
  // The subject must follow the hero, so that section panels on the OPPOSITE
  // side never overlap her.
  const PAD = 0;
  const isRtl = (document.documentElement.dir || "ltr").toLowerCase() === "rtl";
  // Convention: hero on LEFT in Persian RTL, hero on RIGHT in English LTR.
  // So: RTL → subject on LEFT (PAD from left edge)
  //     LTR → subject on RIGHT (vw - dw - PAD from left edge)
  const dx = isRtl ? PAD : (vw - dw - PAD);
  const dy = (vh - dh) / 2;

  // Fill canvas with the EXACT same color as the section panel bg.
  ctx.fillStyle = state.bgColor;  // = CANVAS_BG_FALLBACK
  ctx.fillRect(0, 0, vw, vh);

  ctx.drawImage(img, dx, dy, dw, dh);
}
```

**CRITICAL — counter-claim to the upstream skill's "padded cover mode":**

The upstream skill recommended `Math.max(cw/iw, ch/ih) * IMAGE_SCALE` (cover mode) with `IMAGE_SCALE = 0.85`, claiming this is "safe" and that "Pure contain mode leaves visible border that doesn't match page bg."

This is wrong based on the ROKH project's experiments:
1. **Cover mode crops the subject's head and feet** on portrait video displayed in a landscape viewport. The ROKH video (480×854 portrait) on a 1920×1080 viewport cropped 963px vertically — the lady's head was cut off. User feedback: "lady's head cut off."
2. **The "visible border" claim is false** when bg color is hardcoded identically in JS and CSS. The visible border appears ONLY when the canvas bg color differs from the panel bg color — which is fixed by Step 5.1 (hardcode same hex in both files), NOT by switching to cover mode.
3. **The fix is**: contain mode (`Math.min`) + hardcoded bg color match + side-offset to position the subject on the hero side.

**Do NOT use cover mode for portrait video.** Use cover mode only for landscape video where the subject is centered and the video is meant to be full-bleed.

**Common bugs prevented:**
1. Cover mode crops subject's head → use `Math.min` (contain).
2. Subject centered → overlaps section panel → offset subject to hero side.
3. Visible seam between canvas bg and panel bg → hardcode identical hex in both files.
4. DPR blurriness → multiply canvas.width by `Math.min(devicePixelRatio, 2)`.
5. Subject doesn't move on language toggle → re-call `drawFrame(state.currentFrame)` in `applyLanguage()`.

#### 9c-alt. Video Scrub Renderer (Path A) — Apple-style `<video>` + `currentTime`

Path A replaces the entire canvas renderer with a single `<video>` element bound to scroll progress via `video.currentTime`. This is dramatically simpler than Path B (no preloader, no `drawFrame`, no bg color sampling, no DPR handling), but has its own set of gotchas.

**HTML (use this in `index.html` instead of `<canvas>`):**

```html
<div class="video-wrap" id="videoWrap">
  <video id="scrub-video"
         muted
         playsinline
         webkit-playsinline
         preload="auto"
         disablepictureinpicture
         disableremoteplayback></video>
</div>
```

**CSS — full-bleed video with side-offset via `object-position` (CRITICAL):**

```css
.video-wrap {
  position: fixed; top: 0; left: 0;
  width: 100vw; height: 100vh;
  z-index: 10;
  background: var(--bg-dark);   /* MUST match --bg-dark for letterboxing */
  clip-path: circle(0% at 50% 50%);
  -webkit-clip-path: circle(0% at 50% 50%);
  overflow: hidden;
}
#scrub-video {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  /* CRITICAL: contain mode, NOT cover.
     Cover mode crops the portrait video to fill the landscape viewport,
     cutting off the subject's head/feet and zooming past them at mid-scroll.
     Contain mode shows the FULL video frame, matching Path B's Math.min
     canvas behavior. The empty letterbox space is filled by video-wrap's
     bg-dark, which matches --bg-dark exactly so the seam is invisible. */
  object-fit: contain;
  object-position: right center;  /* default LTR: lady on RIGHT */
  display: block;
  pointer-events: none;
}
html[dir="rtl"] #scrub-video {
  object-position: left center;  /* RTL: lady on LEFT */
}
```

**Why `object-fit: contain` is correct for Path A (counter-claim to earlier advice):**

Earlier versions of this skill recommended `object-fit: cover` for Path A, reasoning that "the browser's video decoder preserves the center of the frame when cropping, so a centered subject stays in frame." This is **wrong** for portrait videos of people, for three reasons:

1. **Cover mode crops aggressively on portrait video in landscape viewport.** A 640×1139 portrait video in a 1440×900 viewport scales by `max(1440/640, 900/1139) = 2.25x`, becoming 1440×2563. Then it's cropped vertically to 900px — losing 831px from top and bottom each. The subject's head and feet are cut off.

2. **The subject moves within the frame during the video.** At scroll=15%, the subject might be in the center of the video frame (visible in the cropped viewport). At scroll=50%, the subject might have moved to the top or bottom of the video frame (cropped out entirely). User feedback: "by 50% scroll the lady is gone, only blurry fabric visible."

3. **It diverges from Path B's behavior.** If you're delivering BOTH a Path A and Path B version (as the ROKH project did), users expect them to look identical. Path B uses `Math.min` (contain). Path A with `cover` looks completely different — zoomed, cropped, jumpy. Use `contain` in Path A to match.

**`object-position` with `contain` mode** moves the entire fitted video to one side of the element's box, leaving the other side as bg-dark. This is the Path A equivalent of Path B's canvas `dx` offset. Same logical effect (subject on one side, empty bg on the other), different mechanism (CSS vs JS math).

**When `object-fit: cover` IS correct for Path A:**
- The source video is **landscape** (16:9 or wider) and you want full-bleed background
- The video has the subject **centered throughout the entire duration** (no movement that would push them out of the cropped zone)
- You're building a single-version deliverable (no Path B equivalent to match)

For portrait videos of people, always use `contain`.

**JS — load video + bind scroll to `currentTime`:**

```js
const VIDEO_URL = "video/output.mp4";   // or base64 data URI for standalone
const FRAME_SPEED = 1.0;                 // 1.0 = video.duration maps 1:1 to scroll progress

function loadVideo() {
  // Fake loader progress while video downloads
  let fakeProgress = 0;
  const fakeInterval = setInterval(() => {
    fakeProgress = Math.min(fakeProgress + 8, 90);
    loaderProgress.style.width = fakeProgress + "%";
    loaderPercent.textContent = fakeProgress + "%";
  }, 200);

  video.src = VIDEO_URL;

  const onLoadedMetadata = () => {
    state.videoReady = true;
    state.videoDuration = video.duration || 10;
    clearInterval(fakeInterval);
    loaderProgress.style.width = "100%";
    loaderPercent.textContent = "100%";

    // CRITICAL: seek to 0.001 to force decoder to render first frame.
    // Without this, video stays black until first scroll event triggers a seek.
    try { video.currentTime = 0.001; } catch (e) {}

    setTimeout(() => {
      loader.classList.add("hidden");
      if (typeof ScrollTrigger !== "undefined") ScrollTrigger.refresh();
    }, 400);
  };

  // CRITICAL: check readyState first — if video is cached, loadedmetadata
  // may have ALREADY fired before we attached the listener, and the event
  // will never come. Without this check, loader hangs forever on cache hit.
  if (video.readyState >= 1) {
    onLoadedMetadata();
  } else {
    video.addEventListener("loadedmetadata", onLoadedMetadata, { once: true });
  }

  // Fallback: force-hide loader after 6s if metadata never fires
  // (network issue, broken src, etc.)
  setTimeout(() => {
    if (!state.videoReady) {
      clearInterval(fakeInterval);
      state.videoReady = true;
      state.videoDuration = video.duration || 10;
      loader.classList.add("hidden");
      if (typeof ScrollTrigger !== "undefined") ScrollTrigger.refresh();
    }
  }, 6000);
}

function setupVideoScrubBinding() {
  if (typeof ScrollTrigger === "undefined") return;

  // CRITICAL: throttle seeks with requestAnimationFrame.
  // ScrollTrigger fires onUpdate on every scroll event (potentially 60+/sec).
  // Setting video.currentTime directly on each event causes jank — the
  // browser queues seeks faster than it can decode them.
  // Coalesce: if a seek is pending, drop the new request; the next rAF
  // will pick up the latest progress.
  let pendingSeek = null;
  const seekVideo = (progress) => {
    if (!state.videoReady || !state.videoDuration) return;
    const accelerated = Math.min(progress * FRAME_SPEED, 1);
    const targetTime = accelerated * state.videoDuration;
    if (pendingSeek) return;
    pendingSeek = requestAnimationFrame(() => {
      pendingSeek = null;
      // CRITICAL: skip seek if delta < 16ms (~1 frame at 60fps).
      // Avoids redundant decoder work when scroll position barely changed.
      if (Math.abs(video.currentTime - targetTime) > 0.016) {
        try { video.currentTime = targetTime; } catch (e) {}
      }
    });
  };

  ScrollTrigger.create({
    trigger: scrollContainer,
    start: "top top",
    end: "bottom bottom",
    scrub: 0.4,
    onUpdate: (self) => seekVideo(self.progress),
  });
}

// CRITICAL: update object-position on language toggle so the lady
// moves to the opposite side when hero text flips. Without this,
// the lady stays on her old side and overlaps the new hero text.
function updateVideoPosition() {
  const isRtl = (document.documentElement.dir || "ltr").toLowerCase() === "rtl";
  video.style.objectPosition = isRtl ? "left center" : "right center";
}

// In applyLanguage(), call updateVideoPosition() after setting dir.
// (No need to "redraw" anything — CSS handles the position change.)
```

**Common bugs prevented (Path A specific):**
1. **Video stays black until first scroll** → seek to `0.001` after `loadedmetadata` to force first-frame decode.
2. **Loader hangs on cache hit** → check `video.readyState >= 1` before attaching `loadedmetadata` listener.
3. **Scroll scrubbing feels janky** → throttle `video.currentTime` seeks via `requestAnimationFrame` + 16ms delta threshold.
4. **iOS Safari forces fullscreen** → add `playsinline` AND `webkit-playsinline` attributes (the latter is the legacy iOS 8-9 name, still required for older devices).
5. **iOS blocks video loading** → `muted` attribute is mandatory; without it, iOS requires a user gesture before any video can load.
6. **Lady on wrong side after language toggle** → call `updateVideoPosition()` in `applyLanguage()` to flip `object-position`.
7. **Chrome shows Picture-in-Picture / Cast buttons on hover** → add `disablepictureinpicture` and `disableremoteplayback` attributes.
8. **Seeking stutters / lags** → your video has too few keyframes. Re-encode with `-g 10 -keyint_min 10` (see Step 4a).
9. **Standalone HTML file is too large (>5MB)** → your video is too big. Target ≤2MB MP4 before base64 inlining (which adds 33%).
10. **Video doesn't span full viewport on Safari** → ensure CSS `width: 100%; height: 100%; object-fit: cover` is set. Safari sometimes ignores `width: 100vw` on `<video>`.

#### 9d. Frame-to-Scroll Binding (Path B) / Video-to-Scroll Binding (Path A)

```js
const FRAME_SPEED = 1.0; // 1.0 = frames map 1:1 to scroll progress (frame 0 at 0%, frame N at 100%). DO NOT use >1.5 — frames will deplete before scroll ends and canvas freezes mid-page.

ScrollTrigger.create({
  trigger: scrollContainer,
  start: "top top",
  end: "bottom bottom",
  scrub: 0.4,
  onUpdate: (self) => {
    const accelerated = Math.min(self.progress * FRAME_SPEED, 1);

    // Path B: frame index
    const index = Math.min(Math.floor(accelerated * FRAME_COUNT), FRAME_COUNT - 1);
    if (index !== state.currentFrame) {
      state.currentFrame = index;
      requestAnimationFrame(() => drawFrame(index));
    }

    // Path A: video.currentTime
    // video.currentTime = accelerated * video.duration;
  }
});
```

#### 9e. Continuous Parallax Crossfade Engine (v2.1 Kinematic Standard)

**Why v2.1 replaced binary transitions:**
In v2.0, sections set `opacity = 1` instantly upon entering, remained static for 75% of their scroll duration, and had non-overlapping ranges with dead space between them. On fast or regular mouse wheel scrolling, cards felt disjointed ("instant transition between cards").

v2.1 introduces the **Continuous 3-Phase Smoothstep Crossfade Engine**:
1. **Phase 1 (Glide-in, 0.00 → 0.28 relative progress)**: Card ascends smoothly from `translateY(36px)` to `0px`, and opacity fades from 0.0 to 1.0 using the Smoothstep polynomial (e = t^2 \cdot (3 - 2t)).
2. **Phase 2 (Stable hold, 0.28 → 0.72 relative progress)**: Card sits stably at `translateY(0px)`, `opacity: 1` for reading.
3. **Phase 3 (Glide-out, 0.72 → 1.00 relative progress)**: Card ascends from `translateY(0px)` to `translateY(-32px)`, and opacity dissolves from 1.0 to 0.0.
4. **Overlapping intervals**: Adjacent sections overlap by 4–6% (e.g. Card 1 active 0.16–0.34, Card 2 active 0.30–0.48). As Card 1 fades out, Card 2 is already ascending into view, eliminating all dead space.

```js
// Helper: Smoothstep cubic polynomial for natural cinematic easing
const smoothstep = (min, max, value) => {
  const x = Math.max(0, Math.min(1, (value - min) / (max - min)));
  return x * x * (3 - 2 * x);
};

function setupSectionAnimations() {
  const sections = document.querySelectorAll(".scroll-section");
  if (!sections.length || typeof ScrollTrigger === "undefined") return;

  sections.forEach((section) => {
    const enterPct = parseFloat(section.dataset.enter) / 100;
    const leavePct = parseFloat(section.dataset.leave) / 100;
    const persist = section.dataset.persist === "true";
    const inner = section.querySelector(".section-inner") || section;

    // Center card vertically at midpoint of its range
    const midPct = (enterPct + leavePct) / 2;
    section.style.top = (midPct * 100) + "%";
    section.style.transform = "translateY(-35%)";

    // Continuous scroll-driven kinematic update
    ScrollTrigger.create({
      trigger: scrollContainer,
      start: "top top",
      end: "bottom bottom",
      scrub: 0.4,
      onUpdate: (self) => {
        const p = self.progress;

        if (p < enterPct) {
          // Before card's entrance
          section.style.opacity = 0;
          section.style.pointerEvents = "none";
          inner.style.transform = "translateY(36px)";
        } else if (p >= enterPct && p <= leavePct) {
          // Inside active range: 3-Phase Smoothstep
          const rel = (p - enterPct) / (leavePct - enterPct);
          let opacity = 1;
          let yOffset = 0;

          if (rel < 0.28) {
            // Phase 1: Glide-in (0.0 -> 0.28)
            const t = smoothstep(0, 0.28, rel);
            opacity = t;
            yOffset = (1 - t) * 36; // Glide from +36px to 0px
          } else if (rel > 0.72 && !persist) {
            // Phase 3: Glide-out (0.72 -> 1.0)
            const t = smoothstep(0.72, 1.0, rel);
            opacity = 1 - t;
            yOffset = -t * 32;       // Glide from 0px to -32px
          } else {
            // Phase 2: Stable Hold (0.28 -> 0.72)
            opacity = 1;
            yOffset = 0;
          }

          section.style.opacity = opacity;
          section.style.pointerEvents = opacity > 0.2 ? "auto" : "none";
          inner.style.transform = `translateY(${yOffset}px)`;
        } else {
          // After card's range
          if (persist) {
            section.style.opacity = 1;
            section.style.pointerEvents = "auto";
            inner.style.transform = "translateY(0px)";
          } else {
            section.style.opacity = 0;
            section.style.pointerEvents = "none";
            inner.style.transform = "translateY(-32px)";
          }
        }
      }
    });
  });
}
```

#### 9f. Counter Animations

```js
document.querySelectorAll(".stat-number").forEach(el => {
  const target = parseFloat(el.dataset.value);
  const decimals = parseInt(el.dataset.decimals || "0");
  gsap.from(el, {
    textContent: 0,
    duration: 2.2,
    ease: "power1.out",
    snap: { textContent: decimals === 0 ? 1 : 0.01 },
    scrollTrigger: {
      trigger: el.closest(".scroll-section"),
      start: "top 70%",
      toggleActions: "play none none reverse"
    },
    onUpdate: () => {
      // Persian digits if RTL
      if (decimals === 0) {
        el.textContent = Math.round(el.textContent).toLocaleString(
          state.lang === "fa" ? "fa-IR" : "en-US"
        );
      }
    }
  });
});
```

#### 9g. Horizontal Text Marquee (OPT-IN ONLY — do not build by default)

> This is no longer part of the default build. Skip this entirely unless the user explicitly asks for a sliding/marquee text banner. Do not add a `.marquee-wrap` element to the HTML, CSS, or JS otherwise.

```js
document.querySelectorAll(".marquee-wrap").forEach(el => {
  const speed = parseFloat(el.dataset.scrollSpeed) || -25;
  gsap.to(el.querySelector(".marquee-text"), {
    xPercent: speed,
    ease: "none",
    scrollTrigger: {
      trigger: scrollContainer,
      start: "top top",
      end: "bottom bottom",
      scrub: 1
    }
  });

  // Fade marquee in/out around a clean 4% range
  const mEnter = 0.78, mLeave = 0.82, mFade = 0.012;
  ScrollTrigger.create({
    trigger: scrollContainer,
    start: "top top",
    end: "bottom bottom",
    scrub: 0.5,
    onUpdate: (self) => {
      const p = self.progress;
      let opacity = 0;
      if (p >= mEnter - mFade && p <= mEnter) opacity = (p - (mEnter - mFade)) / mFade;
      else if (p > mEnter && p < mLeave) opacity = 1;
      else if (p >= mLeave && p <= mLeave + mFade) opacity = 1 - (p - mLeave) / mFade;
      el.classList.toggle("visible", opacity > 0.05);
      el.style.opacity = opacity;
    }
  });
});
```

#### 9h. Dark Overlay

```js
function setupDarkOverlay(enter, leave) {
  const overlay = document.getElementById("dark-overlay");
  const fadeRange = 0.04;
  ScrollTrigger.create({
    trigger: scrollContainer,
    start: "top top",
    end: "bottom bottom",
    scrub: 0.5,
    onUpdate: (self) => {
      const p = self.progress;
      let opacity = 0;
      if (p >= enter - fadeRange && p <= enter) {
        opacity = (p - (enter - fadeRange)) / fadeRange * 0.92;
      } else if (p > enter && p < leave) {
        opacity = 0.92;
      } else if (p >= leave && p <= leave + fadeRange) {
        opacity = 0.92 * (1 - (p - leave) / fadeRange);
      }
      overlay.style.opacity = opacity;
    }
  });
}
```

#### 9i. Circle-Wipe Hero Reveal

```js
function setupHeroTransition() {
  ScrollTrigger.create({
    trigger: scrollContainer,
    start: "top top",
    end: "bottom bottom",
    scrub: 0.6,
    onUpdate: (self) => {
      const p = self.progress;
      // Hero fades out fast as scroll begins
      const heroOpacity = Math.max(0, 1 - p * 14);
      heroSection.style.opacity = heroOpacity;
      heroSection.style.pointerEvents = heroOpacity > 0.1 ? "auto" : "none";

      // Canvas reveals via expanding circle clip-path
      const wipeProgress = Math.min(1, Math.max(0, (p - 0.005) / 0.06));
      const radius = wipeProgress * 75;
      canvasWrap.style.clipPath = `circle(${radius}% at 50% 50%)`;
      canvasWrap.style.webkitClipPath = `circle(${radius}% at 50% 50%)`;
    }
  });
}
```

#### 9j. Language Toggle (FA/EN, RTL/LTR) — when applicable

```js
function applyLanguage(lang) {
  state.lang = lang;
  document.documentElement.setAttribute("lang", lang);
  document.documentElement.setAttribute("dir", lang === "fa" ? "rtl" : "ltr");
  localStorage.setItem("brand-lang", lang);

  // Swap all data-fa/data-en elements
  document.querySelectorAll("[data-fa], [data-en]").forEach((el) => {
    const text = lang === "fa" ? el.dataset.fa : el.dataset.en;
    if (text !== undefined && text !== "") el.textContent = text;
  });

  // CRITICAL: re-draw canvas so the subject moves to the opposite side
  // (matching the new hero side) immediately after language switch.
  setTimeout(() => drawFrame(state.currentFrame), 60);

  // Refresh ScrollTrigger — RTL changes layout flow
  if (typeof ScrollTrigger !== "undefined") {
    setTimeout(() => ScrollTrigger.refresh(), 100);
  }
}
```

**Bug to prevent**: if you only set `dir` without re-drawing the canvas, the lady stays on her old side while the hero text flips — causing immediate overlap.

### Step 10: QA Loop (Playwright + view tool)

**This step is mandatory. Skipping it = shipping undiagnosed bugs.**

#### 10a. Screenshot script

```python
# /home/claude/scripts/test_<brand>.py
from playwright.sync_api import sync_playwright
import os

OUT = "/home/claude/scripts/test-shots"
os.makedirs(OUT, exist_ok=True)

VIEWPORTS = [
    ("desktop", 1440, 900),
    ("mobile",  390,  844),
]
SCROLL_POSITIONS = [0, 0.15, 0.30, 0.45, 0.60, 0.75, 0.90, 0.98]

errors = []
with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)

    for vp_name, w, h in VIEWPORTS:
        page = browser.new_page(viewport={"width": w, "height": h})
        page.on("console", lambda msg: errors.append(msg.text) if msg.type == "error" else None)
        page.goto("file:///home/claude/<brand>-site/index.html")
        page.wait_for_selector("#loader.hidden", timeout=30000)
        page.wait_for_timeout(1000)

        for i, scroll in enumerate(SCROLL_POSITIONS):
            page.evaluate(f"window.scrollTo(0, document.body.scrollHeight * {scroll})")
            page.wait_for_timeout(800)
            fname = f"{i:02d}-{vp_name}-{int(scroll*100)}pct.png"
            page.screenshot(path=os.path.join(OUT, fname), full_page=False)

        # English version (desktop only)
        if vp_name == "desktop":
            page.click("#langToggle")
            page.wait_for_timeout(1500)
            page.evaluate("window.scrollTo(0, 0)")
            page.wait_for_timeout(800)
            page.screenshot(path=os.path.join(OUT, "en-hero.png"))

    browser.close()

assert len(errors) == 0, f"Console errors: {errors}"
print(f"Screenshots saved to {OUT}")
```

#### 10b. Review each screenshot yourself with the `view` tool

There is no separate "VLM" tool in this environment — Claude is already vision-capable, so **you** are the reviewer. Open every screenshot from `10a` with the `view` tool (not just a sample — all of them) and check each one against this list:

1. Is the subject's full body (head AND feet, if applicable) visible? (yes/no)
2. Is the section text fully readable? (yes/no)
3. Does the section text overlap the subject image? (yes/no)
4. Are there any UI elements that look broken, misaligned, or "falling off"? (yes/no + which)
5. Does the design feel "luxury" or "amateur"? (luxury/amateur + why)
6. Any unexpected content (wrong product category, emoji, missing-glyph boxes)? (yes/no + what)

Be specific about pixel locations when you spot an issue.

**Rule**: 0 issues allowed. If you spot any issue while viewing a screenshot, patch the code, re-run Playwright to regenerate that screenshot, and view it again. Do not deliver until every screenshot passes on re-inspection.

#### 10c. Test at 4 viewport widths minimum

Always test at 1440, 1280, 1024, and 900 pixels wide. The ROKH project had a bug that only manifested at 900px (panel overlapped lady by 2vw) — invisible at 1440px.

