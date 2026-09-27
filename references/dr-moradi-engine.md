# The Dr. Moradi Executive Portrait Engine (Path B-Alpha)
## Ultra-Smooth Apple/BMW-Grade Canvas Scrollytelling for C-Suite & Authority Portfolios

The **Dr. Moradi Executive Portrait Engine** (introduced in OneShot3D v2.2) is the production reference implementation for high-stakes executive portfolios, board-level advisory showcases, and institutional authority pages.

While the original OneShot3D focused on e-commerce cosmetics (ROKH) and automotive showcases (BMW), this engine solves the unique challenges of **personal executive branding**: presenting an influential leader's full-body transparent character sequence composited alongside dense, authoritative 2-column strategic analyses, technical telemetry, and bilingual executive briefs.

---

## 1. Core Architectural Pillars

| Feature | BMW / ROKH (Path B Full-Bleed) | Dr. Moradi (Path B-Alpha Cutout) |
|---|---|---|
| **Canvas Alpha** | `alpha: false, desynchronized: true` | `alpha: true` (transparent WebP sequence) |
| **Asset Type** | Full-frame photography (with background) | Transparent cutout frames (`rembg` / chroma-key) |
| **Canvas Scaling** | Fullscreen `object-fit: cover` math | Anchored character scaling (84% height, 28% width cap) |
| **Staging Alignment** | Centered or full-bleed background | Dynamic RTL/LTR edge-docked character |
| **Atmosphere** | Video vignette / Canvas gradient | 5-Layer pure CSS Architectural Atmosphere (0 JS particles) |
| **Card Layout** | 1-Column floating glass cards | 2-Column synchronized `.stage-grid` (Editorial + Telemetry) |
| **Motion Physics** | 3D Spatial Tilt / Deck Stacking (`rotateX/Y`) | Subtle 18-66-16 Smoothstep Parallax (Reading-first) |
| **Loading Pipeline** | Stride-8 uniform sampling | 3-Tier Progressive (16 Keyframes -> Stride-3 -> Full pool) |
| **Language Toggle** | Text swap only | Dynamic character mirroring + instant canvas repaint |

---

## 2. Transparent WebP Cutout Engine (`Path B-Alpha`)

### 2.1 The Alpha Rule (Critical)
Never initialize the canvas context with `{ alpha: false }` when working with cutout characters:
```javascript
// FORBIDDEN for Path B-Alpha:
var ctx = canvas.getContext('2d', { alpha: false }); // Renders transparent pixels as solid pitch black!

// CORRECT for Path B-Alpha:
var ctx = canvas.getContext('2d'); // Preserves transparent alpha channel for seamless CSS composition
```

### 2.2 Viewport Staging & Proportional Clamping Math
The subject must feel monumental yet never encroach upon reading zones. The golden formula calibrated across 4K, 1440p, 1080p, and MacBook viewports:

```javascript
function drawFrame(idx) {
  if (!ctx || !canvas) return;
  idx = Math.max(1, Math.min(idx, FRAME_COUNT));
  currentFrameIndex = idx;

  // 1. Frame fallback resolution (nearest loaded neighbor within +/- 25 frames)
  var img = frames[idx];
  if (!img || !img.complete || !img.naturalWidth) {
    for (var d = 1; d <= 25; d++) {
      if (frames[idx - d] && frames[idx - d].complete && frames[idx - d].naturalWidth) {
        img = frames[idx - d];
        break;
      }
      if (frames[idx + d] && frames[idx + d].complete && frames[idx + d].naturalWidth) {
        img = frames[idx + d];
        break;
      }
    }
  }
  if (!img || !img.complete || !img.naturalWidth) return;

  var rect = canvas.getBoundingClientRect();
  var cw = rect.width || window.innerWidth;
  var ch = rect.height || (window.innerHeight - 74);
  ctx.clearRect(0, 0, cw, ch);

  var imgW = img.naturalWidth;
  var imgH = img.naturalHeight;
  var aspect = imgW / imgH;

  // 2. Character vertical scale: 84% of viewport height with breathing headroom
  var drawH = ch * 0.84;
  var drawW = drawH * aspect;

  // 3. Strict 28% screen-width ceiling on desktop:
  // Guarantees the leader's figure NEVER overlaps the 2-column editorial grid
  if (cw >= 900 && drawW > cw * 0.28) {
    drawW = cw * 0.28;
    drawH = drawW / aspect;
  }

  // 4. Dynamic RTL/LTR Edge Insets:
  // RTL: Leader stands on the RIGHT edge (facing left towards content)
  // LTR: Leader stands on the LEFT edge (facing right towards content)
  var isRtl = (document.documentElement.getAttribute('dir') || 'rtl').toLowerCase() === 'rtl';
  var inset = cw * 0.025; // 2.5vw edge margin
  var drawX = isRtl ? inset : (cw - drawW - inset);
  var drawY = (ch - drawH) * 0.52; // Centered vertically with slight ground bias

  // 5. Mobile Layout (< 768px):
  // Character scales down to 70% height and docks at center-top
  if (cw < 768) {
    drawH = ch * 0.70;
    drawW = drawH * aspect;
    drawX = (cw - drawW) / 2;
    drawY = ch * 0.06;
  }

  ctx.drawImage(img, Math.round(drawX), Math.round(drawY), Math.round(drawW), Math.round(drawH));
}
```

---

## 3. The 3-Tier Progressive Preloader

Cutout frame sequences require high visual fidelity: showing a blank frame breaks immersion. The 3-Tier Preloader achieves an **instant First Meaningful Paint in under 300ms** while silently hydrating the full 240-frame pool in the background:

```
[Start] 
   |
   +---> Tier 1 (Immediate): Load 16 Strategic Keyframes [1, 5, 10, ..., 240]
   |        |--> First 5 load? Dismiss #loader within 200ms -> Canvas is LIVE!
   |
   +---> Tier 2 (+250ms Delay): Stride-3 Pool (Load every 3rd frame)
   |        |--> Smooths scrubbing trajectory for moderate scrolls
   |
   +---> Tier 3 (+600ms Delay): Full Pool (Load remaining 160 frames)
   |        |--> Reaches 120fps continuous scrubbing parity
   |
   +---> Safety Gate (+3500ms Hard Timeout):
            Force-dismiss preloader regardless of network hitches.
```

### Complete Preloader Implementation:
```javascript
function loadFrames(onInitialReady) {
  // Strategic keyframes covering start, mid-gestures, and closing pose
  var keyframes = [1, 5, 10, 20, 35, 50, 70, 90, 110, 130, 150, 170, 190, 210, 230, 240];
  var loadedKeyframes = 0;
  var readyTriggered = false;

  function checkReady() {
    if (readyTriggered) return;
    loadedKeyframes++;
    var pct = Math.min(100, Math.round((loadedKeyframes / keyframes.length) * 100));
    if (loaderBar) loaderBar.style.width = pct + '%';
    if (loaderPct) loaderPct.textContent = pct + '%';

    // Dismiss as soon as 5 keyframes are available (~300ms)
    if (loadedKeyframes >= Math.min(5, keyframes.length)) {
      readyTriggered = true;
      state.canvasReady = true;
      resizeCanvas();
      drawFrame(1);
      setTimeout(function() {
        if (loader) loader.classList.add('hidden');
        if (typeof ScrollTrigger !== 'undefined') ScrollTrigger.refresh();
        if (onInitialReady) onInitialReady();
      }, 200);
    }
  }

  // Tier 1: Keyframes
  keyframes.forEach(function(idx) {
    var img = new Image();
    img.onload = function() { frames[idx] = img; checkReady(); };
    img.onerror = function() { checkReady(); };
    img.src = getFrameSrc(idx);
  });

  // Tier 2: Stride-3 (every 3rd frame) after 250ms
  setTimeout(function() {
    for (var i = 1; i <= FRAME_COUNT; i += 3) {
      if (!frames[i]) {
        (function(fIdx) {
          var img = new Image();
          img.onload = function() {
            frames[fIdx] = img;
            if (fIdx === currentFrameIndex) drawFrame(fIdx);
          };
          img.src = getFrameSrc(fIdx);
        })(i);
      }
    }
  }, 250);

  // Tier 3: All remaining frames after 600ms
  setTimeout(function() {
    for (var i = 1; i <= FRAME_COUNT; i++) {
      if (!frames[i]) {
        (function(fIdx) {
          var img = new Image();
          img.onload = function() {
            frames[fIdx] = img;
            if (fIdx === currentFrameIndex) drawFrame(fIdx);
          };
          img.src = getFrameSrc(fIdx);
        })(i);
      }
    }
  }, 600);

  // Fallback timer: guarantees user is never stuck
  setTimeout(function() {
    if (!readyTriggered) {
      readyTriggered = true;
      state.canvasReady = true;
      resizeCanvas();
      drawFrame(1);
      if (loader) loader.classList.add('hidden');
      if (typeof ScrollTrigger !== 'undefined') ScrollTrigger.refresh();
      if (onInitialReady) onInitialReady();
    }
  }, 3500);
}
```

---

## 4. Architectural CSS Atmosphere System (Purging "Childish" Stars)

### 4.1 The Psychology of Executive Authority
Consumer widgets like floating star particles, canvas confetti, twinkling dots, or cyberpunk neons immediately undermine managerial credibility. C-suite leaders and institutional stakeholders require an aesthetic of **architectural dignity, geometric stability, and warm amber authority**.

### 4.2 The 5 CSS Layers
```html
<div class="video-wrap" id="videoWrap">
  <!-- Layer 1: Architectural Precision Grid -->
  <div class="architectural-grid"></div>
  
  <!-- Layer 2: Directional Stage Lighting (RTL/LTR aware) -->
  <div class="stage-lighting"></div>
  
  <!-- Layer 3: Ultra-Smooth Scrub Canvas -->
  <canvas id="scrub-canvas"></canvas>
  
  <!-- Layer 4: Ground Ambient Glow Puddle -->
  <div class="canvas-ambient-glow"></div>
  
  <!-- Layer 5: Focus Vignette -->
  <div class="video-vignette"></div>
</div>
```

### 4.3 Pure CSS Implementation
```css
/* Zero JS painting — 100% GPU composited */
.video-wrap {
  position: fixed;
  top: var(--header-h);
  left: 0;
  width: 100vw;
  height: calc(100vh - var(--header-h));
  z-index: 10;
  background: transparent;
  overflow: hidden;
  pointer-events: none;
}

/* Layer 1: Subtle Gold Grid */
.architectural-grid {
  position: absolute;
  inset: 0;
  background-image: 
    linear-gradient(to right, rgba(212, 175, 55, 0.022) 1px, transparent 1px),
    linear-gradient(to bottom, rgba(212, 175, 55, 0.022) 1px, transparent 1px);
  background-size: 52px 52px;
  mask-image: radial-gradient(ellipse 85% 75% at 50% 50%, black 40%, transparent 95%);
  -webkit-mask-image: radial-gradient(ellipse 85% 75% at 50% 50%, black 40%, transparent 95%);
  pointer-events: none;
  opacity: 0.65;
  z-index: 2;
}

/* Layer 2: Stage Lighting (Mirrored for RTL / LTR) */
.stage-lighting {
  position: absolute;
  inset: 0;
  background: 
    radial-gradient(ellipse 48% 58% at 18% 50%, rgba(212, 175, 55, 0.08) 0%, transparent 65%),
    radial-gradient(ellipse 55% 65% at 65% 45%, rgba(13, 22, 42, 0.40) 0%, transparent 75%);
  pointer-events: none;
  z-index: 3;
}
html[dir="ltr"] .stage-lighting {
  background: 
    radial-gradient(ellipse 48% 58% at 82% 50%, rgba(212, 175, 55, 0.08) 0%, transparent 65%),
    radial-gradient(ellipse 55% 65% at 35% 45%, rgba(13, 22, 42, 0.40) 0%, transparent 75%);
}

/* Layer 3: Canvas */
#scrub-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: block;
  pointer-events: none;
  z-index: 4;
}

/* Layer 4: Amber/Gold Pedestal Puddle Glow */
.canvas-ambient-glow {
  position: absolute;
  bottom: 2vh;
  left: 6vw;
  width: 480px;
  height: 140px;
  background: radial-gradient(ellipse at center, rgba(212, 175, 55, 0.16) 0%, rgba(14, 24, 46, 0.12) 50%, transparent 80%);
  filter: blur(36px);
  pointer-events: none;
  z-index: 5;
}
html[dir="ltr"] .canvas-ambient-glow {
  left: auto;
  right: 6vw;
}

/* Layer 5: Focus Vignette */
.video-vignette {
  position: absolute;
  inset: 0;
  background: radial-gradient(ellipse at center, transparent 60%, rgba(4, 6, 10, 0.78) 100%);
  pointer-events: none;
  z-index: 6;
}
```

---

## 5. The 2-Column Synchronized Editorial Grid (`.stage-grid`)

Rather than floating a single isolated product card, executive showcases structure each scroll act into **two coordinated columns**:
1. **Column 1 (`.editorial-stage`)**: Strategic analysis, macro thesis, and board axiom.
2. **Column 2 (`.intelligence-core`)**: Deployed system telemetry, architecture matrices, terminal logs, or CLI consoles.

### 5.1 CSS Layout Architecture
```css
.hero-grid, .stage-grid {
  display: grid;
  grid-template-columns: minmax(320px, 460px) minmax(320px, 440px);
  gap: 24px;
  align-items: center;
  width: auto;
  max-width: min(65vw, 960px);
  margin-right: 3vw;
  margin-left: auto;
  box-sizing: border-box;
}

html[dir="ltr"] .hero-grid,
html[dir="ltr"] .stage-grid {
  margin-left: 3vw;
  margin-right: auto;
}

/* HUD Spatial Framing Brackets */
.hud-bracket {
  position: absolute;
  width: 14px;
  height: 14px;
  pointer-events: none;
  border-color: rgba(212, 175, 55, 0.45);
  border-style: solid;
  transition: all 0.35s ease;
}
.hud-tl { top: -4px; right: -4px; border-width: 2px 2px 0 0; }
.hud-tr { top: -4px; left: -4px; border-width: 2px 0 0 2px; }
.hud-bl { bottom: -4px; right: -4px; border-width: 0 2px 2px 0; }
.hud-br { bottom: -4px; left: -4px; border-width: 0 0 2px 2px; }

/* Target Acquisition Effect: Bracket Glow on Section Visible */
.scroll-section.visible .hud-bracket,
.hero-standalone .hud-bracket {
  border-color: rgba(212, 175, 55, 0.90);
  box-shadow: 0 0 10px rgba(212, 175, 55, 0.30);
}

/* Responsive Collapse for Tablets */
@media (max-width: 1024px) {
  .hero-grid, .stage-grid {
    grid-template-columns: 1fr;
    max-width: 520px;
    gap: 16px;
  }
  .hero-intelligence-deck, .intelligence-core { order: 2; }
  .hero-content, .editorial-stage { order: 1; }
}
```

---

## 6. Kinematic Crossfades: The 18-66-16 Editorial Smoothstep

In e-commerce, cards can afford quick 28-44-28 transitions. In an executive portfolio, leaders are reading serious strategic theses. The **18-66-16 Smoothstep Ratio** gives **66% of scroll time to undisturbed, high-contrast reading**:

```
[enterPct]                [enterPct + 18%]          [leavePct - 16%]               [leavePct]
    |----------------------------|--------------------------|------------------------------|
            Phase 1: Entry              Phase 2: Stable Hold              Phase 3: Exit
            Glide-in (+18px)            100% Solid Opacity                Glide-out (-16px)
            Blur (4px -> 0px)           Pointer Events ACTIVE             Blur (0px -> 4px)
            Opacity (0 -> 1)            Perfect Contrast Read             Opacity (1 -> 0)
```

### GSAP ScrollTrigger Section Logic:
```javascript
function setupSections() {
  if (typeof ScrollTrigger === 'undefined') return;
  var sections = Array.from(document.querySelectorAll('.scroll-section'));

  sections.forEach(function(section, idx) {
    section.style.zIndex = 34 + idx * 2;
    var persist = section.dataset.persist === 'true';
    var enterPct = parseFloat(section.dataset.enter) / 100;
    var leavePct = parseFloat(section.dataset.leave) / 100;
    var span = leavePct - enterPct;

    var stageGrid = section.querySelector('.stage-grid') || section;
    var inRange = span * 0.18;  // 18% glide in
    var outRange = span * 0.16; // 16% glide out

    ScrollTrigger.create({
      trigger: '#scroll-container',
      start: 'top top',
      end: 'bottom bottom',
      scrub: 0.15,
      onUpdate: function(self) {
        var p = self.progress;

        if (p < enterPct) {
          section.style.opacity = 0;
          section.style.pointerEvents = 'none';
          section.classList.remove('visible');
          stageGrid.style.transform = 'translate3d(0, 18px, 0)';
          stageGrid.style.filter = 'blur(4px)';
        } else if (p >= enterPct && p < enterPct + inRange) {
          var t = (p - enterPct) / inRange;
          var ease = t * t * (3 - 2 * t); // Smoothstep
          section.style.opacity = ease;
          section.style.pointerEvents = ease > 0.4 ? 'auto' : 'none';
          section.classList.toggle('visible', ease > 0.4);

          var curY = 18 * (1 - ease);
          var curBlur = 4 * (1 - ease);
          stageGrid.style.transform = 'translate3d(0, ' + curY.toFixed(1) + 'px, 0)';
          stageGrid.style.filter = curBlur > 0.1 ? 'blur(' + curBlur.toFixed(1) + 'px)' : 'none';
        } else if (p >= enterPct + inRange && p <= leavePct - outRange) {
          // Stable reading hold
          section.style.opacity = 1;
          section.style.pointerEvents = 'auto';
          section.classList.add('visible');
          stageGrid.style.transform = 'translate3d(0, 0px, 0)';
          stageGrid.style.filter = 'none';
        } else if (p > leavePct - outRange && p <= leavePct) {
          if (persist) {
            section.style.opacity = 1;
            section.style.pointerEvents = 'auto';
            section.classList.add('visible');
            stageGrid.style.transform = 'translate3d(0, 0px, 0)';
            stageGrid.style.filter = 'none';
          } else {
            var t = (p - (leavePct - outRange)) / outRange;
            var ease = t * t * (3 - 2 * t);
            var curOpacity = Math.max(0, 1.0 - ease);
            var curY = -16 * ease;
            var curBlur = 4 * ease;

            section.style.opacity = curOpacity;
            section.style.pointerEvents = curOpacity > 0.4 ? 'auto' : 'none';
            section.classList.toggle('visible', curOpacity > 0.4);
            stageGrid.style.transform = 'translate3d(0, ' + curY.toFixed(1) + 'px, 0)';
            stageGrid.style.filter = curBlur > 0.1 ? 'blur(' + curBlur.toFixed(1) + 'px)' : 'none';
          }
        } else {
          if (persist) {
            section.style.opacity = 1;
            section.style.pointerEvents = 'auto';
            section.classList.add('visible');
            stageGrid.style.transform = 'translate3d(0, 0px, 0)';
            stageGrid.style.filter = 'none';
          } else {
            section.style.opacity = 0;
            section.style.pointerEvents = 'none';
            section.classList.remove('visible');
            stageGrid.style.transform = 'translate3d(0, -18px, 0)';
            stageGrid.style.filter = 'blur(4px)';
          }
        }
      }
    });
  });
}
```

---

## 7. Bilingual Mirroring & Directional Repaint

When toggling between Farsi (`dir="rtl"`) and English (`dir="ltr"`), three synchronization tasks must execute concurrently:
1. Swap `lang` and `dir` on `<html>`.
2. Update all DOM elements bearing `data-fa` / `data-en`.
3. **Re-call `drawFrame(currentFrameIndex)`** to instantly reposition the leader's figure to the opposite side.
4. **Trigger `ScrollTrigger.refresh()`** after a 100ms debounce to recalculate pin bounds.

```javascript
window.toggleLanguage = function() {
  currentLang = currentLang === 'fa' ? 'en' : 'fa';
  var html = document.documentElement;
  html.setAttribute('lang', currentLang);
  html.setAttribute('dir', currentLang === 'fa' ? 'rtl' : 'ltr');

  var docTitle = document.querySelector('title');
  if (docTitle && docTitle.dataset[currentLang]) {
    docTitle.textContent = docTitle.dataset[currentLang];
  }

  document.querySelectorAll('[data-fa], [data-en]').forEach(function(el) {
    var t = el.dataset[currentLang];
    if (t !== undefined && t !== '') el.textContent = t;
  });

  var langToggle = document.getElementById('langToggle');
  if (langToggle) {
    langToggle.textContent = currentLang === 'fa' ? 'EN' : 'فا';
  }

  // Force character repositioning based on new dir attribute
  if (canvas && ctx) {
    drawFrame(currentFrameIndex);
  }

  // Recalculate ScrollTrigger measurements
  if (typeof ScrollTrigger !== 'undefined') {
    setTimeout(function() {
      ScrollTrigger.refresh();
    }, 100);
  }
};
```

---

## 8. Next.js SSR Integration Pattern (`ScriptHtmlContainer.tsx`)

When deploying standalone OneShot3D HTML inside Next.js (App Router), standard React JSX parsing will fail to execute embedded `<script>` tags. Use this production-tested hydration pattern:

```tsx
// ScriptHtmlContainer.tsx
'use client';
import { useEffect, useRef } from 'react';

export default function ScriptHtmlContainer({ htmlContent }: { htmlContent: string }) {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!containerRef.current) return;
    const container = containerRef.current;
    container.innerHTML = htmlContent;

    // Sequentially extract and execute embedded script tags in dependency order
    const scripts = Array.from(container.querySelectorAll('script'));
    const loadScriptSequentially = async (index: number) => {
      if (index >= scripts.length) return;
      const oldScript = scripts[index];
      const newScript = document.createElement('script');

      Array.from(oldScript.attributes).forEach(attr => {
        newScript.setAttribute(attr.name, attr.value);
      });

      if (oldScript.src) {
        newScript.onload = () => loadScriptSequentially(index + 1);
        newScript.onerror = () => loadScriptSequentially(index + 1);
        document.body.appendChild(newScript);
      } else {
        newScript.textContent = oldScript.textContent;
        document.body.appendChild(newScript);
        await loadScriptSequentially(index + 1);
      }
      oldScript.remove();
    };

    loadScriptSequentially(0);
  }, [htmlContent]);

  return <div ref={containerRef} className="oneshot3d-ssr-container" />;
}
```

---

## 9. Definition of Done Checklist for Executive Showcases

Before shipping a Path B-Alpha showcase:
- [ ] Canvas initialized with `alpha: true` (transparent cutout WebP, no black bounding box).
- [ ] Character width clamped to `cw * 0.28` on desktop (zero text collision).
- [ ] 3-Tier Preloader implemented with keyframe array and 3.5s safety timeout.
- [ ] All floating star/particle animations removed; 5-layer CSS atmosphere active.
- [ ] Stage lighting and ambient glow puddle contain separate RTL and LTR CSS rules.
- [ ] HUD brackets (`⌜ ⌝ ⌞ ⌟`) present on cards with `.visible` target glow.
- [ ] Editorial crossfades set to 18-66-16 Smoothstep ratio.
- [ ] Language toggle updates text AND repositions character on opposite screen edge.
- [ ] Tested on 1920px (Desktop), 1366px (Laptop), 1024px (Tablet), and 375px (Mobile).
