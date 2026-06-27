---
name: ios-theme-color
description: "Diagnose and fix Safari iOS rendering issues related to theme-color, Liquid Glass, safe areas, viewport, and browser chrome. Three-layer defense-in-depth for cross-browser toolbar color control."
---

# Skill: Next.js + Safari iOS UI Chrome Expert

## Mission

Diagnose and fix Safari iOS rendering issues related to:
- browser toolbar
- address bar
- status bar
- Liquid Glass
- theme-color
- header rendering
- safe areas
- viewport behavior

Always identify whether the issue is caused by CSS, Next.js, or Safari/WebKit before modifying code.

## Technical Context

Default stack:
- Next.js 14+
- App Router
- React
- Safari iOS
- WebKit

## Next.js Rules

### Use the Viewport API

Use:

```ts
export const viewport = {
  themeColor: "#ffffff",
};
```

Do not introduce `metadata.themeColor` for new implementations.

## Safari iOS Rules

### Safari iOS 26+

Safari's Liquid Glass UI derives browser chrome colors directly from the rendered page.

Assume:
- `<meta name="theme-color">` is no longer the primary source for toolbar colors.
- Runtime JavaScript updates to `<meta name="theme-color">` are ignored.
- Even static theme-color may not determine the final toolbar appearance.

This is expected WebKit behavior.
It is NOT a Next.js limitation.

## CSS is now the source of truth

Safari determines toolbar appearance from what is actually rendered at the top of the viewport.

Always inspect:
- html
- body
- first rendered element
- fixed header
- sticky header

before changing metadata.

## Preferred CSS

Root should have an explicit background.

```css
html,
body {
    background: white;
}
```

If using a fixed or sticky header:

```css
header {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    background: white;
}
```

Solid backgrounds are preferred.

## Defense-in-depth: three-layer approach

No single API works across all Safari versions and browsers. The reliable solution combines three complementary layers:

### Layer 1: Static viewport export (fallback initial value)

```ts
export const viewport: Viewport = {
  themeColor: "#dae2df",
};
```

Use a **single static value** without media queries. Multiple `prefers-color-scheme` media queries conflict with JS-driven updates. A single value gives JS a clean target to override.

### Layer 2: `body::before` CSS-driven fixed element (Safari 26+)

```css
body::before {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 6px;
  z-index: 1;
  pointer-events: none;
  background: var(--color-light);
  transition: background 500ms;
}
html.dark-scheme body::before {
  background: var(--color-dark);
}
```

Safari 26+ Liquid Glass samples `position: fixed` elements at viewport edges. `body::before` is a dedicated invisible fixed element whose background changes via CSS class toggle (no JS style mutations). The CSS `transition` triggers Safari's renderer to re-sample the toolbar color.

### Layer 3: JS-driven `theme-color` meta update (Chrome, older Safari)

```js
document.documentElement.classList.toggle("dark-scheme", isDark)

const meta = document.querySelector('meta[name="theme-color"]')
if (meta) meta.setAttribute("content", isDark ? "#231123" : "#dae2df")
```

JS updates to `theme-color` work reliably on Chrome/Android/Arc and older Safari versions (<26). On Safari 26+ they are ignored, but the `body::before` layer covers that case.

### Why combine all three?

| Layer | Targets | Mechanism |
|-------|---------|-----------|
| Static viewport | Initial HTML render | Single `<meta>` tag without media queries |
| `body::before` | Safari 26+ Liquid Glass | CSS class toggle on `<html>`, re-sampled by WebKit |
| JS meta update | Chrome, Arc, Safari <26 | `setAttribute` driven by scroll/state change |

Do NOT rely on any single layer in isolation.

## Transparency

Avoid transparent top-level backgrounds.

Problematic patterns include:
- `background: transparent`
- `background: rgba(..., 0)`
- `backdrop-filter: blur(...)` without an actual background color

Safari's color sampling may produce unexpected toolbar colors.

## Investigation Workflow

Before editing code determine whether the issue comes from:
- Liquid Glass color sampling
- CSS background
- safe-area
- viewport-fit
- fixed header
- sticky header
- z-index
- stacking context
- backdrop-filter
- color-scheme
- dark mode
- WebKit rendering bug
- Next.js metadata

Never assume theme-color is responsible.

## Safe Area

Verify usage of:

```css
padding-top: env(safe-area-inset-top);
```

when implementing custom headers.

## color-scheme

Check whether `color-scheme: light` or `color-scheme: dark` is influencing Safari UI.
Do not modify it unless necessary.

## PWAs

For installed PWAs or legacy Safari compatibility:

```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
```

Use them only when the application is intended to behave as a PWA or fullscreen web app.
Do not recommend them for ordinary Safari browsing unless explicitly relevant.

## Key constraints for `body::before` layer

- The element must be `position: fixed` or `position: sticky`.
- Must be within 4px from viewport edge.
- Must be at least 3px tall and 80%+ wide on iOS.
- Must have an explicit `background-color`.
- `z-index` must be lower than the page header to remain invisible.
- `pointer-events: none` prevents interaction interference.

## WebKit Bugs

When behavior differs across iOS versions, search for:
- WebKit bugs
- Safari release notes
- Apple documentation

Never invent browser behavior.
Distinguish between: documented behavior, observed browser bugs, community workarounds.

## Output Requirements

Every solution must include:
1. Root cause
2. Whether it is: CSS issue, Next.js issue, WebKit limitation, or Safari bug
3. Confidence level
4. Minimal code diff
5. References when relying on WebKit behavior or Safari-specific limitations

Prefer minimal, standards-based solutions.
When dynamic toolbar colors are required, always use the three-layer defense-in-depth approach:
1. Static viewport export (single value, no media queries)
2. `body::before` CSS-driven fixed element (Safari 26+)
3. JS-driven `theme-color` update (Chrome, older Safari)
Never rely on any single layer alone.
