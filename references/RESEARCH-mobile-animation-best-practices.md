# Mobile Web Animation Best Practices: Comprehensive Research Report
## Detection Patterns, Common Bugs, and Programmatic Fixes (2024-2026)

---

## Table of Contents
1. [Mobile Animation Performance Fundamentals](#1-mobile-animation-performance-fundamentals)
2. [Scroll-Driven Animations on Mobile](#2-scroll-driven-animations-on-mobile)
3. [GSAP on Mobile](#3-gsap-on-mobile)
4. [Framer Motion on Mobile](#4-framer-motion-on-mobile)
5. [Lenis / Locomotive Scroll on Mobile](#5-lenis--locomotive-scroll-on-mobile)
6. [View Transitions API on Mobile](#6-view-transitions-api-on-mobile)
7. [Reduced Motion Preferences](#7-reduced-motion-preferences)
8. [Mobile-Specific Animation Bugs](#8-mobile-specific-animation-bugs)
9. [Programmatic Detection of Animation Issues](#9-programmatic-detection-of-animation-issues)
10. [Visual Verification via Screenshots](#10-visual-verification-via-screenshots)

---

## 1. Mobile Animation Performance Fundamentals

### The Rendering Pipeline

Every frame the browser renders follows this pipeline:

```
JavaScript -> Style -> Layout -> Paint -> Composite
```

Each step triggers everything after it. If you animate a property that triggers **Layout**, you also trigger **Paint** and **Composite**. The goal on mobile is to animate only properties that skip Layout and Paint entirely.

### GPU Compositing: The Key to 60fps on Mobile

Only two CSS properties can be animated purely on the GPU compositor thread, skipping both layout and paint:

- **`transform`** (translate, scale, rotate, skew)
- **`opacity`**

Everything else forces work on the main thread and will cause jank on low-end mobile devices.

**Properties that trigger Layout (most expensive):**
```
width, height, padding, margin, top, right, bottom, left,
display, position, float, overflow, border-width,
font-size, font-weight, line-height, text-align,
min-height, max-width, flex, align-items, align-content
```

**Properties that trigger Paint only (medium cost):**
```
color, background-color, background-image, border-color,
border-style, border-radius, box-shadow, outline,
text-decoration, visibility
```

**Properties that trigger Composite only (cheapest):**
```
transform, opacity, filter (when on own layer), will-change
```

### Detection Pattern: Finding layout-triggering animations in code

```bash
# Grep patterns to find CSS animations on layout-triggering properties
# In CSS files:
grep -rn "animation.*\(width\|height\|top\|left\|right\|bottom\|margin\|padding\)" --include="*.css"
grep -rn "transition.*\(width\|height\|top\|left\|right\|bottom\|margin\|padding\)" --include="*.css"

# In JS files (inline styles):
grep -rn "\.style\.\(width\|height\|top\|left\|margin\|padding\)" --include="*.js" --include="*.ts" --include="*.tsx"

# GSAP animations on layout properties:
grep -rn "gsap\.\(to\|from\|fromTo\|set\).*\(width\|height\|top\|left\|margin\|padding\)" --include="*.js" --include="*.ts" --include="*.tsx"
```

### The FLIP Technique

FLIP (First, Last, Invert, Play) converts expensive layout animations into cheap transform animations:

```javascript
// FLIP Implementation
function flipAnimate(element, applyChange) {
  // FIRST: Record current position
  const first = element.getBoundingClientRect();

  // Apply the DOM change that causes layout shift
  applyChange();

  // LAST: Record new position
  const last = element.getBoundingClientRect();

  // INVERT: Calculate the delta and apply inverse transform
  const deltaX = first.left - last.left;
  const deltaY = first.top - last.top;
  const deltaW = first.width / last.width;
  const deltaH = first.height / last.height;

  element.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${deltaW}, ${deltaH})`;
  element.style.transformOrigin = 'top left';

  // PLAY: Animate back to final position
  requestAnimationFrame(() => {
    element.style.transition = 'transform 300ms ease';
    element.style.transform = 'none';

    element.addEventListener('transitionend', () => {
      element.style.transition = '';
      element.style.transformOrigin = '';
    }, { once: true });
  });
}
```

**Mobile performance tip:** When FLIPing many elements, batch all `getBoundingClientRect()` reads before any writes to avoid layout thrashing.

### CSS vs JS Animations: When to Use Each on Mobile

| Scenario | Recommended | Why |
|----------|-------------|-----|
| Simple hover/focus states | CSS Transitions | No JS overhead, browser-optimized |
| Scroll-linked animations | CSS `animation-timeline` or WAAPI | Runs off main thread |
| Complex orchestrated sequences | GSAP or Motion | Better control, timeline management |
| Layout animations (list reorder) | FLIP + transform | Avoids layout-triggering properties |
| Gesture-driven animations | Motion (Framer) or GSAP | Need JS for touch input processing |
| Infinite/looping decorative | CSS `@keyframes` | Zero JS, fully compositor-driven |

### requestAnimationFrame Best Practices

```javascript
// GOOD: Use rAF for visual updates
let ticking = false;
function onScroll() {
  if (!ticking) {
    requestAnimationFrame(() => {
      updateAnimation();
      ticking = false;
    });
    ticking = true;
  }
}

// BAD: Never use setTimeout/setInterval for animations
// setInterval(() => { element.style.transform = ...; }, 16); // DON'T

// GOOD: Cancel animations when not visible
let rafId;
function startAnimation() {
  rafId = requestAnimationFrame(animate);
}
function stopAnimation() {
  cancelAnimationFrame(rafId);
}

// Use IntersectionObserver to start/stop animations based on visibility
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      startAnimation();
    } else {
      stopAnimation();
    }
  });
});
```

### Web Animations API (WAAPI) on Mobile

```javascript
// WAAPI runs off the main thread for transform/opacity
element.animate([
  { transform: 'translateX(0)', opacity: 1 },
  { transform: 'translateX(100px)', opacity: 0 }
], {
  duration: 300,
  easing: 'ease-out',
  fill: 'forwards'
});

// WAAPI + ScrollTimeline (Chrome 115+)
element.animate(
  { transform: ['translateY(0)', 'translateY(-100px)'] },
  {
    timeline: new ScrollTimeline({ source: document.scrollingElement }),
    rangeStart: '0%',
    rangeEnd: '100%'
  }
);
```

### `will-change` Usage (and Misuse)

```css
/* GOOD: Apply just before animation starts */
.element:hover {
  will-change: transform;
}
.element.animating {
  will-change: transform, opacity;
}

/* BAD: Never apply globally or to many elements */
/* This creates too many compositor layers and INCREASES memory usage */
* {
  will-change: transform; /* DON'T DO THIS - especially on mobile */
}
```

**Detection pattern for will-change misuse:**
```bash
# Find overly broad will-change usage
grep -rn "will-change" --include="*.css" --include="*.scss" --include="*.less"
# Flag if applied to universal selectors, body, or many elements
grep -rn "\*.*will-change\|body.*will-change" --include="*.css"
```

---

## 2. Scroll-Driven Animations on Mobile

### CSS Scroll-Driven Animations API

The new CSS scroll-driven animations specification (Chrome 115+, Safari 18+) enables scroll-linked animations without JavaScript:

```css
/* Scroll Progress Timeline */
@keyframes reveal {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}

.element {
  animation: reveal linear both;
  animation-timeline: scroll();       /* Linked to nearest scroll ancestor */
  animation-range: entry 0% entry 100%; /* Animate as element enters viewport */
}

/* View Progress Timeline */
.element {
  animation: reveal linear both;
  animation-timeline: view();          /* Linked to element's visibility */
  animation-range: entry 25% cover 50%;
}

/* Named Scroll Timeline */
.scroll-container {
  scroll-timeline-name: --my-scroller;
  scroll-timeline-axis: block;
}
.animated-child {
  animation-timeline: --my-scroller;
}
```

### Browser Support on Mobile (as of early 2026)

| Feature | Chrome Android | Safari iOS | Firefox Android | Samsung Internet |
|---------|---------------|------------|-----------------|-----------------|
| `animation-timeline: scroll()` | 115+ | 18+ (partial) | Not yet | 23+ |
| `animation-timeline: view()` | 115+ | 18+ (partial) | Not yet | 23+ |
| `scroll-timeline-name` | 115+ | 18+ | Not yet | 23+ |
| `view-timeline-name` | 115+ | 18+ | Not yet | 23+ |

### Progressive Enhancement Pattern

```css
/* Base: No animation (works everywhere) */
.element {
  opacity: 1;
  transform: none;
}

/* Enhanced: Only if scroll-driven animations are supported */
@supports (animation-timeline: scroll()) {
  .element {
    opacity: 0;
    transform: translateY(30px);
    animation: fadeInUp linear both;
    animation-timeline: view();
    animation-range: entry 10% cover 40%;
  }

  @keyframes fadeInUp {
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
}
```

### JavaScript Fallback with IntersectionObserver

```javascript
// Feature detection
if (!CSS.supports('animation-timeline', 'scroll()')) {
  // Fallback: IntersectionObserver for scroll-triggered animations
  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        entry.target.classList.add('is-visible');
        observer.unobserve(entry.target); // One-shot animation
      }
    });
  }, {
    threshold: 0.1,
    rootMargin: '0px 0px -50px 0px'
  });

  document.querySelectorAll('.element').forEach(el => observer.observe(el));
}
```

### IntersectionObserver vs Scroll Events on Mobile

**Performance comparison (simulated 6x CPU slowdown / older mobile):**
- IntersectionObserver: Main thread 43% more available for user input
- Throttled scroll listener (16ms): Moderate main thread blocking
- Un-throttled scroll listener: Severe jank, dropped frames

```javascript
// BAD: Scroll event listener (blocks main thread on mobile)
window.addEventListener('scroll', () => {
  const rect = element.getBoundingClientRect(); // Forces layout!
  if (rect.top < window.innerHeight) {
    element.classList.add('visible');
  }
});

// GOOD: IntersectionObserver (async, off main thread)
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      entry.target.classList.toggle('visible', entry.isIntersecting);
    });
  },
  { threshold: [0, 0.25, 0.5, 0.75, 1] }
);
```

### Polyfill for Scroll-Driven Animations

```html
<!-- scroll-timeline polyfill for browsers without native support -->
<script src="https://flackr.github.io/scroll-timeline/dist/scroll-timeline.js"></script>
```

**Important caveat:** The polyfill runs on the main thread in JavaScript, so you lose the performance benefit of native scroll-driven animations. On mobile, this means the polyfilled version will not be as smooth as native.

### Common Pitfalls on Mobile

1. **Scroll jank when combining with touch-action:** Ensure `touch-action: pan-y` is set on scrollable containers
2. **Overuse of scroll timelines:** Each timeline consumes resources; limit to visible elements
3. **Missing `animation-range`:** Without specifying a range, the animation spans the entire scroll distance, which may be undesirable
4. **iOS Safari partial support:** Test thoroughly; some edge cases may not work

### Detection Patterns

```bash
# Find scroll-driven animation usage without feature detection
grep -rn "animation-timeline" --include="*.css" --include="*.scss"
# Check if @supports is used alongside
grep -rn "@supports.*animation-timeline" --include="*.css" --include="*.scss"

# Find scroll event listeners that could be replaced
grep -rn "addEventListener.*scroll" --include="*.js" --include="*.ts" --include="*.tsx"
grep -rn "onScroll\|onscroll" --include="*.js" --include="*.ts" --include="*.tsx" --include="*.jsx"

# Find getBoundingClientRect inside scroll handlers (layout thrashing)
grep -rn "getBoundingClientRect" --include="*.js" --include="*.ts" --include="*.tsx"
```

---

## 3. GSAP on Mobile

### Common GSAP ScrollTrigger Issues on Mobile

#### Pin Spacing Problems

When GSAP pins an element, it wraps it in a `pin-spacer` div and applies `position: fixed`. This frequently causes issues on mobile:

```javascript
// PROBLEM: Pin spacer height mismatch on mobile
ScrollTrigger.create({
  trigger: '.section',
  pin: true,
  start: 'top top',
  end: '+=500',
  // On mobile, address bar show/hide changes viewport height
  // causing pin spacing to be incorrect
});

// FIX: Use invalidateOnRefresh and handle resize
ScrollTrigger.create({
  trigger: '.section',
  pin: true,
  start: 'top top',
  end: '+=500',
  invalidateOnRefresh: true,
  // Recalculate on address bar changes
});

// Additional fix: Refresh on resize/orientationchange
ScrollTrigger.config({
  autoRefreshEvents: 'visibilitychange,DOMContentLoaded,load,resize'
});

// For mobile address bar changes specifically:
let lastHeight = window.innerHeight;
window.addEventListener('resize', () => {
  if (Math.abs(window.innerHeight - lastHeight) > 100) {
    // Likely address bar toggle, not keyboard
    ScrollTrigger.refresh();
    lastHeight = window.innerHeight;
  }
});
```

#### Horizontal Scroll Sections on Mobile

```javascript
// Common horizontal scroll setup
const sections = gsap.utils.toArray('.horizontal-panel');
const scrollTween = gsap.to(sections, {
  xPercent: -100 * (sections.length - 1),
  ease: 'none',
  scrollTrigger: {
    trigger: '.horizontal-container',
    pin: true,
    scrub: 1,
    end: () => '+=' + document.querySelector('.horizontal-container').offsetWidth,
    invalidateOnRefresh: true,
    // IMPORTANT: Snap doesn't work with containerAnimation
    // snap: 1 / (sections.length - 1), // This WON'T work inside containerAnimation
  }
});

// MOBILE FIX: Disable horizontal scroll on narrow screens
if (window.matchMedia('(max-width: 768px)').matches) {
  // Convert to vertical scroll on mobile
  // Or adjust the end value for mobile viewport
}
```

#### Transform on Pinned Container Breaks `position: fixed`

```javascript
// PROBLEM: If a parent has a CSS transform, position:fixed won't work
// This causes pin jittering on mobile

// FIX: Ensure no parent of pinned element has transform
// Or use pinType: 'transform' instead
ScrollTrigger.create({
  trigger: '.section',
  pin: true,
  pinType: 'transform', // Uses transform instead of position:fixed
});
```

### GSAP Performance Optimization for Mobile

```javascript
// 1. Use GPU-friendly properties only
gsap.to('.element', {
  x: 100,        // GOOD: maps to translateX
  y: 50,         // GOOD: maps to translateY
  rotation: 45,  // GOOD: maps to rotate
  scale: 1.2,    // GOOD: maps to scale
  opacity: 0.5,  // GOOD: compositor-only
  // AVOID on mobile:
  // width: 200,   // BAD: triggers layout
  // top: 100,     // BAD: triggers layout
  // margin: 20,   // BAD: triggers layout
});

// 2. Lag smoothing for mobile
gsap.ticker.lagSmoothing(1000, 16);

// 3. Batch ScrollTrigger animations
ScrollTrigger.batch('.reveal-element', {
  onEnter: (elements) => {
    gsap.to(elements, {
      opacity: 1,
      y: 0,
      stagger: 0.1,
      overwrite: true
    });
  },
  once: true // Don't re-trigger
});

// 4. Use stagger instead of individual tweens
// BAD: Creates many tween instances
document.querySelectorAll('.item').forEach(item => {
  gsap.to(item, { opacity: 1, y: 0 });
});

// GOOD: Single tween with stagger
gsap.to('.item', {
  opacity: 1,
  y: 0,
  stagger: 0.05
});

// 5. Avoid fromTo when possible
// BAD: Forces extra initial calculation
gsap.fromTo('.el', { x: 0 }, { x: 100 });

// BETTER: Set initial state in CSS, use .to()
gsap.to('.el', { x: 100 });

// 6. Kill unused ScrollTriggers
// Especially important on mobile SPA navigation
ScrollTrigger.getAll().forEach(st => st.kill());

// 7. Use matchMedia for responsive animations
const mm = gsap.matchMedia();
mm.add('(max-width: 768px)', () => {
  // Mobile-specific animations (simpler, less elements)
  gsap.to('.hero-text', { y: -20, opacity: 1 });
  return () => {
    // Cleanup when breakpoint changes
  };
});
mm.add('(min-width: 769px)', () => {
  // Desktop animations (can be more complex)
  gsap.to('.hero-text', { y: -50, opacity: 1, scale: 1.1 });
});
```

### GSAP + React Cleanup (Critical for Mobile)

```javascript
// In React: ALWAYS clean up GSAP in useEffect
import { useEffect, useRef } from 'react';
import { gsap } from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';

function AnimatedSection() {
  const containerRef = useRef(null);

  useEffect(() => {
    const ctx = gsap.context(() => {
      gsap.to('.box', {
        x: 100,
        scrollTrigger: {
          trigger: '.box',
          start: 'top center',
        }
      });
    }, containerRef);

    return () => {
      ctx.revert(); // Cleans up all GSAP animations and ScrollTriggers
    };
  }, []);

  // If animations misbehave on mount (React StrictMode):
  useEffect(() => {
    const timeout = setTimeout(() => {
      ScrollTrigger.refresh();
    }, 100);
    return () => clearTimeout(timeout);
  }, []);

  return <div ref={containerRef}>...</div>;
}
```

### Detection Patterns for GSAP Issues

```bash
# Find GSAP animations on layout-triggering properties
grep -rn "gsap\.\(to\|from\|fromTo\).*width\|height\|top\|left\|margin\|padding" --include="*.js" --include="*.ts" --include="*.tsx"

# Find ScrollTrigger without cleanup in React
grep -rn "ScrollTrigger" --include="*.tsx" --include="*.jsx"
# Cross-reference with presence of ctx.revert() or ScrollTrigger.kill()

# Find pin without invalidateOnRefresh
grep -rn "pin:\s*true" --include="*.js" --include="*.ts" --include="*.tsx"

# Find horizontal scroll setups (potential mobile issues)
grep -rn "xPercent.*-100\|horizontal" --include="*.js" --include="*.ts"

# Find missing gsap.context in React components
grep -rn "gsap\.\(to\|from\|timeline\)" --include="*.tsx" --include="*.jsx"
# Should be wrapped in gsap.context()
```

---

## 4. Framer Motion (Motion) on Mobile

### Name Change: Framer Motion -> Motion (2025)

In 2025, Framer Motion was rebranded to **Motion** and published as a standalone package:
```bash
# Old:
npm install framer-motion
import { motion } from 'framer-motion';

# New (2025+):
npm install motion
import { motion } from 'motion/react';
```

### Common Mobile Issues

#### Layout Animations Causing Reflow

```tsx
// PROBLEM: layout prop on too many elements causes excessive reflow
<motion.div layout>        {/* Animates position AND size changes */}
  <motion.div layout>      {/* Every child also recalculates */}
    <motion.div layout>    {/* Cascading reflow on mobile */}
      <p>Content</p>
    </motion.div>
  </motion.div>
</motion.div>

// FIX: Only apply layout to elements that actually need it
<div>                               {/* Static wrapper */}
  <motion.div layout>               {/* Only this moves */}
    <div>                            {/* Static children */}
      <p>Content</p>
    </div>
  </motion.div>
</div>

// FIX 2: Use layout="position" to only animate position, not size
<motion.div layout="position">
  {/* Cheaper: skips size interpolation */}
</motion.div>
```

#### Scrollable Container Issues

```tsx
// PROBLEM: Layout animations break inside scrollable containers
<div style={{ overflow: 'auto', height: '400px' }}>
  <AnimatePresence>
    {items.map(item => (
      <motion.div
        key={item.id}
        layout              // Animation offset is wrong after scrolling
        exit={{ opacity: 0 }}
      />
    ))}
  </AnimatePresence>
</div>

// FIX: Add layoutScroll to the scrollable container
<motion.div
  layoutScroll             // Tells Motion to measure scroll offset
  style={{ overflow: 'auto', height: '400px' }}
>
  <AnimatePresence>
    {items.map(item => (
      <motion.div
        key={item.id}
        layout
        exit={{ opacity: 0 }}
      />
    ))}
  </AnimatePresence>
</motion.div>
```

#### Gesture Handling Conflicts on Mobile

```tsx
// PROBLEM: Drag conflicts with native scroll on mobile
<motion.div
  drag="x"
  // On mobile, this fights with horizontal scroll
/>

// FIX: Use touch-action CSS to disable conflicting native behavior
<motion.div
  drag="x"
  style={{ touchAction: 'pan-y' }}  // Allow vertical scroll, capture horizontal drag
  dragConstraints={{ left: -200, right: 0 }}
  dragElastic={0.1}
/>

// PROBLEM: Pan gestures conflict with scroll
<motion.div
  onPan={(e, info) => { /* ... */ }}
  // This prevents scrolling on mobile
/>

// FIX: Add touch-action
<motion.div
  onPan={(e, info) => { /* ... */ }}
  style={{ touchAction: 'pan-y' }}  // Explicitly allow vertical scrolling
/>
```

#### Performance Optimization for Mobile

```tsx
// 1. Use transform-based values instead of layout properties
<motion.div
  animate={{ x: 100 }}      // GOOD: uses transform
  // NOT: animate={{ left: 100 }}  // BAD: triggers layout
/>

// 2. Reduce the number of motion components
// BAD: Every list item is a motion component
{items.map(item => (
  <motion.div
    key={item.id}
    initial={{ opacity: 0, y: 20 }}
    animate={{ opacity: 1, y: 0 }}
    transition={{ delay: item.index * 0.1 }}
  />
))}

// BETTER: Use variants with staggerChildren
const containerVariants = {
  hidden: {},
  visible: {
    transition: {
      staggerChildren: 0.05, // Much more efficient
      delayChildren: 0.1,
    }
  }
};
const itemVariants = {
  hidden: { opacity: 0, y: 20 },
  visible: { opacity: 1, y: 0 }
};

<motion.div variants={containerVariants} initial="hidden" animate="visible">
  {items.map(item => (
    <motion.div key={item.id} variants={itemVariants} />
  ))}
</motion.div>

// 3. Disable layout animations on mobile if too expensive
const isMobile = window.matchMedia('(max-width: 768px)').matches;
<motion.div layout={!isMobile}>
  {/* Layout animations only on desktop */}
</motion.div>

// 4. Use useReducedMotion hook
import { useReducedMotion } from 'motion/react';
function MyComponent() {
  const shouldReduceMotion = useReducedMotion();
  return (
    <motion.div
      animate={{
        x: shouldReduceMotion ? 0 : 100,
        opacity: shouldReduceMotion ? 1 : [0, 1]
      }}
    />
  );
}
```

### Detection Patterns for Framer Motion Issues

```bash
# Find layout prop usage (potential reflow on mobile)
grep -rn "layout[^S]" --include="*.tsx" --include="*.jsx" | grep "motion\."

# Find drag without touch-action
grep -rn "drag=" --include="*.tsx" --include="*.jsx"
# Cross-reference with touchAction

# Find AnimatePresence without layoutScroll in scrollable parents
grep -rn "AnimatePresence" --include="*.tsx" --include="*.jsx"

# Find motion components animating layout properties
grep -rn "animate.*\(width\|height\|top\|left\|margin\|padding\)" --include="*.tsx" --include="*.jsx"

# Find missing layoutScroll on scrollable containers
grep -rn "overflow.*auto\|overflow.*scroll" --include="*.tsx" --include="*.jsx"
```

---

## 5. Lenis / Locomotive Scroll on Mobile

### Lenis on Mobile

Lenis is a lightweight smooth-scroll library that has largely replaced Locomotive Scroll for new projects.

#### When Lenis Helps on Mobile
- Normalizes scroll behavior across browsers
- Provides smooth momentum scrolling with consistent feel
- Syncs well with GSAP ScrollTrigger for animation timing

#### When Lenis Hurts on Mobile
- Adds overhead to native scrolling (which is already smooth on iOS)
- Can conflict with native scroll behaviors (pull-to-refresh, rubber band)
- `syncTouch: true` can cause iOS < 16 issues

#### Recommended Mobile Configuration

```javascript
import Lenis from 'lenis';

// BASIC: Let Lenis handle desktop, native on mobile
const lenis = new Lenis({
  // Touch device configuration
  syncTouch: false,          // Disable sync on touch for better perf
  touchMultiplier: 1,        // Default touch speed (0 = stop immediately)
  touchInertiaMultiplier: 35, // Momentum after finger lifts
  autoRaf: true,             // Manage its own RAF loop
});

// ADVANCED: With GSAP integration
const lenis = new Lenis({
  autoRaf: false,            // GSAP manages the RAF loop
  syncTouch: false,          // Better mobile performance
});

// Sync with GSAP ticker
lenis.on('scroll', ScrollTrigger.update);
gsap.ticker.add((time) => {
  lenis.raf(time * 1000);
});
gsap.ticker.lagSmoothing(0);

// ANDROID FIX: Prevent immediate stop on finger lift
// On Android, screen stops immediately when user lifts finger
// instead of continuing momentum
const lenis = new Lenis({
  touchMultiplier: 0,        // Workaround for Android momentum issue
});
```

#### Disabling Lenis on Mobile

```javascript
// Option 1: Don't initialize on mobile
const isTouchDevice = 'ontouchstart' in window || navigator.maxTouchPoints > 0;
let lenis;
if (!isTouchDevice) {
  lenis = new Lenis();
}

// Option 2: Destroy on mobile breakpoint
const mq = window.matchMedia('(max-width: 768px)');
mq.addEventListener('change', (e) => {
  if (e.matches) {
    lenis?.destroy();
    lenis = null;
  } else {
    lenis = new Lenis();
  }
});
```

### Locomotive Scroll on Mobile

Locomotive Scroll v5 is built on top of Lenis, so it inherits its mobile behavior.

#### V5 Key Changes for Mobile
- Built on Lenis (inherits its touch handling)
- Auto-disables parallax on mobile for native scrolling performance
- Smart touch detection

#### Common Mobile Problems

```javascript
// PROBLEM: Horizontal scroll breaks on mobile in v5
// data-scroll-direction="horizontal" may not work after upgrade

// FIX: Use media query to disable horizontal on mobile
const scroll = new LocomotiveScroll({
  el: document.querySelector('[data-scroll-container]'),
  smooth: true,
  smartphone: {
    smooth: false,   // Fall back to native scroll on phones
  },
  tablet: {
    smooth: false,   // Fall back to native scroll on tablets
  },
});
```

### Detection Patterns

```bash
# Find Lenis or Locomotive Scroll usage
grep -rn "new Lenis\|import.*lenis\|require.*lenis" --include="*.js" --include="*.ts" --include="*.tsx"
grep -rn "LocomotiveScroll\|locomotive-scroll" --include="*.js" --include="*.ts" --include="*.tsx"

# Check if syncTouch is misconfigured
grep -rn "syncTouch" --include="*.js" --include="*.ts"

# Check if smooth scroll is disabled on mobile
grep -rn "smartphone.*smooth\|tablet.*smooth\|isTouchDevice\|ontouchstart" --include="*.js" --include="*.ts"

# Check for deprecated package
grep -rn "@studio-freight/lenis" --include="*.json"
# Should be using "lenis" package instead
```

---

## 6. View Transitions API on Mobile

### Browser Support (as of early 2026)

| Feature | Chrome Android | Safari iOS | Firefox Android |
|---------|---------------|------------|-----------------|
| Same-document (SPA) | 111+ | 18+ | 144+ |
| Cross-document (MPA) | 126+ | 18.2+ | Not yet |

The API reached **Baseline Newly Available** status in October 2025, covering ~89% of global browser traffic.

### Basic Implementation

```javascript
// Same-document view transition (SPA)
function navigateTo(newContent) {
  if (!document.startViewTransition) {
    // Fallback: Just update the DOM
    updateDOM(newContent);
    return;
  }

  const transition = document.startViewTransition(() => {
    updateDOM(newContent);
  });
}

// Cross-document view transition (MPA) - via CSS
// In both pages:
@view-transition {
  navigation: auto;
}
```

### Mobile Performance Considerations

**Critical finding:** View Transitions can add ~70ms to LCP on mobile. If your mobile LCP is already above 2.0 seconds, this overhead puts you closer to failing Core Web Vitals thresholds.

```javascript
// OPTIMIZATION: Use Speculation Rules API to prerender next page
// This eliminates the LCP penalty because both states are already rendered
<script type="speculationrules">
{
  "prerender": [
    {
      "where": { "href_matches": "/product/*" },
      "eagerness": "moderate"
    }
  ]
}
</script>
```

### Mobile-Optimized View Transitions

```css
/* Keep transitions simple on mobile */
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 200ms; /* Shorter on mobile */
}

/* Shared element transitions */
.product-image {
  view-transition-name: product-hero;
}

::view-transition-old(product-hero),
::view-transition-new(product-hero) {
  animation-duration: 250ms;
  animation-timing-function: ease-out;
}

/* Reduce complexity on mobile */
@media (max-width: 768px) {
  ::view-transition-old(root),
  ::view-transition-new(root) {
    animation-duration: 150ms; /* Even shorter on small screens */
  }
}

/* Respect reduced motion */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(root),
  ::view-transition-new(root) {
    animation-duration: 0.01ms !important;
  }
}
```

### Detection Patterns

```bash
# Find view transition usage
grep -rn "startViewTransition\|view-transition-name\|::view-transition" --include="*.js" --include="*.ts" --include="*.css" --include="*.tsx"

# Check for feature detection
grep -rn "startViewTransition" --include="*.js" --include="*.ts"
# Should be wrapped in: if (document.startViewTransition)

# Check for reduced motion respect
grep -rn "view-transition" --include="*.css"
# Should have accompanying prefers-reduced-motion media query
```

---

## 7. Reduced Motion Preferences

### Detection Methods

```css
/* CSS: Opt-in approach (recommended) */
/* Default: No animation */
.element {
  /* Static state */
}

/* Only animate if user is OK with motion */
@media (prefers-reduced-motion: no-preference) {
  .element {
    animation: slideIn 300ms ease-out;
    transition: transform 200ms ease;
  }
}

/* CSS: Opt-out approach (common but less ideal) */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

```javascript
// JavaScript detection
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)');

// Check current preference
if (prefersReducedMotion.matches) {
  // User prefers reduced motion
}

// Listen for changes (user can toggle during session)
prefersReducedMotion.addEventListener('change', (event) => {
  if (event.matches) {
    disableAnimations();
  } else {
    enableAnimations();
  }
});

// React hook
function useReducedMotion() {
  const [prefersReduced, setPrefersReduced] = useState(
    window.matchMedia('(prefers-reduced-motion: reduce)').matches
  );

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    const handler = (e) => setPrefersReduced(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);

  return prefersReduced;
}
```

### What to Reduce vs Disable

| Animation Type | Reduce | Disable | Replace With |
|----------------|--------|---------|-------------|
| Parallax scrolling | | X | Static positioning |
| Background video/motion | | X | Static image |
| Auto-playing carousels | | X | Manual controls |
| Page transition effects | X | | Simple fade or instant |
| Hover effects (scale, etc.) | X | | Subtle opacity change |
| Loading spinners | | | Static "Loading..." text |
| Progress indicators | | | Static progress bar |
| Scroll-triggered reveals | X | | Instant appearance |
| Micro-interactions (button feedback) | X | | Opacity/color only |
| Toast/notification entry | X | | Instant appearance |
| Decorative floating elements | | X | Nothing |

### Accessible Animation Patterns

```css
/* Pattern 1: Crossfade instead of slide */
@media (prefers-reduced-motion: reduce) {
  .modal {
    /* Instead of sliding in, just fade */
    animation: fadeIn 200ms ease;
  }
}

/* Pattern 2: Reduced distance */
@media (prefers-reduced-motion: no-preference) {
  .element { transform: translateY(50px); }
}
@media (prefers-reduced-motion: reduce) {
  .element { transform: translateY(10px); } /* Much less movement */
}

/* Pattern 3: Essential motion indicators still work */
.recording-indicator {
  /* Always show a red dot */
  background: red;
  border-radius: 50%;
}
@media (prefers-reduced-motion: no-preference) {
  .recording-indicator {
    animation: pulse 1.5s infinite; /* Pulsing animation */
  }
}
/* In reduced motion: solid red dot (still communicates "recording") */
```

### GSAP Reduced Motion Integration

```javascript
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (prefersReducedMotion) {
  // Option 1: Set all to final state immediately
  gsap.globalTimeline.timeScale(1000); // Everything plays instantly

  // Option 2: Disable ScrollTrigger animations
  ScrollTrigger.config({ limitCallbacks: true });

  // Option 3: Use matchMedia
  const mm = gsap.matchMedia();
  mm.add('(prefers-reduced-motion: no-preference)', () => {
    // Full animations here
    gsap.to('.hero', { y: -100, scrollTrigger: { ... } });
  });
  mm.add('(prefers-reduced-motion: reduce)', () => {
    // Minimal or no animations
    gsap.set('.hero', { opacity: 1 }); // Just ensure visibility
  });
}
```

### Pause Requirement (WCAG 2.2.2)

Even without `prefers-reduced-motion`, any animation lasting more than 5 seconds MUST have a pause/stop mechanism:

```javascript
// Provide a pause control for long-running animations
const timeline = gsap.timeline({ repeat: -1 });
// ...animation setup...

document.querySelector('.pause-btn').addEventListener('click', () => {
  if (timeline.paused()) {
    timeline.play();
  } else {
    timeline.pause();
  }
});
```

### Detection Patterns

```bash
# Find animations without reduced motion handling
grep -rn "animation:\|transition:" --include="*.css" --include="*.scss"
# Cross-reference with:
grep -rn "prefers-reduced-motion" --include="*.css" --include="*.scss"
# If animations exist but no reduced motion query -> issue

# Find JS animations without reduced motion check
grep -rn "gsap\.\(to\|from\|timeline\)\|\.animate(" --include="*.js" --include="*.ts" --include="*.tsx"
# Cross-reference with:
grep -rn "prefers-reduced-motion\|useReducedMotion\|shouldReduceMotion" --include="*.js" --include="*.ts" --include="*.tsx"

# Find auto-playing animations (should have pause control)
grep -rn "animation-iteration-count.*infinite\|repeat.*-1\|repeat.*Infinity" --include="*.css" --include="*.js" --include="*.ts"
```

---

## 8. Mobile-Specific Animation Bugs

### iOS Safari Bugs

#### 8.1 `backdrop-filter` + `transform` Conflict

```css
/* PROBLEM: backdrop-filter disappears when parent has transform */
.parent {
  transform: translateX(0); /* This breaks backdrop-filter on children in iOS */
}
.child {
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px); /* Still needed on older Safari */
}

/* FIX 1: Move backdrop-filter to a separate stacking context */
.child {
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  isolation: isolate; /* Creates new stacking context */
  z-index: 1;
}

/* FIX 2: Avoid transform on parent, use position instead */
.parent {
  position: relative;
  left: 0; /* Use positioning instead of transform */
}

/* FIX 3: Replace box-shadow with filter: drop-shadow */
.glass-element {
  backdrop-filter: blur(10px);
  /* Instead of: box-shadow: 0 4px 6px rgba(0,0,0,0.1); */
  filter: drop-shadow(0 4px 6px rgba(0,0,0,0.1));
}
```

#### 8.2 `position: fixed` + Animation

```css
/* PROBLEM: Fixed elements jump/flicker during iOS Safari animations */
/* iOS Safari recalculates fixed positioning during scroll momentum */

/* FIX 1: Use transform: translateZ(0) to promote to own layer */
.fixed-header {
  position: fixed;
  top: 0;
  transform: translateZ(0);
  -webkit-transform: translateZ(0);
}

/* FIX 2: On iOS 26+, fixed elements below navigation controls don't render */
/* Workaround: Use sticky instead of fixed where possible */
.header {
  position: sticky;
  top: 0;
  z-index: 100;
}

/* FIX 3: For modals/overlays, use viewport units carefully */
.modal {
  position: fixed;
  inset: 0;
  /* Use dvh instead of vh on iOS to account for dynamic toolbar */
  height: 100dvh;
}
```

#### 8.3 3D Transform Flickering in Safari

```css
/* PROBLEM: 3D transforms cause flickering on Safari */
.card-flip {
  transform-style: preserve-3d;
  perspective: 1000px;
  /* Elements flicker during rotation */
}

/* FIX: Apply backface-visibility and force GPU layer */
.card-flip .front,
.card-flip .back {
  -webkit-backface-visibility: hidden;
  backface-visibility: hidden;
  -webkit-transform: translateZ(0);
  transform: translateZ(0);
}
```

#### 8.4 Animation Flicker on Page Load

```css
/* PROBLEM: Elements flash/flicker before animations start on iOS */

/* FIX 1: Hide elements initially, reveal via JS when ready */
.animate-on-load {
  opacity: 0; /* Hidden by default */
}

/* FIX 2: Use will-change temporarily */
.animate-on-load {
  -webkit-backface-visibility: hidden;
  backface-visibility: hidden;
  -webkit-perspective: 1000;
  perspective: 1000;
}
```

```javascript
// FIX 3: Delay animation start until page is fully rendered
window.addEventListener('load', () => {
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      // Double rAF ensures the browser has painted
      document.querySelectorAll('.animate-on-load').forEach(el => {
        el.classList.add('is-ready');
      });
    });
  });
});
```

### Android Chrome Bugs

#### 8.5 Viewport Resize During Keyboard Open

```javascript
// PROBLEM: Virtual keyboard resizes viewport, triggering animations/media queries
// Viewport height changes cause CSS animations to restart

// FIX 1: Use VirtualKeyboard API (Chrome Android only)
if ('virtualKeyboard' in navigator) {
  navigator.virtualKeyboard.overlaysContent = true;
  // Now keyboard overlays instead of resizing viewport
  // Use env(keyboard-inset-*) to adjust layout
}

// FIX 2: Detect keyboard vs real resize
let lastWidth = window.innerWidth;
window.addEventListener('resize', () => {
  if (window.innerWidth === lastWidth) {
    // Width didn't change = likely keyboard open/close
    // Don't refresh animations
    return;
  }
  lastWidth = window.innerWidth;
  // Real resize: refresh animations
  ScrollTrigger?.refresh();
});
```

```css
/* FIX 3: Use CSS env() for keyboard-aware layouts */
.fixed-bottom-bar {
  position: fixed;
  bottom: env(keyboard-inset-bottom, 0px);
  transition: bottom 100ms ease;
}
```

#### 8.6 Orientation Change Issues

```javascript
// PROBLEM: Animations break after orientation change on Android

// FIX: Debounced refresh after orientation settles
let resizeTimeout;
window.addEventListener('resize', () => {
  clearTimeout(resizeTimeout);
  resizeTimeout = setTimeout(() => {
    // Refresh all animation calculations
    ScrollTrigger?.refresh();
    // Recalculate any cached dimensions
    recalculateAnimationBounds();
  }, 300); // Wait for orientation change to settle
});

// Better: Use screen.orientation API
screen.orientation?.addEventListener('change', () => {
  setTimeout(() => {
    ScrollTrigger?.refresh(true); // Force recalculation
  }, 500); // Longer delay for orientation animation to complete
});
```

### Cross-Platform Issues

#### 8.7 SVG Animation Performance on Mobile

```css
/* PROBLEM: SVG animations are CPU-intensive on mobile */

/* FIX 1: Avoid SMIL animations - use CSS transforms instead */
/* BAD: SMIL (not hardware accelerated in WebKit) */
<animate attributeName="cx" from="0" to="100" dur="1s" />

/* GOOD: CSS transform on SVG element */
.svg-circle {
  animation: moveCircle 1s ease infinite;
}
@keyframes moveCircle {
  to { transform: translateX(100px); }
}

/* FIX 2: Reduce SVG complexity for mobile */
/* Use fewer path points, simpler shapes */
/* Optimize SVGs with SVGO before animating */

/* FIX 3: Use <img> or background-image for complex static SVGs */
/* Only inline SVGs that need animation */
```

```javascript
// FIX 4: Animate SVG with WAAPI for compositor thread
const svgElement = document.querySelector('.animated-svg-group');
svgElement.animate([
  { transform: 'translateX(0) scale(1)' },
  { transform: 'translateX(100px) scale(1.2)' }
], {
  duration: 1000,
  iterations: Infinity,
  easing: 'ease-in-out'
});
```

#### 8.8 `overflow: hidden` + `border-radius` + Transform (Safari)

```css
/* PROBLEM: Rounded corners disappear during transform animation in Safari */
.rounded-card {
  border-radius: 16px;
  overflow: hidden;
  /* During transform animation, content bleeds outside rounded corners */
}

/* FIX: Force new compositing layer */
.rounded-card {
  border-radius: 16px;
  overflow: hidden;
  -webkit-mask-image: -webkit-radial-gradient(white, black);
  /* Or: */
  transform: translateZ(0);
  /* Or: */
  isolation: isolate;
}
```

### Detection Patterns for Mobile-Specific Bugs

```bash
# 1. backdrop-filter + transform conflict
grep -rn "backdrop-filter" --include="*.css" --include="*.scss"
# Check if parent elements have transform

# 2. position:fixed elements (iOS issues)
grep -rn "position:\s*fixed" --include="*.css" --include="*.scss"

# 3. 3D transforms without backface-visibility (Safari flicker)
grep -rn "transform-style.*preserve-3d\|rotateX\|rotateY\|perspective" --include="*.css" --include="*.scss"
# Cross-reference with:
grep -rn "backface-visibility" --include="*.css" --include="*.scss"

# 4. vh units (problematic on mobile)
grep -rn "[0-9]*vh" --include="*.css" --include="*.scss"
# Should use dvh, svh, or lvh instead

# 5. SMIL animations in SVGs
grep -rn "<animate\|<animateTransform\|<animateMotion" --include="*.svg" --include="*.html" --include="*.tsx" --include="*.jsx"

# 6. overflow:hidden + border-radius (Safari rendering bug)
grep -rn "overflow.*hidden" --include="*.css" --include="*.scss"
# Check if combined with border-radius and transform

# 7. Missing -webkit- prefixes for older iOS Safari
grep -rn "backdrop-filter\|backface-visibility" --include="*.css" --include="*.scss"
# Should have -webkit- prefix versions

# 8. Animations without will-change or translateZ(0) (no GPU layer)
grep -rn "@keyframes\|animation:" --include="*.css" --include="*.scss"
```

---

## 9. Programmatic Detection of Animation Issues

### Comprehensive Code Analysis Patterns

The following patterns can be used with grep/ripgrep to scan codebases for potential mobile animation problems:

#### Category 1: Layout-Triggering Animations (High Impact)

```bash
# CSS animations on layout properties
grep -rn "transition.*\b(width|height|top|left|right|bottom|margin|padding|border-width|font-size|line-height)\b" --include="*.css" --include="*.scss" --include="*.less"

# CSS keyframe animations on layout properties
grep -rn "@keyframes" -A 10 --include="*.css" --include="*.scss" | grep -E "width|height|top|left|margin|padding"

# JS style mutations on layout properties
grep -rn "\.style\.(width|height|top|left|margin|padding)" --include="*.js" --include="*.ts" --include="*.tsx"

# GSAP tweens on layout properties
grep -rn 'gsap\.(to|from|fromTo|set)' -A 5 --include="*.js" --include="*.ts" --include="*.tsx" | grep -E '"(width|height|top|left|margin|padding)"'
```

#### Category 2: Scroll Performance Issues

```bash
# Scroll event listeners without throttling (layout thrashing risk)
grep -rn "addEventListener.*['\"]scroll['\"]" --include="*.js" --include="*.ts" --include="*.tsx"

# getBoundingClientRect calls (may be inside scroll handlers)
grep -rn "getBoundingClientRect\|offsetTop\|offsetLeft\|offsetWidth\|offsetHeight\|scrollTop\|scrollLeft" --include="*.js" --include="*.ts" --include="*.tsx"

# Missing IntersectionObserver (could replace scroll listeners)
grep -rn "IntersectionObserver" --include="*.js" --include="*.ts" --include="*.tsx"
```

#### Category 3: Accessibility Issues

```bash
# Animations without reduced-motion handling
# Step 1: Find all animation declarations
grep -rn "animation:|transition:|@keyframes" --include="*.css" --include="*.scss" --include="*.less"
# Step 2: Check for reduced-motion queries
grep -rn "prefers-reduced-motion" --include="*.css" --include="*.scss" --include="*.less"
# If step 1 has results but step 2 doesn't -> Issue

# Infinite animations without pause mechanism
grep -rn "animation.*infinite\|iteration-count.*infinite\|repeat.*-1" --include="*.css" --include="*.js" --include="*.ts"

# Auto-playing video/animation content
grep -rn "autoplay\|autoPlay" --include="*.html" --include="*.tsx" --include="*.jsx"
```

#### Category 4: iOS Safari Specific Risks

```bash
# backdrop-filter without -webkit- prefix
grep -rn "backdrop-filter" --include="*.css" --include="*.scss" | grep -v "webkit"

# position:fixed (problematic on iOS)
grep -rn "position:\s*fixed" --include="*.css" --include="*.scss"

# 100vh usage (doesn't account for mobile browser chrome)
grep -rn "100vh" --include="*.css" --include="*.scss"
# Should be 100dvh or use JS-based height

# 3D transforms without backface-visibility fix
grep -rn "preserve-3d\|rotateX\|rotateY\|rotate3d" --include="*.css" --include="*.scss"

# overflow:hidden + border-radius (Safari rendering)
grep -rn "overflow.*hidden" --include="*.css" --include="*.scss"
```

#### Category 5: Library-Specific Issues

```bash
# GSAP without cleanup in React
grep -rn "gsap\.\(to\|from\|timeline\)" --include="*.tsx" --include="*.jsx"
grep -rn "gsap\.context\|ctx\.revert\|\.kill()" --include="*.tsx" --include="*.jsx"

# Framer Motion layout without layoutScroll
grep -rn "layout[^S]" --include="*.tsx" --include="*.jsx" | grep "motion"
grep -rn "layoutScroll" --include="*.tsx" --include="*.jsx"
grep -rn "overflow.*scroll\|overflow.*auto" --include="*.tsx" --include="*.jsx"

# Lenis deprecated package
grep -rn "@studio-freight/lenis" --include="package.json"

# GSAP ScrollTrigger pin without invalidateOnRefresh
grep -rn "pin.*true" --include="*.js" --include="*.ts" --include="*.tsx"
grep -rn "invalidateOnRefresh" --include="*.js" --include="*.ts" --include="*.tsx"

# will-change overuse
grep -c "will-change" --include="*.css" --include="*.scss"
# More than ~10 instances is suspicious
```

#### Category 6: Performance Anti-Patterns

```bash
# setTimeout/setInterval for animations (should use rAF)
grep -rn "setInterval.*style\|setTimeout.*style\|setInterval.*transform\|setTimeout.*transform" --include="*.js" --include="*.ts"

# Forced synchronous layout (read-write-read pattern)
grep -rn "offsetWidth\|offsetHeight\|getComputedStyle" --include="*.js" --include="*.ts"
# These force layout recalculation if preceded by style writes

# Large DOM animations (too many animated elements)
grep -rn "querySelectorAll.*forEach.*animate\|querySelectorAll.*forEach.*style" --include="*.js" --include="*.ts"

# Missing feature detection for modern APIs
grep -rn "startViewTransition\|animation-timeline\|ScrollTimeline" --include="*.js" --include="*.ts" --include="*.css"
# Should have @supports or if-check wrappers
```

### Automated Scanning Script

```javascript
/**
 * Mobile Animation Issue Scanner
 * Run this in the browser console on any page to detect common issues
 */
function scanForMobileAnimationIssues() {
  const issues = [];

  // 1. Check for layout-triggering animations
  const allElements = document.querySelectorAll('*');
  const layoutProps = ['width', 'height', 'top', 'left', 'right', 'bottom',
                       'margin', 'padding', 'border-width', 'font-size'];

  allElements.forEach(el => {
    const style = getComputedStyle(el);
    const transition = style.transition;
    const animation = style.animationName;

    if (transition && transition !== 'none' && transition !== 'all 0s ease 0s') {
      layoutProps.forEach(prop => {
        if (transition.includes(prop)) {
          issues.push({
            type: 'LAYOUT_ANIMATION',
            severity: 'high',
            element: el,
            message: `Element animates layout property "${prop}" via transition`,
            fix: `Replace with transform equivalent`
          });
        }
      });
    }

    // Check for will-change overuse
    if (style.willChange && style.willChange !== 'auto') {
      issues.push({
        type: 'WILL_CHANGE',
        severity: 'medium',
        element: el,
        message: `will-change: ${style.willChange} detected`,
        fix: 'Ensure will-change is applied only during animation, not permanently'
      });
    }
  });

  // 2. Check for reduced motion support
  const stylesheets = [...document.styleSheets];
  let hasReducedMotionQuery = false;
  try {
    stylesheets.forEach(sheet => {
      try {
        [...sheet.cssRules].forEach(rule => {
          if (rule.media?.mediaText.includes('prefers-reduced-motion')) {
            hasReducedMotionQuery = true;
          }
        });
      } catch (e) { /* cross-origin stylesheet */ }
    });
  } catch (e) {}

  if (!hasReducedMotionQuery) {
    issues.push({
      type: 'ACCESSIBILITY',
      severity: 'high',
      message: 'No prefers-reduced-motion media query found',
      fix: 'Add @media (prefers-reduced-motion: reduce) rules'
    });
  }

  // 3. Check for 100vh usage (problematic on mobile)
  try {
    stylesheets.forEach(sheet => {
      try {
        [...sheet.cssRules].forEach(rule => {
          if (rule.cssText?.includes('100vh') && !rule.cssText?.includes('dvh')) {
            issues.push({
              type: 'MOBILE_VIEWPORT',
              severity: 'medium',
              message: `100vh used without dvh fallback: ${rule.selectorText}`,
              fix: 'Use 100dvh or JS-based viewport height'
            });
          }
        });
      } catch (e) {}
    });
  } catch (e) {}

  // 4. Check for scroll event listeners
  // (Can only detect listeners added via addEventListener on window)
  const originalAddEventListener = EventTarget.prototype.addEventListener;
  // This is a post-hoc check; for real detection, instrument before page load

  // 5. Report
  console.group('Mobile Animation Issues Report');
  console.log(`Found ${issues.length} potential issues`);

  const grouped = {};
  issues.forEach(issue => {
    if (!grouped[issue.type]) grouped[issue.type] = [];
    grouped[issue.type].push(issue);
  });

  Object.entries(grouped).forEach(([type, typeIssues]) => {
    console.group(`${type} (${typeIssues.length} issues)`);
    typeIssues.forEach(issue => {
      console.warn(`[${issue.severity.toUpperCase()}] ${issue.message}`);
      if (issue.element) console.log('  Element:', issue.element);
      console.log('  Fix:', issue.fix);
    });
    console.groupEnd();
  });

  console.groupEnd();
  return issues;
}

scanForMobileAnimationIssues();
```

### Performance Observer for Runtime Detection

```javascript
/**
 * Monitor for layout shifts caused by animations at runtime
 */
function monitorAnimationPerformance() {
  // 1. Detect layout shifts (CLS)
  if ('PerformanceObserver' in window) {
    const clsObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.value > 0.01) {
          console.warn('Layout shift detected:', {
            value: entry.value,
            sources: entry.sources?.map(s => ({
              node: s.node,
              previousRect: s.previousRect,
              currentRect: s.currentRect
            }))
          });
        }
      }
    });
    clsObserver.observe({ type: 'layout-shift', buffered: true });

    // 2. Detect long animation frames
    const lafObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > 50) { // Longer than one frame at 60fps
          console.warn('Long animation frame:', {
            duration: entry.duration,
            blockingDuration: entry.blockingDuration,
            scripts: entry.scripts?.map(s => s.sourceURL)
          });
        }
      }
    });
    try {
      lafObserver.observe({ type: 'long-animation-frame', buffered: true });
    } catch (e) {
      // long-animation-frame not supported
    }
  }
}
```

---

## 10. Visual Verification via Screenshots

### Playwright-Based Animation Testing

```javascript
// playwright.config.js
import { defineConfig } from '@playwright/test';

export default defineConfig({
  use: {
    viewport: { width: 375, height: 812 },   // iPhone viewport
    deviceScaleFactor: 3,
    isMobile: true,
    hasTouch: true,
  },
  projects: [
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 13'] },
    },
  ],
});
```

```javascript
// tests/animation-visual.spec.js
import { test, expect } from '@playwright/test';

test.describe('Mobile Animation Visual Tests', () => {

  // Test 1: Screenshot at different scroll positions
  test('scroll-triggered animations render correctly', async ({ page }) => {
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    // Disable CSS animations for baseline
    await page.addStyleTag({
      content: `*, *::before, *::after {
        animation-duration: 0s !important;
        transition-duration: 0s !important;
      }`
    });

    // Take screenshots at different scroll positions
    const scrollPositions = [0, 500, 1000, 1500, 2000];

    for (const position of scrollPositions) {
      await page.evaluate((y) => window.scrollTo(0, y), position);
      await page.waitForTimeout(100); // Let scroll settle

      await expect(page).toHaveScreenshot(
        `scroll-position-${position}.png`,
        { maxDiffPixels: 50 }
      );
    }
  });

  // Test 2: Check for layout shifts during scroll
  test('no layout shifts during scroll', async ({ page }) => {
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    // Inject CLS observer
    const cls = await page.evaluate(async () => {
      return new Promise((resolve) => {
        let clsValue = 0;
        const observer = new PerformanceObserver((list) => {
          for (const entry of list.getEntries()) {
            if (!entry.hadRecentInput) {
              clsValue += entry.value;
            }
          }
        });
        observer.observe({ type: 'layout-shift', buffered: true });

        // Scroll through the page
        let y = 0;
        const interval = setInterval(() => {
          y += 100;
          window.scrollBy(0, 100);
          if (y >= document.body.scrollHeight - window.innerHeight) {
            clearInterval(interval);
            setTimeout(() => {
              observer.disconnect();
              resolve(clsValue);
            }, 500);
          }
        }, 50);
      });
    });

    expect(cls).toBeLessThan(0.1); // Good CLS score
  });

  // Test 3: Animation frame consistency
  test('animations maintain consistent frame rate', async ({ page }) => {
    await page.goto('/animated-section');

    // Start tracing for performance analysis
    await page.tracing.start({
      screenshots: true,
      categories: ['devtools.timeline']
    });

    // Trigger animation
    await page.click('.start-animation-btn');
    await page.waitForTimeout(2000); // Let animation play

    const trace = await page.tracing.stop();
    // Analyze trace for dropped frames
    // trace can be loaded into Chrome DevTools for analysis
  });

  // Test 4: Reduced motion screenshots
  test('animations respect prefers-reduced-motion', async ({ page }) => {
    // Emulate reduced motion preference
    await page.emulateMedia({ reducedMotion: 'reduce' });
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    await expect(page).toHaveScreenshot('reduced-motion.png', {
      maxDiffPixels: 100
    });
  });

  // Test 5: Screenshot with virtual keyboard
  test('animations work with virtual keyboard open', async ({ page }) => {
    await page.goto('/form-page');

    // Take screenshot before keyboard
    await expect(page).toHaveScreenshot('before-keyboard.png');

    // Focus input to trigger virtual keyboard
    await page.tap('input[type="text"]');
    await page.waitForTimeout(500);

    // Take screenshot with keyboard
    await expect(page).toHaveScreenshot('with-keyboard.png', {
      maxDiffPixels: 200 // Allow for keyboard overlay
    });
  });

  // Test 6: Orientation change
  test('animations survive orientation change', async ({ page }) => {
    await page.goto('/');
    await page.setViewportSize({ width: 375, height: 812 }); // Portrait

    await expect(page).toHaveScreenshot('portrait.png');

    await page.setViewportSize({ width: 812, height: 375 }); // Landscape
    await page.waitForTimeout(500);

    await expect(page).toHaveScreenshot('landscape.png');

    await page.setViewportSize({ width: 375, height: 812 }); // Back to portrait
    await page.waitForTimeout(500);

    await expect(page).toHaveScreenshot('portrait-after-rotation.png');
  });

  // Test 7: Page load animation flicker detection
  test('no flicker during page load animations', async ({ page }) => {
    // Take rapid screenshots during page load
    const screenshots = [];

    page.on('load', async () => {
      for (let i = 0; i < 5; i++) {
        screenshots.push(await page.screenshot());
        await page.waitForTimeout(100);
      }
    });

    await page.goto('/');
    await page.waitForTimeout(1000);

    // Compare consecutive screenshots for unexpected changes
    for (let i = 1; i < screenshots.length; i++) {
      // Use image comparison library (pixelmatch) to detect large diffs
      // that would indicate flicker
    }
  });
});
```

### Puppeteer-Based Approach for CI/CD

```javascript
const puppeteer = require('puppeteer');
const { PNG } = require('pngjs');
const pixelmatch = require('pixelmatch');

async function captureScrollSequence(url, device = 'iPhone 12') {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();

  // Emulate mobile device
  await page.emulate(puppeteer.KnownDevices[device]);

  await page.goto(url, { waitUntil: 'networkidle2' });

  const scrollHeight = await page.evaluate(() => document.body.scrollHeight);
  const viewportHeight = await page.evaluate(() => window.innerHeight);
  const steps = Math.ceil(scrollHeight / viewportHeight);

  const screenshots = [];

  for (let i = 0; i <= steps; i++) {
    const y = i * viewportHeight;
    await page.evaluate((scrollY) => window.scrollTo(0, scrollY), y);
    await page.waitForTimeout(200); // Wait for scroll animations

    const screenshot = await page.screenshot({ type: 'png' });
    screenshots.push({ position: y, buffer: screenshot });
  }

  await browser.close();
  return screenshots;
}

// Compare two runs for visual regression
async function compareRuns(baseline, current) {
  const results = [];

  for (let i = 0; i < baseline.length && i < current.length; i++) {
    const img1 = PNG.sync.read(baseline[i].buffer);
    const img2 = PNG.sync.read(current[i].buffer);
    const diff = new PNG({ width: img1.width, height: img1.height });

    const numDiffPixels = pixelmatch(
      img1.data, img2.data, diff.data,
      img1.width, img1.height,
      { threshold: 0.1 }
    );

    const diffPercentage = (numDiffPixels / (img1.width * img1.height)) * 100;

    results.push({
      scrollPosition: baseline[i].position,
      diffPixels: numDiffPixels,
      diffPercentage: diffPercentage.toFixed(2),
      passed: diffPercentage < 1 // Less than 1% difference
    });
  }

  return results;
}
```

### Chrome DevTools Protocol for Animation Inspection

```javascript
// Use CDP to inspect animations programmatically
const { chromium } = require('playwright');

async function inspectAnimations(url) {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  // Enable Animation domain via CDP
  const client = await page.context().newCDPSession(page);
  await client.send('Animation.enable');

  const animations = [];

  client.on('Animation.animationStarted', (event) => {
    animations.push({
      id: event.animation.id,
      name: event.animation.name,
      type: event.animation.type, // CSSTransition, CSSAnimation, WebAnimation
      duration: event.animation.source?.duration,
      delay: event.animation.source?.delay,
    });
  });

  await page.goto(url, { waitUntil: 'networkidle' });

  // Scroll to trigger scroll-based animations
  await page.evaluate(async () => {
    for (let y = 0; y < document.body.scrollHeight; y += 100) {
      window.scrollTo(0, y);
      await new Promise(r => setTimeout(r, 50));
    }
  });

  console.log(`Detected ${animations.length} animations:`);
  animations.forEach(anim => {
    console.log(`  [${anim.type}] ${anim.name || 'unnamed'} - ${anim.duration}ms`);
  });

  await browser.close();
  return animations;
}
```

---

## Quick Reference: Detection Pattern Summary

| Issue Category | What to Search For | Risk Level |
|---|---|---|
| Layout-triggering animations | `transition: width\|height\|top\|left\|margin\|padding` | HIGH |
| Scroll listeners without throttle | `addEventListener('scroll')` without `requestAnimationFrame` | HIGH |
| Missing reduced-motion | CSS with `animation:` but no `prefers-reduced-motion` | HIGH |
| `100vh` on mobile | `100vh` without `dvh` fallback | MEDIUM |
| `position: fixed` + iOS | `position: fixed` with animations | MEDIUM |
| `will-change` overuse | More than 10 `will-change` declarations | MEDIUM |
| GSAP without cleanup | `gsap.to()` in React without `gsap.context()` | HIGH |
| Framer Motion layout spam | `layout` prop on deeply nested motion elements | MEDIUM |
| `backdrop-filter` + `transform` | Both on same/parent-child elements | HIGH (Safari) |
| 3D transforms without `backface-visibility` | `preserve-3d\|rotateX\|rotateY` without `backface-visibility: hidden` | HIGH (Safari) |
| Deprecated Lenis package | `@studio-freight/lenis` in package.json | LOW |
| Infinite animation without pause | `animation.*infinite` without pause control | MEDIUM (a11y) |
| SMIL SVG animations | `<animate>` elements in SVG | MEDIUM (perf) |
| ScrollTrigger pin without refresh | `pin: true` without `invalidateOnRefresh: true` | MEDIUM (mobile) |
| Missing feature detection | `startViewTransition\|animation-timeline` without `@supports` or `if` check | MEDIUM |

---

## Sources

### Mobile Animation Performance
- [GPU's Role in the Critical Rendering Path - DEV Community](https://dev.to/yorgie7/the-power-behind-web-animations-gpus-role-in-the-critical-rendering-path-27hg)
- [CSS and JavaScript animation performance - MDN](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/CSS_JavaScript_animation_performance)
- [Animation Performance Guide - Motion](https://motion.dev/docs/performance)
- [CSS GPU Animation: Doing It Right - Smashing Magazine](https://www.smashingmagazine.com/2016/12/gpu-animation-doing-it-right/)
- [Myth Busting: CSS Animations vs. JavaScript - CSS-Tricks](https://css-tricks.com/myth-busting-css-animations-vs-javascript/)
- [What forces layout/reflow - Paul Irish](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)
- [FLIP Your Animations - Aerotwist](https://aerotwist.com/blog/flip-your-animations/)
- [The FLIP technique for list animations - Josh Comeau](https://www.joshwcomeau.com/react/animating-the-unanimatable/)
- [The Web Animation Performance Tier List - Motion Magazine](https://motion.dev/magazine/web-animation-performance-tier-list)

### Scroll-Driven Animations
- [CSS Scroll-Driven Animations - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Scroll-driven_animations)
- [Animate elements on scroll - Chrome for Developers](https://developer.chrome.com/docs/css-ui/scroll-driven-animations)
- [Scroll-driven animations case studies - Chrome for Developers](https://developer.chrome.com/blog/css-ui-ecommerce-sda)
- [Introduction to CSS Scroll-Driven Animations - Smashing Magazine](https://www.smashingmagazine.com/2024/12/introduction-css-scroll-driven-animations/)
- [Two approaches to fallback CSS scroll driven animations](https://cydstumpel.nl/two-approaches-to-fallback-css-scroll-driven-animations/)
- [Scroll-timeline polyfill - GitHub](https://github.com/flackr/scroll-timeline)

### IntersectionObserver
- [IntersectionObserver vs Scroll Event Listeners](https://koketsomawasha.hashnode.dev/performance-intersection-observer-vs-scroll-event-listeners)
- [1v1: Scroll Listener vs Intersection Observers](https://itnext.io/1v1-scroll-listener-vs-intersection-observers-469a26ab9eb6)

### GSAP
- [ScrollTrigger Documentation - GSAP](https://gsap.com/docs/v3/Plugins/ScrollTrigger/)
- [GSAP Mobile Optimization - GSAP Forums](https://gsap.com/community/forums/topic/35654-gsap-mobile-optimization/)
- [Optimizing GSAP & Canvas for Smooth Performance](https://www.augustinfotech.com/blogs/optimizing-gsap-and-canvas-for-smooth-performance-and-responsive-design/)
- [ScrollTrigger Pin Issues - GSAP Forums](https://gsap.com/community/forums/topic/25649-scrolltrigger-pin-spacing-issue/)
- [Fixing GSAP ScrollTrigger Pin Issue in React](https://dev.to/abdulwahidkahar/how-to-fixing-gsap-scrolltrigger-pin-issue-in-react-153k)

### Framer Motion / Motion
- [Framer Motion vs Motion One Mobile Performance 2025](https://reactlibraries.com/blog/framer-motion-vs-motion-one-mobile-animation-performance-in-2025)
- [Layout Animation - Motion Docs](https://motion.dev/docs/react-layout-animations)
- [Layout Transition Not Working on Mobile - Framer Community](https://www.framer.community/c/developers/framer-motion-layout-transition-not-working-on-mobile-device)
- [GSAP vs Motion - Motion Docs](https://motion.dev/docs/gsap-vs-motion)
- [Layout animations broken with scroll - GitHub Issue](https://github.com/framer/motion/issues/1471)

### Lenis / Locomotive Scroll
- [Lenis - GitHub](https://github.com/darkroomengineering/lenis)
- [Building Smooth Scroll in 2025 with Lenis](https://www.edoardolunardi.dev/blog/building-smooth-scroll-in-2025-with-lenis)
- [Lenis Mobile Performance Discussions](https://github.com/darkroomengineering/lenis/discussions/431)
- [Disable Lenis on Mobile - GitHub Discussion](https://github.com/darkroomengineering/lenis/discussions/322)
- [Locomotive Scroll Releases](https://github.com/locomotivemtl/locomotive-scroll/releases)

### View Transitions API
- [View Transition API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API)
- [View Transitions API Browser Support - Can I Use](https://caniuse.com/view-transitions)
- [Smooth transitions with the View Transition API - Chrome Developers](https://developer.chrome.com/docs/web-platform/view-transitions)
- [Impact of CSS View Transitions on Web Performance](https://www.corewebvitals.io/pagespeed/view-transition-web-performance)
- [Mastering Page Transitions with View Transitions API 2026](https://dev.to/krish_kakadiya_5f0eaf6342/mastering-smooth-page-transitions-with-the-view-transitions-api-in-2026-31of)

### Reduced Motion
- [prefers-reduced-motion - CSS-Tricks](https://css-tricks.com/almanac/rules/m/media/prefers-reduced-motion/)
- [Design accessible animation - Pope Tech](https://blog.pope.tech/2025/12/08/design-accessible-animation-and-movement/)
- [CSS prefers-reduced-motion - W3C WAI](https://www.w3.org/WAI/WCAG21/Techniques/css/C39)
- [Accessible Animations in React - Josh Comeau](https://www.joshwcomeau.com/react/prefers-reduced-motion/)
- [prefers-reduced-motion - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-reduced-motion)

### iOS Safari Bugs
- [Safari WebKit CSS Bugs and Workarounds](https://docs.bswen.com/blog/2026-03-12-safari-css-issues-workarounds/)
- [Backdrop Filter Rendering Issues in Safari](https://copyprogramming.com/howto/backdrop-filter-blur-box-shadow-not-rendering-properly-in-safari)
- [iOS CSS Transition Flickering Fix](https://www.pixeldock.com/blog/how-to-avoid-the-ugly-flickering-effect-when-using-css-transitions-in-ios/)
- [3D Transform Flickering in Safari - Webflow Forum](https://discourse.webflow.com/t/3d-transform-flickering-glitch-safari-only-issue/127805)
- [Safari border-radius + overflow + transform fix](https://gist.github.com/ayamflow/b602ab436ac9f05660d9c15190f4fd7b)
- [Severe bugs with CSS transitions on iOS WebKit](https://github.com/necolas/react-native-web/issues/484)

### Android / Virtual Keyboard
- [VirtualKeyboard API - Chrome Developers](https://developer.chrome.com/docs/web-platform/virtual-keyboard)
- [VirtualKeyboard API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/VirtualKeyboard_API)
- [Virtual Keyboard API - Ahmad Shadeed](https://ishadeed.com/article/virtual-keyboard-api/)
- [Orientation and media queries on Android](https://www.robinosborne.co.uk/2014/09/12/android-browsers-keyboards-and-media-queries/)

### SVG Animation
- [SVG Animation Methods Compared - Xyris](https://xyris.app/blog/svg-animation-methods-compared-css-smil-and-javascript/)
- [Weighing SVG Animation Techniques with Benchmarks - CSS-Tricks](https://css-tricks.com/weighing-svg-animation-techniques-benchmarks/)
- [Optimizing SVG Animations for Desktop and Mobile](https://www.zigpoll.com/content/how-can-i-optimize-svg-animations-to-run-smoothly-on-both-desktop-and-mobile-browsers-without-significant-performance-loss)

### Performance Auditing & Detection
- [AI Animation Performance Audit - Motion](https://motion.dev/docs/animation-performance-audit)
- [How To Fix Forced Reflows - DebugBear](https://www.debugbear.com/blog/forced-reflows)
- [CSS Triggers List](https://csstriggers.com/)
- [Rendering Performance - web.dev](https://web.dev/articles/rendering-performance)
- [Cumulative Layout Shift - web.dev](https://web.dev/articles/cls)
- [CLS Debugger](https://webvitals.dev/cls)

### Visual Regression Testing
- [Visual Comparisons - Playwright Docs](https://playwright.dev/docs/test-snapshots)
- [Playwright Visual Regression Testing Guide](https://testgrid.io/blog/playwright-visual-regression-testing/)
- [Visual Regression Testing - Vitest](https://vitest.dev/guide/browser/visual-regression-testing)
