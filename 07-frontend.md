# 07 — Frontend: CSS & JavaScript

> The marketing/CMS frontend is pure SSR — no build step, no Vue, no bundler.
> CSS is organised into 4 layers. JavaScript is 3 small files plus CDN-loaded EditorJS.

---

## CSS Architecture

```
design-system.css  (~1370 lines)   Tokens, resets, base typography, app-wide components
       │
       ├── marketing.css  (~1250 lines)   Public marketing page styles (blocks, blog, nav, footer)
       │
        ├── cms.css  (~1360 lines)          CMS admin panel styles (sidebar, forms, editor, dashboard, action bars)
       │
       └── homepage.css             Homepage-only styles (hero, mock widgets, sections)
```

### Loading rules

| Template | CSS loaded |
|----------|-----------|
| `marketing/base.html` (public pages) | `design-system.css` + `marketing.css` |
| `marketing/home.html` (homepage, standalone) | `design-system.css` + `homepage.css` |
| `marketing/cms/base.html` (CMS admin) | `design-system.css` + `cms.css` |

The homepage is a standalone template (`home.html` extends nothing — it's self-contained) and does not load `marketing.css`.

---

## Production Static Assets (Manifest Storage)

This repo uses WhiteNoise manifest static storage in production. That means any change to files under `static/` requires `collectstatic` to regenerate hashed filenames.

If you edit `static/js/cms-editor.js`, `static/css/*.css`, or any images, ensure your deploy/build pipeline runs:

```bash
python manage.py collectstatic --noinput
```

Otherwise the CMS may reference stale hashed asset paths and break.

---

## Porting A Full Site From A Provided Design (Non-Negotiables)

This is a common failure mode when "turning a design into a CMS". The goal is not "close enough"; it's a faithful port.

- Match the provided typography (families, weights, line-heights) before tweaking layout
- Reproduce spacing/scale as a system (tokens), not one-off pixel fixes
- Keep component proportions and radii consistent; don't approximate cards/buttons
- Validate responsive breakpoints against the design (mobile/tablet/desktop), not just one viewport
- Preserve the intended visual direction; do not swap fonts/colors unless explicitly approved

### Making the Homepage Editable via CMS

When your design includes a complex homepage with multiple sections (hero, features, testimonials, etc.), you have two options:

**Option A: Standalone Homepage (Simpler)**
Keep the homepage as a standalone template (`home.html`) with hardcoded content. This works when the homepage is a "landing page" that doesn't need frequent updates. Updates require developer involvement.

**Option B: Homepage as CMS Blocks (Recommended for Frequent Updates)**
If the homepage needs to be editable by non-developers, create blocks for each major section:

1. **Create homepage-specific block types** in `blocks.py` and `cms-editor.js`:
   - `hero_section` — headline, subheadline, CTA buttons, background image
   - `feature_showcase` — icon + title + description grid
   - `testimonial_slider` — customer quotes with photos
   - `cta_band` — call-to-action section
   - `stats_row` — key metrics/numbers

2. **Add a "Homepage" model or use Page with a special flag** to enable homepage-specific blocks

3. **Render homepage blocks in order** — the view queries for the homepage and renders each block type in sequence

> **Important**: Don't assume standard CMS blocks (rich_text, callout, etc.) are sufficient for homepage designs. Most professional designs have unique sections (hero with chat widget mockup, trust bar with logos, animated feature showcases) that need custom block types.

### Adding Drag-and-Drop to Page/Blog Lists

The CMS page and blog post lists should support drag-and-drop reordering for:
- **Ordering pages** — When you have many pages, being able to drag-and-drop to set display order is essential
- **Blog post ordering** — Particularly important for featured posts at the top

Add an `order` field to your Page/Post models and update list views to support drag-and-drop reordering via AJAX. This is a common missing feature that makes content management tedious.

> **Tip**: The blocks builder already supports drag-and-drop for reordering blocks within a page. Ensure both "Add block" bars (top and bottom) are always visible so users can add new blocks at any position, not just at the extremes.

---

## Design Tokens (`design-system.css`)

All tokens are CSS custom properties on `:root`. The exact palette and typography should be derived from the provided design system.

### Raw palette

```css
--d-obsidian: #0F172A;          /* Main background */
--d-obsidian-light: #1E293B;    /* Elevated surfaces */
--d-obsidian-mid: #334155;      /* Secondary surfaces */

--d-stone: #F8FAFC;             /* Primary text */
--d-stone-dim: #E2E8F0;         /* Secondary text */
--d-stone-mid: #94A3B8;         /* Muted text */

--d-amber: #F59E0B;             /* Primary accent / CTAs */
--d-amber-soft: #FDE68A;        /* Soft amber for highlights */
--d-amber-glow: rgba(245, 158, 11, 0.15);

--d-violet: #8B5CF6;            /* AI/secondary accent */
--d-violet-bright: #A78BFA;     /* Lighter violet */
--d-violet-glow: rgba(139, 92, 246, 0.12);

--d-teal: #0D9488;              /* Success / confirmation */
--d-teal-glow: rgba(13, 148, 136, 0.15);

--d-bdr: rgba(255, 255, 255, 0.08);       /* Default border */
--d-bdr-hover: rgba(255, 255, 255, 0.16); /* Hover border */
```

### Semantic tokens (preferred for use)

```css
--color-bg: var(--d-obsidian);
--color-bg-elevated: var(--d-obsidian-light);
--color-bg-elevated-2: var(--d-obsidian-mid);

--color-text: var(--d-stone);
--color-text-dim: var(--d-stone-dim);
--color-text-muted: var(--d-stone-mid);

--color-primary: var(--d-amber);        /* Primary brand color */
--color-ai: var(--d-violet);            /* AI-related elements */
--color-success: var(--d-teal);         /* Success states */
--color-danger: #EF4444;               /* Errors */

--color-border: var(--d-bdr);
--color-border-hover: var(--d-bdr-hover);
--color-divider: rgba(255, 255, 255, 0.06);
```

### Legacy aliases

The design system maintains backward-compatible aliases so old templates still work:

```css
--brand-blue: var(--color-primary);
--brand-purple: var(--color-ai);
--bg-void: var(--color-bg);
--bg-surface: var(--color-bg-elevated);
--text-primary: var(--color-text);
--text-secondary: var(--color-text-dim);
```

### Fonts

```css
--font-display: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
--font-body: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
--font-mono: 'Space Mono', ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
--font-signature: 'Instrument Serif', Georgia, 'Times New Roman', serif;
```

Loaded via Google Fonts import at the top of `design-system.css`:

```css
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Space+Mono:wght@400;700&family=Instrument+Serif:ital@0;1&display=swap');
```

---

## Key CSS Class Names

### Block classes (from `marketing.css`, rendered by `renderers.py`)

| Block type | CSS class | Notes |
|-----------|-----------|-------|
| rich_text | `.cms-prose` | Prose styles for headings, paragraphs, lists |
| cta | `.cms-cta` | Call-to-action band with button |
| callout | `.cms-callout` | Highlighted box with amber left border |
| feature_grid | `.cms-feature-grid` | CSS Grid of feature cards |
| faq | `.cms-faq` | Accordion-style FAQ items |
| quote | `.cms-quote` | Styled blockquote with author |
| comparison_table | `.cms-comparison-table` | Styled data table |
| pricing_table | `.cms-pricing` | Pricing plan cards in grid |
| logo_cloud | `.cms-logo-cloud` | Logo row with grayscale filter |
| image_gallery | `.cms-gallery-wrap` | Image grid with captions |
| table | `.cms-table-wrap` | Generic table wrapper |

### Homepage section classes (from `homepage.css`)

| Section | CSS class | Purpose |
|---------|-----------|---------|
| Hero | `.d-hero` | Above-the-fold hero with mock widget |
| Trust bar | `.d-trust` | Logo bar showing trusted companies |
| Section heading | `.d-sh` | Reusable section heading component |
| How it works | `.d-how` | Three-step explanation |
| Contract types | `.d-contracts` | Grid of contract type cards |
| CTA band | `.d-cta-band` | Full-width call-to-action |
| Quality/stats | `.d-quality` | Statistics/metrics section |
| Features | `.d-features` | Feature showcase grid |
| Footer disclaimer | `.d-footer-disc` | Legal disclaimer above footer |
| Footer | `.d-footer` | Site footer |

### Homepage layout container

```css
.version-a {
  --content-max: 1240px;
  --gutter: clamp(18px, 4vw, 48px);
}
```

All sections constrained to `max-width: var(--content-max)` with auto margins.

### CMS action bar & inline button classes (from `cms.css`)

| CSS Class | Purpose |
|-----------|---------|
| `.cms-form-actions-bar` | Flexbox bar at top/bottom of form templates (card-bg background, border, rounded) |
| `.cms-form-actions-left` | Left-aligned flex container (Save Draft, Publish/Unpublish buttons) |
| `.cms-form-actions-right` | Right-aligned flex container (Delete button) |
| `.cms-btn-warning` | Amber warning button for "Unpublish" action |
| `.cms-confirm-delete` | Inline danger banner (red background) for delete confirmation in forms |
| `.cms-confirm-actions` | Flex container for Cancel/Confirm buttons inside delete banner |
| `.cms-inline-form` | `display: inline` wrapper for forms inside table cells and dashboard lists |
| `.cms-btn-inline` | Minimal unstyled button base for inline row actions |
| `.cms-btn-inline-success` | Teal colored inline button (Publish) |
| `.cms-btn-inline-warning` | Amber colored inline button (Unpublish) |
| `.cms-btn-inline-danger` | Red colored inline button (Delete) |

---

## JavaScript Files

### Overview

| File | Lines | Purpose |
|------|-------|---------|
| `static/js/scroll-reveal.js` | 164 | Scroll-triggered reveal animations + image fade-in |
| `static/js/cms-editor.js` | 919 | Block editor for CMS admin forms |
| `static/js/cms-json.js` | 39 | JSON field collapse/expand for CMS forms |

`scroll-reveal.js` is loaded on public marketing pages.
`cms-editor.js` and `cms-json.js` are loaded on CMS admin pages.

---

## Complete Source: `static/js/scroll-reveal.js`

```javascript
/**
 * scroll-reveal.js
 * ---------------------------------------------------------
 * Lightweight scroll-reveal + image-fade system.
 *
 * 1. Elements with [data-reveal] fade/slide up as they enter the viewport.
 * 2. Images with [data-img-reveal] stay invisible until loaded, then fade in.
 * 3. On page load, any element with .reveal-on-load gets the entrance animation.
 *
 * Auto-initialises on DOMContentLoaded. No dependencies.
 */

(function () {
  'use strict';

  /* -- Config ------------------------------------------------ */
  var REVEAL_THRESHOLD = 0.12;        // 12% visible triggers reveal
  var REVEAL_ROOT_MARGIN = '0px 0px -40px 0px';  // slight bottom inset
  var STAGGER_DELAY = 80;             // ms between staggered children

  /* -- Scroll reveal (IntersectionObserver) ------------------- */
  function initScrollReveal() {
    var els = document.querySelectorAll('[data-reveal]');
    if (!els.length) return;

    // Fallback: if IntersectionObserver not supported, reveal all immediately
    if (!('IntersectionObserver' in window)) {
      els.forEach(function (el) { el.classList.add('revealed'); });
      return;
    }

    var observer = new IntersectionObserver(function (entries) {
      entries.forEach(function (entry) {
        if (!entry.isIntersecting) return;
        var el = entry.target;

        // Support staggered children: [data-reveal="stagger"]
        if (el.getAttribute('data-reveal') === 'stagger') {
          var children = el.querySelectorAll('[data-reveal-child]');
          children.forEach(function (child, i) {
            child.style.transitionDelay = (i * STAGGER_DELAY) + 'ms';
            child.classList.add('revealed');
          });
        }

        el.classList.add('revealed');
        observer.unobserve(el);
      });
    }, {
      threshold: REVEAL_THRESHOLD,
      rootMargin: REVEAL_ROOT_MARGIN,
    });

    els.forEach(function (el) { observer.observe(el); });
  }

  /* -- Image fade-in on load --------------------------------- */
  function initImageReveal() {
    var imgs = document.querySelectorAll('img[data-img-reveal]');
    if (!imgs.length) {
      // Auto-apply to common marketing images if no explicit attribute
      imgs = document.querySelectorAll(
        '.cms-post-cover img, .cms-post-card-img img, .cms-gallery-item img'
      );
    }

    imgs.forEach(function (img) {
      // Already loaded (cached)
      if (img.complete && img.naturalWidth > 0) {
        img.classList.add('img-revealed');
        return;
      }

      // Not yet loaded -- start invisible
      img.classList.add('img-loading');

      img.addEventListener('load', function () {
        // Small RAF delay so the browser has painted the decoded frame
        requestAnimationFrame(function () {
          img.classList.remove('img-loading');
          img.classList.add('img-revealed');
        });
      });

      img.addEventListener('error', function () {
        // On error, still reveal so layout isn't broken
        img.classList.remove('img-loading');
        img.classList.add('img-revealed');
      });
    });
  }

  /* -- Auto-reveal sections on marketing pages --------------- */
  function autoMarkRevealTargets() {
    // Mark common block-level marketing elements for scroll-reveal
    // so we don't have to add data-reveal to every template manually.
    var selectors = [
      '.cms-prose',
      '.cms-callout',
      '.cms-cta',
      '.cms-feature-grid',
      '.cms-comparison-table',
      '.cms-table-wrap',
      '.cms-faq',
      '.cms-quote',
      '.cms-logo-cloud',
      '.cms-pricing',
      '.cms-gallery-wrap',
      '.cms-post-cover',
      '.cms-post-card',
      '.d-hero',
      '.d-trust',
      '.d-sh',
      '.d-how',
      '.d-contracts',
      '.d-cta-band',
      '.d-quality',
      '.d-features',
      '.d-footer-disc',
      '.d-footer',
    ];

    var all = document.querySelectorAll(selectors.join(','));
    all.forEach(function (el) {
      if (!el.hasAttribute('data-reveal')) {
        el.setAttribute('data-reveal', '');
      }
    });

    // Stagger grid children
    var grids = document.querySelectorAll(
      '.cms-feature-grid, .cms-pricing, .d-contracts-grid, .d-fg, .cms-post-grid'
    );
    grids.forEach(function (grid) {
      grid.setAttribute('data-reveal', 'stagger');
      Array.prototype.forEach.call(grid.children, function (child) {
        child.setAttribute('data-reveal-child', '');
      });
    });
  }

  /* -- Page-load entrance ------------------------------------ */
  function initPageEntrance() {
    document.documentElement.classList.add('page-ready');
  }

  /* -- Init -------------------------------------------------- */
  function init() {
    autoMarkRevealTargets();
    initScrollReveal();
    initImageReveal();
    // Slight delay for page entrance so CSS transition can catch
    requestAnimationFrame(function () {
      requestAnimationFrame(initPageEntrance);
    });
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', init);
  } else {
    init();
  }
})();
```

### How scroll-reveal works

1. **`autoMarkRevealTargets()`** runs first. It scans the DOM for known block/section CSS classes and automatically adds `data-reveal` attributes. Grid containers get `data-reveal="stagger"` and their children get `data-reveal-child`. This means **you never need to manually add `data-reveal` to templates** — the script does it automatically for all known block types.

2. **`initScrollReveal()`** creates one `IntersectionObserver` watching all `[data-reveal]` elements. When 12% of an element enters the viewport (with a 40px bottom inset), it adds the `.revealed` class. The element is then unobserved (one-shot animation).

3. **Staggered reveals** — when `data-reveal="stagger"`, each child with `[data-reveal-child]` gets an increasing `transition-delay` (0ms, 80ms, 160ms, ...) before getting `.revealed`. This creates the cascade effect on feature grids and pricing cards.

4. **`initImageReveal()`** handles image fade-in. It auto-applies to images inside `.cms-post-cover`, `.cms-post-card-img`, and `.cms-gallery-item`. Images start with `.img-loading` (invisible), then get `.img-revealed` on the `load` event (with a `requestAnimationFrame` delay for smooth painting). Errors also trigger reveal to prevent layout breakage.

5. **`initPageEntrance()`** adds `.page-ready` to `<html>` after two rAF ticks, enabling any CSS page-entrance animations.

### Required CSS (in `marketing.css`)

The JS relies on these CSS rules existing:

```css
[data-reveal] {
  opacity: 0;
  transform: translateY(24px);
  transition: opacity 0.6s ease, transform 0.6s ease;
}

[data-reveal].revealed,
[data-reveal-child].revealed {
  opacity: 1;
  transform: translateY(0);
}

.img-loading {
  opacity: 0;
  transition: opacity 0.4s ease;
}

.img-revealed {
  opacity: 1;
}
```

---

## Mobile Menu Pattern

The navigation bar includes a mobile hamburger menu. The pattern:

- **HTML** — `<button class="d-nav-toggle">` toggles `.menu-open` on the `<nav>` element
- **CSS** — the nav links are hidden on mobile and shown in a dropdown panel when `.menu-open` is active
- **No JS file** — the toggle is a tiny inline `onclick` handler in `base.html`:

```html
<button class="d-nav-toggle" onclick="this.closest('nav').classList.toggle('menu-open')">
  <span></span><span></span><span></span>
</button>
```

The three `<span>` elements are styled as hamburger lines and animate to an X on `.menu-open`.

---

## Replicating the Frontend in a New Project

1. Copy `design-system.css` as your token foundation — update colors as needed
2. Keep the semantic token layer (`--color-bg`, `--color-primary`, etc.) — it makes theming trivial
3. Copy `scroll-reveal.js` as-is — update the `selectors` array in `autoMarkRevealTargets()` to match your block class names
4. The image reveal system works automatically for any images inside containers matching the selector list
5. Load Google Fonts (Inter + Space Mono + Instrument Serif) or swap to your own
6. The homepage uses a standalone template pattern — good for landing pages that need different layouts
