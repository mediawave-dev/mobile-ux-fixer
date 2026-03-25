# Mobile Web Standards Research Report (2024-2026)

> Comprehensive research for building a mobile UX code audit tool.
> Covers SEO, meta tags, PWA, performance metrics, fonts, third-party scripts,
> dark mode, responsive design, security, and RTL/internationalization.

---

## Table of Contents

1. [Mobile SEO Requirements](#1-mobile-seo-requirements)
2. [Meta Tags and Head Configuration](#2-meta-tags-and-head-configuration-for-mobile)
3. [PWA Checklist](#3-pwa-checklist)
4. [Mobile-Specific Performance Metrics](#4-mobile-specific-performance-metrics)
5. [Font Loading on Mobile](#5-font-loading-on-mobile)
6. [Third-Party Script Impact](#6-third-party-script-impact-on-mobile)
7. [Mobile Dark Mode](#7-mobile-dark-mode)
8. [Responsive Design Patterns](#8-responsive-design-patterns)
9. [Mobile-Specific Security](#9-mobile-specific-security)
10. [International/RTL Mobile Issues](#10-internationalrtl-mobile-issues)

---

## 1. Mobile SEO Requirements

### Current Status (2024-2026)

Google completed its mobile-first indexing rollout on **July 5, 2024**. Mobile-first is now the **universal default for 100% of websites**. There are no longer any desktop-only crawling exceptions. Google uses only the mobile version of a page for indexing and ranking.

### Key Requirements

**Content Parity:**
- Identical content between mobile and desktop versions
- Same `meta` tags, structured data, headings, and metadata on both versions
- Same alt text for images across both versions
- No content hidden behind tabs/accordions that desktop shows openly

**Mobile Usability Signals:**
- Responsive design (not separate mobile URLs preferred)
- No horizontal scrolling required
- Text readable without zooming (16px base minimum)
- Touch targets adequately sized and spaced (48x48dp minimum)
- No intrusive interstitials blocking content

**Page Speed as Ranking Factor:**
- Core Web Vitals are a confirmed ranking signal
- Mobile page speed is independently measured (separate from desktop)
- Sites failing CWV on mobile risk ranking penalties even if desktop passes

**Structured Data (Schema.org / JSON-LD):**
- Google recommends JSON-LD format (89.4% market share as of 2024)
- JSON-LD must be present on mobile pages (since only mobile is crawled)
- Place JSON-LD in `<head>` for easy maintenance across viewport versions
- Shops with mobile-optimized schema see 47.3% higher mobile engagement

### Detection Patterns for Code Audit

```
# Check: Missing viewport meta tag
Pattern: <head> section without <meta name="viewport"

# Check: Content hidden on mobile only
Pattern: class=".*hidden.*md:block|display:\s*none.*@media.*min-width

# Check: Structured data only on desktop
Pattern: application/ld\+json (verify present in mobile-served HTML)

# Check: Separate mobile URL without canonical
Pattern: <link rel="alternate" media="only screen and (max-width
Without corresponding: <link rel="canonical"

# Check: Robots meta blocking mobile
Pattern: <meta name="robots".*noindex (on mobile-served pages)
```

### Fixes

```html
<!-- Required viewport meta tag -->
<meta name="viewport" content="width=device-width, initial-scale=1">

<!-- JSON-LD in head (works for both desktop and mobile) -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebPage",
  "name": "Page Title",
  "description": "Page description"
}
</script>

<!-- Responsive images with alt text -->
<img src="image.webp"
     srcset="image-400.webp 400w, image-800.webp 800w"
     sizes="(max-width: 600px) 100vw, 50vw"
     alt="Descriptive alt text matching desktop version"
     width="800" height="600"
     loading="lazy">
```

---

## 2. Meta Tags and Head Configuration for Mobile

### Essential Meta Tags

| Meta Tag | Purpose | Required? |
|----------|---------|-----------|
| `viewport` | Controls mobile layout and zoom | **YES** |
| `theme-color` | Colors browser chrome on Android/iOS | Recommended |
| `color-scheme` | Opts into light/dark UA defaults | Recommended |
| `description` | SEO snippet for search results | **YES** |
| `robots` | Crawling instructions | Situational |
| `format-detection` | Prevents auto-linking phone numbers | Recommended |

### Viewport Meta Tag (Critical)

```html
<!-- GOOD: Accessible, responsive -->
<meta name="viewport" content="width=device-width, initial-scale=1">

<!-- BAD: Blocks zoom - WCAG AA violation -->
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
```

**WCAG Requirements (2024):**
- `user-scalable=no` is a WCAG 2.0 Level AA violation
- `maximum-scale` must not be less than 2.0 (best practice: allow 5x zoom)
- People with low vision rely on zoom to read content
- Remove `maximum-scale` and `user-scalable=no` entirely

### Apple Mobile Web App Tags (Deprecation Notice 2024)

```html
<!-- DEPRECATED: These meta tags are no longer recommended -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="App Name">

<!-- MODERN APPROACH: Use web app manifest instead -->
<link rel="manifest" href="/manifest.json">
```

Safari now falls back to `apple-mobile-web-app-capable` only if it cannot load the manifest. Using deprecated tags without a manifest creates a poor home screen experience (no start_url, no scope).

### Theme-Color with Dark Mode Support

```html
<!-- Light and dark theme colors -->
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#1a1a2e" media="(prefers-color-scheme: dark)">

<!-- Color scheme declaration -->
<meta name="color-scheme" content="light dark">
```

### Open Graph for Mobile Sharing

```html
<!-- Essential OG tags -->
<meta property="og:title" content="Page Title">
<meta property="og:description" content="Description under 200 chars">
<meta property="og:image" content="https://example.com/og-image.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:type" content="website">
<meta property="og:url" content="https://example.com/page">

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Page Title">
<meta name="twitter:description" content="Description">
<meta name="twitter:image" content="https://example.com/twitter-image.jpg">
```

**Image Dimensions:**
- Open Graph: **1200x630px** (1.91:1 ratio) -- works on Facebook, LinkedIn, WhatsApp
- Twitter summary_large_image: **1200x675px**
- Twitter summary (square): **144x144** to **4096x4096px**
- Keep focal point centered (platforms may crop edges)
- Max file size: 5MB (smaller is better for mobile sharing speed)

### Miscellaneous Useful Meta Tags

```html
<!-- Prevent phone number auto-linking (iOS) -->
<meta name="format-detection" content="telephone=no">

<!-- DNS prefetch for critical third-party origins -->
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

### Detection Patterns

```
# Check: Missing viewport meta tag
Pattern: <head>.*</head> without name="viewport"

# Check: Zoom-blocking viewport
Pattern: user-scalable\s*=\s*(no|0)|maximum-scale\s*=\s*1([^0-9]|$)

# Check: Deprecated apple-mobile-web-app-capable without manifest
Pattern: apple-mobile-web-app-capable without <link rel="manifest"

# Check: Missing OG image
Pattern: <head>.*</head> without og:image

# Check: OG image with wrong dimensions
Pattern: og:image:width.*(value != 1200) or og:image:height.*(value != 630)

# Check: Missing theme-color
Pattern: <head>.*</head> without name="theme-color"
```

---

## 3. PWA Checklist

### Installability Criteria (2024-2025 Update)

As of 2024, **service workers are no longer strictly required** for installability in Chrome and Edge. The minimum requirements are:

1. **HTTPS** (or localhost for development)
2. **Web App Manifest** with required fields
3. Manifest must be linked from the HTML page

However, service workers remain **essential** for offline functionality, caching, and a quality PWA experience.

### Web App Manifest Required Fields

```json
{
  "name": "Full Application Name",
  "short_name": "ShortName",
  "description": "Description of the application",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#319197",
  "orientation": "portrait-primary",
  "scope": "/",
  "icons": [
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-maskable-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "/icons/icon-maskable-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

### Icon Requirements

| Size | Purpose | Notes |
|------|---------|-------|
| 192x192 | `any` | **Required** by Chrome for installability |
| 512x512 | `any` | **Required** by Chrome for installability and splash screen |
| 192x192 | `maskable` | Recommended for adaptive icons on Android |
| 512x512 | `maskable` | Recommended -- keep content in inner 80% safe zone |

**Maskable Icon Safe Zone:** Important content must stay within the inner 80% circle of the image. The OS may crop up to 20% from edges into circles, rounded squares, or squircles.

**Avoid:** `"purpose": "any maskable"` -- this forces the "any" icon to be masked, which often crops poorly. Use separate icons for each purpose.

### Service Worker Basics to Check

```javascript
// Minimal service worker registration
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js')
      .then(reg => console.log('SW registered:', reg.scope))
      .catch(err => console.log('SW registration failed:', err));
  });
}
```

**Essential Service Worker Features:**

1. **Offline Fallback Page** (minimum viable):
```javascript
// sw.js - Using Workbox (used by 54% of mobile sites)
import { offlineFallback } from 'workbox-recipes';
import { setDefaultHandler } from 'workbox-routing';
import { NetworkOnly } from 'workbox-strategies';

setDefaultHandler(new NetworkOnly());
offlineFallback({ pageFallback: '/offline.html' });
```

2. **Cache Strategies:**
   - **Cache First**: Static assets (CSS, JS, images) -- serve from cache, update in background
   - **Network First**: HTML pages, API responses -- try network, fall back to cache
   - **Stale While Revalidate**: Frequently updated content -- serve cache, update cache from network

3. **Pre-caching Essential Files:**
```javascript
import { precacheAndRoute } from 'workbox-precaching';
precacheAndRoute([
  { url: '/offline.html', revision: '1' },
  { url: '/css/critical.css', revision: '1' },
  { url: '/icons/icon-192x192.png', revision: '1' }
]);
```

### Detection Patterns

```
# Check: Missing manifest link
Pattern: <head>.*</head> without <link rel="manifest"

# Check: Manifest missing required fields
Parse manifest.json and verify: name, icons, start_url, display

# Check: Missing required icon sizes
Parse manifest.json icons array -- must contain 192x192 AND 512x512

# Check: Maskable icon with wrong purpose
Pattern: "purpose":\s*"any maskable" (should be separate entries)

# Check: No service worker registration
Pattern: Search all JS for navigator.serviceWorker.register
If absent, no SW is registered

# Check: No offline fallback
Fetch /offline.html or check SW for offline handling logic

# Check: HTTPS not enforced
Pattern: start_url or scope starting with http://
```

### Fixes

```html
<!-- Link manifest in HTML head -->
<link rel="manifest" href="/manifest.json">

<!-- Apple touch icon fallback -->
<link rel="apple-touch-icon" href="/icons/apple-touch-icon-180x180.png">
```

---

## 4. Mobile-Specific Performance Metrics

### Core Web Vitals Thresholds (2024-2026)

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| **LCP** (Largest Contentful Paint) | < 2.5s | 2.5s - 4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | < 200ms | 200ms - 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | < 0.1 | 0.1 - 0.25 | > 0.25 |

**Assessment Rule:** To pass, at least **75% of page visits** must meet the "Good" threshold.

**Mobile Reality (2025 Web Almanac):**
- Only **62% of mobile pages** achieve good LCP (hardest CWV to pass)
- **77% of mobile pages** achieve good INP
- INP replaced FID as a Core Web Vital in **March 2024**

### LCP -- Detection of Code Issues

**Issue 1: Lazy-loading the LCP image**
```
# DETECT: LCP image has loading="lazy"
Pattern: <img.*loading="lazy" on hero/above-fold images
Pattern: data-src (lazy-load library) on hero images without src
```

```html
<!-- BAD: Hero image lazy-loaded delays LCP significantly -->
<img src="hero.jpg" loading="lazy" alt="Hero">

<!-- GOOD: Hero image eager + priority hint -->
<img src="hero.jpg" fetchpriority="high" alt="Hero">
```

**Issue 2: Render-blocking CSS/JS**
```
# DETECT: Non-critical CSS in head without media or async loading
Pattern: <link rel="stylesheet" href="..." without media= attribute
(for stylesheets that are not critical-path)

# DETECT: Synchronous scripts in head
Pattern: <script src="..." without defer or async in <head>
```

```html
<!-- BAD: Render-blocking -->
<link rel="stylesheet" href="/non-critical.css">
<script src="/analytics.js"></script>

<!-- GOOD: Non-blocking -->
<link rel="stylesheet" href="/non-critical.css" media="print" onload="this.media='all'">
<script src="/analytics.js" defer></script>
```

**Issue 3: Unoptimized LCP resource discovery**
```
# DETECT: LCP image set via CSS background-image (late discovery)
Pattern: background-image:\s*url\( on hero/banner elements

# DETECT: LCP image loaded via JavaScript (late discovery)
Pattern: new Image\(\)|\.src\s*= on hero image initialization
```

```html
<!-- GOOD: Preload the LCP image for early discovery -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high">
```

### INP -- Detection of Code Issues

INP measures the full latency from user input through JavaScript processing to visual update.

**Issue 1: Long tasks blocking the main thread**
```
# DETECT: Synchronous heavy computation in event handlers
Pattern: addEventListener\(['"]click|addEventListener\(['"]touchstart
followed by heavy loops, DOM manipulation, or synchronous API calls

# DETECT: Large bundle without code splitting
Check: main.js bundle size > 200KB (compressed)
```

```javascript
// BAD: Long synchronous task in click handler
button.addEventListener('click', () => {
  // 500ms of synchronous work blocks INP
  processAllItems(largeArray);
  updateDOM();
});

// GOOD: Yield to main thread
button.addEventListener('click', async () => {
  updateDOM(); // Immediate visual feedback
  await scheduler.yield(); // Let browser process pending work
  processAllItems(largeArray); // Heavy work after yield
});
```

**Issue 2: Third-party scripts blocking interactions**
```
# DETECT: Synchronous third-party scripts
Pattern: <script src="https://(third-party-domain)"> without async or defer
```

### CLS -- Detection of Code Issues

**Issue 1: Images without dimensions**
```
# DETECT: img tags without width/height or aspect-ratio
Pattern: <img(?!.*(?:width|aspect-ratio))

# DETECT: CSS background images on elements without fixed height
Pattern: background-image on elements without explicit height/aspect-ratio
```

```html
<!-- BAD: No dimensions causes layout shift when image loads -->
<img src="photo.jpg" alt="Photo">

<!-- GOOD: Explicit dimensions or aspect-ratio -->
<img src="photo.jpg" alt="Photo" width="800" height="600">
<!-- OR -->
<img src="photo.jpg" alt="Photo" style="aspect-ratio: 4/3; width: 100%;">
```

**Issue 2: Dynamically injected content**
```
# DETECT: DOM insertions without reserved space
Pattern: insertBefore|appendChild|innerHTML\s*=|insertAdjacentHTML
In ad slots, banners, notification bars

# DETECT: Web font layout shift
Pattern: @font-face without font-display
Pattern: @font-face.*font-display:\s*block (block causes FOIT then shift)
```

**Issue 3: Animations triggering layout**
```
# DETECT: CSS transitions on layout properties
Pattern: transition.*(?:width|height|top|left|right|bottom|margin|padding)
Pattern: animation.*(?:width|height|top|left|right|bottom|margin|padding)
```

```css
/* BAD: Animating layout properties causes CLS */
.expanding { transition: height 0.3s; }

/* GOOD: Use transform instead */
.expanding { transition: transform 0.3s; }
```

---

## 5. Font Loading on Mobile

### The Problem

Mobile devices on cellular connections face:
- Higher latency for font file downloads (100-500ms per font)
- Slower CPU for font parsing and rendering
- FOIT (Flash of Invisible Text): text hidden until font loads (~3s default in Chrome)
- FOUT (Flash of Unstyled Text): fallback shown, then swapped (causes CLS)

### font-display Strategies

| Value | Behavior | Best For |
|-------|----------|----------|
| `swap` | Show fallback immediately, swap when ready | Body text, headings |
| `optional` | Use font only if loaded within ~100ms | Best CWV scores |
| `fallback` | Short invisible period (100ms), then fallback | Compromise |
| `block` | Invisible up to 3s, then fallback | Icon fonts only |
| `auto` | Browser decides (usually = block) | **Avoid** |

### Recommended Setup

```css
/* Critical fonts with swap */
@font-face {
  font-family: 'Brand Font';
  src: url('/fonts/brand.woff2') format('woff2');
  font-display: swap;
  font-weight: 400;
  font-style: normal;
}

/* For best CLS: optional (no layout shift at all) */
@font-face {
  font-family: 'Brand Font';
  src: url('/fonts/brand.woff2') format('woff2');
  font-display: optional;
}
```

### Preloading Critical Fonts

```html
<!-- Preload 1-2 critical fonts (saves 200-800ms) -->
<link rel="preload" href="/fonts/brand-regular.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/fonts/brand-bold.woff2" as="font" type="font/woff2" crossorigin>
```

**Rules:**
- Only preload **1-2** fonts (preloading too many negates the benefit)
- Always use `crossorigin` attribute (even for same-origin fonts)
- Always use `woff2` format (best compression, 98%+ browser support)

### Variable Fonts for Mobile

Variable fonts combine multiple weights/styles into a single file, reducing HTTP requests.

```css
@font-face {
  font-family: 'Brand Variable';
  src: url('/fonts/brand-variable.woff2') format('woff2-variations');
  font-display: swap;
  font-weight: 100 900; /* Full weight range in one file */
}
```

**Benefit:** One 50-80KB variable font file replaces 3-5 separate files (150-400KB total).

### System Font Stack as Fallback

```css
/* High-performance fallback that matches metrics across platforms */
body {
  font-family: 'Brand Font',
    -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
    'Helvetica Neue', Arial, 'Noto Sans', sans-serif;
}
```

### Detection Patterns

```
# Check: Missing font-display
Pattern: @font-face(?![\s\S]*?font-display)[\s\S]*?\}

# Check: font-display: block (causes FOIT)
Pattern: font-display:\s*block

# Check: font-display: auto (unpredictable)
Pattern: font-display:\s*auto

# Check: Too many preloaded fonts (>2)
Count: <link rel="preload".*as="font" -- warn if > 2

# Check: Non-woff2 font formats
Pattern: url\(.*\.(ttf|otf|eot|woff(?!2))

# Check: Missing crossorigin on font preload
Pattern: <link rel="preload".*as="font"(?!.*crossorigin)

# Check: Google Fonts loaded synchronously
Pattern: <link.*href="https://fonts.googleapis.com"
Without: <link rel="preconnect" href="https://fonts.gstatic.com"
```

### Fixes

```html
<!-- Optimal Google Fonts loading -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet"
      href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap"
      media="print" onload="this.media='all'">
<noscript>
  <link rel="stylesheet"
        href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap">
</noscript>
```

---

## 6. Third-Party Script Impact on Mobile

### Scale of the Problem

- Average website uses **21 different third-party scripts** on mobile
- Over **94%** of all websites use third-party resources
- **53% of mobile visits** abandoned if page takes longer than 3 seconds
- YouTube embeds block main thread for **4.5 seconds** on 10% of sites
- Analytics, chat widgets, and ad networks add **500-1500ms** to load times

### Common Offenders and Their Impact

| Script Type | Typical Size | Main Thread Block | CWV Impact |
|-------------|-------------|-------------------|------------|
| Google Analytics (GA4) | 30-50KB | 50-100ms | Minor |
| Google Tag Manager | 80-120KB | 100-300ms | INP, LCP |
| Facebook Pixel | 60-80KB | 100-200ms | INP |
| Intercom/Drift chat | 200-400KB | 300-800ms | LCP, INP |
| YouTube embed | 500KB+ | 1500-4500ms | LCP, CLS, INP |
| Social share buttons | 100-300KB | 200-500ms | CLS, LCP |
| Cookie consent (CMP) | 150-500KB | 200-500ms | LCP, INP, CLS |

### Detection Patterns

```
# Check: Synchronous third-party scripts in head
Pattern: <script\s+src="https?://(?!your-domain).*?"(?!\s*(async|defer))

# Check: Third-party scripts without async/defer
Pattern: <script\s+src="https?://(www\.googletagmanager|connect\.facebook|cdn\.segment|js\.intercom|widget\.crisp|embed\.tawk)"

# Check: YouTube iframe embeds (eager loading)
Pattern: <iframe.*src="https?://www\.youtube\.com/embed"(?!.*loading="lazy")

# Check: Social media embed scripts
Pattern: <script.*src=".*(platform\.twitter|connect\.facebook|apis\.google|linkedin\.com/in\.js)"

# Check: Multiple analytics scripts
Count: googletagmanager|google-analytics|segment\.com|mixpanel|amplitude|hotjar -- warn if > 2

# Check: Chat widget loaded eagerly
Pattern: <script.*src=".*(intercom|drift|crisp|tawk|livechat|zendesk)"
Without evidence of lazy/deferred loading
```

### Fixes

**1. Defer non-critical scripts:**
```html
<!-- BAD -->
<script src="https://www.googletagmanager.com/gtag/js?id=GA_ID"></script>

<!-- GOOD -->
<script async src="https://www.googletagmanager.com/gtag/js?id=GA_ID"></script>
```

**2. Lazy-load chat widgets on interaction:**
```javascript
// Load chat widget only when user shows intent to interact
let chatLoaded = false;
const loadChat = () => {
  if (chatLoaded) return;
  chatLoaded = true;
  const script = document.createElement('script');
  script.src = 'https://widget.intercom.io/widget/APP_ID';
  script.async = true;
  document.head.appendChild(script);
};

// Trigger on scroll past 50% or click on chat button
window.addEventListener('scroll', () => {
  if (window.scrollY > document.body.scrollHeight * 0.5) loadChat();
}, { once: true, passive: true });
document.querySelector('.chat-trigger')?.addEventListener('click', loadChat);
```

**3. Use facade pattern for YouTube embeds:**
```html
<!-- Lite YouTube embed (loads only on click) -->
<lite-youtube videoid="VIDEO_ID" playlabel="Play: Video Title">
  <a href="https://youtube.com/watch?v=VIDEO_ID" class="lty-playbtn">
    <span class="lyt-visually-hidden">Play</span>
  </a>
</lite-youtube>
<script src="/lite-youtube-embed.js" defer></script>
```

**4. Move third-party scripts to a Web Worker:**
```javascript
// Using Partytown to run third-party scripts off the main thread
// partytown.js runs analytics, tag managers, etc. in a web worker
<script type="text/partytown" src="https://www.googletagmanager.com/gtag/js?id=GA_ID"></script>
```

**5. Cookie Consent Banner Optimization:**
```html
<!-- BAD: Render-blocking CMP -->
<script src="https://cdn.cookielaw.org/consent.js"></script>

<!-- GOOD: Async with CSP support -->
<script async src="https://cdn.cookielaw.org/consent.js"></script>
```

Best practices for cookie consent:
- Load CMP script with `async` or `defer`
- Use `min-height` on banner container to prevent CLS
- Server-side consent state: render banner only for new visitors
- Dismissing the banner is often the first interaction -- optimize for INP

---

## 7. Mobile Dark Mode

### prefers-color-scheme Support

```css
/* Base (light) styles */
:root {
  --bg-color: #ffffff;
  --text-color: #1a1a1a;
  --surface-color: #f5f5f5;
}

/* Dark mode override */
@media (prefers-color-scheme: dark) {
  :root {
    --bg-color: #1a1a2e;
    --text-color: #e0e0e0;
    --surface-color: #2d2d44;
  }
}
```

### Meta Theme-Color for Dark Mode

```html
<!-- Dual theme-color for browser chrome -->
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#1a1a2e" media="(prefers-color-scheme: dark)">

<!-- Color-scheme meta tag (opts into UA dark defaults) -->
<meta name="color-scheme" content="light dark">
```

The `color-scheme` meta tag tells the browser to render UA-provided elements (scrollbars, form controls, CSS system colors) in the user's preferred color scheme.

### Common Dark Mode Issues on Mobile

**Issue 1: Image Contrast**
```css
/* BAD: Images with white backgrounds look glaring in dark mode */
/* No handling for dark mode images */

/* GOOD: Reduce brightness of images in dark mode */
@media (prefers-color-scheme: dark) {
  img:not([src*=".svg"]) {
    filter: brightness(0.9);
  }

  /* Invert diagrams/line art that have white backgrounds */
  img.diagram,
  img[src*="chart"],
  img[src*="diagram"] {
    filter: invert(1) hue-rotate(180deg);
  }
}
```

**Issue 2: SVG and Icon Colors**
```css
/* SVGs should use currentColor for dark mode compatibility */
@media (prefers-color-scheme: dark) {
  svg {
    fill: currentColor;
  }
}
```

**Issue 3: Hardcoded Colors**
```css
/* BAD: Hardcoded colors break in dark mode */
.card { background: white; color: black; }

/* GOOD: Use CSS custom properties */
.card {
  background: var(--surface-color);
  color: var(--text-color);
}
```

**Issue 4: Box Shadows Invisible in Dark Mode**
```css
/* Light mode shadows are invisible against dark backgrounds */
@media (prefers-color-scheme: dark) {
  .card {
    /* Replace shadow with border or lighter shadow */
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
    /* OR use a subtle border */
    border: 1px solid rgba(255, 255, 255, 0.1);
  }
}
```

**Issue 5: Contrast Ratios**
- WCAG requires 4.5:1 contrast for normal text, 3:1 for large text
- Colors that pass contrast in light mode may fail in dark mode
- Gray text (#666) on white (21:1) vs gray text (#999) on dark (#1a1a2e) may fail

### Detection Patterns

```
# Check: No dark mode support
Pattern: Absence of prefers-color-scheme in any CSS file

# Check: Missing meta theme-color for dark mode
Pattern: name="theme-color".*media=".*prefers-color-scheme: dark"
(should be present if dark mode is supported)

# Check: Missing color-scheme meta tag
Pattern: <head>.*</head> without <meta name="color-scheme"

# Check: Hardcoded color values (potential dark mode breakage)
Pattern: color:\s*#(?:000|fff|white|black)|background:\s*#(?:000|fff|white|black)
(without corresponding dark mode override)

# Check: Images without dark mode handling
Pattern: <img that doesn't have a dark mode filter or picture source

# Check: SVGs with hardcoded fill colors
Pattern: fill="#(?!current|none)" or fill="white|black|#fff|#000"
```

---

## 8. Responsive Design Patterns

### Modern Responsive Techniques (2024-2026)

**Mobile Traffic Reality:**
- 62.54% of website visits come from mobile (Statista Q4 2024)
- Mobile-first CSS is the standard approach

### Fluid Typography with clamp()

```css
/* BAD: Fixed font sizes with breakpoints */
h1 { font-size: 24px; }
@media (min-width: 768px) { h1 { font-size: 36px; } }
@media (min-width: 1200px) { h1 { font-size: 48px; } }

/* GOOD: Fluid typography scales smoothly */
h1 {
  /* min: 24px, preferred: 5vw, max: 48px */
  font-size: clamp(1.5rem, 1rem + 2.5vw, 3rem);
}

p {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
}
```

### Container Queries (93.92% Browser Support as of Dec 2025)

```css
/* Define a containment context */
.card-container {
  container-type: inline-size;
  container-name: card;
}

/* Respond to container size, not viewport */
@container card (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 1fr 2fr;
  }
}

@container card (max-width: 399px) {
  .card {
    display: flex;
    flex-direction: column;
  }
}
```

### Common Breakpoints (2025)

| Device Category | Width Range | Common Devices |
|----------------|-------------|----------------|
| Small Mobile | 320-480px | iPhone SE, Galaxy S mini |
| Mobile | 481-767px | iPhone 14, Pixel 7 |
| Tablet | 768-1023px | iPad, Galaxy Tab |
| Laptop | 1024-1366px | MacBook Air, Surface |
| Desktop | 1367px+ | External monitors |

**Tailwind CSS defaults:** sm:640px, md:768px, lg:1024px, xl:1280px, 2xl:1536px
**Bootstrap 5 defaults:** sm:576px, md:768px, lg:992px, xl:1200px, xxl:1400px

### Mobile-First vs Desktop-First CSS

```css
/* MOBILE-FIRST (Recommended): Base styles for mobile, enhance for larger */
.nav {
  flex-direction: column; /* Mobile: stacked */
}
@media (min-width: 768px) {
  .nav {
    flex-direction: row; /* Tablet+: horizontal */
  }
}

/* DESKTOP-FIRST: Base for desktop, override for smaller (not recommended) */
.nav {
  flex-direction: row; /* Desktop: horizontal */
}
@media (max-width: 767px) {
  .nav {
    flex-direction: column; /* Mobile: override to stacked */
  }
}
```

### Common Breakpoint Issues

**Issue 1: Content overflow between breakpoints**
```css
/* BAD: Gap between 480 and 768 where layout breaks */
@media (max-width: 480px) { /* small mobile */ }
@media (min-width: 768px) { /* tablet */ }
/* 481-767px range has no styles! */

/* GOOD: Use min-width progressive enhancement */
/* Base: mobile */
@media (min-width: 640px) { /* large mobile / small tablet */ }
@media (min-width: 768px) { /* tablet */ }
@media (min-width: 1024px) { /* laptop */ }
```

**Issue 2: Fixed-width elements on mobile**
```
# DETECT: Fixed pixel widths that may overflow mobile
Pattern: width:\s*\d{4,}px|min-width:\s*\d{4,}px
Pattern: width:\s*(5\d{2}|[6-9]\d{2})px (widths 500px+)
```

### Detection Patterns

```
# Check: No mobile styles (desktop-only site)
Pattern: Absence of @media.*max-width or @media.*min-width
AND absence of responsive framework (Tailwind, Bootstrap)

# Check: Desktop-first approach (max-width overrides)
Count: @media.*max-width vs @media.*min-width
If max-width count > min-width count, likely desktop-first

# Check: Missing responsive images
Pattern: <img(?!.*srcset)(?!.*sizes) (img without srcset or sizes)

# Check: Fixed-width containers that break on mobile
Pattern: width:\s*\d{3,}px(?!.*max-width|@media)

# Check: No container queries where beneficial
Pattern: Components with size-dependent layouts using only viewport queries

# Check: Horizontal scroll on mobile
Pattern: overflow-x:\s*visible|overflow:\s*visible on containers
With fixed-width children
```

---

## 9. Mobile-Specific Security

### HTTPS Requirements

As of 2024, HTTPS is required for:
- **PWA installability** (manifest requires secure context)
- **Service worker registration** (requires secure context)
- **Geolocation API** (requires secure context)
- **Camera/Microphone access** (requires secure context)
- **Web Push notifications** (requires secure context)
- **Google ranking signal** (HTTPS is a confirmed factor)

### Mixed Content Issues

Mixed content occurs when HTTPS pages load resources over HTTP.

**Types:**
- **Active mixed content** (blocked by browsers): Scripts, stylesheets, iframes, fetch/XHR
- **Passive mixed content** (may be auto-upgraded): Images, video, audio

**Modern browser behavior (2024):**
- Browsers automatically upgrade `http://` images to `https://`
- Active mixed content is blocked entirely
- `upgrade-insecure-requests` CSP directive handles most cases

### Content Security Policy for Mobile

```html
<!-- Recommended CSP header -->
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self' https://www.googletagmanager.com;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com;
  img-src 'self' data: https:;
  connect-src 'self' https://www.google-analytics.com;
  upgrade-insecure-requests;
">
```

**Key CSP directives for mobile:**
- `upgrade-insecure-requests` -- automatically upgrades HTTP to HTTPS (replaces deprecated `block-all-mixed-content`)
- `frame-ancestors 'self'` -- prevents clickjacking in mobile WebViews
- CSP adoption reached ~20% of sites in 2024-2025 (14% relative increase year-over-year)

### Cookie Consent on Mobile (UX Impact)

Cookie consent banners have significant mobile UX impact:
- **Performance**: Heavy CMPs add 200-500ms to mobile load time
- **CLS**: Banners without reserved space cause layout shift
- **INP**: Dismissing the banner is often the first interaction
- **Screen real estate**: Banners can cover 30-50% of mobile viewport

```css
/* Reserve space for cookie banner to prevent CLS */
.cookie-banner-placeholder {
  min-height: 120px; /* Reserve space before banner loads */
}

/* Ensure banner doesn't cover too much mobile viewport */
@media (max-width: 768px) {
  .cookie-banner {
    max-height: 40vh; /* Never cover more than 40% of screen */
    overflow-y: auto;
  }
}
```

### Detection Patterns

```
# Check: Mixed content (HTTP resources on HTTPS page)
Pattern: (src|href|action)="http://(?!localhost|127\.0\.0\.1)
Pattern: url\("?http://(?!localhost|127\.0\.0\.1)

# Check: Missing upgrade-insecure-requests
Pattern: Content-Security-Policy header without upgrade-insecure-requests

# Check: Missing HTTPS redirect
Pattern: HTTP server configuration without redirect to HTTPS

# Check: Insecure form actions
Pattern: <form.*action="http://

# Check: Cookie consent render-blocking
Pattern: <script src=".*cookie(consent|law|bot|yes).*"(?!\s*(async|defer))

# Check: Cookie banner without CLS prevention
Pattern: Cookie/consent banner element without min-height or fixed positioning
```

---

## 10. International/RTL Mobile Issues

### RTL Layout Fundamentals

RTL (Right-to-Left) affects Hebrew, Arabic, Persian, and Urdu content. On mobile, RTL issues are amplified due to narrow viewports and touch interactions.

### HTML Direction Setup

```html
<!-- Document-level direction -->
<html lang="he" dir="rtl">

<!-- Or per-element direction -->
<div dir="rtl">Hebrew content here</div>

<!-- For bidirectional content -->
<p dir="auto">This adapts based on first strong character</p>
```

### CSS Logical Properties (Full Browser Support 2024)

CSS logical properties automatically adapt to text direction. They replace physical properties:

| Physical Property | Logical Property |
|-------------------|-----------------|
| `margin-left` | `margin-inline-start` |
| `margin-right` | `margin-inline-end` |
| `padding-left` | `padding-inline-start` |
| `padding-right` | `padding-inline-end` |
| `border-left` | `border-inline-start` |
| `border-right` | `border-inline-end` |
| `text-align: left` | `text-align: start` |
| `text-align: right` | `text-align: end` |
| `float: left` | `float: inline-start` |
| `float: right` | `float: inline-end` |
| `left` (positioning) | `inset-inline-start` |
| `right` (positioning) | `inset-inline-end` |
| `width` | `inline-size` |
| `height` | `block-size` |

```css
/* BAD: Physical properties break in RTL */
.sidebar {
  margin-left: 20px;
  padding-right: 10px;
  border-left: 2px solid #333;
  text-align: left;
}

/* GOOD: Logical properties adapt automatically */
.sidebar {
  margin-inline-start: 20px;
  padding-inline-end: 10px;
  border-inline-start: 2px solid #333;
  text-align: start;
}
```

### Common RTL Bugs on Mobile

**Bug 1: letter-spacing on Arabic text**
```css
/* BAD: Disconnects Arabic connected letters */
.arabic-text {
  letter-spacing: 0.5px; /* Breaks Arabic ligatures! */
}

/* GOOD: Never use letter-spacing on Arabic */
[lang="ar"] * {
  letter-spacing: 0 !important;
}
```

**Bug 2: CSS transforms not mirrored**
```css
/* BAD: Arrow icon points wrong direction in RTL */
.arrow-icon {
  transform: translateX(10px);
}

/* GOOD: Use logical direction or mirror in RTL */
[dir="rtl"] .arrow-icon {
  transform: translateX(-10px) scaleX(-1);
}
```

**Bug 3: Flexbox direction not reversed**
```css
/* Flexbox auto-reverses with dir="rtl" */
/* But explicit flex-direction: row does NOT reverse */
/* This is the expected behavior -- flex follows document direction */

/* BAD: Overriding flex direction for RTL manually */
[dir="rtl"] .flex-row { flex-direction: row-reverse; }

/* GOOD: Let flexbox handle it naturally via dir="rtl" on parent */
.flex-row { display: flex; /* direction follows document */ }
```

**Bug 4: Horizontal scroll position**
```javascript
// Scroll position behavior differs in RTL across browsers!
// In RTL mode:
// - Chrome: scrollLeft is 0 at right edge, negative going left
// - Firefox: scrollLeft is 0 at right edge, negative going left
// - Safari (older): scrollLeft is positive from left edge

// GOOD: Use scrollIntoView instead of manual scrollLeft
element.scrollIntoView({ behavior: 'smooth', inline: 'start' });
```

**Bug 5: Font sizing for Hebrew/Arabic**
```css
/* Arabic/Hebrew text needs larger sizes than Latin for same readability */
[lang="he"], [lang="ar"] {
  font-size: 1.1em; /* 10% larger than base */
  line-height: 1.6; /* More line height for diacritics */
}
```

**Bug 6: Bidirectional text with numbers and URLs**
```html
<!-- BAD: Numbers and URLs may render in wrong position -->
<p dir="rtl">Visit example.com for more info about 2024</p>

<!-- GOOD: Isolate embedded LTR content -->
<p dir="rtl">Visit <bdi dir="ltr">example.com</bdi> for more info about <bdi dir="ltr">2024</bdi></p>
```

**Bug 7: The destructive bidi-override**
```css
/* CATASTROPHIC: This completely breaks ALL RTL text */
* {
  direction: ltr;
  unicode-bidi: bidi-override;
}
/* NEVER DO THIS */
```

### Detection Patterns

```
# Check: Physical properties instead of logical (RTL-unfriendly)
Pattern: margin-left|margin-right|padding-left|padding-right
Pattern: border-left|border-right
Pattern: text-align:\s*(left|right)
Pattern: float:\s*(left|right)
(Should use logical equivalents)

# Check: letter-spacing on Arabic/Hebrew text
Pattern: letter-spacing on elements with lang="ar" or lang="he" or dir="rtl"

# Check: Missing dir attribute on html element
Pattern: <html(?!.*dir=)

# Check: Missing lang attribute on html element
Pattern: <html(?!.*lang=)

# Check: Hardcoded LTR transforms
Pattern: translateX\(\d on elements that may be in RTL context

# Check: unicode-bidi: bidi-override (destructive)
Pattern: unicode-bidi:\s*bidi-override

# Check: Mixed bidirectional content without bdi/bdo
Pattern: dir="rtl".*[a-zA-Z].*[0-9] without <bdi or unicode-bidi: isolate

# Check: direction: ltr forced globally
Pattern: \*\s*\{[^}]*direction:\s*ltr
```

### RTL Testing Checklist for Mobile

1. Navigation: Does hamburger menu open from correct side?
2. Swipe gestures: Are swipe directions mirrored?
3. Progress bars: Do they fill from right to left?
4. Carousels/sliders: Do they scroll in correct direction?
5. Form fields: Are labels and inputs aligned correctly?
6. Icons: Are directional icons (arrows, back buttons) mirrored?
7. Numbers in text: Do they read correctly in mixed content?
8. Phone numbers: Are they displayed LTR within RTL text?
9. Horizontal scroll: Does the scrollbar behave correctly?
10. Keyboard: Does the mobile keyboard switch to appropriate layout?

---

## Appendix A: Complete Mobile Audit Checklist

### Phase 1: Viewport and Layout Stability
- [ ] Viewport meta tag present and accessible (no zoom blocking)
- [ ] No `100dvh` on hero sections (use `100svh` for stability)
- [ ] Safe area insets on fixed/sticky elements
- [ ] No horizontal overflow on any element

### Phase 2: SEO
- [ ] Mobile content parity with desktop
- [ ] Structured data (JSON-LD) present on mobile-served HTML
- [ ] Mobile-friendly test passing
- [ ] No `noindex` on mobile pages
- [ ] Canonical tags consistent between mobile and desktop

### Phase 3: Meta Tags
- [ ] Viewport meta tag (without zoom restrictions)
- [ ] Theme-color meta tag (light and dark)
- [ ] Color-scheme meta tag
- [ ] OG tags with proper image dimensions (1200x630)
- [ ] Twitter Card tags
- [ ] Web app manifest linked
- [ ] Description meta tag present

### Phase 4: PWA
- [ ] Web app manifest with all required fields
- [ ] Icons: 192x192 and 512x512 (both "any" and "maskable")
- [ ] Service worker registered
- [ ] Offline fallback page
- [ ] HTTPS enforced
- [ ] start_url defined and valid

### Phase 5: Performance (Core Web Vitals)
- [ ] LCP element not lazy-loaded
- [ ] LCP image preloaded
- [ ] No render-blocking CSS/JS for above-fold content
- [ ] Images have explicit dimensions (width/height or aspect-ratio)
- [ ] No animations on layout properties (use transform/opacity)
- [ ] No excessive `will-change` usage
- [ ] Long tasks broken up (< 50ms chunks)

### Phase 6: Font Loading
- [ ] `font-display: swap` or `optional` on all @font-face
- [ ] Critical fonts preloaded (max 2)
- [ ] Using woff2 format
- [ ] `crossorigin` on font preloads
- [ ] System font fallback stack defined

### Phase 7: Third-Party Scripts
- [ ] All third-party scripts are `async` or `defer`
- [ ] Chat widgets lazy-loaded on interaction
- [ ] YouTube/video embeds use lite/facade pattern
- [ ] Cookie consent banner loaded async
- [ ] No more than 2-3 analytics scripts

### Phase 8: Dark Mode
- [ ] `prefers-color-scheme` media query implemented
- [ ] Dual `theme-color` meta tags (light/dark)
- [ ] `color-scheme` meta tag present
- [ ] Images handled in dark mode (brightness/invert)
- [ ] No hardcoded black/white colors
- [ ] Contrast ratios pass in both modes
- [ ] SVGs use `currentColor`

### Phase 9: Responsive Design
- [ ] Mobile-first CSS approach
- [ ] Fluid typography with `clamp()`
- [ ] Container queries for component-level responsiveness
- [ ] Responsive images with `srcset` and `sizes`
- [ ] No fixed-width elements exceeding 320px
- [ ] Breakpoints cover all device ranges (no gaps)

### Phase 10: Security
- [ ] HTTPS enforced site-wide
- [ ] No mixed content (HTTP resources on HTTPS pages)
- [ ] CSP header with `upgrade-insecure-requests`
- [ ] Cookie consent banner doesn't block rendering
- [ ] Secure form actions (HTTPS)

### Phase 11: Touch and Interaction
- [ ] Touch targets minimum 48x48dp (recommended) or 24x24px (WCAG 2.2 AA)
- [ ] 8px spacing between adjacent touch targets
- [ ] Form inputs have font-size >= 16px (prevents iOS zoom)
- [ ] No hover-only interactions (must work with touch)

### Phase 12: RTL/International
- [ ] `dir` attribute on `<html>` element
- [ ] `lang` attribute on `<html>` element
- [ ] CSS logical properties used (not physical)
- [ ] No `letter-spacing` on Arabic text
- [ ] No global `direction: ltr; unicode-bidi: bidi-override`
- [ ] Bidirectional content uses `<bdi>` elements
- [ ] Font sizes adjusted for non-Latin scripts
- [ ] Tested on real mobile devices in RTL mode

---

## Appendix B: Source References

### Mobile SEO
- [Google Mobile-First Indexing Best Practices](https://developers.google.com/search/docs/crawling-indexing/mobile/mobile-sites-mobile-first-indexing)
- [Google Mobile-First Indexing 2025 Guide](https://zaphyrpro.com/googles-mobile-first-indexing-seo-impact)
- [Mobile-First Indexing July 2024 Update](https://www.whitelabeliq.com/blog/googles-mobile-first-indexing-checklist-july-2024-update/)

### Meta Tags
- [Apple Supported Meta Tags](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariHTMLRef/Articles/MetaTags.html)
- [apple-mobile-web-app-capable Deprecation](https://github.com/vercel/next.js/issues/70272)
- [Meta Theme-Color MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name/theme-color)
- [Color-Scheme Meta MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name/color-scheme)

### PWA
- [Chrome Installable Manifest Requirements](https://developer.chrome.com/docs/lighthouse/pwa/installable-manifest)
- [MDN Making PWAs Installable](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable)
- [web.dev Install Criteria](https://web.dev/articles/install-criteria)
- [PWA Manifest Cheat Sheet 2024](https://www.zeepalm.com/blog/pwa-manifest-cheat-sheet-2024)
- [MDN Define App Icons](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/How_to/Define_app_icons)
- [PWA Icon Requirements 2025 Checklist](https://dev.to/albert_nahas_cdc8469a6ae8/pwa-icon-requirements-the-complete-2025-checklist-i3g)

### Core Web Vitals
- [Google Core Web Vitals Documentation](https://developers.google.com/search/docs/appearance/core-web-vitals)
- [web.dev Web Vitals](https://web.dev/articles/vitals)
- [Core Web Vitals 2026](https://www.corewebvitals.io/core-web-vitals)
- [web.dev Optimize LCP](https://web.dev/articles/optimize-lcp)
- [web.dev INP](https://web.dev/articles/inp)
- [web.dev CLS](https://web.dev/articles/cls)

### Font Loading
- [OneNine Font Loading Optimization Guide](https://onenine.com/ultimate-guide-to-font-loading-optimization/)
- [Font Loading Strategies 2025](https://font-converters.com/guides/font-loading-strategies)
- [Chrome Lighthouse font-display](https://developer.chrome.com/docs/lighthouse/performance/font-display)
- [NitroPack Font Optimization](https://nitropack.io/blog/post/font-loading-optimization)

### Third-Party Scripts
- [web.dev Third-Party JavaScript](https://web.dev/articles/optimizing-content-efficiency-loading-third-party-javascript)
- [Chrome Blog Third-Party Scripts](https://developer.chrome.com/blog/third-party-scripts)
- [GTmetrix Reduce Third-Party Code](https://gtmetrix.com/reduce-the-impact-of-third-party-code.html)

### Dark Mode
- [web.dev Color Scheme](https://web.dev/articles/color-scheme)
- [MDN prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-color-scheme)
- [CSS-Tricks Dark Mode Guide](https://css-tricks.com/a-complete-guide-to-dark-mode-on-the-web/)
- [CSS-Tricks Meta Theme Color](https://css-tricks.com/meta-theme-color-and-trickery/)

### Responsive Design
- [Responsive Web Design Techniques 2026](https://lovable.dev/guides/responsive-web-design-techniques-that-work)
- [CSS Breakpoints for Responsive Design](https://blog.logrocket.com/css-breakpoints-responsive-design/)
- [BrowserStack Responsive Design Breakpoints 2025](https://www.browserstack.com/guide/responsive-design-breakpoints)

### Security
- [Web Almanac Security 2024](https://almanac.httparchive.org/en/2024/security)
- [MDN Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)
- [MDN Mixed Content](https://developer.mozilla.org/en-US/docs/Web/Security/Mixed_content)
- [SpeedCurve Cookie Consent Performance](https://www.speedcurve.com/blog/web-performance-cookie-consent/)
- [DebugBear Cookie Consent Performance](https://www.debugbear.com/blog/cookie-consent-banner-performance)

### Viewport Accessibility
- [W3C Viewport Zoom Rule](https://www.w3.org/WAI/standards-guidelines/act/rules/b4f0c3/)
- [MDN Viewport Meta Tag](https://developer.mozilla.org/en-US/docs/Web/HTML/Viewport_meta_tag)
- [lukeplant.me.uk Stop Using user-scalable=no](https://lukeplant.me.uk/blog/posts/you-can-stop-using-user-scalable-no-and-maximum-scale-1-in-viewport-meta-tags-now/)

### Touch Targets
- [web.dev Accessible Tap Targets](https://web.dev/articles/accessible-tap-targets)
- [WCAG 2.5.8 Target Size Minimum](https://testparty.ai/blog/wcag-target-size-guide)
- [Smashing Magazine Tap Target Sizes](https://www.smashingmagazine.com/2023/04/accessible-tap-target-sizes-rage-taps-clicks/)

### RTL / International
- [RTL Styling 101](https://rtlstyling.com/posts/rtl-styling/)
- [W3C Structural Markup for RTL](https://www.w3.org/International/questions/qa-html-dir)
- [Material Design Bidirectionality](https://m2.material.io/design/usability/bidirectionality.html)
- [CSS Logical Properties MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Logical_properties_and_values/Margins_borders_padding)
- [RTL in Web Platform (CSS Logical Properties)](https://dev.to/pffigueiredo/css-logical-properties-rtl-in-a-web-platform-2-6-5hin)

### Open Graph / Social
- [OG Image Dimensions Guide 2024](https://www.ogimage.gallery/libary/the-ultimate-guide-to-og-image-dimensions-2024-update)
- [EverywhereMarketer Social Meta Tags Guide](https://www.everywheremarketer.com/blog/ultimate-guide-to-social-meta-tags-open-graph-and-twitter-cards)

### Service Workers
- [Chrome Workbox Caching Strategies](https://developer.chrome.com/docs/workbox/caching-strategies-overview/)
- [Chrome Workbox Fallback Responses](https://developer.chrome.com/docs/workbox/managing-fallback-responses)
- [Service Worker Caching Strategies](https://www.zeepalm.com/blog/service-worker-caching-5-offline-fallback-strategies)
