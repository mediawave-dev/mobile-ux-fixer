# Mobile UX Fixer - Detailed Phase Checks

> Reference file with all detect patterns and fix code for each check.
> Consult this file when running the audit phases.

---

## Phase 1: Viewport and Layout Stability

### Check 1.1: Address Bar Scroll Jump

**The Problem:** On mobile browsers, scrolling hides/shows the address bar, which changes `window.innerHeight`. Any JS that uses `window.innerHeight` during scroll will JUMP when the address bar toggles.

**Detect:**
```
Pattern: addEventListener.*scroll.*innerHeight|innerHeight.*scroll|onScroll.*innerHeight
Files: *.jsx, *.tsx, *.js, *.ts, *.vue, *.svelte
Also check: window.innerHeight in rAF callbacks or IntersectionObserver
```

**Fix:**
```javascript
let vh = window.innerHeight
let lastWidth = window.innerWidth
const onResize = () => {
  if (window.innerWidth !== lastWidth) {
    vh = window.innerHeight
    lastWidth = window.innerWidth
  }
}
window.addEventListener('resize', onResize)
// Use vh instead of window.innerHeight in all scroll calculations
```

### Check 1.2: Dynamic Viewport Height (dvh vs svh)

**The Problem:** `100dvh` changes dynamically when address bar shows/hides, shifting ALL content below.

**Detect:**
```
Pattern: h-\[100dvh\]|height:\s*100dvh|min-h-\[100dvh\]|min-height:\s*100dvh
Also find: height:\s*100vh|min-height:\s*100vh|h-screen
```

**Fix:**
```css
/* Use svh for stable elements */
height: 100svh;

/* Fallback for older browsers */
@supports not (height: 100svh) {
  height: 100vh;
}
```

**When to keep dvh:** Only for overlays/modals that need to fill the current visible area.

### Check 1.3: Fixed/Sticky Elements and Safe Areas

**The Problem:** Fixed elements can be hidden behind notch, home indicator, or rounded corners.

**Detect:**
```
Pattern: (position:\s*(fixed|sticky)(?!.*safe-area))|(fixed|sticky)(?!.*safe-area)
Check viewport meta for: viewport-fit=cover
```

**Fix:**
```css
.fixed-bottom-cta {
  padding-bottom: env(safe-area-inset-bottom);
}
.fixed-top-nav {
  padding-top: env(safe-area-inset-top);
}
```

### Check 1.4: Body Scroll Lock for Modals (iOS)

**The Problem:** iOS Safari ignores `overflow: hidden` on body. Background scrolls through modals.

**Detect:**
```
Pattern: overflow:\s*hidden.*body|body.*overflow:\s*hidden
Check: any modal/dialog component without scroll lock implementation
```

**Fix (Modern -- inert attribute):**
```html
<main inert>...</main>
<dialog open>...</dialog>
```

**Fix (Classic -- position fixed):**
```javascript
let scrollY = 0;
function lockScroll() {
  scrollY = window.scrollY;
  document.body.style.position = 'fixed';
  document.body.style.top = `-${scrollY}px`;
  document.body.style.width = '100%';
}
function unlockScroll() {
  document.body.style.position = '';
  document.body.style.top = '';
  window.scrollTo(0, scrollY);
}
```

### Check 1.5: Horizontal Overflow

**The Problem:** Elements wider than viewport cause horizontal scroll.

**Detect (code):**
```
Pattern: width:\s*\d{4,}px|min-width:\s*\d{3,}px
```

**Detect (Playwright):**
```javascript
document.querySelectorAll('*').forEach(el => {
  if (el.scrollWidth > document.documentElement.clientWidth) {
    console.log('Overflow:', el.tagName, el.className);
  }
});
```

**Fix:**
```css
html, body { overflow-x: hidden; }
img { max-width: 100%; height: auto; }
.table-wrapper { overflow-x: auto; -webkit-overflow-scrolling: touch; }
```

---

## Phase 2: Parallax and Scroll Effects

### Check 2.1: background-attachment: fixed

**The Problem:** Completely broken on iOS Safari and most mobile browsers.

**Detect:**
```
Pattern: background-attachment:\s*fixed|\.parallax|backgroundAttachment
```

**Fix:** Replace with actual img element + JS translateY parallax:
```jsx
<div className="relative overflow-hidden">
  <img src="..." className="absolute inset-0 w-full h-full object-cover" data-parallax="-0.08" />
</div>
```

### Check 2.2: Ken Burns / CSS Animation Scale Consistency

**Detect:**
```
Pattern: @keyframes.*kenburns|@keyframes.*zoom|@keyframes.*scale
Verify: consistent scale start/end values across all variants
```

**Fix:** Same scale range across all variants, only vary translate direction.

### Check 2.3: GSAP ScrollTrigger Mobile Issues

**The Problem:** Pin-spacer height mismatch with address bar. Horizontal scroll sections break on touch.

**Detect:**
```
Pattern: ScrollTrigger\.create|scrollTrigger:|pin:\s*true|scrub:\s*true
Files: *.js, *.ts, *.jsx, *.tsx
```

**Fix:**
```javascript
ScrollTrigger.normalizeScroll(true);
ScrollTrigger.matchMedia({
  "(max-width: 768px)": function() {
    // Mobile-specific -- avoid pins
  },
  "(min-width: 769px)": function() {
    // Desktop -- pins are fine
  }
});
gsap.ticker.lagSmoothing(0);
```

### Check 2.4: Framer Motion Layout Animation Issues

**The Problem:** Layout animations break when container is scrolled. Missing `layoutScroll` prop.

**Detect:**
```
Pattern: layout\s*=|<motion\.|AnimatePresence|layoutId
Files: *.jsx, *.tsx
Check: scrollable parent containers missing layoutScroll prop
```

**Fix:**
```jsx
<motion.div layoutScroll style={{ overflow: 'auto' }}>
  <motion.div layout>...</motion.div>
</motion.div>
```

### Check 2.5: Lenis/Smooth Scroll Mobile Configuration

**The Problem:** `syncTouch: true` causes performance issues on mobile. Wrong autoRaf config with GSAP.

**Detect:**
```
Pattern: new Lenis|import.*lenis|import.*locomotive
```

**Fix:**
```javascript
const lenis = new Lenis({
  syncTouch: false,
  touchMultiplier: 1.5,
  autoRaf: false,
});
lenis.on('scroll', ScrollTrigger.update);
gsap.ticker.add((time) => lenis.raf(time * 1000));
gsap.ticker.lagSmoothing(0);
```

### Check 2.6: Scroll-Snap iOS Bugs

**The Problem:** iOS WebKit caches snap points. Fast flick scrolls to end instead of next item.

**Detect:**
```
Pattern: scroll-snap-type|scroll-snap-align
```

**Fix:**
```css
.scroll-container { scroll-snap-type: x mandatory; }
.scroll-item {
  scroll-snap-align: start;
  scroll-snap-stop: always; /* Prevents skip-scrolling on iOS */
}
```

---

## Phase 3: Animations and Motion

### Check 3.1: Layout-Triggering Animations

**The Problem:** Animating `width`, `height`, `top`, `left`, `margin`, `padding` triggers layout recalc on every frame.

**Detect:**
```
Pattern: transition.*(?:width|height|top|left|right|bottom|margin|padding)
Also: gsap\.(to|from|fromTo).*(?:width|height|top|left|margin|padding)
```

**Fix:** Only animate `transform` and `opacity`:
```css
/* BAD */ transition: width 0.3s, left 0.3s;
/* GOOD */ transition: transform 0.3s, opacity 0.3s;
```

### Check 3.2: will-change Overuse

**Detect:**
```
Pattern: will-change
Count occurrences. More than 5-10 is suspicious.
```

**Fix:** Only on actively animated elements. Add/remove dynamically.

### Check 3.3: Reduced Motion Preferences

**The Problem:** WCAG requires respecting `prefers-reduced-motion`.

**Detect:**
```
Pattern: prefers-reduced-motion
Expected: At least ONE occurrence in CSS. If ZERO and site has animations = FAIL.
```

**Fix:**
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

For GSAP:
```javascript
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
if (prefersReducedMotion) {
  gsap.globalTimeline.timeScale(20);
}
```

### Check 3.4: iOS backdrop-filter + Transform Bug

**The Problem:** Combining `backdrop-filter` with `background-color` causes white rendering on iOS.

**Detect:**
```
Pattern: backdrop-filter.*blur
Check: if same element has both backdrop-filter and background-color
```

**Fix:** Separate into pseudo-element:
```css
.glass-element { position: relative; }
.glass-element::before {
  content: '';
  position: absolute;
  inset: 0;
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  z-index: -1;
}
```

### Check 3.5: Page Load Animation Flicker

**The Problem:** Elements flash at non-animated state before JS initializes.

**Detect:**
```
Pattern: gsap\.from|animate\(|useSpring|initial=\{
```

**Fix (double-rAF):**
```javascript
document.documentElement.classList.add('js-loading');
requestAnimationFrame(() => {
  requestAnimationFrame(() => {
    document.documentElement.classList.remove('js-loading');
  });
});
```
```css
.js-loading [data-animate] { visibility: hidden; }
```

### Check 3.6: Scroll-Driven Animations Fallback

**The Problem:** `animation-timeline: scroll()` not supported in Safari < 18, Firefox.

**Detect:**
```
Pattern: animation-timeline|scroll-timeline|view-timeline
Check: @supports fallback exists
```

**Fix:**
```css
@supports (animation-timeline: scroll()) {
  .parallax-element {
    animation: parallax linear;
    animation-timeline: scroll();
  }
}
```

---

## Phase 4: Images and Media

### Check 4.1: Image Cropping on Mobile

**Detect:** Look for `object-cover` without explicit `object-position`.
**Fix:** `object-position: center 30%;`

### Check 4.2: Image Loading Performance

**Detect:**
```
FAIL if: hero image has loading="lazy"
FAIL if: hero image missing fetchpriority="high"
```

**Fix:**
```html
<img fetchpriority="high" src="hero.webp" width="1200" height="600" alt="..." />
<img loading="lazy" src="content.webp" width="800" height="400" alt="..." />
```

### Check 4.3: Missing Image Dimensions (CLS)

**Detect:**
```
Pattern: <img(?![^>]*width)[^>]*>
```

**Fix:** Add `width`/`height` attributes or use CSS `aspect-ratio`.

### Check 4.4: Modern Image Formats

**Detect:**
```
Pattern: src="[^"]*\.(jpg|jpeg|png)"
```

**Fix:**
```html
<picture>
  <source srcset="image.avif" type="image/avif" />
  <source srcset="image.webp" type="image/webp" />
  <img src="image.jpg" alt="..." width="800" height="400" />
</picture>
```

### Check 4.5: Art Direction for Mobile

**Detect:** Hero images without `<picture>` + `<source media>` for different viewports.

**Fix:**
```html
<picture>
  <source media="(max-width: 768px)" srcset="hero-mobile.webp" />
  <source media="(min-width: 769px)" srcset="hero-desktop.webp" />
  <img src="hero-desktop.jpg" alt="..." />
</picture>
```

---

## Phase 5: Touch, Forms, and Interaction

### Check 5.1: Touch Target Size

**The Problem:** WCAG 2.5.8 (AA) requires 24x24px min. Best practice 44x44px.

**Detect (Playwright):**
```javascript
const small = [];
document.querySelectorAll('a, button, [role="button"], input, select, textarea').forEach(el => {
  const rect = el.getBoundingClientRect();
  if (rect.width < 44 || rect.height < 44) {
    small.push({ tag: el.tagName, text: el.textContent?.slice(0,30), w: rect.width, h: rect.height });
  }
});
JSON.stringify(small);
```

**Fix:**
```css
.touch-target { min-width: 44px; min-height: 44px; }
.icon-btn { position: relative; }
.icon-btn::after { content: ''; position: absolute; inset: -8px; }
```

### Check 5.2: iOS Font Size Zoom

**The Problem:** iOS auto-zooms when input font-size < 16px.

**Detect:**
```
Pattern: text-xs|text-sm|font-size:\s*(1[0-5]|[0-9])px on input/select/textarea
```

**Fix:**
```css
input, select, textarea { font-size: max(16px, 1rem); }
```
Tailwind: `text-base md:text-sm`. NEVER use `user-scalable=no`.

### Check 5.3: Passive Event Listeners

**The Problem:** Non-passive touch/wheel listeners block scrolling.

**Detect:**
```
Pattern: addEventListener\s*\(\s*['"](?:touchstart|touchmove|wheel)['"]
Check: { passive: true } present?
```

**Fix:**
```javascript
element.addEventListener('touchstart', handler, { passive: true });
```

### Check 5.4: Input Type and Inputmode

**Detect:**
```
Pattern: <input(?![^>]*inputmode)[^>]*type="text"[^>]*>
```

**Fix:**
```html
<input type="email" inputmode="email" enterkeyhint="next" />
<input type="tel" inputmode="tel" enterkeyhint="next" />
<input type="text" inputmode="numeric" pattern="[0-9]*" enterkeyhint="done" />
```

### Check 5.5: Virtual Keyboard Handling

**Detect:** Fixed bottom elements without `visualViewport` handling.

**Fix:**
```javascript
if (window.visualViewport) {
  window.visualViewport.addEventListener('resize', () => {
    const offset = window.innerHeight - window.visualViewport.height;
    bottomEl.style.transform = `translateY(-${offset}px)`;
  });
}
```

### Check 5.6: Overscroll Behavior

**Detect:**
```
Pattern: overscroll-behavior
If site has modals and no overscroll-behavior = ISSUE.
```

**Fix:**
```css
.modal-content { overscroll-behavior: contain; }
html, body { overscroll-behavior-y: none; }
```

---

## Phase 6: Typography and Readability

### Check 6.1: Minimum Font Sizes

**Detect:** `font-size:\s*(1[0-1]|[0-9])px`
**Fix:** Body 16px min, UI elements 14px min, never below 12px.

### Check 6.2: Fluid Typography with clamp()

**Detect:** Fixed px font sizes on headings without media query overrides.
**Fix:**
```css
h1 { font-size: clamp(1.75rem, 4vw + 1rem, 3.5rem); }
h2 { font-size: clamp(1.5rem, 3vw + 0.75rem, 2.5rem); }
p  { font-size: clamp(1rem, 1vw + 0.75rem, 1.125rem); }
```

### Check 6.3: Line Height

**Detect:** `line-height:\s*(0\.\d+|1(\.[0-3])?)\s*(;|})`
**Fix:** Body 1.5, headings 1.2, long-form 1.6.

### Check 6.4: Line Length

**Detect:** Text containers without max-width constraint.
**Fix:** `max-width: 65ch; padding-inline: 1rem;`

---

## Phase 7: Core Web Vitals (Mobile)

### Check 7.1: LCP

**Target:** Under 2.5s.
**Detect:**
```
FAIL if: hero image has loading="lazy"
FAIL if: hero image missing fetchpriority="high"
FAIL if: render-blocking CSS/JS in head
Pattern: <script(?!.*defer|async)[^>]*src= in head
```

**Fix:** `fetchpriority="high"` on hero, `defer` scripts, preload critical CSS.

### Check 7.2: CLS from Font Loading

**Detect:** `@font-face` without `font-display`. Google Fonts without `&display=`.
**Fix:** `font-display: swap;` + preload max 2 critical fonts with `crossorigin`.

### Check 7.3: INP/TBT from Long Tasks

**Detect:** Large sync loops, heavy computation in event handlers.
**Fix:** `scheduler.yield()` or `requestIdleCallback` to break up long tasks.

### Check 7.4: Third-Party Script Impact

**Detect:** External scripts (YouTube embeds, chat widgets, analytics without async).
**Fix:** Facade patterns, lazy-load on interaction.

### Check 7.5: Font Loading Strategy

**Detect:** `.ttf`/`.otf` formats, more than 4 font files, missing `crossorigin`.
**Fix:** woff2 only, preload max 2, variable fonts, `size-adjust`/`ascent-override`.

---

## Phase 8: SEO, Meta Tags, and PWA

### Check 8.1: Viewport Meta Tag

**Detect:** Missing `<meta name="viewport">`. FAIL if contains `user-scalable=no`.
**Expected:** `width=device-width, initial-scale=1`

### Check 8.2: Theme Color

**Fix:**
```html
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)" />
<meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)" />
```

### Check 8.3: Open Graph

**Detect:** Missing `og:title`, `og:description`, `og:image`. Image should be 1200x630px.

### Check 8.4: PWA Manifest

**Detect:** Check for `name`, `short_name`, icons (192x192, 512x512), `start_url`, `display`, `theme_color`.

---

## Phase 9: Accessibility on Mobile

### Check 9.1: Hamburger Menu

**Detect:** `hamburger|menu-toggle|mobile-menu|nav-toggle` without `aria-expanded`/`aria-label`.
**Fix:**
```html
<button aria-label="Open navigation menu" aria-expanded="false" aria-controls="mobile-nav">
  <span class="sr-only">Menu</span>
</button>
```

### Check 9.2: Focus Management

**Detect:** `outline:\s*none|outline:\s*0` without `:focus-visible` replacement.
**Fix:**
```css
:focus:not(:focus-visible) { outline: none; }
:focus-visible { outline: 2px solid var(--focus-color); outline-offset: 2px; }
```

### Check 9.3: Skip to Content

**Detect:** Missing `skip.*content` link.
**Fix:** Add visually hidden skip link targeting `#main-content`.

### Check 9.4: ARIA on Interactive Elements

**Detect:** `<div.*onClick|<span.*onClick` without `role`, `tabindex`, keyboard handler.

---

## Phase 10: RTL and Dark Mode

### Check 10.1: RTL Layout

**Detect:** If RTL content exists, check for CSS logical properties (`margin-inline`, `padding-inline`, `inset-inline`).
**FAIL if:** Using physical properties (`margin-left`, `padding-right`) for layout.

**Fix:**
```css
/* BAD */ margin-left: 1rem;    /* GOOD */ margin-inline-start: 1rem;
/* BAD */ text-align: left;     /* GOOD */ text-align: start;
/* BAD */ left: 0;              /* GOOD */ inset-inline-start: 0;
```

### Check 10.2: Dark Mode

**Detect:** `prefers-color-scheme` -- at least one occurrence expected.
**Fix:**
```css
@media (prefers-color-scheme: dark) {
  :root { --bg: #1a1a1a; --text: #e5e5e5; }
  img:not([src*=".svg"]) { filter: brightness(0.9); }
}
```
