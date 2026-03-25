# Mobile UX Fixer - Claude Code Skill

> Comprehensive mobile UX audit, visual inspection, and auto-fix skill for Claude Code.
> 40+ automated checks across 10 phases. Every check comes from real production bugs.

## What It Does

Run `mobile audit` (or `תקן מובייל`) and Claude Code will automatically:

1. **See your site** on mobile via Playwright MCP (auto-installs if missing)
2. **Scan your code** for 40+ known mobile UX issues
3. **Fix problems** directly in your codebase
4. **Report results** with file paths, line numbers, and explanations

## 10 Audit Phases

| Phase | Checks | What It Catches |
|-------|--------|-----------------|
| **0. Visual Inspection** | 4 | Screenshots at mobile viewports, scroll-through, overflow, landscape |
| **1. Viewport & Layout** | 5 | Address bar jumps, dvh/svh, safe areas, scroll lock, overflow |
| **2. Scroll & Parallax** | 6 | background-attachment, GSAP ScrollTrigger, Framer Motion, Lenis, scroll-snap |
| **3. Animations** | 6 | Layout-triggering animations, will-change, reduced motion, backdrop-filter, flicker |
| **4. Images & Media** | 5 | Cropping, lazy/priority loading, dimensions (CLS), WebP/AVIF, art direction |
| **5. Touch & Forms** | 6 | Touch targets (44px), iOS font zoom, passive listeners, inputmode, keyboard |
| **6. Typography** | 4 | Min font sizes, fluid clamp(), line-height, line length |
| **7. Core Web Vitals** | 5 | LCP, CLS from fonts, INP/TBT, third-party scripts, font loading |
| **8. SEO & PWA** | 4 | Viewport meta, theme-color, Open Graph, PWA manifest |
| **9. Accessibility** | 4 | Hamburger menu a11y, focus management, skip-to-content, ARIA |
| **10. RTL & Dark Mode** | 2 | Logical properties, prefers-color-scheme |

## Installation

### Option 1: Add as Claude Code skill (recommended)

```bash
claude skill add --url https://github.com/mediawave-dev/mobile-ux-fixer
```

### Option 2: Manual installation

1. Copy the `SKILL.md` file to `~/.claude/skills/mobile-ux-fixer/SKILL.md`
2. Optionally copy the `references/` folder for detailed research docs

## Usage

Just tell Claude Code:

```
fix mobile
mobile audit
check mobile
תקן מובייל
בדיקת מובייל
```

Or be specific:

```
check my site for mobile scroll issues
run the full mobile audit checklist
fix Core Web Vitals on mobile
check mobile accessibility
```

## Playwright MCP (Visual Inspection)

The skill auto-installs Playwright MCP if not present, giving Claude Code "eyes" to:

- Take mobile screenshots at 320px, 375px, 390px, 412px viewports
- Scroll through the page and capture each section
- Detect horizontal overflow visually
- Verify animations by comparing before/after screenshots
- Test landscape orientation

## Frameworks Supported

Works with any web project. Has specific checks for:

- **React / Next.js** - Framer Motion layout animations, React-specific patterns
- **GSAP** - ScrollTrigger mobile issues, pin-spacer bugs
- **Lenis** - syncTouch configuration, GSAP ticker integration
- **Tailwind CSS** - Class-based detection patterns
- **Vue / Svelte** - Component file scanning
- **Vanilla CSS/JS** - Universal patterns

## Research

Built from extensive research across 50+ sources. Full research reports are in `references/`:

- Mobile UX issues 2024-2026
- Playwright MCP mobile testing methodology
- Mobile animation best practices (GSAP, Framer Motion, Lenis, CSS)
- Mobile SEO, PWA, fonts, dark mode, RTL

## License

MIT
