# Lessons From the Field — Annotated Case Studies

Three real production postmortems (frame-based canvas / Path B, video-scrub / Path A, and multi-bitrate compression) that every rule in SKILL.md and workflow-detail.md was distilled from. Read this when: (a) you want to understand *why* a MUST/FORBIDDEN rule exists, (b) you hit a bug that feels familiar and want to check if it's already been diagnosed here, or (c) the user pushes back on a rule and you need the original reasoning.

**Note**: these postmortems refer to "VLM" review — that was the original authoring environment's separate vision-model CLI. In this environment there's no separate tool: the visual judgment described below is just you, Claude, looking at the screenshot with the `view` tool. Read "VLM confirmed/flagged/said" below as "visual inspection confirmed/flagged/showed."


## Lessons From the Field (annotated case study)

The ROKH cosmetics project (10-second portrait video of a woman with cosmetics) required **11 iterations** of UX fixes before reaching user acceptance. Each iteration addressed real, reproducible bugs. Here is the annotated timeline:

**Iteration 1 (initial build)**: Bug — section `top: Xvh` instead of `top: X%`. All sections invisible except the first. Fix: switched to `top: X%`.

**Iteration 2 (lady's head cut off)**: Bug — cover mode on portrait video cropped 963px vertically. Fix: `Math.min` (contain mode).

**Iteration 3 (saffron on cosmetics site)**: Bug — wrong botanical references in copy. Fix: replaced with rose/jasmine.

**Iteration 4 (text unreadable)**: Bug — soft gradient backdrop insufficient. Fix: solid glass card with `backdrop-filter: blur(18px)`.

**Iteration 5 (poetry instead of products)**: Bug — content was philosophy, not e-commerce. Fix: complete content rewrite, 14 product cards with prices and Add to Bag buttons.

**Iteration 6 (pop-up menus, emojis, missing product images)**: Bug — three separate UX issues. Fix: removed scroll-pop nav, flattened header nav, fetched 15 real VLM-vetted product images.

**Iteration 7-10 (scroll UX)**: 4 iterations to nail section/lady/hero side alignment. Each iteration moved the lady to a different side, tightened padding, and fixed overlaps. Final state: lady on LEFT (Persian) / RIGHT (English), hero text + section panels on OPPOSITE side, gap ≤ 6% at all viewports.

**Iteration 11 (15% discount text falling off hero)**: Bug — `.hero-corner-tl` positioned at `top: calc(header-h + 24px); right: 5vw;` with `font-size: 0.7rem` fell outside visible area on narrow viewports and overlapped the language toggle. Fix: removed the element entirely. If a promo is needed, integrate into the hero-meta-row.

**Iteration 12 (4 regression bugs in delivered ZIP)**: User said "برگشتی به عصر حجر" (you went back to the stone age) — the delivered ZIP had 4 bugs that all stemmed from earlier decisions:

1. **Hero text overlapping lady canvas**: `.hero-standalone { justify-content: flex-end }` in RTL puts hero text on LEFT, same side as lady canvas (dx=0). During the circle-wipe transition (0→7% scroll), hero text covers the lady's face. Fix: changed to `flex-start` — flex-start = LEFT in LTR, RIGHT in RTL, so hero text is always on the OPPOSITE side of the lady.

2. **Section titles not visible**: `translateY(-50%)` vertical centering + 4 product cards per section (2×2 grid) made sections taller than viewport. At 47% scroll, VLM confirmed: title above viewport, only product cards visible. Fix: changed to `translateY(-35%)` AND reduced all product sections from 4 cards to 2 cards (matching bestsellers).

3. **Stats section + "Free Shipping" text on hero**: User said these "should not exist". Removed: entire stats section (6 stat cards), hero-meta-row ("ارسال رایگان بالای ۵۰۰ هزار تومان"), hero-corner-br ("۲۴/۷ پشتیبانی آنلاین"), marquee block (entirely), dark-overlay div. Made JS setup functions null-safe (`if (!element) return;`) to prevent console errors.

4. **Frames run out mid-scroll**: `FRAME_SPEED = 2.0` caused frames to deplete at 50% scroll — canvas frozen on last frame for entire second half of page. Fix: changed to `FRAME_SPEED = 1.0` (1:1 mapping — frame 0 at 0% scroll, frame N at 100% scroll). VLM confirmed lady's pose is different at 20%, 61%, and 95% scroll, proving frames still animating.

**Key lesson**: `justify-content: flex-end` in RTL flexbox = LEFT (not RIGHT as the name suggests). The flex-end "end" refers to the end of the main axis, which flips with `direction`. When building RTL/LTR bilingual layouts, always verify with a screenshot in BOTH directions — the CSS comment can lie.

**Takeaway**: every "rule" in this document is a scar from a specific bug. When in doubt, follow the rule. When tempted to add a new feature, ask: "does this introduce a new popup, corner element, or position-dependent widget that could break on a different viewport?" If yes, don't add it.

---

## Lessons From the Field — Path A (video-scrub) case study

After delivering the Path B (frame-based) ROKH standalone, the user requested a **second** standalone HTML using the **video-scrub technique** (Apple AirPods Pro style) instead of frame-by-frame canvas rendering. The build took ~2 hours and surfaced 6 new Path A-specific bugs not covered by the Path B lessons above.

**Path A iteration 1 (video stays black on load)**: Bug — `<video>` element with `loadedmetadata` listener attached, but the decoder doesn't render frame 0 until a seek or play is requested. Loader hidden, but video shows black rectangle until first scroll. Fix: after `loadedmetadata` fires, immediately `video.currentTime = 0.001` to force decoder to render frame 0.

**Path A iteration 2 (loader hangs on page reload)**: Bug — on second visit, the video may already have `readyState >= 1` (cached by browser). The `loadedmetadata` event already fired before JS attached the listener, so the listener never triggers and the loader hangs forever. Fix: check `video.readyState >= 1` BEFORE attaching the listener. If true, call the handler directly. Always add a 6-second timeout fallback to force-hide the loader if metadata never fires.

**Path A iteration 3 (scroll scrubbing feels janky)**: Bug — setting `video.currentTime` directly on every ScrollTrigger `onUpdate` callback. ScrollTrigger fires 60+ times/sec during scroll, but the browser's video decoder can only process seeks at ~30/sec. The queue piles up, causing visible stutter and CPU spikes. Fix: throttle seeks via `requestAnimationFrame` — if a seek is already pending, drop the new request. Also add a 16ms delta threshold: if `Math.abs(video.currentTime - targetTime) < 0.016`, skip the seek entirely (sub-frame changes are invisible).

**Path A iteration 4 (lady on wrong side after language toggle)**: Bug — `<video>` doesn't automatically move when `dir` changes. Only flexbox children and text reflows. The lady stayed on her old side after toggling FA→EN, overlapping the new hero text. Fix: add `updateVideoPosition()` function that sets `video.style.objectPosition = isRtl ? "left center" : "right center"`, and call it from `applyLanguage()`.

**Path A iteration 5 (seeking stutters on long GOP video)**: Bug — initial ffmpeg re-encode used default `-g 250` (keyframe every 250 frames). Seeking to a non-keyframe position required decoding up to 250 frames forward, causing 200-500ms seek lag. Scroll-scrub felt completely broken. Fix: re-encode with `-g 10 -keyint_min 10` (keyframe every 10 frames). Seek time dropped to <30ms. Verified by `ffprobe ... -show_entries frame=pict_type | grep -c I` returning ~3 I-frames in the first 30 frames.

**Path A iteration 6 (file size concern)**: Bug — initial video encode at `scale=1080:-2 -crf 22` produced a 3.4MB MP4. With base64 inlining (+33%), the standalone HTML would be ~5MB. Fix: re-encode at `scale=720:-2 -crf 24` → 1.6MB MP4 → 2.1MB base64 → 2.7MB final HTML (with images + CSS + JS). The 720px width is invisible to the eye because the video is displayed at viewport size and the browser upscales during render.

**Path A iteration 7 (lady cropped, head cut off, disappears at mid-scroll)**: Bug — used `object-fit: cover` on the `<video>` element, thinking it would behave like Apple's full-bleed product pages. Instead, the portrait video (640×1139) in a landscape viewport (1440×900) was scaled 2.25x and cropped to 900px tall — losing 831px from top and bottom. The lady's head and feet were cut off. Worse, as the video scrubbed forward, the lady moved within the video frame, sometimes disappearing entirely from the cropped zone. User feedback: "هیرو درست همونجایی که حالت قبلی به درستی قرار داده بود قرار بگیره. درستش کن" ("the hero should be placed where the previous version did correctly — fix it"). VLM comparison of frame-based vs video-scrub at 15%, 30%, 50% scroll confirmed: frame-based showed the lady fully on the left; video-scrub showed her cropped, zoomed, or absent. Fix: changed `object-fit: cover` → `object-fit: contain`. This matches Path B's `Math.min` canvas behavior — the full video frame is visible, positioned on one side via `object-position: left/right center`, with the empty letterbox space filled by `video-wrap`'s `bg-dark` (matching `--bg-dark` exactly so the seam is invisible). Verified: video-scrub screenshots at 15%, 30%, 50% now pixel-match the frame-based layout.

**Final Path A delivery verification** (Playwright):
- 0 console errors, 0 console warnings
- Loader hides within 2.5s on clean cache
- Video duration: 10s, readyState: 4, videoWidth: 640, videoHeight: 1139 (portrait)
- Scrub mapping verified: 25% scroll → 3.0s, 50% → 6.0s, 75% → 9.0s, 95% → 10.0s (perfect 1:1)
- Language toggle: RTL → `object-position: left center` (lady LEFT), LTR → `object-position: right center` (lady RIGHT)
- Standalone HTML size: 2.7MB (vs Path B's 3.3MB — 18% smaller)

**Key Path A lessons**:
1. **`<video>` is NOT a drop-in replacement for `<canvas>`** — it has its own set of decoder-related gotchas (first-frame rendering, keyframe density, seek throttling, readyState races).
2. **Short GOP is non-negotiable for scrub video** — `-g 10 -keyint_min 10` is the difference between smooth scrubbing and broken scrolling. Default ffmpeg settings are for streaming playback, not random access.
3. **`object-position` is the Path A equivalent of canvas `dx` offset** — same logical purpose (move subject to one side), different mechanism (CSS vs JS math). Path A is simpler but only supports symmetric center-preserving crop, not arbitrary pixel offsets.
4. **Path A vs Path B trade-off**: Path A is ~20% smaller and 60% less code, but loses the ability to do asymmetric subject positioning. If your layout needs the subject offset by a specific number of pixels (not just left/right/center), use Path B.
5. **base64 inlining adds 33% overhead** — a 1.6MB MP4 becomes a 2.1MB data URI. For Path A, the video dominates the file size; for Path B, the 150 frames dominate. Either way, target ≤2MB raw media before inlining.
6. **`object-fit: contain` (NOT `cover`) for portrait videos of people** — cover mode crops aggressively (2.25x zoom on 640×1139 portrait in 1440×900 viewport, losing 831px top+bottom). The subject's head/feet are cut off, and they disappear entirely at scroll positions where they've moved to the cropped zone. Contain mode shows the full video frame, matching Path B's `Math.min` canvas behavior. The empty letterbox space is filled by `video-wrap`'s `bg-dark`, which matches `--bg-dark` exactly so the seam is invisible. Cover mode is only correct for landscape video with a centered-static subject.

---

## Lessons From the Field — Multi-bitrate compression case study

After delivering Path A at 2.59MB, the user flagged it as too heavy: "حس می کنم صفحه سنگین شده. سرچ کن ببین چه راهکارهایی وجود داره. 2.6 مگابایت خیلی زیاده" (page feels heavy, search for solutions, 2.6MB is too much). The research and compression pass that followed surfaced 5 new lessons specific to size optimization.

**Compression iteration 1 (suggesting external video file)**: Bug — proposed splitting HTML from MP4 to reduce HTML size from 2.59MB to 550KB. User correctly rejected: "گزینه 1 که هیچ تفاوتی نمی کنه. اگه اینترنت کاربر کند باشه صفحه خیلی دیر بالا میاد" (Option 1 makes no difference, on slow internet the page still loads late). The total download is the same — browser still needs both files before scrubbing works. External file splitting only helps when video streaming (HTTP range requests) is supported, which doesn't apply to local standalone HTML. **Lesson: don't suggest file splitting as a size optimization for standalone HTML.**

**Compression iteration 2 (CRF mode can't target size)**: Bug — tried lowering CRF from 24 to 28, 30, 32 to reduce size. But CRF mode gives constant quality with variable bitrate — same CRF on different content produces wildly different sizes. Had to encode multiple times to find the right CRF for a target size. **Lesson: use 2-pass VBR (`-b:v <target> -pass 1/2`) when targeting a specific file size.** Pass 1 analyzes complexity, pass 2 distributes bits intelligently. Same size, better perceptual quality than 1-pass VBR.

**Compression iteration 3 (VLM side-by-side too picky)**: Bug — VLM side-by-side comparisons flagged every bitrate below source as "visible quality loss," even at 1200kbps vs 1290kbps source (only 7% smaller). This led to over-conservative compression. **Lesson: test with SOLO assessment** — show VLM only the compressed frame and ask "is this acceptable for a luxury cosmetics website?" Users never see the original next to the compressed version. Soft-focus/fashion content gets "acceptable" ratings even at 300kbps. The aesthetic itself hides compression artifacts.

**Compression iteration 4 (codec experiments)**: Tested AV1, WebM/VP9, AVIF — all failed. AV1 was 5-10× slower to encode with no size advantage and spotty browser support (Safari < 16.4 has none). WebM/VP9 produced 3.2MB at CRF 32 — **worse** than H.264's 1.5MB at CRF 24, because VP9's complex intra-frame prediction makes random access (scrubbing) slower. AVIF was 18% smaller than WebP for Path B frames but VLM flagged visible quality loss (hair/fabric detail blurred). **Lesson: stick with H.264 + WebP.** Both are already optimal for their respective use cases. Newer codecs don't help for scrub video.

**Compression iteration 5 (ultra-light version rejected)**: Bug — delivered 300kbps (395KB video → 1.04MB HTML, 60% smaller) as the primary deliverable. User rejected: "رزولوشنش خیلی کم شد. فایده نداره" (resolution dropped too much, useless). VLM had said "acceptable" but actual viewing disappointed the user. **Lesson: deliver THREE versions** — original quality, recommended (600kbps), and ultra-light (300kbps). Let the user choose. The recommended version is the actual deliverable; the others are options for different network conditions.

**Final compression delivery** (3 standalone HTML files):
- `rokh-standalone-videoscrub-original.html` — 2.59MB, 1290kbps source, full quality
- `rokh-standalone-videoscrub.html` — 1.52MB, 600kbps 2-pass, recommended balance (41% smaller)
- `rokh-standalone-videoscrub-300k.html` — 1.04MB, 300kbps 2-pass, ultra-light (60% smaller)

**Key compression lessons**:
1. **2-pass VBR beats CRF for size-targeted compression** — CRF gives constant quality with no size target; 2-pass gives target size with intelligent bit distribution.
2. **VLM solo assessment > side-by-side** — users never see the original next to compressed version. Soft-focus content hides artifacts that side-by-side comparison exaggerates.
3. **H.264 + WebP are already optimal** — don't waste time on AV1/VP9/AVIF for scrub video. The newer codecs' advantages are for streaming or tiny thumbnails, not full-size scrubbing media.
4. **Deliver multiple bitrates, not one** — single-bitrate delivery is fragile. Three files (original / recommended / ultra-light) let the user pick the right balance.
5. **The original video is already well-compressed** — re-encoding at the same visual quality (CRF 24-26) produces same or larger files. Real savings require accepting lower visual quality, which soft-focus content tolerates well.
6. **Don't suggest file splitting for standalone HTML** — splitting HTML from media doesn't reduce total download. Browser still needs both files before scrubbing works. The correct solution is to reduce total bytes via 2-pass compression.

---

## Lessons From the Field — why v2 moved compression to Step 0

The multi-bitrate case study above worked, but only after the full
sequence: research → build → deliver → user flags it as heavy → search
for solutions → re-encode → re-deliver. That's the entire skill's
workflow run twice because the size/quality trade-off wasn't surfaced
until the very end.

**v2 lesson**: everything needed to have this conversation — file size,
codec, resolution, aspect ratio — is available from a single `ffprobe`
call that takes under a second, before any research or build work
starts. There's no reason to wait for the user to notice the page feels
heavy when the source file already told you it would be. v2's Step 0
gate asks the same underlying question ("heavy vs. compressed, which do
you want?") at the point where the answer is cheapest to act on, instead
of the point where it's most expensive. See `SKILL.md` Step 0 and
`references/qc-and-compression-research.md` for the implementation.

This doesn't invalidate the case study above — its ffmpeg recipes,
codec findings, and "don't split files" lesson are all still exactly
right and still used by Step 0's "robust/compressed" path. It just
changes *when* the decision gets made.
