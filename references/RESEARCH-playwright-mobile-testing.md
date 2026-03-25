# Playwright for Mobile Website Testing & Visual Inspection with Claude Code

> Comprehensive research report covering Playwright MCP integration, mobile emulation, visual testing, animation testing, performance measurement, and accessibility auditing.

---

## Table of Contents

1. [Playwright MCP for Claude Code](#1-playwright-mcp-for-claude-code)
2. [Playwright Mobile Emulation](#2-playwright-mobile-emulation)
3. [Visual Testing with Playwright on Mobile](#3-visual-testing-with-playwright-on-mobile)
4. [Animation and Transition Testing](#4-animation-and-transition-testing)
5. [Performance Testing on Mobile](#5-performance-testing-on-mobile)
6. [Accessibility Testing](#6-accessibility-testing)
7. [Best Practices for Claude Code Visual Analysis](#7-best-practices-for-claude-code-visual-analysis)

---

## 1. Playwright MCP for Claude Code

### Overview

The **Playwright MCP** (Model Context Protocol) server is Microsoft's official package (`@playwright/mcp`) that gives coding agents like Claude Code direct browser control. It exposes Playwright's browser automation as structured MCP tools that an AI can invoke.

**Key architecture choice**: The server uses the browser's **accessibility tree** for interactions, making them faster and more token-efficient than screenshot-based approaches. Claude works directly with structured data -- no vision model needed for basic interactions.

### Installation

```bash
# Add Playwright MCP to Claude Code
claude mcp add playwright npx "@playwright/mcp@latest"
```

This persists in `~/.claude.json`. To verify, run `/mcp` inside Claude Code and navigate to the `playwright` entry.

### Configuration for Mobile Testing

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

### Full Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `--browser` | Browser type: `chrome`, `firefox`, `webkit`, `msedge` | `chrome` |
| `--headless` | Run in headless mode (no visible window) | `false` |
| `--device` | Device preset (e.g., `"iPhone 15"`, `"Pixel 7"`, `"Galaxy S24"`) | none |
| `--viewport-size` | Viewport dimensions (e.g., `"375x812"`) | browser default |
| `--user-agent` | Custom user agent string | browser default |
| `--caps` | Extra capabilities: `vision`, `pdf`, `devtools` | none |
| `--timeout-action` | Action timeout in ms | `5000` |
| `--timeout-navigation` | Navigation timeout in ms | `60000` |
| `--isolated` | Each session uses a separate profile | `false` |
| `--storage-state` | Path to cookies/localStorage file | none |
| `--proxy-server` | Proxy server URL | none |
| `--output-dir` | Directory for output files | none |
| `--snapshot-mode` | `"incremental"`, `"full"`, or `"none"` | `incremental` |
| `--image-responses` | `"allow"` or `"omit"` | `allow` |

### Available MCP Tools (Complete List)

The official Microsoft Playwright MCP server exposes these tools:

**Navigation:**
- `browser_navigate` -- Navigate to a URL
- `browser_navigate_back` -- Go back in history

**Interaction:**
- `browser_click` -- Click an element
- `browser_type` -- Type text into an element
- `browser_fill_form` -- Fill a form field
- `browser_select_option` -- Select a dropdown option
- `browser_hover` -- Hover over an element
- `browser_drag` -- Drag an element
- `browser_press_key` -- Press a keyboard key
- `browser_file_upload` -- Upload a file
- `browser_handle_dialog` -- Accept/dismiss dialogs

**Inspection:**
- `browser_take_screenshot` -- Capture a screenshot (returns image)
- `browser_snapshot` -- Get accessibility tree snapshot
- `browser_console_messages` -- Read console messages
- `browser_network_requests` -- View network activity

**Advanced:**
- `browser_evaluate` -- Execute JavaScript in the page context
- `browser_run_code` -- Run a Playwright script
- `browser_resize` -- Resize the viewport
- `browser_tabs` -- List and switch between tabs
- `browser_close` -- Close the browser
- `browser_install` -- Install a browser

### Practical Usage Examples

```
# Navigate to a URL
"Use playwright mcp to navigate to https://example.com"

# Take a screenshot
"Take a screenshot of the current page"

# Get accessibility snapshot
"Get a snapshot of the page structure"

# Execute JavaScript
"Use browser_evaluate to get document.body.scrollHeight"

# Resize viewport for mobile
"Resize the browser to 375x812 pixels"
```

### Authentication Workflow

A practical approach for authenticated pages:
1. Have Claude navigate to the login page
2. Log in manually with your credentials
3. Tell Claude to continue -- cookies persist for the session

---

## 2. Playwright Mobile Emulation

### Built-in Device Descriptors

Playwright ships with 143+ pre-configured device presets. Each descriptor includes viewport, user agent, device scale factor, touch support, and mobile flag.

#### Common Mobile Device Descriptors

| Device | Viewport | Scale | User Agent (abbreviated) |
|--------|----------|-------|--------------------------|
| `iPhone 12` | 390x844 | 3 | Mozilla/5.0 (iPhone; CPU iPhone OS 15_0...) |
| `iPhone 13` | 390x844 | 3 | Mozilla/5.0 (iPhone; CPU iPhone OS 15_0...) |
| `iPhone 14` | 390x844 | 3 | Mozilla/5.0 (iPhone; CPU iPhone OS 16_0...) |
| `iPhone 15` | 393x852 | 3 | Mozilla/5.0 (iPhone; CPU iPhone OS 17_0...) |
| `iPhone 15 Pro Max` | 430x932 | 3 | Mozilla/5.0 (iPhone; CPU iPhone OS 17_0...) |
| `Pixel 5` | 393x851 | 2.75 | Mozilla/5.0 (Linux; Android 11...) |
| `Pixel 7` | 412x915 | 2.625 | Mozilla/5.0 (Linux; Android 13...) |
| `Galaxy S20 Ultra` | 412x915 | 3.5 | Mozilla/5.0 (Linux; Android 10...) |
| `Galaxy S24` | 360x780 | 3 | Mozilla/5.0 (Linux; Android 14...) |
| `iPad Pro 11` | 834x1194 | 2 | Mozilla/5.0 (iPad; CPU OS 15_0...) |

#### Using Device Presets in Playwright Test Config

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 15'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 7'] },
    },
    {
      name: 'Galaxy S24',
      use: { ...devices['Galaxy S24'] },
    },
    {
      name: 'Tablet',
      use: { ...devices['iPad Pro 11'] },
    },
  ],
});
```

#### Using Device Presets Programmatically

```typescript
import { chromium, devices } from 'playwright';

const iPhone = devices['iPhone 15'];
const browser = await chromium.launch();
const context = await browser.newContext({
  ...iPhone,
  // Override specific properties if needed:
  locale: 'en-US',
  geolocation: { longitude: -122.4194, latitude: 37.7749 },
  permissions: ['geolocation'],
});
const page = await context.newPage();
await page.goto('https://example.com');
```

#### Custom Device Configuration

```typescript
const context = await browser.newContext({
  viewport: { width: 414, height: 896 },
  userAgent: 'Mozilla/5.0 (Linux; Android 14; SM-S928B) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36',
  deviceScaleFactor: 3,
  isMobile: true,
  hasTouch: true,
  // Additional options:
  colorScheme: 'dark',
  reducedMotion: 'reduce',
  forcedColors: 'active',
});
```

### Network Throttling (Slow 3G/4G Simulation)

Playwright uses Chrome DevTools Protocol for network throttling. **This only works with Chromium-based browsers.**

```typescript
// Create a CDP session
const session = await page.context().newCDPSession(page);

// Enable network domain
await session.send('Network.enable');

// Slow 3G
await session.send('Network.emulateNetworkConditions', {
  offline: false,
  downloadThroughput: (0.4 * 1024 * 1024) / 8,  // 0.4 Mbps
  uploadThroughput: (0.4 * 1024 * 1024) / 8,     // 0.4 Mbps
  latency: 2000,                                   // 2000ms
  connectionType: 'cellular3g',
});

// Fast 3G
await session.send('Network.emulateNetworkConditions', {
  offline: false,
  downloadThroughput: (1.6 * 1024 * 1024) / 8,   // 1.6 Mbps
  uploadThroughput: (0.75 * 1024 * 1024) / 8,     // 750 kbps
  latency: 562,                                     // 562ms
  connectionType: 'cellular3g',
});

// Slow 4G
await session.send('Network.emulateNetworkConditions', {
  offline: false,
  downloadThroughput: (3 * 1024 * 1024) / 8,      // 3 Mbps
  uploadThroughput: (1.5 * 1024 * 1024) / 8,      // 1.5 Mbps
  latency: 150,                                     // 150ms
  connectionType: 'cellular4g',
});

// Regular 4G / LTE
await session.send('Network.emulateNetworkConditions', {
  offline: false,
  downloadThroughput: (12 * 1024 * 1024) / 8,     // 12 Mbps
  uploadThroughput: (6 * 1024 * 1024) / 8,         // 6 Mbps
  latency: 50,                                      // 50ms
  connectionType: 'cellular4g',
});
```

### CPU Throttling

```typescript
const session = await page.context().newCDPSession(page);

// Throttle CPU to simulate slow mobile device
// rate: 1 = no throttle, 4 = 4x slower, 6 = 6x slower
await session.send('Emulation.setCPUThrottlingRate', { rate: 4 });

// For very slow devices (low-end Android)
await session.send('Emulation.setCPUThrottlingRate', { rate: 6 });
```

### Using with Playwright MCP

For Playwright MCP, configure mobile emulation at the server level:

```json
{
  "mcpServers": {
    "playwright-mobile": {
      "command": "npx",
      "args": [
        "@playwright/mcp@latest",
        "--device", "iPhone 15",
        "--headless",
        "--caps", "vision"
      ]
    }
  }
}
```

Or dynamically resize within a session:
```
"Use browser_resize to set viewport to 375x812"
"Use browser_evaluate to check if the mobile menu is visible"
```

---

## 3. Visual Testing with Playwright on Mobile

### Full-Page Mobile Screenshots

```typescript
// Full page screenshot (captures entire scrollable content)
await page.screenshot({
  path: 'mobile-full-page.png',
  fullPage: true,
});

// Viewport-only screenshot (what the user sees)
await page.screenshot({
  path: 'mobile-viewport.png',
  fullPage: false,
});
```

### Element-Specific Screenshots

```typescript
// Screenshot a specific element
const header = page.locator('header');
await header.screenshot({ path: 'header.png' });

// Screenshot the navigation menu
const nav = page.locator('nav.mobile-menu');
await nav.screenshot({ path: 'mobile-nav.png' });

// Screenshot a hero section
const hero = page.locator('.hero-section');
await hero.screenshot({ path: 'hero.png' });
```

### Clipped Region Screenshots

```typescript
// Capture a specific region of the page
await page.screenshot({
  path: 'region.png',
  clip: {
    x: 0,
    y: 0,
    width: 375,
    height: 600,
  },
});
```

### Visual Comparison (Regression Detection)

```typescript
// Built-in visual comparison with Playwright Test
import { test, expect } from '@playwright/test';

test('mobile homepage visual regression', async ({ page }) => {
  await page.goto('https://example.com');

  // Compare against baseline (auto-created on first run)
  await expect(page).toHaveScreenshot('homepage-mobile.png', {
    fullPage: true,
    maxDiffPixels: 100,          // Allow up to 100 different pixels
    // OR
    maxDiffPixelRatio: 0.01,     // Allow 1% difference
    threshold: 0.2,               // Per-pixel color diff threshold (0-1)
  });
});

// Compare specific elements
test('mobile nav visual check', async ({ page }) => {
  await page.goto('https://example.com');
  const nav = page.locator('nav');
  await expect(nav).toHaveScreenshot('mobile-nav.png');
});
```

#### Masking Dynamic Content

```typescript
await expect(page).toHaveScreenshot('page.png', {
  mask: [
    page.locator('.timestamp'),
    page.locator('.ad-banner'),
    page.locator('.user-avatar'),
  ],
  maskColor: '#FF00FF',  // Visible mask color for debugging
});
```

### Detecting Horizontal Overflow

```typescript
// Check if page has horizontal overflow (common mobile issue)
const hasHorizontalOverflow = await page.evaluate(() => {
  return document.documentElement.scrollWidth > document.documentElement.clientWidth;
});

// Get all elements causing horizontal overflow
const overflowingElements = await page.evaluate(() => {
  const docWidth = document.documentElement.clientWidth;
  const results = [];
  const allElements = document.querySelectorAll('*');

  for (const el of allElements) {
    const rect = el.getBoundingClientRect();
    if (rect.right > docWidth || rect.left < 0) {
      results.push({
        tag: el.tagName,
        class: el.className,
        id: el.id,
        right: rect.right,
        width: rect.width,
        overflow: rect.right - docWidth,
      });
    }
  }
  return results;
});

console.log('Elements causing horizontal overflow:', overflowingElements);
```

### Verifying Touch Target Sizes

```typescript
// Check all interactive elements meet minimum touch target size
// WCAG 2.5.5 (AAA): 44x44px | WCAG 2.5.8 (AA): 24x24px
const touchTargetIssues = await page.evaluate(() => {
  const MIN_SIZE = 44; // or 48 for Material Design
  const interactiveSelectors = 'a, button, input, select, textarea, [role="button"], [role="link"], [role="checkbox"], [role="radio"], [tabindex]';
  const elements = document.querySelectorAll(interactiveSelectors);
  const issues = [];

  for (const el of elements) {
    const rect = el.getBoundingClientRect();
    if (rect.width > 0 && rect.height > 0) {
      if (rect.width < MIN_SIZE || rect.height < MIN_SIZE) {
        issues.push({
          tag: el.tagName,
          text: el.textContent?.trim().substring(0, 50),
          class: el.className,
          width: Math.round(rect.width),
          height: Math.round(rect.height),
          tooSmallBy: {
            width: Math.max(0, MIN_SIZE - rect.width),
            height: Math.max(0, MIN_SIZE - rect.height),
          },
        });
      }
    }
  }
  return issues;
});

console.log('Touch target issues:', touchTargetIssues);
```

### Systematic Scroll-and-Capture

```typescript
// Capture screenshots while scrolling through the entire page
async function scrollAndCapture(page, outputDir) {
  const viewportHeight = page.viewportSize().height;
  const totalHeight = await page.evaluate(() => document.body.scrollHeight);
  const screenshots = [];
  let currentScroll = 0;
  let index = 0;

  while (currentScroll < totalHeight) {
    await page.evaluate((scrollY) => window.scrollTo(0, scrollY), currentScroll);
    await page.waitForTimeout(300); // Wait for any scroll-triggered animations

    const path = `${outputDir}/scroll-${index}.png`;
    await page.screenshot({ path, fullPage: false });
    screenshots.push(path);

    currentScroll += viewportHeight;
    index++;
  }

  return screenshots;
}
```

---

## 4. Animation and Transition Testing

### Testing CSS Transitions

```typescript
test('fade-in animation works correctly', async ({ page }) => {
  await page.goto('https://example.com');

  const element = page.locator('.animated-element');

  // Verify initial state
  await expect(element).toHaveCSS('opacity', '0');

  // Trigger the animation
  await page.click('#trigger-button');

  // Wait for animation to complete using waitForFunction
  await page.waitForFunction(() => {
    const el = document.querySelector('.animated-element');
    return parseFloat(getComputedStyle(el).opacity) > 0.99;
  });

  // Verify final state
  await expect(element).toHaveCSS('opacity', '1');
});
```

### Testing Slide/Transform Animations

```typescript
test('slide panel opens correctly', async ({ page }) => {
  await page.goto('https://example.com');

  const panel = page.locator('.slide-panel');

  // Verify panel is off-screen initially
  await expect(panel).toHaveCSS('transform', 'matrix(1, 0, 0, 1, -300, 0)');

  // Open the panel
  await page.click('#open-panel');

  // Wait for transition to complete
  await page.waitForFunction(() => {
    const el = document.querySelector('.slide-panel');
    const transform = getComputedStyle(el).transform;
    return transform === 'none' || transform === 'matrix(1, 0, 0, 1, 0, 0)';
  });

  // Verify panel is visible
  await expect(panel).toHaveCSS('transform', 'none');
});
```

### Testing Scroll-Based Animations

```typescript
test('parallax effect triggers on scroll', async ({ page }) => {
  await page.goto('https://example.com');

  // Get initial state
  const initialTransform = await page.evaluate(() => {
    const el = document.querySelector('.parallax-element');
    return getComputedStyle(el).transform;
  });

  // Scroll down programmatically
  await page.evaluate(() => window.scrollBy(0, 500));
  await page.waitForTimeout(500); // Allow scroll-based animation to process

  // Check that transform has changed
  const afterScrollTransform = await page.evaluate(() => {
    const el = document.querySelector('.parallax-element');
    return getComputedStyle(el).transform;
  });

  expect(afterScrollTransform).not.toBe(initialTransform);
});
```

### Capturing Video of Animations

```typescript
// Playwright Test configuration for video recording
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  use: {
    video: {
      mode: 'on',           // 'on' | 'off' | 'on-first-retry' | 'retain-on-failure'
      size: { width: 375, height: 812 },
    },
  },
  projects: [
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 15'] },
    },
  ],
});
```

```typescript
// Programmatic video recording
const context = await browser.newContext({
  recordVideo: {
    dir: './videos/',
    size: { width: 375, height: 812 },
  },
});

const page = await context.newPage();
await page.goto('https://example.com');

// Perform scroll to capture animation
for (let i = 0; i < 10; i++) {
  await page.evaluate(() => window.scrollBy(0, 100));
  await page.waitForTimeout(100);
}

// Close context to save video
await context.close();
```

### Testing Reduced Motion Preference

```typescript
test('respects prefers-reduced-motion', async ({ page }) => {
  // Emulate reduced motion preference
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('https://example.com');

  // Verify animations are disabled or simplified
  const animationDuration = await page.evaluate(() => {
    const el = document.querySelector('.animated-element');
    return getComputedStyle(el).animationDuration;
  });

  // Should be 0s or very short when reduced motion is preferred
  expect(parseFloat(animationDuration)).toBeLessThanOrEqual(0.01);
});
```

### Detecting Layout Thrashing During Scroll

```typescript
// Inject performance observer before scrolling
await page.evaluate(() => {
  window.__layoutShifts = [];
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      window.__layoutShifts.push({
        value: entry.value,
        startTime: entry.startTime,
        hadRecentInput: entry.hadRecentInput,
        sources: entry.sources?.map(s => ({
          node: s.node?.nodeName,
          previousRect: s.previousRect,
          currentRect: s.currentRect,
        })),
      });
    }
  });
  observer.observe({ type: 'layout-shift', buffered: true });
});

// Perform scroll
for (let i = 0; i < 20; i++) {
  await page.evaluate(() => window.scrollBy(0, 50));
  await page.waitForTimeout(50);
}

// Collect layout shift data
const layoutShifts = await page.evaluate(() => window.__layoutShifts);
const totalCLS = layoutShifts
  .filter(s => !s.hadRecentInput)
  .reduce((sum, s) => sum + s.value, 0);

console.log('Layout shifts during scroll:', layoutShifts.length);
console.log('Total CLS during scroll:', totalCLS);
```

### Disabling Animations for Faster Testing

```typescript
// Disable all animations via CSS injection
await page.addStyleTag({
  content: `
    *, *::before, *::after {
      animation-duration: 0.001ms !important;
      animation-delay: 0ms !important;
      transition-duration: 0.001ms !important;
      transition-delay: 0ms !important;
    }
  `,
});
```

---

## 5. Performance Testing on Mobile

### Measuring Core Web Vitals

#### Largest Contentful Paint (LCP)

```typescript
const lcp = await page.evaluate(() => {
  return new Promise((resolve) => {
    new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lastEntry = entries[entries.length - 1];
      resolve(lastEntry.startTime);
    }).observe({ type: 'largest-contentful-paint', buffered: true });
  });
});

console.log(`LCP: ${lcp}ms`);
// Good: < 2500ms | Needs Improvement: 2500-4000ms | Poor: > 4000ms
```

#### Cumulative Layout Shift (CLS)

```typescript
const cls = await page.evaluate(() => {
  return new Promise((resolve) => {
    let CLS = 0;
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!entry.hadRecentInput) {
          CLS += entry.value;
        }
      }
    });
    observer.observe({ type: 'layout-shift', buffered: true });

    // Wait a bit for all shifts to be recorded
    setTimeout(() => resolve(CLS), 3000);
  });
});

console.log(`CLS: ${cls}`);
// Good: < 0.1 | Needs Improvement: 0.1-0.25 | Poor: > 0.25
```

#### First Contentful Paint (FCP)

```typescript
const fcp = await page.evaluate(() => {
  const paintEntries = performance.getEntriesByType('paint');
  const fcpEntry = paintEntries.find(e => e.name === 'first-contentful-paint');
  return fcpEntry ? fcpEntry.startTime : null;
});

console.log(`FCP: ${fcp}ms`);
// Good: < 1800ms | Needs Improvement: 1800-3000ms | Poor: > 3000ms
```

#### Time to First Byte (TTFB)

```typescript
const ttfb = await page.evaluate(() => {
  const nav = performance.getEntriesByType('navigation')[0];
  return nav.responseStart - nav.requestStart;
});

console.log(`TTFB: ${ttfb}ms`);
// Good: < 800ms
```

#### Total Blocking Time (TBT) - Lab Metric for FID

```typescript
// Inject observer BEFORE navigation
await page.addInitScript(() => {
  window.__longTasks = [];
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      window.__longTasks.push({
        duration: entry.duration,
        startTime: entry.startTime,
        name: entry.name,
      });
    }
  });
  observer.observe({ type: 'longtask', buffered: true });
});

await page.goto('https://example.com');
await page.waitForLoadState('networkidle');

const tbt = await page.evaluate(() => {
  const LONG_TASK_THRESHOLD = 50; // ms
  return window.__longTasks.reduce((total, task) => {
    const blockingTime = Math.max(0, task.duration - LONG_TASK_THRESHOLD);
    return total + blockingTime;
  }, 0);
});

console.log(`TBT: ${tbt}ms`);
// Good: < 200ms | Needs Improvement: 200-600ms | Poor: > 600ms
```

#### Interaction to Next Paint (INP)

```typescript
const inp = await page.evaluate(() => {
  return new Promise((resolve) => {
    let maxINP = 0;
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > maxINP) {
          maxINP = entry.duration;
        }
      }
    });
    observer.observe({ type: 'event', buffered: true, durationThreshold: 16 });

    setTimeout(() => resolve(maxINP), 5000);
  });
});

console.log(`INP: ${inp}ms`);
// Good: < 200ms | Needs Improvement: 200-500ms | Poor: > 500ms
```

### Complete Performance Test with Throttling

```typescript
import { test, expect, chromium, devices } from '@playwright/test';

test('mobile performance audit', async () => {
  const browser = await chromium.launch();
  const iPhone = devices['iPhone 15'];

  const context = await browser.newContext({
    ...iPhone,
  });

  const page = await context.newPage();

  // Set up CDP session for throttling
  const session = await page.context().newCDPSession(page);

  // Enable performance metrics
  await session.send('Performance.enable');

  // Throttle CPU (4x slowdown simulates mid-range mobile)
  await session.send('Emulation.setCPUThrottlingRate', { rate: 4 });

  // Throttle network to Fast 3G
  await session.send('Network.enable');
  await session.send('Network.emulateNetworkConditions', {
    offline: false,
    downloadThroughput: (1.6 * 1024 * 1024) / 8,
    uploadThroughput: (0.75 * 1024 * 1024) / 8,
    latency: 562,
    connectionType: 'cellular3g',
  });

  // Inject performance observers
  await page.addInitScript(() => {
    window.__perfMetrics = {
      lcp: 0,
      cls: 0,
      fcp: 0,
      longTasks: [],
    };

    new PerformanceObserver((list) => {
      const entries = list.getEntries();
      window.__perfMetrics.lcp = entries[entries.length - 1].startTime;
    }).observe({ type: 'largest-contentful-paint', buffered: true });

    new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!entry.hadRecentInput) {
          window.__perfMetrics.cls += entry.value;
        }
      }
    }).observe({ type: 'layout-shift', buffered: true });

    new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        window.__perfMetrics.longTasks.push(entry.duration);
      }
    }).observe({ type: 'longtask', buffered: true });
  });

  // Navigate and wait
  await page.goto('https://example.com');
  await page.waitForLoadState('networkidle');
  await page.waitForTimeout(3000); // Allow metrics to stabilize

  // Collect all metrics
  const metrics = await page.evaluate(() => {
    const nav = performance.getEntriesByType('navigation')[0];
    const paintEntries = performance.getEntriesByType('paint');
    const fcpEntry = paintEntries.find(e => e.name === 'first-contentful-paint');

    const LONG_TASK_THRESHOLD = 50;
    const tbt = window.__perfMetrics.longTasks.reduce((total, duration) => {
      return total + Math.max(0, duration - LONG_TASK_THRESHOLD);
    }, 0);

    return {
      ttfb: nav.responseStart - nav.requestStart,
      fcp: fcpEntry?.startTime || 0,
      lcp: window.__perfMetrics.lcp,
      cls: window.__perfMetrics.cls,
      tbt: tbt,
      domContentLoaded: nav.domContentLoadedEventEnd - nav.startTime,
      loadComplete: nav.loadEventEnd - nav.startTime,
      longTaskCount: window.__perfMetrics.longTasks.length,
    };
  });

  console.log('Performance Metrics:', metrics);

  // Assert performance budgets
  expect(metrics.lcp).toBeLessThan(2500);
  expect(metrics.fcp).toBeLessThan(1800);
  expect(metrics.cls).toBeLessThan(0.1);
  expect(metrics.ttfb).toBeLessThan(800);
  expect(metrics.tbt).toBeLessThan(200);

  // Get Chrome DevTools metrics
  const cdpMetrics = await session.send('Performance.getMetrics');
  console.log('CDP Metrics:', cdpMetrics.metrics);

  await browser.close();
});
```

### Performance Trace Capture

```typescript
// Start a performance trace
const session = await page.context().newCDPSession(page);

await session.send('Tracing.start', {
  categories: [
    'devtools.timeline',
    'disabled-by-default-devtools.timeline',
    'disabled-by-default-devtools.timeline.frame',
  ].join(','),
});

await page.goto('https://example.com');
await page.waitForLoadState('networkidle');

// Stop tracing and save
const trace = await session.send('Tracing.end');
// The trace data arrives via the Tracing.tracingComplete event
```

### Playwright Built-in Tracing

```typescript
// Start tracing
await context.tracing.start({
  screenshots: true,
  snapshots: true,
  sources: true,
});

await page.goto('https://example.com');
// ... perform actions ...

// Stop and save trace
await context.tracing.stop({ path: 'trace.zip' });
// View with: npx playwright show-trace trace.zip
```

---

## 6. Accessibility Testing

### Setup: Axe-Core Integration

```bash
npm install @axe-core/playwright
```

### Basic Accessibility Scan

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('mobile page passes accessibility checks', async ({ page }) => {
  await page.goto('https://example.com');

  const results = await new AxeBuilder({ page }).analyze();

  // Assert no violations
  expect(results.violations).toEqual([]);
});
```

### WCAG-Specific Scanning

```typescript
test('WCAG 2.1 AA compliance', async ({ page }) => {
  await page.goto('https://example.com');

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();

  // Log violations for debugging
  for (const violation of results.violations) {
    console.log(`${violation.id}: ${violation.description}`);
    console.log(`  Impact: ${violation.impact}`);
    console.log(`  Nodes: ${violation.nodes.length}`);
    for (const node of violation.nodes) {
      console.log(`    - ${node.target.join(' > ')}`);
      console.log(`      ${node.failureSummary}`);
    }
  }

  expect(results.violations).toEqual([]);
});
```

### Scanning Specific Page Sections

```typescript
test('mobile navigation accessibility', async ({ page }) => {
  await page.goto('https://example.com');

  // Open mobile menu first
  await page.getByRole('button', { name: 'Menu' }).click();
  await page.locator('#mobile-nav').waitFor();

  // Scan only the mobile navigation
  const results = await new AxeBuilder({ page })
    .include('#mobile-nav')
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### Excluding Known Issues

```typescript
const results = await new AxeBuilder({ page })
  .exclude('#third-party-widget')       // Exclude elements
  .disableRules(['duplicate-id'])        // Disable specific rules
  .withTags(['wcag2a', 'wcag2aa'])
  .analyze();
```

### Keyboard/Focus Navigation Testing

```typescript
test('keyboard navigation works on mobile menu', async ({ page }) => {
  await page.goto('https://example.com');

  // Tab through interactive elements
  await page.keyboard.press('Tab');
  let focused = await page.evaluate(() => {
    const el = document.activeElement;
    return {
      tag: el.tagName,
      text: el.textContent?.trim(),
      role: el.getAttribute('role'),
      ariaLabel: el.getAttribute('aria-label'),
    };
  });
  console.log('First tab stop:', focused);

  // Continue tabbing and collect focus order
  const focusOrder = [];
  for (let i = 0; i < 20; i++) {
    await page.keyboard.press('Tab');
    const info = await page.evaluate(() => {
      const el = document.activeElement;
      return {
        tag: el.tagName,
        text: el.textContent?.trim().substring(0, 40),
        visible: el.offsetParent !== null,
        hasOutline: getComputedStyle(el).outlineStyle !== 'none',
      };
    });
    focusOrder.push(info);

    // Check that focused element is visible
    expect(info.visible).toBe(true);
  }

  console.log('Focus order:', focusOrder);
});
```

### Focus Trap Detection

```typescript
test('modal does not trap focus', async ({ page }) => {
  await page.goto('https://example.com');

  // Open modal
  await page.click('#open-modal');
  await page.locator('.modal').waitFor();

  // Tab through all elements in the modal
  const focusedElements = new Set();
  let loopDetected = false;

  for (let i = 0; i < 50; i++) {
    await page.keyboard.press('Tab');
    const activeId = await page.evaluate(() => {
      const el = document.activeElement;
      return el.id || el.className || el.tagName;
    });

    if (focusedElements.has(activeId) && focusedElements.size < 3) {
      loopDetected = true;
      break;
    }
    focusedElements.add(activeId);
  }

  // For modals: focus SHOULD be trapped within the modal
  // For other components: focus should NOT be trapped
  console.log('Unique focus targets:', focusedElements.size);
  console.log('Focus loop detected:', loopDetected);
});
```

### ARIA Attribute Verification

```typescript
test('ARIA attributes are correctly set', async ({ page }) => {
  await page.goto('https://example.com');

  // Check ARIA on interactive elements
  const ariaIssues = await page.evaluate(() => {
    const issues = [];

    // Check buttons
    document.querySelectorAll('button').forEach(btn => {
      if (!btn.textContent?.trim() && !btn.getAttribute('aria-label') && !btn.getAttribute('aria-labelledby')) {
        issues.push({ element: 'button', class: btn.className, issue: 'No accessible name' });
      }
    });

    // Check images
    document.querySelectorAll('img').forEach(img => {
      if (!img.getAttribute('alt') && img.getAttribute('role') !== 'presentation') {
        issues.push({ element: 'img', src: img.src, issue: 'Missing alt text' });
      }
    });

    // Check form inputs
    document.querySelectorAll('input, select, textarea').forEach(input => {
      const id = input.id;
      const label = id ? document.querySelector(`label[for="${id}"]`) : null;
      const ariaLabel = input.getAttribute('aria-label');
      const ariaLabelledBy = input.getAttribute('aria-labelledby');

      if (!label && !ariaLabel && !ariaLabelledBy) {
        issues.push({
          element: input.tagName,
          type: input.type,
          name: input.name,
          issue: 'No associated label or aria-label',
        });
      }
    });

    return issues;
  });

  console.log('ARIA issues:', ariaIssues);
  expect(ariaIssues).toEqual([]);
});
```

### Color Contrast Checking

```typescript
test('text has sufficient color contrast', async ({ page }) => {
  await page.goto('https://example.com');

  // Use axe-core for automated contrast checking
  const results = await new AxeBuilder({ page })
    .withRules(['color-contrast'])
    .analyze();

  for (const violation of results.violations) {
    for (const node of violation.nodes) {
      console.log(`Contrast issue: ${node.target.join(' > ')}`);
      console.log(`  ${node.failureSummary}`);
    }
  }

  expect(results.violations).toEqual([]);
});

// Manual contrast check for specific elements
const contrastData = await page.evaluate(() => {
  const results = [];
  const textElements = document.querySelectorAll('p, h1, h2, h3, h4, h5, h6, span, a, li, td, th, label');

  for (const el of Array.from(textElements).slice(0, 50)) {
    const style = getComputedStyle(el);
    results.push({
      text: el.textContent?.trim().substring(0, 30),
      color: style.color,
      backgroundColor: style.backgroundColor,
      fontSize: style.fontSize,
      fontWeight: style.fontWeight,
    });
  }
  return results;
});
```

### Reusable Accessibility Fixture

```typescript
// fixtures.ts
import { test as base } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

export const test = base.extend({
  makeAxeBuilder: async ({ page }, use) => {
    const makeAxeBuilder = () =>
      new AxeBuilder({ page })
        .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
        .exclude('#cookie-banner')
        .exclude('#third-party-chat');
    await use(makeAxeBuilder);
  },
});

// usage in tests:
test('page is accessible', async ({ page, makeAxeBuilder }) => {
  await page.goto('https://example.com');
  const results = await makeAxeBuilder().analyze();
  expect(results.violations).toEqual([]);
});
```

---

## 7. Best Practices for Claude Code Visual Analysis

### The Core Principle

> "The single highest-leverage thing you can do is include tests, screenshots, or expected outputs so Claude can check itself. Claude performs dramatically better when it can verify its own work."

### Recommended Viewport Sizes to Test

| Breakpoint | Width | Description |
|-----------|-------|-------------|
| 320px | 320x568 | Small mobile (iPhone SE) |
| 375px | 375x812 | Standard mobile (iPhone 12/13/14) |
| 393px | 393x852 | Modern mobile (iPhone 15) |
| 414px | 414x896 | Large mobile (iPhone Plus) |
| 430px | 430x932 | Extra large mobile (iPhone Pro Max) |
| 768px | 768x1024 | Tablet portrait (iPad) |
| 834px | 834x1194 | Tablet (iPad Pro 11") |
| 1024px | 1024x768 | Tablet landscape |
| 1280px | 1280x720 | Small desktop |
| 1920px | 1920x1080 | Standard desktop |

### Systematic Page Analysis Workflow

**Step 1: Take a full-page screenshot at each key viewport**

```
For each breakpoint (375, 768, 1024, 1920):
  1. Navigate to the page
  2. Resize viewport to breakpoint
  3. Take a full-page screenshot
  4. Take a viewport-only screenshot (above the fold)
```

**Step 2: Check for horizontal overflow**

```
Use browser_evaluate:
  document.documentElement.scrollWidth > document.documentElement.clientWidth
```

**Step 3: Scroll through and capture sections**

```
For each section of the page:
  1. Scroll to section
  2. Wait 300ms for scroll-triggered animations
  3. Take viewport screenshot
  4. Check for layout issues
```

**Step 4: Test interactive states**

```
For mobile menu:
  1. Screenshot closed state
  2. Click hamburger menu
  3. Screenshot open state
  4. Navigate through menu items

For forms:
  1. Screenshot empty state
  2. Fill with test data
  3. Screenshot filled state
  4. Submit and screenshot result
```

### What to Look for in Screenshots

When Claude Code analyzes screenshots, it should check for:

1. **Layout Issues**
   - Elements overflowing the viewport horizontally
   - Text truncated or overlapping
   - Images stretched, squished, or breaking layout
   - Uneven spacing or alignment problems
   - Content hidden behind fixed headers/footers

2. **Typography**
   - Text too small to read on mobile (< 16px body text)
   - Line lengths too long (> 70-80 characters per line)
   - Poor contrast between text and background
   - Orphaned words or awkward line breaks

3. **Navigation**
   - Hamburger menu visible and functional on mobile
   - Touch targets too small (< 44x44px)
   - Navigation items too close together
   - Dropdown menus clipped or overflowing

4. **Images and Media**
   - Images not responsive (fixed width breaking layout)
   - Images too large (wasting bandwidth)
   - Missing alt text indicators
   - Videos not fitting viewport

5. **Forms**
   - Input fields too small for touch
   - Labels not associated or not visible
   - Error messages not visible
   - Submit buttons too small

### Screenshot Quality Guidelines

- Use images at least 1000x1000 pixels for detailed analysis
- Capture at the device's native scale factor (2x or 3x for mobile)
- Save as PNG for UI analysis (lossless compression)
- For comparison, use the same viewport and scroll position

### Practical Workflow with Playwright MCP

```
1. "Use playwright MCP to open https://example.com with device iPhone 15"
2. "Take a full-page screenshot"
3. "Check if there is horizontal overflow using browser_evaluate"
4. "Get all elements wider than the viewport"
5. "Scroll to the bottom of the page and take another screenshot"
6. "Resize to 768px wide and take a full-page screenshot"
7. "Click the mobile menu button and take a screenshot"
8. "Run browser_evaluate to check all touch target sizes"
```

### MCP Configuration Presets for Testing

#### Mobile-First Testing

```json
{
  "mcpServers": {
    "mobile-safari": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--device", "iPhone 15", "--headless", "--caps", "vision"]
    },
    "mobile-android": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--device", "Pixel 7", "--headless", "--caps", "vision"]
    },
    "tablet": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--device", "iPad Pro 11", "--headless", "--caps", "vision"]
    }
  }
}
```

### Iterative Visual Feedback Loop

The most effective workflow for Claude Code:

1. **Build** -- Claude Code makes frontend changes
2. **Render** -- Run through a real browser via Playwright MCP
3. **Capture** -- Take screenshots of the result
4. **Inspect** -- Claude Code analyzes the screenshots for issues
5. **Fix** -- Claude Code fixes any issues found
6. **Repeat** -- Continue until the page looks correct

This round-trip verification dramatically improves the quality of Claude Code's frontend work.

---

## Sources

### Playwright MCP
- [Microsoft Playwright MCP - GitHub](https://github.com/microsoft/playwright-mcp)
- [How to Use Playwright MCP Server with Claude Code - Builder.io](https://www.builder.io/blog/playwright-mcp-server-claude-code)
- [Using Playwright MCP with Claude Code - Simon Willison](https://til.simonwillison.net/claude-code/playwright-mcp-claude-code)
- [Taking screenshots with Playwright MCP - Shipyard](https://shipyard.build/blog/playwright-mcp-screenshots/)
- [Playwright MCP Claude Code - Testomat](https://testomat.io/blog/playwright-mcp-claude-code/)

### Mobile Emulation
- [Emulation - Playwright Official Docs](https://playwright.dev/docs/emulation)
- [Emulating Mobile Devices - Checkly Docs](https://www.checklyhq.com/docs/learn/playwright/emulating-mobile-devices/)
- [Device Descriptors Source - Playwright GitHub](https://github.com/microsoft/playwright/blob/main/packages/playwright-core/src/server/deviceDescriptorsSource.json)
- [Mobile Web Testing Best Practices - Alphabin](https://www.alphabin.co/blog/using-playwright-for-mobile-web-testing)

### Visual Testing
- [Visual Comparisons - Playwright Official Docs](https://playwright.dev/docs/test-snapshots)
- [Visual Regression Testing Guide - TestGrid](https://testgrid.io/blog/playwright-visual-regression-testing/)
- [Screenshots - Playwright Official Docs](https://playwright.dev/docs/screenshots)
- [Playwright Visual Testing - Codoid](https://codoid.com/automation-testing/playwright-visual-testing-a-comprehensive-guide-to-ui-regression/)

### Animation Testing
- [Automating Animation Testing with Playwright - The Green Report](https://www.thegreenreport.blog/articles/automating-animation-testing-with-playwright-a-practical-guide/automating-animation-testing-with-playwright-a-practical-guide.html)

### Performance Testing
- [Testing Web Application Performance - FocusReactive](https://focusreactive.com/testing-web-application-performance-with-playwright/)
- [Measuring Page Performance - Checkly Docs](https://www.checklyhq.com/docs/learn/playwright/performance/)
- [Performance Testing with Playwright - TestingBot](https://testingbot.com/support/web-automate/playwright/performance)
- [Network Throttling in Playwright - SDetective](https://sdetective.blog/blog/qa_auto/pw-cdp/networking-throttle_en)

### Accessibility Testing
- [Accessibility Testing - Playwright Official Docs](https://playwright.dev/docs/accessibility-testing)
- [axe-playwright - NPM](https://www.npmjs.com/package/axe-playwright)
- [WCAG 2.5.8 Target Size - W3C](https://www.w3.org/WAI/WCAG21/Understanding/target-size.html)

### Claude Code Visual Analysis
- [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Giving Claude Code Eyes: Round-Trip Screenshot Testing](https://medium.com/@rotbart/giving-claude-code-eyes-round-trip-screenshot-testing-ce52f7dcc563)
- [Using Claude Code to Debug Visual Regressions - Vizzly](https://vizzly.dev/blog/claude-code-ai-visual-testing/)
