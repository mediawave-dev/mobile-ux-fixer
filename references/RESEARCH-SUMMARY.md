# Mobile UX Fixer v2.0 - Research Summary

> 4 parallel research agents conducted deep web research on 2024-2026 mobile UX best practices.
> Date: 2026-03-25

## Research Reports

1. **mobile-ux-research-2024-2026.md** - CSS issues, JS gotchas, Core Web Vitals, navigation, iOS/Android bugs, forms, typography, images, accessibility
2. **RESEARCH-playwright-mobile-testing.md** - Playwright MCP setup, device emulation, visual testing, animation testing, performance testing, accessibility testing
3. **RESEARCH-mobile-animation-best-practices.md** - GSAP, Framer Motion, Lenis, scroll-driven animations, View Transitions, reduced motion, mobile animation bugs
4. **mobile-web-standards-research-2024-2026.md** - SEO, meta tags, PWA, performance metrics, fonts, 3rd-party scripts, dark mode, responsive design, security, RTL

## Key Findings for Skill Upgrade

### New Capabilities to Add

**Playwright MCP Visual Inspection (Phase 0)**
- Configure with `--device "iPhone 15" --headless --caps vision`
- Take screenshots at 375px, 390px, 412px mobile viewports
- Scroll through page systematically to capture all sections
- Use `browser_evaluate` to run JS checks in-page (overflow detection, touch targets, CLS)
- Check animations by scrolling and taking before/after screenshots

**Scroll-Snap iOS Bugs (NEW)**
- iOS WebKit caches snap points, causing misaligned snaps
- Fix: `scroll-snap-stop: always` + force reflow with `void child.offsetHeight`

**Body Scroll Lock for Modals (NEW)**
- iOS ignores `overflow: hidden` on body
- Fix: `position: fixed` + save/restore scroll position, or `inert` attribute

**Passive Event Listeners (NEW)**
- All `touchstart`, `touchmove`, `wheel` must be `{ passive: true }`
- Fix: add passive flag or use `touch-action` CSS property

**Overscroll Behavior (NEW)**
- `overscroll-behavior: contain` prevents scroll chaining
- `overscroll-behavior-y: none` on html/body prevents pull-to-refresh

**GSAP ScrollTrigger Mobile Issues (NEW)**
- Pin-spacer height mismatch with address bar
- Fix: `ScrollTrigger.normalizeScroll(true)` or disable pins on mobile

**Framer Motion Layout Animation (NEW)**
- `layoutScroll` prop required on scrollable containers
- Layout animations break when container is scrolled

**Lenis Configuration (NEW)**
- `syncTouch: false` for better mobile performance
- `autoRaf: false` when using GSAP ticker integration

**Reduced Motion (NEW)**
- Must check for `prefers-reduced-motion: reduce`
- Apply to all animations including GSAP, Framer Motion, CSS

**CSS Scroll-Driven Animations (NEW)**
- `animation-timeline: scroll()` / `view()` - Chrome 115+, Safari 18+ partial
- Must provide `@supports` fallback with IntersectionObserver

**iOS Backdrop-Filter + Transform Bug (NEW)**
- Combining `backdrop-filter` with `background-color` causes white rendering
- Fix: use pseudo-element layer separation

**Page Load Animation Flicker (NEW)**
- Animations flash briefly on load
- Fix: double-rAF technique or hidden-until-ready class

**Core Web Vitals Checks (NEW)**
- LCP: hero images need `fetchpriority="high"`, must NOT have `loading="lazy"`
- CLS: images need `width`/`height` attributes, fonts need `font-display`
- INP/TBT: `scheduler.yield()` breaks long tasks, defer non-critical JS

**Font Loading Strategy (NEW)**
- `font-display: swap` for body text, `optional` for best CWV
- Preload max 2 critical fonts with `crossorigin`
- Use `size-adjust` and `ascent-override` to match fallbacks

**Third-Party Scripts (NEW)**
- YouTube embeds block main thread 4.5s
- Fix: facade patterns, lazy-load on interaction, Partytown for web workers

**Mobile SEO Meta Tags (NEW)**
- Validate viewport meta, theme-color, OG images
- Check for `user-scalable=no` (accessibility violation)

**PWA Checklist (NEW)**
- Manifest.json validation, icons (192x192, 512x512), theme-color
- Service workers no longer required for installability (Chrome 2024)

**Dark Mode Support (NEW)**
- `prefers-color-scheme` media query
- Dual `theme-color` meta tags
- Image contrast issues in dark mode

**RTL/Hebrew Mobile (NEW)**
- CSS logical properties check
- `dir="rtl"` attribute
- Bidirectional text with numbers/URLs

**Typography Best Practices (NEW)**
- 16px min body, clamp() for fluid sizing
- Line-height 1.5 for body
- Max 45 chars per line on mobile (`max-width: 65ch`)

**Input Optimization (NEW)**
- `inputmode` attribute for optimal keyboard
- `enterkeyhint` for action button label
- Virtual keyboard handling with `visualViewport` API

**Hamburger Menu Accessibility (NEW)**
- `aria-expanded`, `aria-label`, focus trap
- `inert` on background content when menu open

### Enhanced Existing Checks

- **Touch targets**: WCAG 2.5.8 requires 24x24px minimum (AA), best practice 44x44px
- **Image loading**: Add AVIF/WebP format check, art direction with `<picture>`
- **Horizontal overflow**: Add Playwright-based visual verification
- **Font zoom**: Add responsive approach `text-base md:text-sm`
