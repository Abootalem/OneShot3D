# OneShot3D

A [Claude Skill](https://github.com/anthropics/skills) that turns a single video file into a premium, scroll-driven "Apple-style" website — the video scrubs as the visitor scrolls, sections reveal in sync, and the whole thing ships as one standalone HTML file with no server required.

Forked from a production skill hardened over 11+ real build iterations, with a v2 pass that adds a **mandatory pre-flight video QC gate**, **network-adaptive delivery**, and **accessibility fallbacks**. Full audit and rationale for every v2 change: [`improvements.txt`](./improvements.txt).

> **One prompt, one video, one professional website.** With this skill, a single prompt plus a single video is enough to produce a complete, professional scroll-driven site — no back-and-forth required. The source video can either be uploaded directly by the user, or generated on the fly through a connected video-generation MCP (e.g. Higgsfield) if the user doesn't have one ready.

## What it does

Point it at a video and it researches comparable premium sites, decides between video-scrub and frame-extraction rendering, and builds a Lenis + GSAP ScrollTrigger scroll experience — animated section reveals, a horizontal marquee, counter animations, bilingual RTL/LTR support — all QA'd against 53 documented anti-patterns before delivery.

## What's new in v2

- **Step 0 pre-flight QC** — probes file size, codec, resolution, and aspect ratio *before* any design work starts, and asks a single up-front question (heavy/cinematic vs. robust/compressed vs. network-adaptive) instead of discovering the deliverable is too heavy after the fact.
- **Audio stripped unconditionally** — every build mutes its video anyway, so a source audio track is pure dead weight; it's removed by default, not offered as a choice.
- **Network-adaptive delivery** — ships light + recommended video tiers together and picks the right one per visitor at runtime via the Network Information API.
- **`prefers-reduced-motion` fallback** — required, not optional, given the whole technique is scroll-jacking.
- **Codec guidance reconciled with 2026 encoder data** — keeps H.264 as the safe default for scrub video, with a measured (not assumed) path to testing SVT-AV1 when a user wants the smallest possible file.
- **"3D" scoped honestly** — default output is 2D canvas/video parallax, not WebGL geometry; true Three.js 3D is opt-in only, behind its own performance budget check.

## Live examples

Built with the workflow this skill encodes:

- [ROKH Cosmetics](https://ai-1.ir/showcase/beautyorg) — the original 11-iteration case study documented in `references/lessons-learned.md`
- [BMW Parts Pro](https://ai-1.ir/showcase/bmw)
- [Lotus Jewels](https://ai-1.ir/showcase/lotus-standalone)

## Install

**Claude Code / Claude Cowork:**
```bash
git clone https://github.com/Abootalem/OneShot3D
cp -r OneShot3D ~/.claude/skills/OneShot3D
```

**Claude.ai:** upload the `OneShot3D` folder as a custom skill from the skills picker.

Requires `ffmpeg`/`ffprobe` and (for the QA loop) Python + Playwright available in the environment.

## Structure

```
OneShot3D/
├── SKILL.md                              # entry point — read this first
├── improvements.txt                      # audit of the v1 skill + rationale for every v2 change, with sources
└── references/
    ├── workflow-detail.md                # full step-by-step build commands (ffmpeg, HTML/CSS/JS)
    ├── lessons-learned.md                 # annotated production postmortems
    ├── templates-and-scripts.md           # copy-paste CSS/JS blocks
    └── qc-and-compression-research.md     # v2: QC thresholds, compression recipes, a11y patterns, sources
```

## License

MIT — see [LICENSE](./LICENSE).

## Author

**Abootaleb Moradi** — [ai-1.ir](https://ai-1.ir) (AI-1 Academy: prompt engineering, AI agent systems, and applied AI tooling)
Contact: [Telegram](https://t.me/+989398770326) · [email](mailto:Abootalebmoradi@gmail.com)

---

## درباره

این اسکیل توسط **دکتر ابوطالب مرادی** توسعه داده شده — بر پایهٔ تجربهٔ واقعی ساخت چند وب‌سایت اسکرولی لوکس (رُخ، بی‌ام‌و، لوتوس) که نمونه‌هاشون در بالا لینک شده. برای آموزش تخصصی مهندسی پرامپت، سیستم‌های ایجنتیک و ابزارهای هوش مصنوعی کاربردی، به [ai-1.ir](https://ai-1.ir) سر بزنید.

با این اسکیل، تنها با **یک پرامپت و یک ویدیو** می‌توان یک وب‌سایت کاملاً حرفه‌ای و اسکرولی ساخت. ویدیوی مورد نیاز را کاربر می‌تواند خودش آپلود کند، یا در صورت نداشتن ویدیوی آماده، از طریق یک MCP تولید ویدیو (مثل Higgsfield) در همان لحظه بسازد.
