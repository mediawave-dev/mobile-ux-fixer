---
name: mobile-ux-fixer
description: "Comprehensive mobile UX audit, visual inspection, and auto-fix skill for web projects. Detects and fixes 40+ mobile issues across 10 phases: visual inspection via Playwright MCP, viewport/layout stability, scroll/parallax effects, animations/motion, images/media, touch/forms/interaction, typography, Core Web Vitals, SEO/meta/PWA, accessibility, RTL, and dark mode. Use when user says 'fix mobile', 'mobile issues', 'looks broken on phone', 'scroll jump', 'address bar jump', 'parallax mobile', 'mobile audit', 'check mobile', 'mobile test', 'mobile performance', 'core web vitals mobile', 'תקן מובייל', 'בעיות במובייל', 'קופץ בגלילה', 'נראה רע בטלפון', 'בדיקת מובייל', or after deploying a site and testing on a real device."
---

# Mobile UX Fixer v2.0

> Systematic mobile audit, visual inspection, and auto-fix skill.
> 40+ checks across 10 phases. Every check comes from real production bugs.
> Now with Playwright MCP visual inspection for animations and layout verification.

---

## When to Use

- After deploying to production and testing on a real phone
- When user reports scroll jumps, layout shifts, or broken visuals on mobile
- Before launch as a pre-flight mobile checklist
- When parallax or animations look wrong on mobile devices
- When Core Web Vitals scores are poor on mobile
- When accessibility issues are reported on mobile
- As part of QA before any production deploy

---

## Phase 0: Visual Inspection with Playwright MCP

> **Use Playwright MCP to SEE the site on mobile before running code checks.**
> This gives Claude Code actual eyes on the mobile layout.

### Auto-Setup

**Before running visual checks, ensure Playwright MCP is installed.**

1. Check if Playwright MCP is already configured:
   ```
   Run: claude mcp list
   Look for: "playwright" in the output
   ```

2. If NOT found, install it automatically:
   ```bash
   claude mcp add playwright -- npx @playwright/mcp@latest --headless --device "iPhone 15" --caps vision
   ```

3. If the `claude` CLI is not available or the command fails, add it manually to the project's `.mcp.json` or `~/.claude.json`:
   ```json
   {
     "mcpServers": {
       "playwright": {
         "command": "npx",
         "args": [
           "@playwright/mcp@latest",
           "--headless",
           "--device", "iPhone 15",
           "--caps", "vision"
         ]
       }
     }
   }
   ```

4. After adding, verify by running `/mcp` and checking that the `playwright` server appears with tools like `browser_navigate`, `browser_take_screenshot`, etc.

**Note:** If Playwright MCP cannot be installed (no npm, restricted environment), skip Phase 0 and proceed directly to Phase 1 (code-level checks still work without it).

### Visual Audit Steps

1. **Navigate** to the site URL using `browser_navigate`
2. **Take screenshot** at initial viewport (mobile portrait)
3. **Scroll down** incrementally and take screenshots at each section:
   ```
   Use browser_evaluate to run: window.scrollBy(0, window.innerHeight)
   Then take screenshot. Repeat until reaching page bottom.
   ```
4. **Check for horizontal overflow** visually and programmatically:
   ```javascript
   // Run via browser_evaluate
   const overflows = [];
   document.querySelectorAll('*').forEach(el => {
     if (el.scrollWidth > document.documentElement.clientWidth) {
       overflows.push({ tag: el.tagName, class: el.className, width: el.scrollWidth });
     }
   });
   JSON.stringify(overflows);
   ```
5. **Resize to different viewports** and screenshot:
   - 320px (iPhone SE / small)
   - 375px (iPhone 12/13/14)
   - 390px (iPhone 15)
   - 412px (Pixel / Android)
6. **Test landscape** by resizing to 812x375

### What to Look For in Screenshots

- Elements overflowing the viewport
- Text too small to read
- Buttons/links too close together
- Images cropped awkwardly
- Navigation not accessible
- Animations that look broken (take before/after scroll screenshots)
- Layout shifts (compare sequential screenshots for element position changes)

---

## Phase 1: Viewport and Layout Stability

These issues cause the most jarring user experience -- layout shifts and jumps.

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
<!-- When modal is open, add inert to main content -->
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

**The Problem:** Elements wider than viewport cause horizontal scroll -- one of the most common mobile bugs.

**Detect (code analysis):**
```
Look for: fixed-width elements, tables without overflow wrapper, images without max-width: 100%
Pattern: width:\s*\d{4,}px|min-width:\s*\d{3,}px
```

**Detect (Playwright):**
```javascript
// Run via browser_evaluate
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
// Normalize scroll for mobile address bar
ScrollTrigger.normalizeScroll(true);

// Use matchMedia to disable pins on mobile
ScrollTrigger.matchMedia({
  "(max-width: 768px)": function() {
    // Mobile-specific ScrollTrigger config -- avoid pins
  },
  "(min-width: 769px)": function() {
    // Desktop -- pins are fine
  }
});

// Always disable lag smoothing when using Lenis
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
// Add layoutScroll to scrollable containers
<motion.div layoutScroll style={{ overflow: 'auto' }}>
  <motion.div layout>...</motion.div>
</motion.div>
```

### Check 2.5: Lenis/Smooth Scroll Mobile Configuration

**The Problem:** `syncTouch: true` causes performance issues on mobile. Wrong autoRaf config with GSAP.

**Detect:**
```
Pattern: new Lenis|import.*lenis|import.*locomotive
Files: *.js, *.ts, *.jsx, *.tsx
```

**Fix:**
```javascript
const lenis = new Lenis({
  syncTouch: false,      // Better mobile performance
  touchMultiplier: 1.5,  // Adjust touch sensitivity
  autoRaf: false,        // When using GSAP ticker
});

// Sync with GSAP
lenis.on('scroll', ScrollTrigger.update);
gsap.ticker.add((time) => lenis.raf(time * 1000));
gsap.ticker.lagSmoothing(0);
```

### Check 2.6: Scroll-Snap iOS Bugs

**The Problem:** iOS WebKit caches snap points. Fast flick scrolls to end instead of next item.

**Detect:**
```
Pattern: scroll-snap-type|scroll-snap-align
Files: *.css, *.scss, *.tsx, *.jsx
```

**Fix:**
```css
.scroll-container {
  scroll-snap-type: x mandatory;
}
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
Also: \.style\.(width|height|top|left|margin|padding)\s*=
```

**Fix:** Only animate `transform` and `opacity`:
```css
/* BAD */ transition: width 0.3s, left 0.3s;
/* GOOD */ transition: transform 0.3s, opacity 0.3s;
```

### Check 3.2: will-change Overuse

**The Problem:** Too many compositor layers exhaust mobile GPU memory.

**Detect:**
```
Pattern: will-change
Count occurrences. More than 5-10 is suspicious.
```

**Fix:** Only on actively animated elements. Add/remove dynamically.

### Check 3.3: Reduced Motion Preferences

**The Problem:** Animations can trigger vestibular disorders. WCAG requires respecting `prefers-reduced-motion`.

**Detect:**
```
Pattern: prefers-reduced-motion
Expected: At least ONE occurrence in CSS. If ZERO found and site has animations, this is a FAIL.
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
  gsap.globalTimeline.timeScale(20); // Skip animations instead of playing
}
```

### Check 3.4: iOS backdrop-filter + Transform Bug

**The Problem:** Combining `backdrop-filter` with `background-color` on same element causes white rendering on iOS.

**Detect:**
```
Pattern: backdrop-filter.*blur
Check: if same element has both backdrop-filter and background-color
```

**Fix:** Separate into pseudo-element:
```css
.glass-element {
  position: relative;
}
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

**The Problem:** Elements flash at their non-animated state before JS initializes animations.

**Detect:**
```
Pattern: gsap\.from|animate\(|useSpring|initial=\{
Check: if animated elements are visible before JS runs
```

**Fix (double-rAF technique):**
```javascript
document.documentElement.classList.add('js-loading');
requestAnimationFrame(() => {
  requestAnimationFrame(() => {
    document.documentElement.classList.remove('js-loading');
    // Start animations here
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
/* Progressive enhancement */
@supports (animation-timeline: scroll()) {
  .parallax-element {
    animation: parallax linear;
    animation-timeline: scroll();
  }
}

/* Fallback with IntersectionObserver */
@supports not (animation-timeline: scroll()) {
  /* JS-based fallback applied via class */
}
```

---

## Phase 4: Images and Media

### Check 4.1: Image Cropping on Mobile

**Detect:** Look for `object-cover` without explicit `object-position`.

**Fix:**
```css
object-position: center 30%; /* Focus on faces */
```

### Check 4.2: Image Loading Performance

**Detect:**
```
Check: hero/above-fold images should NOT have loading="lazy"
Check: hero images should have fetchpriority="high"
Check: below-fold images should have loading="lazy"
Pattern (bad): <img.*fetchpriority.*loading="lazy"|<img.*loading="lazy".*fetchpriority
```

**Fix:**
```html
<!-- Above fold (LCP candidate) -->
<img fetchpriority="high" src="hero.webp" width="1200" height="600" alt="..." />

<!-- Below fold -->
<img loading="lazy" src="content.webp" width="800" height="400" alt="..." />
```

### Check 4.3: Missing Image Dimensions (CLS)

**The Problem:** Images without `width` and `height` cause layout shift when they load.

**Detect:**
```
Pattern: <img(?![^>]*width)[^>]*>
Also check: CSS aspect-ratio as alternative
```

**Fix:**
```html
<img src="..." width="800" height="400" alt="..." />
```
Or use CSS:
```css
img { aspect-ratio: 16/9; width: 100%; height: auto; }
```

### Check 4.4: Modern Image Formats

**Detect:**
```
Pattern: src="[^"]*\.(jpg|jpeg|png)"
Check: Are WebP/AVIF versions available or <picture> element used?
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

**The Problem:** Same image crop doesn't work across 16:9 desktop and 9:16 mobile.

**Detect:**
```
Check: hero images without <picture> + <source media> for different viewports
```

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

**The Problem:** WCAG 2.5.8 (AA) requires 24x24px minimum. Best practice is 44x44px (Apple HIG).

**Detect:**
```
Search: buttons/links with padding < p-2 (8px), icon-only buttons without min-size
Pattern: (min-width|min-height):\s*(1[0-9]|[0-9])px.*(?:button|btn|link|a\b)
```

**Detect (Playwright):**
```javascript
// Run via browser_evaluate
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
/* For icon buttons, expand clickable area with padding or pseudo-element */
.icon-btn { position: relative; }
.icon-btn::after {
  content: ''; position: absolute;
  inset: -8px; /* Expand touch area */
}
```

### Check 5.2: iOS Font Size Zoom

**The Problem:** iOS auto-zooms page when input font-size < 16px. After unfocus, it doesn't zoom back.

**Detect:**
```
Pattern: text-xs|text-sm|font-size:\s*(1[0-5]|[0-9])px
On input/select/textarea elements
```

**Fix:**
```css
input, select, textarea {
  font-size: max(16px, 1rem);
}
```

In Tailwind: use `text-base md:text-sm` (16px on mobile, smaller on desktop).

**NEVER use** `user-scalable=no` or `maximum-scale=1` -- this violates WCAG accessibility.

### Check 5.3: Passive Event Listeners

**The Problem:** Non-passive `touchstart`/`touchmove`/`wheel` listeners block scrolling, causing jank.

**Detect:**
```
Pattern: addEventListener\s*\(\s*['"](?:touchstart|touchmove|wheel)['"]
Check: is { passive: true } present? If not, ISSUE.
Exception: listeners that call preventDefault() need passive: false explicitly.
```

**Fix:**
```javascript
// GOOD
element.addEventListener('touchstart', handler, { passive: true });

// If you need preventDefault, use touch-action CSS instead
element.style.touchAction = 'none'; // Prevents default touch behavior via CSS
```

### Check 5.4: Input Type and Inputmode Optimization

**The Problem:** Wrong input types show the wrong keyboard, hurting mobile form UX.

**Detect:**
```
Pattern: <input(?![^>]*inputmode)[^>]*type="text"[^>]*>
Check: text inputs that should be tel, email, url, or numeric
```

**Fix:**
```html
<input type="email" inputmode="email" enterkeyhint="next" />
<input type="tel" inputmode="tel" enterkeyhint="next" />
<input type="text" inputmode="numeric" pattern="[0-9]*" enterkeyhint="done" />
<input type="url" inputmode="url" enterkeyhint="go" />
<input type="search" inputmode="search" enterkeyhint="search" />
```

### Check 5.5: Virtual Keyboard Handling

**The Problem:** Virtual keyboard pushes content up. Fixed bottom elements get covered or jump.

**Detect:**
```
Pattern: position:\s*fixed.*bottom|fixed.*bottom
Check: any fixed bottom element without visualViewport handling
```

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

**The Problem:** Scroll chaining lets background scroll through modals. Pull-to-refresh interferes with custom gestures.

**Detect:**
```
Pattern: overscroll-behavior
Expected: at least one occurrence on modals/drawers or globally.
If site has modals/sheets and no overscroll-behavior, ISSUE.
```

**Fix:**
```css
/* Prevent scroll chaining on modals */
.modal-content { overscroll-behavior: contain; }

/* Disable pull-to-refresh globally */
html, body { overscroll-behavior-y: none; }
```

---

## Phase 6: Typography and Readability

### Check 6.1: Minimum Font Sizes

**Detect:**
```
Pattern: font-size:\s*(1[0-1]|[0-9])px|text-\[(?:1[0-1]|[0-9])px\]
Also check: any body text below 14px
```

**Fix:**
- Body text: minimum 16px
- UI elements (labels, captions): minimum 14px
- Never below 12px for any visible text

### Check 6.2: Fluid Typography with clamp()

**Detect:**
```
Check: Are heading font-sizes responsive? Fixed px values that don't scale?
Pattern: font-size:\s*\d{2,}px on headings without media query overrides
```

**Fix:**
```css
h1 { font-size: clamp(1.75rem, 4vw + 1rem, 3.5rem); }
h2 { font-size: clamp(1.5rem, 3vw + 0.75rem, 2.5rem); }
p  { font-size: clamp(1rem, 1vw + 0.75rem, 1.125rem); }
```

### Check 6.3: Line Height for Readability

**Detect:**
```
Pattern: line-height:\s*(0\.\d+|1(\.[0-3])?)\s*(;|})
Check: body text with line-height below 1.4
```

**Fix:**
```css
body { line-height: 1.5; }
h1, h2, h3 { line-height: 1.2; }
.long-form { line-height: 1.6; }
```

### Check 6.4: Line Length on Mobile

**Detect:**
```
Check: text containers without max-width constraint
On mobile viewport, lines should be 35-45 characters
```

**Fix:**
```css
.content { max-width: 65ch; padding-inline: 1rem; }
```

---

## Phase 7: Core Web Vitals (Mobile)

### Check 7.1: LCP (Largest Contentful Paint)

**Target:** < 2.5s on mobile. Only 62% of mobile pages pass.

**Detect:**
```
Check: LCP candidate (usually hero image or heading)
FAIL if: hero image has loading="lazy"
FAIL if: hero image missing fetchpriority="high"
FAIL if: render-blocking CSS/JS in <head> without async/defer
Pattern: <link.*rel="stylesheet"(?!.*media=) in <head> (blocks rendering)
Pattern: <script(?!.*defer|async)[^>]*src= in <head>
```

**Fix:**
```html
<!-- Hero image -->
<img fetchpriority="high" src="hero.webp" width="1200" height="600" alt="..." />

<!-- Defer non-critical CSS -->
<link rel="preload" href="non-critical.css" as="style" onload="this.onload=null;this.rel='stylesheet'" />

<!-- Defer scripts -->
<script defer src="app.js"></script>
```

### Check 7.2: CLS from Font Loading

**The Problem:** Font swap causes text reflow and layout shift.

**Detect:**
```
Pattern: font-display
Expected: every @font-face should have font-display
FAIL if: @font-face without font-display
Also: fonts.googleapis.com without &display= parameter
```

**Fix:**
```css
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2') format('woff2');
  font-display: swap; /* or optional for zero CLS */
}
```

Font preload (max 2 critical fonts):
```html
<link rel="preload" href="font.woff2" as="font" type="font/woff2" crossorigin />
```

### Check 7.3: INP/TBT from Long Tasks

**The Problem:** JavaScript tasks > 50ms block the main thread, causing input delay.

**Detect:**
```
Pattern: document\.querySelectorAll\(['"][^'"]*['"]\)\.forEach
Check: large synchronous loops, heavy computation in event handlers
Pattern: JSON\.parse\(.*large|\.map\(.*\.map\(  (nested iterations)
```

**Fix:**
```javascript
// Break up long tasks with scheduler.yield()
async function processItems(items) {
  for (const item of items) {
    processItem(item);
    if (navigator.scheduler?.yield) {
      await navigator.scheduler.yield();
    }
  }
}

// Or use requestIdleCallback
requestIdleCallback(() => { /* non-urgent work */ });
```

### Check 7.4: Third-Party Script Impact

**Detect:**
```
Pattern: <script.*src="(?:https?://(?!(?:localhost|your-domain)))
Common offenders:
- YouTube embeds (blocks 4.5s on mobile)
- Chat widgets (Intercom, Drift, Tawk.to)
- Analytics without async
- Social media embeds
```

**Fix:**
```html
<!-- YouTube: use facade pattern -->
<div class="youtube-facade" data-video-id="abc123">
  <img src="thumbnail.webp" alt="Video" />
  <button aria-label="Play video">Play</button>
</div>
<script>
  // Load iframe only on click
  document.querySelector('.youtube-facade').addEventListener('click', function() {
    this.innerHTML = '<iframe src="https://www.youtube-nocookie.com/embed/' + this.dataset.videoId + '?autoplay=1" allow="autoplay" allowfullscreen></iframe>';
  });
</script>

<!-- Chat widget: load on interaction -->
<script>
  ['scroll', 'mousemove', 'touchstart'].forEach(evt => {
    window.addEventListener(evt, () => loadChatWidget(), { once: true });
  });
</script>
```

### Check 7.5: Font Loading Strategy

**Detect:**
```
Check: @font-face declarations
FAIL if: using .ttf or .otf format (should be woff2)
FAIL if: loading more than 4 font files
FAIL if: missing crossorigin on font preloads
Pattern: url\([^)]*\.(ttf|otf|eot)\)
```

**Fix:**
- Use woff2 format exclusively
- Preload max 2 critical fonts
- Use variable fonts to replace multiple weight files
- Add `size-adjust` and `ascent-override` to match fallback metrics

---

## Phase 8: SEO, Meta Tags, and PWA

### Check 8.1: Viewport Meta Tag

**Detect:**
```
Pattern: <meta.*name="viewport"
FAIL if: missing entirely
FAIL if: contains user-scalable=no or maximum-scale=1 (WCAG violation)
EXPECTED: width=device-width, initial-scale=1
```

### Check 8.2: Theme Color and Mobile Meta Tags

**Detect:**
```
Pattern: <meta.*name="theme-color"
Pattern: <meta.*name="color-scheme"
FAIL if: missing theme-color (affects mobile browser chrome appearance)
```

**Fix:**
```html
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)" />
<meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)" />
<meta name="color-scheme" content="light dark" />
```

### Check 8.3: Open Graph for Mobile Sharing

**Detect:**
```
Pattern: <meta.*property="og:
FAIL if: missing og:title, og:description, og:image
Check: og:image should be 1200x630px for best mobile sharing
```

### Check 8.4: PWA Manifest

**Detect:**
```
Pattern: <link.*rel="manifest"
If manifest exists, check for:
- name and short_name
- icons (192x192 and 512x512, both "any" and "maskable")
- start_url
- display (standalone/fullscreen)
- theme_color and background_color
```

---

## Phase 9: Accessibility on Mobile

### Check 9.1: Hamburger Menu Accessibility

**Detect:**
```
Pattern: hamburger|menu-toggle|mobile-menu|nav-toggle
Check: aria-expanded attribute, aria-label, focus trap implementation
FAIL if: button without aria-label or aria-expanded
```

**Fix:**
```html
<button
  aria-label="Open navigation menu"
  aria-expanded="false"
  aria-controls="mobile-nav"
  onclick="toggleMenu()"
>
  <span class="sr-only">Menu</span>
</button>
<nav id="mobile-nav" role="navigation">...</nav>
```

When open, add `inert` to background content.

### Check 9.2: Focus Management

**Detect:**
```
Pattern: outline:\s*none|outline:\s*0
FAIL if: focus styles removed without replacement
Check: :focus-visible styles exist
```

**Fix:**
```css
/* Remove only mouse-click outline, keep keyboard */
:focus:not(:focus-visible) { outline: none; }
:focus-visible { outline: 2px solid var(--focus-color); outline-offset: 2px; }
```

### Check 9.3: Skip to Content Link

**Detect:**
```
Pattern: skip.*content|skip.*main|skip.*nav
FAIL if: not present on pages with navigation
```

**Fix:**
```html
<a href="#main-content" class="sr-only focus:not-sr-only focus:absolute focus:top-0 focus:left-0 focus:z-50 focus:p-4 focus:bg-white">
  Skip to main content
</a>
```

### Check 9.4: ARIA on Interactive Elements

**Detect:**
```
Check: custom interactive elements (divs with onClick) without role="button"
Pattern: <div.*onClick|<span.*onClick
FAIL if: no role, no tabindex, no keyboard handler
```

---

## Phase 10: RTL and Dark Mode

### Check 10.1: RTL Layout (Hebrew/Arabic)

**Detect:**
```
Pattern: dir="rtl"|direction:\s*rtl
If site has RTL content:
Check: CSS logical properties used (margin-inline, padding-inline, inset-inline)
FAIL if: using physical properties (margin-left, padding-right) for layout
```

**Fix:**
```css
/* BAD */ margin-left: 1rem;
/* GOOD */ margin-inline-start: 1rem;

/* BAD */ text-align: left;
/* GOOD */ text-align: start;

/* BAD */ left: 0; right: auto;
/* GOOD */ inset-inline-start: 0;
```

### Check 10.2: Dark Mode Support

**Detect:**
```
Pattern: prefers-color-scheme
Check: at least one occurrence expected on modern sites
```

**Fix:**
```css
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #1a1a1a;
    --text: #e5e5e5;
  }
}
```

Check for dark mode image issues:
```css
@media (prefers-color-scheme: dark) {
  img:not([src*=".svg"]) { filter: brightness(0.9); }
}
```

---

## Quick Audit Checklist

Run all checks at once. Report: OK / ISSUE FOUND / SKIPPED (not applicable).

```
Phase 0: Visual Inspection (Playwright MCP)
  0.1 Mobile screenshots (375px, 390px)     [ ]
  0.2 Scroll-through capture                [ ]
  0.3 Horizontal overflow check             [ ]
  0.4 Landscape test                        [ ]

Phase 1: Viewport and Layout
  1.1 Address bar scroll jump               [ ]
  1.2 dvh vs svh                            [ ]
  1.3 Safe areas                            [ ]
  1.4 Body scroll lock (modals)             [ ]
  1.5 Horizontal overflow                   [ ]

Phase 2: Parallax and Scroll
  2.1 background-attachment: fixed          [ ]
  2.2 Ken Burns consistency                 [ ]
  2.3 GSAP ScrollTrigger mobile             [ ]
  2.4 Framer Motion layout animations       [ ]
  2.5 Lenis/smooth scroll config            [ ]
  2.6 Scroll-snap iOS bugs                  [ ]

Phase 3: Animations and Motion
  3.1 Layout-triggering animations          [ ]
  3.2 will-change overuse                   [ ]
  3.3 Reduced motion (prefers-reduced-motion)[ ]
  3.4 iOS backdrop-filter bug               [ ]
  3.5 Page load animation flicker           [ ]
  3.6 Scroll-driven animations fallback     [ ]

Phase 4: Images and Media
  4.1 Image cropping / object-position      [ ]
  4.2 Image loading (lazy/priority)         [ ]
  4.3 Missing image dimensions (CLS)        [ ]
  4.4 Modern formats (WebP/AVIF)            [ ]
  4.5 Art direction (mobile crop)           [ ]

Phase 5: Touch, Forms, Interaction
  5.1 Touch target size (44x44px)           [ ]
  5.2 iOS font zoom (16px min)              [ ]
  5.3 Passive event listeners               [ ]
  5.4 Input type / inputmode                [ ]
  5.5 Virtual keyboard handling             [ ]
  5.6 Overscroll behavior                   [ ]

Phase 6: Typography
  6.1 Minimum font sizes                    [ ]
  6.2 Fluid typography (clamp)              [ ]
  6.3 Line height                           [ ]
  6.4 Line length                           [ ]

Phase 7: Core Web Vitals
  7.1 LCP (hero image, render-blocking)     [ ]
  7.2 CLS from fonts                        [ ]
  7.3 INP/TBT long tasks                    [ ]
  7.4 Third-party scripts                   [ ]
  7.5 Font loading strategy                 [ ]

Phase 8: SEO, Meta, PWA
  8.1 Viewport meta tag                     [ ]
  8.2 Theme-color meta                      [ ]
  8.3 Open Graph tags                       [ ]
  8.4 PWA manifest                          [ ]

Phase 9: Accessibility
  9.1 Hamburger menu a11y                   [ ]
  9.2 Focus management                      [ ]
  9.3 Skip to content                       [ ]
  9.4 ARIA on custom interactive elements   [ ]

Phase 10: RTL and Dark Mode
  10.1 RTL logical properties               [ ]
  10.2 Dark mode support                    [ ]
```

---

## Troubleshooting

### "It works on Chrome DevTools mobile emulator but not on real phone"
DevTools does NOT simulate: address bar show/hide, iOS Safari quirks, real touch timing, GPU memory limits. Use Playwright MCP with `--device` flag or test on real devices.

### "Parallax looks smooth on my phone but jumps on others"
Use `requestAnimationFrame` throttling and keep speed values small (under 0.1). Consider disabling parallax on low-end devices.

### "Everything shifts when the keyboard opens"
Use `visualViewport` API. For fixed bottom elements:
```javascript
window.visualViewport?.addEventListener('resize', () => {
  const offset = window.innerHeight - window.visualViewport.height;
  bottomEl.style.transform = `translateY(-${offset}px)`;
});
```

### "GSAP ScrollTrigger pins jump on mobile"
Add `ScrollTrigger.normalizeScroll(true)` or disable pins on mobile with `matchMedia`.

### "Lenis smooth scroll feels laggy on phone"
Set `syncTouch: false` and adjust `touchMultiplier`. Use `autoRaf: false` with GSAP ticker.

### "Animations flash on page load"
Use the double-rAF technique or hide animated elements with CSS until JS is ready.

### "Site looks zoomed in after filling a form on iPhone"
Ensure all input font-sizes are >= 16px. Use `text-base md:text-sm` in Tailwind.

---

## Reference Files

Detailed research backing each check is available in the `references/` directory:
- `mobile-ux-research-2024-2026.md` -- CSS, JS, iOS/Android, forms, typography, images, accessibility
- `RESEARCH-playwright-mobile-testing.md` -- Playwright MCP setup, visual testing, performance
- `RESEARCH-mobile-animation-best-practices.md` -- GSAP, Framer Motion, Lenis, animation bugs
- `mobile-web-standards-research-2024-2026.md` -- SEO, PWA, fonts, dark mode, RTL, security
