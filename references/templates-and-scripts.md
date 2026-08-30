# File Templates & Recovery Scripts

CSS variable blocks, JS CONFIG blocks (Path A and Path B), and copy-paste recovery scripts for common failure modes. Read when scaffolding a new build or recovering from a known bug (see troubleshooting section in SKILL.md).

**Note on "VLM" scripts below**: the original version of this skill assumed a separate `z-ai vision` CLI for image/screenshot review. That tool doesn't exist in this environment — Claude is natively vision-capable, so vetting product images and reviewing QA screenshots is done directly with the `view` tool instead of a subprocess call. The two scripts below are kept as *optional* helpers for mechanical steps (compressing/saving candidate images), not for the vetting judgment itself.


## File Templates

### CSS variables block (luxury cosmetics palette)

```css
:root {
  --cream:      #f5f0e8;
  --bg-dark:    #1a1b1e;     /* MUST match CANVAS_BG_FALLBACK in JS */
  --gold:       #c9a55c;
  --rose-gold:  #b76e79;
  --burgundy:   #6b2c2c;
  --text-light: rgba(245,230,211,0.92);
  --text-mute:  rgba(245,230,211,0.55);

  --font-display: "Cormorant Garamond", "Vazirmatn", serif;
  --font-body:    "Manrope", "Vazirmatn", sans-serif;

  --ease-lux: cubic-bezier(0.22, 1, 0.36, 1);
  --header-h: 80px;
}
```

### JS CONFIG block (Path B)

```javascript
const FRAME_COUNT = 150;
const FRAME_URL_BASE = "frames/frame_";
const IMAGE_SCALE = 0.95;          // contain mode: 0.92-0.97 sweet spot
const FRAME_SPEED = 1.0;           // 1.0 = 1:1 mapping (frame 0 at 0% scroll, frame N at 100%). DO NOT use >1.5 — frames deplete before scroll ends.
const CANVAS_BG_FALLBACK = "#1a1b1e";  // MUST match --bg-dark in CSS
```

### JS CONFIG block (Path A)

```javascript
const VIDEO_URL = "video/output.mp4";
const FRAME_SPEED = 1.0;           // 1.0 = 1:1 mapping. DO NOT use >1.5.
```

### Step 0 QC one-liner (run on every uploaded video, first thing)

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,duration,r_frame_rate,codec_name,bit_rate,pix_fmt \
  -show_entries format=size,bit_rate,duration \
  -of default=noprint_wrappers=1 "$VIDEO_PATH" && du -h "$VIDEO_PATH"
```
Full thresholds and the decision prompt this feeds into: `references/qc-and-compression-research.md` §1.

### `prefers-reduced-motion` gate (wrap around Lenis/ScrollTrigger init)

```javascript
const prefersReducedMotion = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
if (!prefersReducedMotion) {
  // normal Lenis + ScrollTrigger scroll-scrub init
} else {
  // static/instant-scroll fallback — see references/qc-and-compression-research.md §4
}
```

---

## Recovery Scripts

Persist these scripts to `/home/claude/scripts/` so they can be re-run on partial failure without regenerating from scratch.

| Script | Purpose | Re-run on |
|--------|---------|-----------|
| `extract_frames.sh` | ffmpeg frame extraction (Path B) | Change in target fps / quality |
| `reencode_video.sh` | ffmpeg short-GOP re-encode (Path A) | New video input |
| `sample_bg.py` | Sample background color from frame 1 (Path B) | New video input |
| `recompress_frames.py` | Re-compress existing frames (Path B) | Frames too large |
| `fetch_product_images.py` | Search + download product images (e-commerce) | Adding new products |
| — | Vet product image candidates with `view` (no script — this is a direct visual judgment call) | All fetches |
| `test_<brand>.py` | Playwright screenshot suite | After any code change |
| — | Review screenshots with `view` (no script — direct visual judgment call) | After every test run |

