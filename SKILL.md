---
name: mobile-ux-fixer
description: "Comprehensive mobile UX audit, visual inspection, and auto-fix skill for web projects. Detects and fixes 40+ mobile issues across 10 phases: viewport/layout, scroll/parallax, animations, images, touch/forms, typography, Core Web Vitals, SEO/PWA, accessibility, RTL, and dark mode. Uses Playwright MCP for visual inspection. Use when user says 'fix mobile', 'mobile audit', 'check mobile', 'mobile issues', 'looks broken on phone', 'scroll jump', 'address bar jump', 'parallax mobile', 'mobile test', 'mobile performance', 'core web vitals mobile', 'תקן מובייל', 'בעיות במובייל', 'קופץ בגלילה', 'נראה רע בטלפון', 'בדיקת מובייל', or after deploying and testing on a real device."
license: MIT
compatibility: "Requires Node.js and npx for Playwright MCP visual inspection (Phase 0). Code-level checks (Phases 1-10) work in any environment."
metadata:
  author: MediaWave
  version: 2.0.0
  mcp-server: playwright
---

# Mobile UX Fixer v2.0

> Systematic mobile audit, visual inspection, and auto-fix skill.
> 40+ checks across 10 phases. Every check comes from real production bugs.

---

## When to Use

- After deploying to production and testing on a real phone
- When user reports scroll jumps, layout shifts, or broken visuals on mobile
- Before launch as a pre-flight mobile checklist
- When parallax or animations look wrong on mobile devices
- When Core Web Vitals scores are poor on mobile
- When accessibility issues are reported on mobile

---

## Instructions

### Phase 0: Visual Inspection with Playwright MCP

> **Use Playwright MCP to SEE the site on mobile before running code checks.**

#### Auto-Setup

1. Check if Playwright MCP is already configured:
   ```
   Run: claude mcp list
   Look for: "playwright" in the output
   ```

2. If NOT found, install it automatically:
   ```bash
   claude mcp add playwright -- npx @playwright/mcp@latest --headless --device "iPhone 15" --caps vision
   ```

3. If the `claude` CLI is not available or the command fails, add it manually to `.mcp.json` or `~/.claude.json`:
   ```json
   {
     "mcpServers": {
       "playwright": {
         "command": "npx",
         "args": ["@playwright/mcp@latest", "--headless", "--device", "iPhone 15", "--caps", "vision"]
       }
     }
   }
   ```

4. Verify by running `/mcp` and checking that `playwright` server appears.

**Note:** If Playwright MCP cannot be installed, skip Phase 0 and proceed to Phase 1.

#### Visual Audit Steps

1. **Navigate** to the site URL using `browser_navigate`
2. **Screenshot** at initial mobile viewport
3. **Scroll down** incrementally, screenshot each section:
   ```
   browser_evaluate: window.scrollBy(0, window.innerHeight)
   Then screenshot. Repeat until page bottom.
   ```
4. **Check horizontal overflow** programmatically:
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
5. **Test multiple viewports** -- 320px, 375px, 390px, 412px
6. **Test landscape** -- resize to 812x375

#### What to Look For

- Elements overflowing the viewport
- Text too small to read
- Buttons/links too close together
- Images cropped awkwardly
- Navigation not accessible
- Animations that look broken (compare before/after screenshots)
- Layout shifts (compare sequential screenshots)

---

### Phases 1-10: Code-Level Checks

**For each check below, consult `references/phase-checks.md` for detailed detect patterns and fix code.**

#### Phase 1: Viewport and Layout Stability

| Check | What It Catches |
|-------|----------------|
| 1.1 Address bar scroll jump | JS using `window.innerHeight` during scroll |
| 1.2 dvh vs svh | `100dvh` shifting content when address bar toggles |
| 1.3 Safe areas | Fixed elements hidden behind notch/home indicator |
| 1.4 Body scroll lock (iOS) | Background scrolling through modals on iOS Safari |
| 1.5 Horizontal overflow | Elements wider than viewport |

#### Phase 2: Parallax and Scroll Effects

| Check | What It Catches |
|-------|----------------|
| 2.1 background-attachment: fixed | Broken on iOS Safari |
| 2.2 Ken Burns consistency | Inconsistent scale values across animation variants |
| 2.3 GSAP ScrollTrigger | Pin-spacer height mismatch with mobile address bar |
| 2.4 Framer Motion layout | Missing `layoutScroll` on scrollable containers |
| 2.5 Lenis/smooth scroll | `syncTouch: true` performance issues, wrong GSAP config |
| 2.6 Scroll-snap iOS | iOS caches snap points, fast flick skips items |

#### Phase 3: Animations and Motion

| Check | What It Catches |
|-------|----------------|
| 3.1 Layout-triggering animations | Animating width/height/top/left instead of transform |
| 3.2 will-change overuse | Too many compositor layers exhausting GPU memory |
| 3.3 Reduced motion | Missing `prefers-reduced-motion` (WCAG requirement) |
| 3.4 iOS backdrop-filter bug | White rendering when combining backdrop-filter + bg-color |
| 3.5 Page load flicker | Elements flash before JS initializes animations |
| 3.6 Scroll-driven fallback | `animation-timeline: scroll()` without @supports fallback |

#### Phase 4: Images and Media

| Check | What It Catches |
|-------|----------------|
| 4.1 Image cropping | `object-cover` without `object-position` |
| 4.2 Image loading | Hero with `loading="lazy"` or missing `fetchpriority="high"` |
| 4.3 Missing dimensions (CLS) | Images without width/height causing layout shift |
| 4.4 Modern formats | Using jpg/png instead of WebP/AVIF |
| 4.5 Art direction | Same image crop for desktop and mobile |

#### Phase 5: Touch, Forms, and Interaction

| Check | What It Catches |
|-------|----------------|
| 5.1 Touch target size | Targets under 44x44px (WCAG 2.5.8) |
| 5.2 iOS font zoom | Input font-size under 16px triggers auto-zoom |
| 5.3 Passive listeners | Non-passive touchstart/touchmove/wheel blocking scroll |
| 5.4 Inputmode | Missing `inputmode` showing wrong keyboard |
| 5.5 Virtual keyboard | Fixed bottom elements jumping when keyboard opens |
| 5.6 Overscroll behavior | Scroll chaining through modals, unwanted pull-to-refresh |

#### Phase 6: Typography and Readability

| Check | What It Catches |
|-------|----------------|
| 6.1 Min font sizes | Body text below 16px, any text below 12px |
| 6.2 Fluid typography | Fixed px heading sizes that don't scale |
| 6.3 Line height | Body text with line-height below 1.4 |
| 6.4 Line length | Text lines over 45 characters on mobile |

#### Phase 7: Core Web Vitals (Mobile)

| Check | What It Catches |
|-------|----------------|
| 7.1 LCP | Render-blocking resources, lazy hero images |
| 7.2 CLS from fonts | @font-face without font-display, font reflow |
| 7.3 INP/TBT | Long tasks blocking main thread over 50ms |
| 7.4 Third-party scripts | YouTube embeds, chat widgets blocking render |
| 7.5 Font loading | Using ttf/otf, too many font files, missing crossorigin |

#### Phase 8: SEO, Meta Tags, and PWA

| Check | What It Catches |
|-------|----------------|
| 8.1 Viewport meta | Missing or misconfigured viewport tag |
| 8.2 Theme-color | Missing theme-color meta tag |
| 8.3 Open Graph | Missing og:title, og:description, og:image |
| 8.4 PWA manifest | Missing or incomplete manifest.json |

#### Phase 9: Accessibility on Mobile

| Check | What It Catches |
|-------|----------------|
| 9.1 Hamburger menu a11y | Missing aria-expanded, aria-label, focus trap |
| 9.2 Focus management | Focus styles removed without :focus-visible replacement |
| 9.3 Skip to content | Missing skip navigation link |
| 9.4 ARIA on interactive | Clickable divs/spans without role, tabindex, keyboard |

#### Phase 10: RTL and Dark Mode

| Check | What It Catches |
|-------|----------------|
| 10.1 RTL logical properties | Physical CSS properties instead of logical (margin-left vs margin-inline-start) |
| 10.2 Dark mode | Missing prefers-color-scheme support |

---

## Quick Audit Checklist

Run all checks at once. Report: OK / ISSUE FOUND / SKIPPED.

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
  3.3 Reduced motion                        [ ]
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

## Examples

### Example 1: Full mobile audit

User says: "run mobile audit on this site"

Actions:
1. Set up Playwright MCP if not present
2. Navigate to site, take screenshots at 375px and 390px viewports
3. Scroll through and check for horizontal overflow
4. Run all Phase 1-10 code checks using patterns from `references/phase-checks.md`
5. Report findings with file paths, line numbers, and severity

Result: Full audit report with OK / ISSUE FOUND for each check.

### Example 2: Fix specific issue

User says: "the page jumps when I scroll on mobile"

Actions:
1. Check 1.1 (address bar scroll jump) -- search for `window.innerHeight` in scroll handlers
2. Check 1.2 (dvh vs svh) -- search for `100dvh` or `100vh` in CSS
3. Check 2.3 (GSAP ScrollTrigger) -- search for ScrollTrigger pins
4. Apply fixes from `references/phase-checks.md`

Result: Identified cause and applied fix directly in codebase.

### Example 3: Pre-launch checklist

User says: "check mobile before we deploy"

Actions:
1. Run the Quick Audit Checklist above
2. Focus on critical items: LCP, CLS, touch targets, viewport meta
3. Flag any ISSUE FOUND items with severity (critical/warning/info)
4. Fix critical issues, report warnings for review

Result: Deploy-ready mobile audit with all critical issues resolved.

---

## Troubleshooting

### "It works on Chrome DevTools but not on real phone"
DevTools does NOT simulate: address bar show/hide, iOS Safari quirks, real touch timing, GPU memory limits. Use Playwright MCP with `--device` flag or test on real devices.

### "Parallax looks smooth on my phone but jumps on others"
Use `requestAnimationFrame` throttling and keep speed values under 0.1. Consider disabling parallax on low-end devices.

### "Everything shifts when the keyboard opens"
Use `visualViewport` API. See Check 5.5 in `references/phase-checks.md`.

### "GSAP ScrollTrigger pins jump on mobile"
Add `ScrollTrigger.normalizeScroll(true)` or disable pins on mobile with `matchMedia`. See Check 2.3.

### "Lenis smooth scroll feels laggy on phone"
Set `syncTouch: false` and adjust `touchMultiplier`. Use `autoRaf: false` with GSAP ticker. See Check 2.5.

### "Animations flash on page load"
Use the double-rAF technique or hide animated elements with CSS until JS is ready. See Check 3.5.

### "Site looks zoomed in after filling a form on iPhone"
Ensure all input font-sizes are >= 16px. Use `text-base md:text-sm` in Tailwind. See Check 5.2.

---

## Reference Files

Detailed detect patterns and fix code for all checks:
- `references/phase-checks.md` -- All detect patterns and fix code for Phases 1-10

Research backing each check:
- `references/mobile-ux-research-2024-2026.md` -- CSS, JS, iOS/Android, forms, typography, images, a11y
- `references/RESEARCH-playwright-mobile-testing.md` -- Playwright MCP setup, visual testing, performance
- `references/RESEARCH-mobile-animation-best-practices.md` -- GSAP, Framer Motion, Lenis, animation bugs
- `references/mobile-web-standards-research-2024-2026.md` -- SEO, PWA, fonts, dark mode, RTL
