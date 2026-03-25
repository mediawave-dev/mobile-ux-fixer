# Comprehensive Mobile UX Issues, Detection Patterns, and Code Fixes (2024-2026)

> Research compiled from current web sources, browser bug trackers, MDN documentation,
> and developer community reports. Every issue includes programmatic detection patterns
> and specific code-level fixes suitable for automated CLI agent application.

---

## Table of Contents

1. [CSS Issues Specific to Mobile](#1-css-issues-specific-to-mobile)
2. [Mobile-Specific JavaScript Gotchas](#2-mobile-specific-javascript-gotchas)
3. [Core Web Vitals Mobile-Specific Problems](#3-core-web-vitals-mobile-specific-problems)
4. [Mobile Navigation Patterns and Common Failures](#4-mobile-navigation-patterns-and-common-failures)
5. [iOS Safari Specific Bugs (2024-2026)](#5-ios-safari-specific-bugs-2024-2026)
6. [Android Chrome Specific Issues](#6-android-chrome-specific-issues)
7. [Mobile Form UX](#7-mobile-form-ux)
8. [Mobile Typography Best Practices](#8-mobile-typography-best-practices)
9. [Mobile Responsive Image Strategies](#9-mobile-responsive-image-strategies)
10. [Accessibility on Mobile](#10-accessibility-on-mobile)

---

## 1. CSS Issues Specific to Mobile

### 1.1 Scroll-Snap Bugs on iOS Safari

**Problem:** On iOS Safari, CSS scroll-snap has multiple known bugs. A fast flick gesture scrolls all the way to the end of the list instead of snapping to the next item. WebKit caches snap point positions during initial layout and fails to recalculate when child elements are modified programmatically, causing misaligned snaps, jumpy scrolling, and failed snapping.

**Affected:** iOS Safari (all versions), all iOS browsers (they all use WebKit)

**Detection patterns (grep):**
```
# Find scroll-snap usage
Pattern: scroll-snap-type|scroll-snap-align|scroll-snap-stop
Files: *.css, *.scss, *.less, *.tsx, *.jsx, *.vue, *.svelte

# Find programmatic style changes on scroll-snap children
Pattern: style\.(transform|width|height|opacity)\s*=.*(?:snap|scroll|carousel|slider)
```

**Code fix -- Force layout recalculation after modifying snap children:**
```javascript
// BEFORE: Causes jittery snapping on iOS
child.style.transform = `translateX(${offset}px)`;

// AFTER: Force WebKit to recalculate snap points
child.style.transform = `translateX(${offset}px)`;
void child.offsetHeight; // Force synchronous reflow
```

**CSS fix -- Use scroll-snap-stop for reliable snapping:**
```css
.scroll-container {
  scroll-snap-type: x mandatory;
  -webkit-overflow-scrolling: touch;
}
.scroll-item {
  scroll-snap-align: start;
  scroll-snap-stop: always; /* Prevents skip-scrolling on iOS */
}
```

### 1.2 Container Queries on Mobile

**Problem:** CSS container size queries have ~96-97% global browser support as of 2025, but container *style* queries and *scroll-state* queries have very limited support. Only 7% of developers report using style queries. The main mobile pitfall is "container collapse" -- when a container has no intrinsic size, it collapses to zero, breaking child layouts. Also requires explicit `container-type` declaration.

**Affected:** Older iOS Safari (<16), older Android Chrome (<105), all browsers for style queries

**Detection patterns:**
```
# Find container query usage without container-type
Pattern: @container\s
Files: *.css, *.scss

# Then verify parent has container-type set
Pattern: container-type:\s*(inline-size|size|normal)

# Find container style queries (limited support)
Pattern: @container\s+style\(
```

**Code fix -- Always declare container-type and provide fallback:**
```css
/* Required: declare the container */
.card-wrapper {
  container-type: inline-size;
  container-name: card;
}

/* Use container query */
@container card (min-width: 400px) {
  .card { flex-direction: row; }
}

/* Provide fallback for unsupported browsers */
@supports not (container-type: inline-size) {
  @media (min-width: 400px) {
    .card { flex-direction: row; }
  }
}
```

### 1.3 :has() Selector Mobile Support

**Problem:** The `:has()` selector is now supported in all major browsers (Chrome 105+, Firefox 121+, Safari 15.4+). The main issue is performance: complex `:has()` selectors can be expensive on mobile, especially with deeply nested DOM trees. Also, some older Android WebView versions lack support.

**Affected:** Android WebView (< 105), Samsung Internet (< 20), older browsers

**Detection patterns:**
```
# Find :has() usage
Pattern: :has\(
Files: *.css, *.scss

# Find potentially expensive :has() with deep nesting
Pattern: :has\([^)]*\s[^)]*\s[^)]*\)
```

**Code fix -- Provide fallback and limit complexity:**
```css
/* Use @supports for graceful degradation */
@supports selector(:has(*)) {
  .parent:has(.child.active) {
    background: blue;
  }
}

/* Fallback for unsupported browsers */
@supports not selector(:has(*)) {
  .parent.has-active-child {
    background: blue;
  }
}
```

**JavaScript fallback:**
```javascript
// Add class-based fallback for browsers without :has()
if (!CSS.supports('selector(:has(*))')) {
  document.querySelectorAll('.child.active').forEach(child => {
    child.closest('.parent')?.classList.add('has-active-child');
  });
}
```

### 1.4 CSS Subgrid Mobile Issues

**Problem:** CSS Subgrid reached universal browser support in 2024 (~97% global coverage). iOS Safari 16+ and Chrome Android 117+ fully support it. The remaining ~3% of users on older browsers need fallbacks. The main code-level issue is forgetting to set `display: grid` on the subgrid child while also specifying `grid-template-rows: subgrid` or `grid-template-columns: subgrid`.

**Affected:** iOS Safari <16, Chrome Android <117, Samsung Internet <22

**Detection patterns:**
```
# Find subgrid usage
Pattern: grid-template-(rows|columns):\s*subgrid
Files: *.css, *.scss

# Verify @supports fallback exists nearby
Pattern: @supports\s+not.*subgrid
```

**Code fix:**
```css
.grid-parent {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto auto auto;
}

.grid-child {
  display: grid;
  grid-template-rows: subgrid;
  grid-row: span 3;
}

/* Fallback for older browsers */
@supports not (grid-template-rows: subgrid) {
  .grid-child {
    display: flex;
    flex-direction: column;
  }
}
```

### 1.5 100vh / Viewport Height on Mobile

**Problem:** `100vh` on mobile does not account for the browser's address bar, causing content to extend behind it. `100dvh` changes dynamically as the address bar hides/shows, causing layout shifts. `100svh` (small viewport height) is stable but leaves a gap when the address bar is hidden.

**Affected:** All mobile browsers

**Detection patterns:**
```
# Find raw vh usage that should be svh/dvh
Pattern: height:\s*100vh|min-height:\s*100vh|h-screen
Files: *.css, *.scss, *.tsx, *.jsx, *.vue, *.html

# Find dvh on elements that should be stable
Pattern: height:\s*100dvh|min-height:\s*100dvh|h-\[100dvh\]
```

**Code fix:**
```css
/* For hero sections and full-height layouts (stable, no jump) */
.hero {
  height: 100svh;
  /* Fallback for browsers without svh support */
  height: 100vh;
  height: 100svh;
}

/* For modals/overlays that should fill current visible area */
.modal-overlay {
  height: 100dvh;
  height: 100vh; /* fallback */
  height: 100dvh;
}

/* The Tailwind equivalent */
/* Use h-[100svh] instead of h-screen for stable layouts */
```

---

## 2. Mobile-Specific JavaScript Gotchas

### 2.1 Passive Event Listeners

**Problem:** Touch and scroll event listeners block the browser's compositor thread from scrolling until the handler completes. This causes scroll jank on mobile. Chrome and Firefox now default `touchstart` and `touchmove` to passive on document-level listeners, but explicit `addEventListener` calls on elements still default to `{passive: false}`.

**Affected:** All mobile browsers (most impactful on Android Chrome)

**Detection patterns:**
```
# Find non-passive touch/wheel listeners
Pattern: addEventListener\s*\(\s*['"`](touchstart|touchmove|touchend|wheel|mousewheel)['"`]\s*,\s*\w+\s*\)
Files: *.js, *.ts, *.jsx, *.tsx

# Find listeners that call preventDefault on touch events
Pattern: (touchstart|touchmove|wheel).*preventDefault
```

**Code fix:**
```javascript
// BEFORE: Blocks scrolling
element.addEventListener('touchstart', handler);
element.addEventListener('wheel', handler);

// AFTER: Non-blocking, smooth scrolling
element.addEventListener('touchstart', handler, { passive: true });
element.addEventListener('wheel', handler, { passive: true });

// If you NEED preventDefault (e.g., custom scroll container):
element.addEventListener('touchmove', handler, { passive: false });
// But document this explicitly -- it's intentional scroll blocking
```

**Detection in frameworks:**
```
# React: find onTouchStart/onTouchMove without passive option
# React doesn't support passive in JSX props -- must use ref + addEventListener
Pattern: onTouchStart=|onTouchMove=|onWheel=
# These React handlers are NOT passive by default
```

### 2.2 Touch vs Click Delay

**Problem:** Historically, mobile browsers added a 300ms delay after `touchend` before firing `click` to detect double-tap zoom. Modern browsers have eliminated this for pages with `<meta name="viewport" content="width=device-width">`, but some edge cases remain. The real modern issue is ghost clicks (both touch and click firing) and pointer event handling.

**Affected:** All mobile browsers (mostly resolved, but edge cases remain)

**Detection patterns:**
```
# Find dual touch+click handlers (potential ghost click)
Pattern: (addEventListener.*click.*addEventListener.*touch|addEventListener.*touch.*addEventListener.*click|onClick.*onTouch|onTouch.*onClick)

# Find manual fastclick or touch delay workarounds (obsolete)
Pattern: fastclick|fast-click|300ms.*delay|touch.*delay
```

**Code fix:**
```javascript
// MODERN APPROACH: Use pointer events (unified touch + mouse + pen)
element.addEventListener('pointerdown', handleStart);
element.addEventListener('pointerup', handleEnd);
element.addEventListener('pointermove', handleMove);

// If supporting touch + mouse separately, prevent ghost clicks:
element.addEventListener('touchend', (e) => {
  e.preventDefault(); // Prevents subsequent click event
  handleInteraction(e);
});

// Ensure viewport meta is set (eliminates 300ms delay):
// <meta name="viewport" content="width=device-width, initial-scale=1">
```

### 2.3 Overscroll Behavior

**Problem:** Mobile browsers have default overscroll behaviors: pull-to-refresh (Android Chrome), rubber-banding (iOS Safari), and navigation swipe gestures. These can interfere with custom scroll containers, carousels, and swipe-based UIs.

**Affected:** Android Chrome (pull-to-refresh, overscroll glow), iOS Safari (rubber-banding)

**Detection patterns:**
```
# Find custom scroll containers without overscroll-behavior
Pattern: overflow(-y|-x)?:\s*(auto|scroll)(?!.*overscroll)
Files: *.css, *.scss

# Find elements that might need overscroll containment
Pattern: (carousel|slider|drawer|sidebar|modal|sheet|swipe).*overflow
```

**Code fix:**
```css
/* Prevent scroll chaining (child scroll reaching end triggers parent scroll) */
.custom-scroll-container {
  overflow-y: auto;
  overscroll-behavior-y: contain;
}

/* Disable pull-to-refresh on the entire page */
html {
  overscroll-behavior-y: none;
}

/* Disable horizontal swipe navigation */
html {
  overscroll-behavior-x: none;
}

/* For modal/drawer overlays -- prevent background scrolling */
.modal-content {
  overflow-y: auto;
  overscroll-behavior: contain;
}
```

### 2.4 Momentum Scrolling in Custom Containers

**Problem:** On iOS, custom scroll containers (div with `overflow: auto`) historically lacked momentum scrolling (inertial flick). The `-webkit-overflow-scrolling: touch` property fixed this but is now deprecated. Modern iOS Safari applies momentum scrolling by default, but some edge cases with nested scroll containers still fail.

**Affected:** Older iOS Safari (< 13 needed the property), modern iOS for edge cases

**Detection patterns:**
```
# Find deprecated property usage
Pattern: -webkit-overflow-scrolling
Files: *.css, *.scss

# Find nested scroll containers (potential momentum issues)
Pattern: overflow.*scroll.*overflow.*scroll|overflow.*auto.*overflow.*auto
```

**Code fix:**
```css
/* Remove deprecated property -- momentum scrolling is now default */
.scroll-container {
  overflow-y: auto;
  /* REMOVE: -webkit-overflow-scrolling: touch; (deprecated, no longer needed) */
}

/* For nested scroll containers, use overscroll-behavior */
.inner-scroll {
  overflow-y: auto;
  overscroll-behavior: contain; /* Prevents scroll chaining to outer container */
}
```

---

## 3. Core Web Vitals Mobile-Specific Problems

### 3.1 LCP (Largest Contentful Paint) on Mobile

**Problem:** Only 62% of mobile pages achieve a good LCP score (under 2.5s). The main mobile-specific causes: hero images without `fetchpriority="high"`, lazy-loading applied to above-the-fold images, render-blocking CSS/JS, unoptimized image formats, and slow server response on cellular networks.

**Affected:** All mobile browsers (impacts SEO ranking since March 2024)

**Detection patterns:**
```
# Find hero/LCP images without fetchpriority
Pattern: <img[^>]*(?:hero|banner|splash|cover|main-image|featured)[^>]*>
# Then check if fetchpriority="high" is present

# Find above-fold images with lazy loading (anti-pattern)
Pattern: <img[^>]*(loading=["']lazy["'])[^>]*(hero|banner|above-fold|main|first)
# Or vice versa
Pattern: <img[^>]*(hero|banner|above-fold|main|first)[^>]*(loading=["']lazy["'])

# Find render-blocking resources
Pattern: <link[^>]*rel=["']stylesheet["'][^>]*>(?!.*media=)
# In <head> without media attribute or async loading
```

**Code fix:**
```html
<!-- BEFORE: Hero image loads late -->
<img src="hero.jpg" loading="lazy" alt="Hero">

<!-- AFTER: Hero image loads with highest priority -->
<img src="hero.webp" fetchpriority="high" alt="Hero"
     width="1200" height="600"
     decoding="async">

<!-- Preload LCP image in <head> -->
<link rel="preload" as="image" href="hero.webp"
      fetchpriority="high"
      type="image/webp">

<!-- Defer non-critical CSS -->
<link rel="preload" href="non-critical.css" as="style"
      onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="non-critical.css"></noscript>
```

### 3.2 CLS Caused by Late-Loading Fonts

**Problem:** Web fonts loading after initial paint cause text to reflow when the custom font replaces the fallback font. This is especially noticeable on slower mobile connections. The font swap changes character widths, line heights, and line breaks, causing layout shift.

**Affected:** All mobile browsers

**Detection patterns:**
```
# Find @font-face without font-display
Pattern: @font-face\s*\{(?![\s\S]*?font-display)[\s\S]*?\}
Files: *.css, *.scss

# Find @font-face with font-display: swap (can cause CLS)
Pattern: font-display:\s*swap

# Find Google Fonts without display parameter
Pattern: fonts\.googleapis\.com(?!.*display=)

# Find missing font preload
Pattern: <link[^>]*preload[^>]*font
# Check if this exists -- if not, fonts may load late
```

**Code fix:**
```css
/* Use font-display: optional for zero CLS (font only used if already cached) */
@font-face {
  font-family: 'CustomFont';
  src: url('custom-font.woff2') format('woff2');
  font-display: optional; /* Zero CLS -- uses system font if custom not cached */
  /* Alternative: font-display: swap; -- shows text immediately, causes CLS on swap */
}

/* Reduce CLS with font-display: swap by matching fallback metrics */
@font-face {
  font-family: 'CustomFont';
  src: url('custom-font.woff2') format('woff2');
  font-display: swap;
  size-adjust: 105%;        /* Match fallback font width */
  ascent-override: 90%;     /* Match fallback ascent */
  descent-override: 20%;    /* Match fallback descent */
  line-gap-override: 0%;    /* Match fallback line gap */
}
```

```html
<!-- Preload critical fonts -->
<link rel="preload" href="/fonts/custom-font.woff2"
      as="font" type="font/woff2" crossorigin>

<!-- Google Fonts with display=optional -->
<link href="https://fonts.googleapis.com/css2?family=Inter&display=optional"
      rel="stylesheet">
```

### 3.3 TBT/INP from Heavy JavaScript

**Problem:** INP (Interaction to Next Paint) replaced FID in March 2024. Mobile devices have significantly weaker CPUs than desktop, so long JavaScript tasks that are imperceptible on desktop cause noticeable jank on mobile. Any task >50ms blocks the main thread and degrades INP. Common culprits: analytics scripts, large bundle parsing, complex React re-renders, and third-party widgets.

**Affected:** All mobile browsers (3-5x slower JS execution than desktop)

**Detection patterns:**
```
# Find synchronous <script> tags in <head> (render-blocking)
Pattern: <head>[\s\S]*?<script\s+src=(?!.*async|.*defer|.*type=["']module["'])
Files: *.html

# Find large event handlers
Pattern: (onClick|onChange|onInput|onScroll|addEventListener)\s*[=(]\s*(?:async\s*)?\(?[^)]*\)?\s*(?:=>|{)\s*[\s\S]{500,}

# Find expensive operations in render
Pattern: (\.map\(|\.filter\(|\.reduce\(|\.sort\().*\.map\(
# Chained array operations in render = potential main thread blocking

# Find missing dynamic imports
Pattern: import\s+.*from\s+['"](?:lodash|moment|date-fns|chart|three|pdf|xlsx)
# Large libraries imported statically instead of dynamically
```

**Code fix:**
```html
<!-- BEFORE: Render-blocking script -->
<script src="analytics.js"></script>

<!-- AFTER: Non-blocking -->
<script src="analytics.js" defer></script>
<!-- Or for truly non-critical: -->
<script src="analytics.js" async></script>
```

```javascript
// BEFORE: Large import loaded on page load
import { Chart } from 'chart.js';

// AFTER: Dynamic import -- only loaded when needed
const loadChart = async () => {
  const { Chart } = await import('chart.js');
  return Chart;
};

// Break long tasks with scheduler.yield() (Chrome 129+)
async function processItems(items) {
  for (const item of items) {
    processItem(item);
    // Yield to the main thread periodically
    if (navigator.scheduling?.isInputPending?.()) {
      await scheduler.yield();
    }
  }
}

// Fallback for browsers without scheduler.yield:
function yieldToMain() {
  return new Promise(resolve => setTimeout(resolve, 0));
}
```

### 3.4 CLS from Images Without Dimensions

**Problem:** Images without explicit `width` and `height` attributes (or CSS `aspect-ratio`) cause layout shifts when they load, as the browser cannot reserve space. This is the #1 most common CLS issue.

**Affected:** All browsers

**Detection patterns:**
```
# Find img tags without width/height attributes
Pattern: <img(?![^>]*width=)(?![^>]*height=)[^>]*>
Files: *.html, *.jsx, *.tsx, *.vue, *.svelte

# Find img tags without aspect-ratio in accompanying CSS
Pattern: <img(?![^>]*width)[^>]*class=["']([^"']+)["']
# Then check if that class has aspect-ratio defined

# Find dynamic images (CMS/API) that might lack dimensions
Pattern: <img[^>]*src=\{|<img[^>]*:src=|<img[^>]*\[src\]=
```

**Code fix:**
```html
<!-- BEFORE: No dimensions -- causes CLS -->
<img src="photo.jpg" alt="Photo">

<!-- AFTER: Explicit dimensions -- browser reserves space -->
<img src="photo.jpg" alt="Photo" width="800" height="600">

<!-- For responsive images, CSS handles the sizing -->
<style>
img {
  max-width: 100%;
  height: auto;
  /* Modern: explicit aspect-ratio as fallback */
  aspect-ratio: attr(width) / attr(height);
}
</style>
```

```css
/* For containers with dynamic images */
.image-container {
  aspect-ratio: 16 / 9;
  overflow: hidden;
}
.image-container img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

---

## 4. Mobile Navigation Patterns and Common Failures

### 4.1 Hamburger Menu Accessibility

**Problem:** Hamburger menus frequently fail accessibility: missing aria labels, no focus trapping, no keyboard navigation, toggle button not announced as "Menu" by screen readers, and background content remaining interactive when menu is open.

**Affected:** All mobile browsers + screen readers (VoiceOver, TalkBack)

**Detection patterns:**
```
# Find hamburger buttons without accessible label
Pattern: (hamburger|menu-toggle|nav-toggle|menu-btn|burger)(?![\s\S]{0,200}aria-label)
Files: *.html, *.jsx, *.tsx, *.vue, *.svelte

# Find menu toggle without aria-expanded
Pattern: (hamburger|menu-toggle|burger)(?![\s\S]{0,200}aria-expanded)

# Find nav without role or aria attributes
Pattern: <nav(?![^>]*aria-label)[^>]*>

# Find menu without focus trap
Pattern: (mobile-menu|mobile-nav|drawer|sidebar)(?![\s\S]{0,500}(focus-trap|focusTrap|trap.*focus|inert))
```

**Code fix:**
```html
<!-- Accessible hamburger menu -->
<button
  aria-label="Menu"
  aria-expanded="false"
  aria-controls="mobile-nav"
  class="menu-toggle"
>
  <span class="sr-only">Menu</span>
  <!-- Hamburger icon SVG -->
</button>

<nav id="mobile-nav"
     role="navigation"
     aria-label="Main navigation"
     aria-hidden="true"
     class="mobile-nav">
  <!-- Nav items with tabindex management -->
</nav>
```

```javascript
// Focus trap for open menu
function openMenu() {
  menu.setAttribute('aria-hidden', 'false');
  toggle.setAttribute('aria-expanded', 'true');

  // Make background content inert
  document.querySelector('main').setAttribute('inert', '');

  // Focus first menu item
  menu.querySelector('a, button')?.focus();
}

function closeMenu() {
  menu.setAttribute('aria-hidden', 'true');
  toggle.setAttribute('aria-expanded', 'false');
  document.querySelector('main').removeAttribute('inert');
  toggle.focus(); // Return focus to trigger
}
```

### 4.2 Bottom Sheet UX Failures

**Problem:** Bottom sheets on mobile suffer from swipe ambiguity: a vertical swipe starting on a bottom sheet can close the sheet, scroll its content, open the notification drawer, or display the control panel. Without a visible close button, the grab handle is the only dismissal mechanism, which is inaccessible to screen readers and keyboard users.

**Affected:** All mobile browsers, especially on Android (notification drawer interference)

**Detection patterns:**
```
# Find bottom sheet implementations
Pattern: (bottom-sheet|bottomSheet|drawer|action-sheet|sheet-modal)
Files: *.jsx, *.tsx, *.vue, *.svelte, *.css

# Check for missing close button
Pattern: (bottom-sheet|bottomSheet)(?![\s\S]{0,500}(close|dismiss|Cancel))

# Check for missing overscroll containment
Pattern: (bottom-sheet|bottomSheet|drawer)(?![\s\S]{0,500}overscroll-behavior)
```

**Code fix:**
```html
<!-- Bottom sheet with proper accessibility -->
<div class="bottom-sheet"
     role="dialog"
     aria-modal="true"
     aria-label="Options">

  <!-- Visible close button (not just grab handle) -->
  <div class="sheet-header">
    <div class="grab-handle" aria-hidden="true"></div>
    <button class="close-btn" aria-label="Close">
      <span aria-hidden="true">&times;</span>
    </button>
  </div>

  <div class="sheet-content">
    <!-- Content here -->
  </div>
</div>
```

```css
.bottom-sheet .sheet-content {
  overflow-y: auto;
  overscroll-behavior: contain; /* Prevent scroll chaining */
  touch-action: pan-y; /* Only allow vertical scroll */
  -webkit-overflow-scrolling: touch;
}
```

### 4.3 Pull-to-Refresh Interference with Custom UI

**Problem:** Chrome's built-in pull-to-refresh can trigger when the user tries to scroll down within a custom scroll container that's already at the top. This refreshes the page and loses user state/data.

**Affected:** Android Chrome primarily

**Detection patterns:**
```
# Find pages with custom scroll containers but no overscroll prevention
Pattern: overflow(-y)?:\s*(auto|scroll)
# Then check if html/body has overscroll-behavior set

# Find custom pull-to-refresh implementations
Pattern: (pull-to-refresh|pullToRefresh|pull.*refresh)
```

**Code fix:**
```css
/* Disable browser's pull-to-refresh */
html, body {
  overscroll-behavior-y: none;
}

/* If you want to keep pull-to-refresh on body but prevent it on specific containers */
.no-pull-refresh {
  overscroll-behavior-y: contain;
}
```

---

## 5. iOS Safari Specific Bugs (2024-2026)

### 5.1 iOS 26 (2025) Viewport and Fixed Positioning Bug

**Problem:** iOS 26 introduced "Liquid Glass" design with new tab modes for Safari, causing major layout issues. Fixed position elements with `inset: 0` fail to cover the screen if they have a solid background. Modals and overlays do not render behind the area near the address bar, even when `viewport-fit=cover` is set. The bottom safe area is not covered by `position: fixed` elements without explicit safe-area handling.

**Affected:** iOS 26+ Safari, iOS 26+ Chrome (WebKit)

**Detection patterns:**
```
# Find full-screen overlays that may be affected
Pattern: (position:\s*fixed|position:\s*sticky)[^}]*(inset:\s*0|top:\s*0[^}]*bottom:\s*0)
Files: *.css, *.scss

# Find modals without viewport-fit=cover
Pattern: <meta[^>]*viewport[^>]*>
# Check if viewport-fit=cover is present

# Find fixed elements without safe-area-inset
Pattern: position:\s*fixed(?![\s\S]{0,300}safe-area-inset)
```

**Code fix:**
```html
<!-- Ensure viewport-fit=cover is set -->
<meta name="viewport"
      content="width=device-width, initial-scale=1, viewport-fit=cover">
```

```css
/* Full-screen overlay that works on iOS 26 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  /* Extend into safe areas */
  top: calc(0px - env(safe-area-inset-top));
  bottom: calc(0px - env(safe-area-inset-bottom));
  left: calc(0px - env(safe-area-inset-left));
  right: calc(0px - env(safe-area-inset-right));
  /* Use -webkit-fill-available as additional fallback */
  min-height: -webkit-fill-available;
}

/* Fixed bottom elements */
.fixed-bottom {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  padding-bottom: env(safe-area-inset-bottom, 0px);
}

/* Fixed top elements */
.fixed-top {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  padding-top: env(safe-area-inset-top, 0px);
}
```

### 5.2 CSS backdrop-filter Performance on iOS

**Problem:** `backdrop-filter: blur()` combined with `background-color` causes rendering issues on iOS Safari. The element may display as fully white despite having a colored/transparent background. Heavy `backdrop-filter` causes severe performance degradation on older iOS devices, dropping frame rates below 30fps during scrolling.

**Affected:** iOS Safari (all recent versions), especially older devices (iPhone 8, SE)

**Detection patterns:**
```
# Find backdrop-filter usage
Pattern: backdrop-filter:\s*blur|(-webkit-)?backdrop-filter
Files: *.css, *.scss

# Find backdrop-filter combined with background-color
Pattern: backdrop-filter[\s\S]{0,100}background-color|background-color[\s\S]{0,100}backdrop-filter
```

**Code fix:**
```css
/* BEFORE: Can cause white rendering on iOS */
.glass-effect {
  background-color: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
}

/* AFTER: Separate layers for iOS compatibility */
.glass-effect {
  position: relative;
  isolation: isolate; /* Create new stacking context */
}
.glass-effect::before {
  content: '';
  position: absolute;
  inset: 0;
  background: rgba(255, 255, 255, 0.8);
  -webkit-backdrop-filter: blur(10px);
  backdrop-filter: blur(10px);
  z-index: -1;
}

/* For performance on older devices, reduce blur radius */
@media (max-width: 768px) {
  .glass-effect::before {
    -webkit-backdrop-filter: blur(5px); /* Reduced from 10px */
    backdrop-filter: blur(5px);
  }
}

/* Nuclear option: disable on low-end devices via media query */
@media (prefers-reduced-transparency: reduce) {
  .glass-effect::before {
    backdrop-filter: none;
    -webkit-backdrop-filter: none;
    background: rgba(255, 255, 255, 0.95);
  }
}
```

### 5.3 Position Fixed Inside Transformed Parents

**Problem:** On iOS Safari (and all browsers per spec), when any ancestor has a `transform`, `filter`, or `perspective` CSS property, `position: fixed` elements are positioned relative to that ancestor instead of the viewport. This is per-spec but catches many developers off-guard on mobile where modals/drawers are often inside transformed containers.

**Affected:** All browsers (spec behavior), but most commonly encountered on iOS Safari in practice

**Detection patterns:**
```
# Find fixed elements that might be inside transformed parents
Pattern: position:\s*fixed
Files: *.css, *.scss

# Find transforms on potential parent containers
Pattern: transform:|filter:|perspective:
# Cross-reference: if a fixed element is a descendant of a transformed element

# In React/JSX, find fixed elements inside motion/animated wrappers
Pattern: (motion\.|Animated\.|framer-motion|react-spring)[\s\S]{0,500}(fixed|position.*fixed)
```

**Code fix:**
```javascript
// Move fixed elements (modals, toasts) to portal at document root
// React example:
import { createPortal } from 'react-dom';

function Modal({ children }) {
  return createPortal(
    <div className="modal-overlay">{children}</div>,
    document.body // Renders outside any transformed parent
  );
}
```

```css
/* If portal is not possible, remove transform from ancestors when modal is open */
.parent-with-transform {
  transform: translateZ(0); /* GPU acceleration */
}
.parent-with-transform.modal-open {
  transform: none; /* Allow fixed children to work */
}
```

### 5.4 Body Scroll Lock on iOS Safari

**Problem:** iOS Safari does not respect `overflow: hidden` on the `<body>` element when a modal is open. The body continues to scroll behind the modal. This is a decade-old bug. The `window.innerHeight` value is also unreliable when the address bar state changes.

**Affected:** iOS Safari (all versions)

**Detection patterns:**
```
# Find body scroll lock attempts
Pattern: (body|document\.body).*overflow.*hidden|overflow.*hidden.*body
Files: *.js, *.ts, *.jsx, *.tsx, *.css

# Find modal/dialog implementations
Pattern: (modal|dialog|drawer|overlay).*open|open.*(modal|dialog|drawer|overlay)
```

**Code fix:**
```css
/* CSS-only approach (2025): Most reliable for iOS */
html:has(dialog[open]) {
  overflow: hidden;
}

/* Or with a class toggle */
html.scroll-locked {
  position: fixed;
  width: 100%;
  overflow: hidden;
}
```

```javascript
// JavaScript approach that works on iOS Safari
let scrollPosition = 0;

function lockScroll() {
  scrollPosition = window.scrollY;
  document.documentElement.style.position = 'fixed';
  document.documentElement.style.top = `-${scrollPosition}px`;
  document.documentElement.style.width = '100%';
  document.documentElement.style.overflow = 'hidden';
}

function unlockScroll() {
  document.documentElement.style.position = '';
  document.documentElement.style.top = '';
  document.documentElement.style.width = '';
  document.documentElement.style.overflow = '';
  window.scrollTo(0, scrollPosition);
}

// Alternative: use the inert attribute (modern approach)
function openModal() {
  document.querySelector('main').setAttribute('inert', '');
  // The inert attribute prevents scrolling AND interaction
}
function closeModal() {
  document.querySelector('main').removeAttribute('inert');
}
```

---

## 6. Android Chrome Specific Issues

### 6.1 Overscroll Glow / Stretch Effect

**Problem:** Android Chrome shows a blue glow (older versions) or stretch effect (Android 12+) when overscrolling. This can look jarring in custom-themed apps and interferes with custom scroll indicators.

**Affected:** Android Chrome

**Detection patterns:**
```
# Check if overscroll-behavior is set globally
Pattern: overscroll-behavior
Files: *.css, *.scss
# If not found, the default browser overscroll effects are active
```

**Code fix:**
```css
/* Disable overscroll glow/stretch globally */
html, body {
  overscroll-behavior: none;
}

/* Or per-container */
.custom-scroll {
  overscroll-behavior: contain;
}
```

### 6.2 Back Gesture Interference (Predictive Back)

**Problem:** Android 13+ introduced "Predictive Back" gesture -- swiping from the left edge shows a preview of the previous screen. On Android Chrome, horizontal swipe navigation can conflict with carousels, sliders, and swipe-to-dismiss UI patterns. Chrome 122 (2024) improved this by only activating swipe within 12px of the screen edge.

**Affected:** Android Chrome 122+, Android 13+ system gesture

**Detection patterns:**
```
# Find horizontal swipe handlers that may conflict with back gesture
Pattern: (touchstart|pointerdown)[\s\S]{0,300}(clientX|pageX|screenX)
Files: *.js, *.ts, *.jsx, *.tsx

# Find carousels/sliders without edge protection
Pattern: (carousel|slider|swipe|drag)(?![\s\S]{0,500}(edge|threshold|margin))
```

**Code fix:**
```javascript
// Add edge zone protection for swipeable elements
const EDGE_THRESHOLD = 20; // px from screen edge

element.addEventListener('touchstart', (e) => {
  const touch = e.touches[0];
  // Don't initiate swipe if starting from screen edge
  if (touch.clientX < EDGE_THRESHOLD ||
      touch.clientX > window.innerWidth - EDGE_THRESHOLD) {
    return; // Let the browser handle the back gesture
  }
  // ... handle swipe
}, { passive: true });
```

```css
/* Use touch-action to control which gestures are handled */
.carousel {
  touch-action: pan-y pinch-zoom; /* Only allow vertical pan and zoom, not horizontal */
}

/* If horizontal swipe IS needed */
.horizontal-scroll {
  touch-action: pan-x; /* Only horizontal */
  overscroll-behavior-x: contain; /* Don't trigger navigation */
}
```

### 6.3 PWA Issues on Android

**Problem:** PWAs on Android can have issues with: splash screen display (incorrect colors/sizing), orientation lock not working, push notification permission prompts being blocked, and `display: standalone` not properly hiding the URL bar in some Android WebView implementations.

**Affected:** Android Chrome PWA, Samsung Internet PWA

**Detection patterns:**
```
# Check manifest.json for common issues
Pattern: "display":\s*"standalone"|"display":\s*"fullscreen"
Files: manifest.json, manifest.webmanifest

# Check for missing theme-color
Pattern: <meta[^>]*theme-color
Files: *.html

# Check for missing PWA icons at required sizes
Pattern: "sizes":\s*"(192x192|512x512)"
Files: manifest.json, manifest.webmanifest
```

**Code fix:**
```json
// manifest.json -- ensure all required fields
{
  "name": "App Name",
  "short_name": "App",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#000000",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icon-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

```html
<!-- Required meta tags for Android PWA -->
<meta name="theme-color" content="#000000">
<meta name="mobile-web-app-capable" content="yes">
<link rel="manifest" href="/manifest.json">
```

---

## 7. Mobile Form UX

### 7.1 Virtual Keyboard Handling

**Problem:** The virtual keyboard pushes content up on mobile, often hiding the active input field or obscuring important UI elements. Fixed bottom elements (submit buttons, toolbars) can end up behind the keyboard or in an awkward position. The `visualViewport` API provides the actual visible area.

**Affected:** All mobile browsers (behavior varies between iOS and Android)

**Detection patterns:**
```
# Find fixed bottom elements that may be affected by keyboard
Pattern: (position:\s*fixed|position:\s*sticky)[^}]*(bottom:\s*0|bottom:\s*\d)
Files: *.css, *.scss

# Find forms without keyboard handling
Pattern: <form(?![\s\S]{0,1000}(visualViewport|keyboard|resize))
Files: *.html, *.jsx, *.tsx

# Find VirtualKeyboard API usage (or lack thereof)
Pattern: navigator\.virtualKeyboard|virtualKeyboard
```

**Code fix:**
```javascript
// Handle fixed bottom elements when keyboard opens
if ('visualViewport' in window) {
  window.visualViewport.addEventListener('resize', () => {
    const keyboardHeight = window.innerHeight - window.visualViewport.height;
    const bottomBar = document.querySelector('.fixed-bottom-bar');
    if (bottomBar) {
      bottomBar.style.transform = keyboardHeight > 0
        ? `translateY(-${keyboardHeight}px)`
        : 'translateY(0)';
    }
  });
}

// Scroll active input into view when keyboard opens
document.querySelectorAll('input, textarea, select').forEach(input => {
  input.addEventListener('focus', () => {
    setTimeout(() => {
      input.scrollIntoView({ behavior: 'smooth', block: 'center' });
    }, 300); // Wait for keyboard animation
  });
});

// Modern: Use VirtualKeyboard API (Chrome 94+)
if ('virtualKeyboard' in navigator) {
  navigator.virtualKeyboard.overlaysContent = true;
  // Now handle with CSS env() variables
}
```

```css
/* Modern CSS approach with VirtualKeyboard API */
.fixed-bottom-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  /* Adjusts automatically when keyboard is visible */
  bottom: env(keyboard-inset-height, 0px);
  transition: bottom 0.2s ease;
}
```

### 7.2 Autofill Styling

**Problem:** Chrome and Safari aggressively style autofilled inputs with a yellow/light blue background that overrides custom styles. The `:-webkit-autofill` pseudo-class applies styles that use `!important`, making them hard to override with normal CSS.

**Affected:** Chrome (desktop + mobile), Safari, all WebKit browsers

**Detection patterns:**
```
# Find input styling without autofill override
Pattern: (input|textarea|select)\s*\{(?![\s\S]*?autofill)
Files: *.css, *.scss

# Find existing autofill overrides that may be incorrect
Pattern: :-webkit-autofill|:autofill
```

**Code fix:**
```css
/* Override autofill styling */
input:-webkit-autofill,
input:-webkit-autofill:hover,
input:-webkit-autofill:focus,
textarea:-webkit-autofill,
select:-webkit-autofill {
  /* Use box-shadow to cover the yellow background */
  -webkit-box-shadow: 0 0 0px 1000px white inset !important;
  box-shadow: 0 0 0px 1000px white inset !important;

  /* Override text color */
  -webkit-text-fill-color: #333 !important;

  /* Prevent background color transition */
  transition: background-color 5000s ease-in-out 0s;
}

/* For dark theme */
input:-webkit-autofill {
  -webkit-box-shadow: 0 0 0px 1000px #1a1a1a inset !important;
  -webkit-text-fill-color: #ffffff !important;
}

/* Modern standard (use both for compatibility) */
input:autofill {
  -webkit-box-shadow: 0 0 0px 1000px white inset !important;
}
```

### 7.3 Input Type Optimization

**Problem:** Using `type="text"` for all inputs forces mobile users to switch keyboard modes manually. Using the correct `type` and `inputmode` attributes shows the optimal virtual keyboard layout, significantly improving form completion speed (up to 30% improvement with autofill).

**Affected:** All mobile browsers

**Detection patterns:**
```
# Find inputs without optimal type/inputmode
Pattern: type=["']text["'](?![\s\S]{0,50}inputmode)
Files: *.html, *.jsx, *.tsx, *.vue

# Find inputs for specific data types using text type
Pattern: (phone|tel|email|zip|postal|credit.?card|cvv|amount|price|quantity|search|url|otp|pin|code)[\s\S]{0,100}type=["']text["']

# Find inputs without enterkeyhint
Pattern: <input(?![^>]*enterkeyhint)[^>]*>
```

**Code fix:**
```html
<!-- Phone number -->
<input type="tel" inputmode="tel" autocomplete="tel"
       enterkeyhint="next">

<!-- Email -->
<input type="email" inputmode="email" autocomplete="email"
       enterkeyhint="next">

<!-- URL -->
<input type="url" inputmode="url" autocomplete="url"
       enterkeyhint="go">

<!-- Numeric (no decimals, e.g., ZIP code, OTP) -->
<input type="text" inputmode="numeric" pattern="[0-9]*"
       autocomplete="one-time-code"
       enterkeyhint="done">

<!-- Currency / decimal amount -->
<input type="text" inputmode="decimal"
       enterkeyhint="done">

<!-- Search -->
<input type="search" inputmode="search"
       enterkeyhint="search">

<!-- Credit card -->
<input type="text" inputmode="numeric" autocomplete="cc-number"
       pattern="[0-9 ]*" enterkeyhint="next">

<!-- Last field in form -->
<input type="email" enterkeyhint="send">
```

### 7.4 iOS Safari Font Size Auto-Zoom

**Problem:** iOS Safari auto-zooms the viewport when any input field has `font-size` less than 16px. After losing focus, Safari does NOT zoom back out, leaving the page in a zoomed, broken state. This is one of the most common mobile UX complaints.

**Affected:** iOS Safari exclusively

**Detection patterns:**
```
# Find form inputs with small font sizes
Pattern: (input|select|textarea)[\s\S]{0,200}(font-size:\s*(1[0-5]|[0-9])px|text-(xs|sm)|text-\[1[0-5]px\]|text-\[(0\.\d+rem|0\.[0-8]\d*em)\])
Files: *.css, *.scss

# Find Tailwind classes on inputs
Pattern: <(input|select|textarea)[^>]*(text-xs|text-sm|text-\[1[0-4]px\])
Files: *.html, *.jsx, *.tsx, *.vue

# Find global input styles that may set small font
Pattern: (input|select|textarea)\s*\{[\s\S]*?font-size
```

**Code fix:**
```css
/* Ensure all form inputs are at least 16px */
input, select, textarea {
  font-size: max(16px, 1rem);
}

/* If you need visually smaller inputs, use CSS transform */
.small-input {
  font-size: 16px; /* Prevents zoom */
  transform: scale(0.875); /* Visually appears as 14px */
  transform-origin: left top;
  /* Adjust width to compensate for scale */
  width: calc(100% / 0.875);
}

/* Tailwind approach: use text-base on mobile, text-sm on desktop */
/* class="text-base md:text-sm" */
```

**Anti-pattern -- DO NOT DO THIS:**
```html
<!-- NEVER disable user scaling to "fix" zoom -- violates WCAG 2.1 -->
<meta name="viewport" content="..., maximum-scale=1, user-scalable=no">
```

---

## 8. Mobile Typography Best Practices

### 8.1 Minimum Font Sizes

**Problem:** Text smaller than 16px is hard to read on mobile. iOS Safari auto-zooms inputs below 16px. Lighthouse flags text under 12px. WCAG requires sufficient contrast at whatever size is used.

**Affected:** All mobile browsers + all screen readers

**Detection patterns:**
```
# Find excessively small font sizes
Pattern: font-size:\s*(([0-9]|1[0-1])px|0\.[0-6]\d*rem|0\.[0-6]\d*em)
Files: *.css, *.scss

# Find Tailwind tiny text classes
Pattern: text-\[([0-9]|1[0-1])px\]|text-xs
Files: *.html, *.jsx, *.tsx, *.vue

# Find inline styles with small fonts
Pattern: fontSize:\s*['"]?(([0-9]|1[0-1])px|0\.[0-6]rem)
Files: *.jsx, *.tsx, *.vue
```

**Code fix:**
```css
/* Minimum body text: 16px */
body {
  font-size: clamp(16px, 1rem + 0.5vw, 20px);
  line-height: 1.5;
}

/* Minimum for UI elements: 14px */
.caption, .label, .helper-text {
  font-size: max(14px, 0.875rem);
}

/* Minimum for any readable text: 12px */
.fine-print {
  font-size: max(12px, 0.75rem);
}

/* NEVER go below 12px on mobile */
```

### 8.2 Line-Height for Readability

**Problem:** Tight line-heights (1.0-1.2) make text blocks unreadable on small screens. Conversely, excessive line-height wastes valuable screen space. The optimal range for mobile body text is 1.4-1.6.

**Affected:** All mobile browsers

**Detection patterns:**
```
# Find tight line-heights
Pattern: line-height:\s*(0\.\d|1\.?[0-2]?\d?)(;|\s|$|px|em)
Files: *.css, *.scss

# Find Tailwind tight leading
Pattern: leading-(none|tight|3|4|5)
Files: *.html, *.jsx, *.tsx, *.vue

# Find line-height in px (not relative -- doesn't scale)
Pattern: line-height:\s*\d+px
```

**Code fix:**
```css
/* Body text */
body {
  line-height: 1.5; /* Relative, scales with font-size */
}

/* Headings (tighter is OK for large text) */
h1, h2, h3 {
  line-height: 1.2;
}

/* Long-form content */
article, .content {
  line-height: 1.6;
}

/* NEVER use fixed px line-height */
/* BAD: line-height: 18px; */
/* GOOD: line-height: 1.5; */
```

### 8.3 Text Truncation

**Problem:** Long text strings (titles, usernames, descriptions) overflow their containers on narrow mobile screens, causing horizontal scrolling or broken layouts. Truncation must be handled carefully to avoid cutting mid-word and to provide accessible full text.

**Affected:** All mobile browsers

**Detection patterns:**
```
# Find text elements without overflow handling
Pattern: (title|heading|name|label|badge|tag|chip)(?![\s\S]{0,200}(truncat|ellipsis|overflow|line-clamp|text-overflow))
Files: *.css, *.scss

# Find elements that may need truncation
Pattern: (white-space:\s*nowrap)(?![\s\S]{0,100}(text-overflow|overflow))
```

**Code fix:**
```css
/* Single line truncation */
.truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* Multi-line truncation (2 lines) */
.line-clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* Responsive line clamping */
.responsive-clamp {
  display: -webkit-box;
  -webkit-line-clamp: 3; /* Mobile: 3 lines */
  -webkit-box-orient: vertical;
  overflow: hidden;
}
@media (min-width: 768px) {
  .responsive-clamp {
    -webkit-line-clamp: unset; /* Desktop: show all */
  }
}
```

```html
<!-- Accessible truncation with full text available -->
<p class="truncate" title="Full text of the truncated content here">
  Full text of the truncated content here
</p>

<!-- Or with aria-label for screen readers -->
<p class="truncate" aria-label="Full text of the truncated content here">
  Full text of the truncated content here
</p>
```

### 8.4 Optimal Line Length

**Problem:** Lines wider than 45 characters on mobile are hard to track from line end to next line start. Lines narrower than 25 characters create too many line breaks. The optimal range for mobile is 35-45 characters.

**Affected:** All mobile browsers

**Detection patterns:**
```
# Find paragraphs or content areas without max-width constraints
Pattern: (article|\.content|\.text|main|\.body)[\s\S]{0,200}\{(?![\s\S]{0,200}max-width)
Files: *.css, *.scss

# Find text containers using 100% width without padding
Pattern: width:\s*100%(?![\s\S]{0,100}padding)
```

**Code fix:**
```css
/* Limit line length for readability */
article, .content, .prose {
  max-width: 65ch; /* ~65 characters -- good for desktop */
  padding-inline: 1rem; /* Side padding for mobile */
}

/* On mobile, the screen width naturally limits line length */
/* but ensure padding exists */
@media (max-width: 768px) {
  article, .content {
    padding-inline: 1rem;
    /* max-width is naturally the screen width */
  }
}
```

---

## 9. Mobile Responsive Image Strategies

### 9.1 Art Direction with Picture Element

**Problem:** A wide landscape hero image on desktop loses its subject when scaled down to mobile. The `<picture>` element with `<source>` allows serving different crops for different screen sizes (art direction), not just different resolutions.

**Affected:** All browsers (universal support)

**Detection patterns:**
```
# Find hero/banner images without picture element
Pattern: <img[^>]*(hero|banner|feature|cover)[^>]*>(?![\s\S]{0,50}</picture)
Files: *.html, *.jsx, *.tsx, *.vue

# Find large images without srcset
Pattern: <img(?![^>]*srcset)[^>]*src=["'][^"']*\.(jpg|jpeg|png|webp)["'][^>]*>
Files: *.html, *.jsx, *.tsx, *.vue

# Find images using only src (no responsive variants)
Pattern: <img[^>]*src=[^>]*>(?![^>]*srcset)
```

**Code fix:**
```html
<!-- Art direction: different crops for different viewports -->
<picture>
  <!-- Mobile: vertical crop focused on subject -->
  <source media="(max-width: 767px)"
          srcset="hero-mobile.webp 375w,
                  hero-mobile-2x.webp 750w"
          sizes="100vw"
          type="image/webp">
  <source media="(max-width: 767px)"
          srcset="hero-mobile.jpg 375w,
                  hero-mobile-2x.jpg 750w"
          sizes="100vw">

  <!-- Desktop: wide landscape crop -->
  <source media="(min-width: 768px)"
          srcset="hero-desktop.webp 1200w,
                  hero-desktop-2x.webp 2400w"
          sizes="100vw"
          type="image/webp">
  <source media="(min-width: 768px)"
          srcset="hero-desktop.jpg 1200w,
                  hero-desktop-2x.jpg 2400w"
          sizes="100vw">

  <!-- Fallback -->
  <img src="hero-desktop.jpg" alt="Hero description"
       width="1200" height="600"
       loading="eager" fetchpriority="high"
       decoding="async">
</picture>
```

### 9.2 AVIF/WebP Support Detection

**Problem:** AVIF provides 20-50% smaller files than WebP at similar quality, and WebP is 25-35% smaller than JPEG. However, AVIF support is newer. The `<picture>` element's source-order evaluation provides automatic format detection -- no JavaScript needed.

**Affected:** AVIF: Chrome 85+, Firefox 93+, Safari 16.4+. WebP: universal.

**Detection patterns:**
```
# Find images served as JPEG/PNG without modern format alternatives
Pattern: <img[^>]*src=["'][^"']*\.(jpg|jpeg|png)["']
Files: *.html, *.jsx, *.tsx, *.vue

# Check for missing picture element with format sources
Pattern: src=["'][^"']*\.(jpg|jpeg|png)["'](?![\s\S]{0,200}type=["']image/(webp|avif))

# Find picture elements without AVIF source
Pattern: <picture>(?![\s\S]*?type=["']image/avif["'])[\s\S]*?</picture>
```

**Code fix:**
```html
<!-- Progressive format with AVIF -> WebP -> JPEG fallback -->
<picture>
  <source srcset="image.avif" type="image/avif">
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Description"
       width="800" height="600"
       loading="lazy" decoding="async">
</picture>

<!-- With responsive sizes -->
<picture>
  <source srcset="image-400.avif 400w, image-800.avif 800w, image-1200.avif 1200w"
          sizes="(max-width: 768px) 100vw, 50vw"
          type="image/avif">
  <source srcset="image-400.webp 400w, image-800.webp 800w, image-1200.webp 1200w"
          sizes="(max-width: 768px) 100vw, 50vw"
          type="image/webp">
  <img src="image-800.jpg"
       srcset="image-400.jpg 400w, image-800.jpg 800w, image-1200.jpg 1200w"
       sizes="(max-width: 768px) 100vw, 50vw"
       alt="Description"
       width="800" height="600"
       loading="lazy" decoding="async">
</picture>
```

### 9.3 Lazy Loading vs. Priority Loading

**Problem:** Applying `loading="lazy"` to above-the-fold images (especially the LCP element) deprioritizes the request and increases LCP. Conversely, eagerly loading all images wastes bandwidth on mobile networks.

**Affected:** All browsers

**Detection patterns:**
```
# Find hero/LCP images with lazy loading
Pattern: <img[^>]*(hero|banner|above-fold|header-image|main-image|lcp)[^>]*loading=["']lazy["']
Files: *.html, *.jsx, *.tsx, *.vue

# Find images without any loading strategy
Pattern: <img(?![^>]*loading=)[^>]*>
Files: *.html, *.jsx, *.tsx, *.vue

# Find LCP image without fetchpriority
Pattern: <img[^>]*(hero|banner|main)[^>]*>(?![^>]*fetchpriority)
```

**Code fix:**
```html
<!-- Above-the-fold / LCP image: eager + high priority -->
<img src="hero.webp" alt="Hero"
     loading="eager"
     fetchpriority="high"
     decoding="async"
     width="1200" height="600">

<!-- Below-the-fold images: lazy loaded -->
<img src="gallery-1.webp" alt="Gallery item 1"
     loading="lazy"
     decoding="async"
     width="400" height="300">

<!-- Preload LCP image in <head> for fastest loading -->
<link rel="preload" as="image" href="hero.webp"
      type="image/webp" fetchpriority="high">
```

---

## 10. Accessibility on Mobile

### 10.1 Screen Reader Compatibility

**Problem:** Mobile screen readers (VoiceOver on iOS, TalkBack on Android) use swipe gestures to navigate elements, double-tap to activate, and have different focus management than desktop keyboard navigation. Common issues: custom components not announcing roles, dynamic content changes not announced, and touch gestures that conflict with screen reader gestures.

**Affected:** iOS (VoiceOver), Android (TalkBack)

**Detection patterns:**
```
# Find interactive elements without roles
Pattern: <div[^>]*(onClick|on-click|@click|v-on:click)[^>]*>(?![^>]*role=)
Files: *.html, *.jsx, *.tsx, *.vue

# Find buttons using div/span instead of button
Pattern: <(div|span)[^>]*(click|tap|press)[^>]*>
Files: *.html, *.jsx, *.tsx, *.vue

# Find missing aria-live for dynamic content
Pattern: (toast|notification|alert|snackbar|banner|status)(?![\s\S]{0,200}aria-live)
Files: *.html, *.jsx, *.tsx, *.vue

# Find images without alt text
Pattern: <img(?![^>]*alt=)[^>]*>
Files: *.html, *.jsx, *.tsx, *.vue

# Find custom widgets without ARIA
Pattern: (accordion|tab|dropdown|tooltip|carousel|slider|toggle)(?![\s\S]{0,200}(role=|aria-))
```

**Code fix:**
```html
<!-- Dynamic content announcements -->
<div aria-live="polite" aria-atomic="true" class="sr-only">
  <!-- Screen reader will announce changes here -->
</div>

<!-- Custom button (use real button when possible) -->
<div role="button" tabindex="0"
     aria-label="Add to cart"
     onkeydown="if(event.key==='Enter'||event.key===' ')handleClick()">
</div>

<!-- Better: just use a button -->
<button aria-label="Add to cart" onclick="handleClick()">
  <svg aria-hidden="true">...</svg>
</button>

<!-- Toast notification -->
<div role="status" aria-live="polite" aria-atomic="true">
  Item added to cart
</div>

<!-- Alert (important, interrupts user) -->
<div role="alert" aria-live="assertive">
  Error: Payment failed
</div>
```

### 10.2 Reduced Motion Preferences

**Problem:** CSS animations and JavaScript-driven motion can cause vestibular disorders (dizziness, nausea) in affected users (70+ million people). The `prefers-reduced-motion` media query must be respected. This is a WCAG 2.1 Level AAA requirement (2.3.3) and widely expected in modern web apps.

**Affected:** All platforms (iOS, Android, Windows, macOS all support this setting)

**Detection patterns:**
```
# Find CSS animations without reduced-motion override
Pattern: (animation:|@keyframes|transition:)(?![\s\S]{0,500}prefers-reduced-motion)
Files: *.css, *.scss

# Find JS animations without reduced-motion check
Pattern: (animate\(|gsap|framer-motion|react-spring|lottie|anime)(?![\s\S]{0,500}(reduced-motion|reducedMotion|prefersReducedMotion))
Files: *.js, *.ts, *.jsx, *.tsx

# Find missing global reduced-motion styles
Pattern: @media\s*\(prefers-reduced-motion
Files: *.css, *.scss
# If not found at all, no reduced-motion support exists
```

**Code fix:**
```css
/* Global reduced-motion override */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

/* Per-element approach (more refined) */
.animated-element {
  animation: slideIn 0.5s ease-out;
}
@media (prefers-reduced-motion: reduce) {
  .animated-element {
    animation: none;
    /* Or use a simpler, non-motion animation */
    opacity: 1;
  }
}

/* Parallax */
.parallax-layer {
  transform: translateY(var(--parallax-offset));
}
@media (prefers-reduced-motion: reduce) {
  .parallax-layer {
    transform: none !important;
  }
}
```

```javascript
// JavaScript check for reduced motion
const prefersReducedMotion = window.matchMedia(
  '(prefers-reduced-motion: reduce)'
).matches;

if (prefersReducedMotion) {
  // Disable JS animations
  gsap.globalTimeline.pause();
  // Or reduce animation durations
} else {
  // Run animations normally
}

// Listen for changes (user can toggle preference)
window.matchMedia('(prefers-reduced-motion: reduce)')
  .addEventListener('change', (e) => {
    if (e.matches) {
      disableAnimations();
    } else {
      enableAnimations();
    }
  });
```

### 10.3 Focus Management on Touch Devices

**Problem:** Touch devices don't have a visible focus indicator by default, but keyboard/switch control users and screen reader users still need focus management. The challenge: showing focus indicators for keyboard/assistive tech users but hiding them for touch users. The `:focus-visible` pseudo-class solves this.

**Affected:** All mobile browsers

**Detection patterns:**
```
# Find focus styles using :focus instead of :focus-visible
Pattern: :focus\s*\{(?![\s\S]{0,100}:focus-visible)
Files: *.css, *.scss

# Find outline:none or outline:0 without :focus-visible alternative
Pattern: outline:\s*(none|0)(?![\s\S]{0,200}:focus-visible)
Files: *.css, *.scss

# Find missing focus management in modals/dialogs
Pattern: (modal|dialog|drawer|popup)(?![\s\S]{0,500}(focus|tabindex|inert))
Files: *.jsx, *.tsx, *.vue, *.svelte
```

**Code fix:**
```css
/* Remove default focus for mouse/touch, keep for keyboard */
:focus {
  outline: none;
}
:focus-visible {
  outline: 2px solid #4A90D9;
  outline-offset: 2px;
}

/* Skip-to-content link for screen readers */
.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  z-index: 9999;
  padding: 1rem;
  background: white;
  color: black;
}
.skip-link:focus {
  top: 0;
}
```

```html
<!-- Skip navigation link (first element in body) -->
<a href="#main-content" class="skip-link">Skip to main content</a>

<nav>...</nav>

<main id="main-content" tabindex="-1">
  <!-- Main content -->
</main>
```

```javascript
// Focus management for modal open/close
function openModal(modalElement) {
  // Store the element that triggered the modal
  const trigger = document.activeElement;

  modalElement.showModal(); // Native dialog handles focus trapping

  // Or for custom modals:
  modalElement.setAttribute('aria-hidden', 'false');
  const firstFocusable = modalElement.querySelector(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  firstFocusable?.focus();

  // Store trigger for return focus
  modalElement._trigger = trigger;
}

function closeModal(modalElement) {
  modalElement.close(); // Or setAttribute('aria-hidden', 'true')
  // Return focus to trigger
  modalElement._trigger?.focus();
}
```

### 10.4 Touch Target Sizes (WCAG 2.5.8)

**Problem:** WCAG 2.5.8 (Level AA, legally required since June 2025 under EAA) requires interactive targets to be at least 24x24 CSS pixels, or have sufficient spacing from adjacent targets. Best practice is 44x44px (Apple) or 48x48dp (Google Material). Targets smaller than 44px have 3x higher error rates.

**Affected:** All mobile browsers + all assistive technologies

**Detection patterns:**
```
# Find small interactive elements
Pattern: (min-width|min-height|width|height):\s*(([0-9]|1[0-9]|2[0-3])px|1\.(0|1|2|3|4)rem)
# Context: on button, a, input, select, [role="button"] elements

# Find Tailwind tiny targets
Pattern: (w-[3-5]|h-[3-5]|p-[01]|size-[3-5])
# On interactive elements

# Find icon buttons without minimum size
Pattern: <button[^>]*>[^<]*<svg|<button[^>]*>[^<]*<i\s
Files: *.html, *.jsx, *.tsx, *.vue
```

**Code fix:**
```css
/* Global minimum touch target */
button, a, input, select, textarea,
[role="button"], [role="link"], [role="checkbox"],
[role="radio"], [role="switch"], [role="tab"] {
  min-width: 44px;
  min-height: 44px;
}

/* For icon buttons that should be visually small */
.icon-button {
  position: relative;
  /* Visual size */
  width: 24px;
  height: 24px;
}
/* Expand touch target with pseudo-element */
.icon-button::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 44px;
  height: 44px;
  /* Invisible but tappable */
}

/* Ensure spacing between adjacent targets */
.button-group button + button {
  margin-left: 8px; /* Minimum 8px gap between targets */
}
```

---

## Quick Reference: Detection Regex Patterns for CLI Automation

Below is a consolidated list of all detection patterns organized for easy implementation in a CLI tool.

### High Priority (Most Impactful)

| Issue | Pattern | Files |
|-------|---------|-------|
| Missing viewport meta | `<meta[^>]*viewport` (check if exists) | `*.html` |
| 100vh on mobile | `height:\s*100vh\|min-height:\s*100vh\|h-screen` | `*.css, *.html, *.jsx, *.tsx` |
| Lazy-loaded hero image | `(hero\|banner\|main).*loading=["']lazy["']` | `*.html, *.jsx, *.tsx` |
| Missing image dimensions | `<img(?![^>]*width=)(?![^>]*height=)[^>]*>` | `*.html, *.jsx, *.tsx` |
| Input font < 16px | `(input\|select\|textarea).*font-size:\s*(1[0-5]\|[0-9])px` | `*.css` |
| Non-passive listeners | `addEventListener.*['"\x60](touchstart\|touchmove\|wheel)['"\x60].*\)$` | `*.js, *.ts` |
| No reduced-motion | `@media.*prefers-reduced-motion` (check if exists) | `*.css` |
| Missing font-display | `@font-face.*\{(?!.*font-display)` | `*.css` |

### Medium Priority

| Issue | Pattern | Files |
|-------|---------|-------|
| backdrop-filter perf | `backdrop-filter:\s*blur` | `*.css` |
| Missing safe-area | `position:\s*fixed(?!.*safe-area)` | `*.css` |
| Layout animations | `transition.*(?:width\|height\|top\|left\|margin\|padding)` | `*.css` |
| Missing overscroll | `overflow.*scroll(?!.*overscroll)` | `*.css` |
| No alt text | `<img(?![^>]*alt=)[^>]*>` | `*.html, *.jsx, *.tsx` |
| Div as button | `<div[^>]*(onClick\|on-click)[^>]*>(?![^>]*role=)` | `*.jsx, *.tsx, *.vue` |
| Missing aria-expanded | `(hamburger\|menu-toggle)(?!.*aria-expanded)` | `*.html, *.jsx, *.tsx` |

### Lower Priority (Still Important)

| Issue | Pattern | Files |
|-------|---------|-------|
| will-change overuse | `will-change` (count > 10) | `*.css` |
| Deprecated scrolling | `-webkit-overflow-scrolling` | `*.css` |
| Missing inputmode | `type=["']text["'](?!.*inputmode)` | `*.html, *.jsx, *.tsx` |
| Small touch targets | Icon buttons without `min-width: 44px` | `*.css` |
| Missing picture element | `<img.*\.(jpg\|png).*>(?!.*</picture)` | `*.html, *.jsx, *.tsx` |

---

## Sources

### CSS / Scroll-Snap / Container Queries
- [WebKit Bug 173887 - Scroll Snap Jitter on iOS](https://bugs.webkit.org/show_bug.cgi?id=173887)
- [CSS Scroll Snap Visual Glitches on iOS](https://www.xjavascript.com/blog/css-scroll-snap-visual-glitches-on-ios-when-programmatically-setting-style-on-children/)
- [Practical CSS Scroll Snapping - CSS-Tricks](https://css-tricks.com/practical-css-scroll-snapping/)
- [CSS Container Queries - Can I Use](https://caniuse.com/css-container-queries)
- [CSS Container Queries in 2025 - Caisy](https://caisy.io/blog/css-container-queries)
- [Container Queries in 2026 - LogRocket](https://blog.logrocket.com/container-queries-2026/)
- [CSS Subgrid - Can I Use](https://caniuse.com/css-subgrid)
- [CSS Subgrid Browser Support 2025-2026](https://www.frontendtools.tech/blog/mastering-css-grid-2025)

### Mobile JavaScript
- [Passive Event Listeners - Chrome Developers](https://developer.chrome.com/docs/lighthouse/best-practices/uses-passive-event-listeners)
- [Improving Scroll Performance with Passive Listeners - Chrome Blog](https://developer.chrome.com/blog/passive-event-listeners)
- [Overscroll Behavior - Chrome Developers](https://developer.chrome.com/blog/overscroll-behavior)
- [MDN: overscroll-behavior](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/overscroll-behavior)
- [Momentum Scrolling on iOS - CSS-Tricks](https://css-tricks.com/snippets/css/momentum-scrolling-on-ios-overflow-elements/)

### Core Web Vitals
- [Core Web Vitals Explained 2026](https://www.corewebvitals.io/core-web-vitals)
- [How to Fix Core Web Vitals Issues 2025 - NitroPack](https://nitropack.io/blog/post/core-web-vitals-issues)
- [Core Web Vitals 2025 - Google Stricter Standards](https://systemsarchitect.net/core-web-vitals-2025/)
- [Fixing Layout Shifts from Web Fonts - DebugBear](https://www.debugbear.com/blog/web-font-layout-shift)
- [Web Fonts and CLS - Sentry](https://blog.sentry.io/web-fonts-and-the-dreaded-cumulative-layout-shift/)
- [Fixing CLS with Web Fonts - Vincent Bernat](https://vincent.bernat.ch/en/blog/2024-cls-webfonts)
- [Optimize CLS - web.dev](https://web.dev/articles/optimize-cls)
- [Image CLS Prevention - PageSpeed.ONE](https://pagespeed.one/en/know-how/cls-img-width-height)
- [Lazy Loading LCP Penalty - Etavrian](https://www.etavrian.com/news/lazy-loading-lcp-hero-images)

### iOS Safari
- [iOS 26 Viewport Changes - StripEArmy](https://stripearmy.medium.com/ios-26-0-be-prepared-for-viewport-changes-in-safari-e867d7eace43)
- [Fixed iOS Safari Scroll Lock Bug](https://stripearmy.medium.com/i-fixed-a-decade-long-ios-safari-problem-0d85f76caec0)
- [Safari 26 Fixed Overlays Bug - Lunardi](https://www.edoardolunardi.dev/blog/safari-26-and-the-strange-case-of-fixed-overlays)
- [Locking Body Scroll on iOS - Jay Freestone](https://www.jayfreestone.com/writing/locking-body-scroll-ios/)
- [iOS Safari Position Fixed During Scroll - Muffin Man](https://muffinman.io/blog/ios-safari-scroll-position-fixed/)
- [16px Font Prevents iOS Zoom - CSS-Tricks](https://css-tricks.com/16px-or-larger-text-prevents-ios-form-zoom/)
- [No Input Zoom Pixel Perfect Way](https://thingsthemselves.com/no-input-zoom-in-safari-on-iphone-the-pixel-perfect-way/)
- [Defensive CSS - Input Zoom Safari](https://defensivecss.dev/tip/input-zoom-safari/)

### Android Chrome
- [Chrome Overscroll Behavior - Chrome Blog](https://developer.chrome.com/blog/overscroll-behavior)
- [Chrome Back Gesture Changes](https://issuetracker.google.com/issues/172341945)
- [MUI iOS 26 Drawer/Modal Issue](https://github.com/mui/material-ui/issues/46953)

### Mobile Forms
- [VirtualKeyboard API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/VirtualKeyboard_API)
- [inputmode Attribute - MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/inputmode)
- [enterkeyhint Attribute - MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/enterkeyhint)
- [:autofill / :-webkit-autofill - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/:autofill)
- [Change Autocomplete Styles - CSS-Tricks](https://css-tricks.com/snippets/css/change-autocomplete-styles-webkit-browsers/)
- [Mobile Form Statistics 2025 - Tinyform](https://tinyform.com/post/mobile-form-statistics-ux-trends)

### Typography
- [Font Size Guidelines for Responsive Websites 2024 - LearnUI](https://www.learnui.design/blog/mobile-desktop-website-font-size-guidelines.html)
- [Mobile Typography Accessibility - FontFYI](https://fontfyi.com/blog/mobile-typography-accessibility/)
- [Typography in UX Best Practices 2025](https://developerux.com/2025/02/12/typography-in-ux-best-practices-guide/)

### Images
- [Ultimate Guide to Responsive Images - DebugBear](https://www.debugbear.com/blog/responsive-images)
- [picture Element - MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/picture)
- [WebP vs JPEG vs AVIF 2026](https://blog.freeimages.com/post/webp-vs-jpeg-vs-avif-best-format-for-web-photos)
- [Responsive Images Guide - ImageKit](https://imagekit.io/responsive-images/)

### Accessibility
- [Mobile Patterns that Break Accessibility - TestParty](https://testparty.ai/blog/mobile-accessibility-patterns)
- [WCAG 2.5.8 Target Size Guide - TestParty](https://testparty.ai/blog/wcag-target-size-guide)
- [Accessible Tap Target Sizes - Smashing Magazine](https://www.smashingmagazine.com/2023/04/accessible-tap-target-sizes-rage-taps-clicks/)
- [prefers-reduced-motion - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-reduced-motion)
- [prefers-reduced-motion - web.dev](https://web.dev/articles/prefers-reduced-motion)
- [WCAG 2.3.3 Animation from Interactions - W3C](https://www.w3.org/WAI/WCAG21/Understanding/animation-from-interactions.html)
- [Improving Accessibility of Bottom Sheets - InThePocket](https://www.inthepocket.design/articles/improving-the-accessibility-of-bottom-sheets)
- [Bottom Sheets UX Guidelines - NNGroup](https://www.nngroup.com/articles/bottom-sheet/)
