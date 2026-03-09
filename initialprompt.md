# CMSPack — Initial Agent Prompt

Paste this prompt into your AI coding agent (Claude Opus 4.6, GPT-5.4, GLM-4.7, or any frontier-level model) along with the CMSPack documentation files. Then provide your homepage design (static HTML/CSS, a v0 export, a Figma export, or a Next.js mock) and the agent will build your entire site.

---

## Prompt

You are building a Django marketing website with a custom CMS admin panel. You have been given:

1. **A homepage design** — either static HTML/CSS, a v0/Next.js mock, or a Figma export. This is the source of truth for all visual decisions.
2. **The CMSPack documentation** (files `00-index.md` through `09-finalsecuritycheck.md`) — this is the source of truth for all architecture and implementation decisions.

Your job is to implement a complete, production-ready Django marketing site with a block-based CMS that faithfully reproduces the provided design. Every page on the site — About, Pricing, Contact, Blog, and any others — should be built and editable through the CMS block builder.

---

### Phase 1: Project Setup

1. Create a Django project (or use the existing one if provided).
2. Create a `marketing/` Django app.
3. Install dependencies: `beautifulsoup4`, `cloudinary`, and any others from the docs.
4. Set up the models exactly as described in `02-models.md`:
   - `SeoFieldsMixin`, `MediaAsset`, `Category`, `Tag`, `Page`, `Post`, `NavMenu`, `NavItem`, `Footer`
   - Both `Page` and `Post` must override `save()` to render `blocks_json` to `blocks_html`.
5. Set up forms as described in `02-models.md` (bottom section).
6. Create and run migrations.

---

### Phase 2: CMS Backend

1. Implement `permissions.py` with the `@cms_admin_required` decorator (checks `CMS Admins` group) as described in `03-views-and-urls.md`.
2. Implement `blocks.py` — the server-side block renderer with all 12 block types — exactly as specified in `05-blocks-system.md`.
3. Implement `renderers.py` — the EditorJS HTML renderer and sanitizer — exactly as specified in `05-blocks-system.md`.
4. Implement `seo.py` — the SEO payload builder with the full fallback chain — as specified in `03-views-and-urls.md`. **Important**: Make sure it checks `blocks_html` when `body_html` is empty.
5. Implement `cloudinary_utils.py` as specified in `06-cloudinary.md`.
6. Implement all public views (`views.py`) and CMS admin views (`cms_views.py`) as specified in `03-views-and-urls.md`.
7. Implement all URL patterns in `urls.py` as specified in `03-views-and-urls.md`. The `<slug>/` catch-all must be the LAST pattern.
8. Implement the `templatetags/form_extras.py` template filters (`add_class`, `add_attr`, `to_json`).

---

### Phase 3: Homepage — Pixel-Perfect Design Port

This is the most important phase. The homepage must be a faithful reproduction of the provided design.

1. Create `home.html` as a **standalone template** (does NOT extend `base.html` — has its own `<!DOCTYPE html>`, `<head>`, and `<body>`).
2. Match the provided design precisely:
   - **Typography**: Same font families, weights, sizes, and line-heights.
   - **Colors**: Extract the exact color palette and implement as CSS custom properties.
   - **Spacing**: Reproduce margins, paddings, and gaps as a token system, not one-off values.
   - **Components**: Cards, buttons, badges, navigation — match proportions and border-radii exactly.
   - **Layout**: Respect the grid, max-widths, and responsive breakpoints.
   - **Imagery**: Use placeholder images or Cloudinary URLs in the same aspect ratios.
3. The homepage template must:
   - Load nav from `nav_menu.items_json` with hardcoded fallback links.
   - Load footer from `footer.columns_json` with hardcoded fallback links.
   - Include full SEO meta tags (title, description, OG, Twitter Card).
   - Include mobile menu with hamburger toggle.
4. Create a standalone CSS file for homepage styles (e.g., `homepage.css`).
5. Do NOT approximate the design. Do NOT swap fonts or colors. If unsure about a value, match it more closely rather than less.

---

### Phase 4: Design System & Public Templates

1. Create `design-system.css` with CSS custom properties derived from the homepage design:
   - Raw palette tokens (the actual hex/rgb values)
   - Semantic tokens (`--color-bg`, `--color-primary`, `--color-text`, etc.)
   - Font stacks, spacing scale, border-radius scale
2. Create `marketing.css` for public page styles including all block CSS classes:
   - `.cms-prose`, `.cms-cta`, `.cms-callout`, `.cms-feature-grid`, `.cms-faq`, `.cms-quote`, `.cms-comparison`, `.cms-pricing`, `.cms-logo-cloud`, `.cms-gallery-wrap`, `.cms-table`
   - Blog styles: `.cms-post-grid`, `.cms-post-card`, `.cms-post-detail`, `.cms-post-hero`
3. Create `base.html` (public base template) with:
   - Full SEO meta block from `seo` context dict
   - DB-driven nav from `nav_menu.items_json`
   - DB-driven footer from `footer.columns_json`
   - Mobile menu with toggle/close/escape-key
   - Canonical URL tag
4. Create page/post templates:
   - `page.html` — extends `base.html`, renders `{{ page.blocks_html|safe }}`
   - `blog_index.html` — extends `base.html`, shows post cards with category filter chips
   - `blog_post.html` — extends `base.html`, shows cover image, author, date, categories, tags, blocks content
5. Make sure `blog_index.html` uses `post.computed_excerpt` for card descriptions.

---

### Phase 5: CMS Admin UI

1. Create `cms.css` for the CMS admin panel styles.
2. Create `cms/base.html` — the CMS admin shell with:
   - Sidebar navigation (Dashboard, Pages, Blog Posts, Navigation)
   - Active link highlighting via `request.resolver_match.url_name`
   - EditorJS CDN scripts (7 plugins: header, list, quote, table, code, delimiter, warning)
   - `cms-editor.js` and `cms-json.js` script includes
   - The Cloudinary upload widget IIFE (both `.cms-cloudinary-upload` widget handler and `window.cmsUploadImage()` global)
3. Create `cms/dashboard.html` with stats cards and recent pages/posts with inline action forms.
4. Create `cms/page_list.html` and `cms/post_list.html` with data tables and inline toggle/delete forms.
5. Create `cms/page_form.html` and `cms/post_form.html` with:
   - Top and bottom action bars (Save Draft / Publish / Unpublish / Save Changes / Delete)
   - The `data-blocks-input` hidden input and `data-blocks-builder` container
   - Cloudinary upload widgets for image fields
   - Collapsible SEO section
   - Delete confirmation banner OUTSIDE the main form (no nested forms!)
6. Create `cms/navigation.html` with inline JS editors for nav items and footer columns.

---

### Phase 6: Block Builder JavaScript

1. Create `cms-editor.js` (the block builder) as specified in `05-blocks-system.md`:
   - `safeJsonParse()`, `el()` helper, `ICONS`, `BLOCK_LABELS`
   - `blockDefaults()` for all 12 block types
   - Field builders: `fieldInput()`, `fieldTextarea()`
   - Block type renderers for all 12 types
   - Rich text / EditorJS integration with per-block instances
   - Main `initBlocksBuilder()` with:
     - "Add block" bars at BOTH top AND bottom
     - Drag-and-drop reordering
     - Form submit hook that saves all EditorJS instances before submitting
   - **Critical**: The form submit hook must save all EditorJS instances async (Promise.all) BEFORE syncing state to the hidden input and submitting the form. Without this, rich text content is lost.
2. Create `cms-json.js` for JSON field helpers.
3. Create `scroll-reveal.js` for public page animations with IntersectionObserver.

---

### Phase 7: Seed Content & Deployment

1. Create `seed_cms_content.py` management command with:
   - Sample pages (About, Pricing, Contact) using blocks with real, meaningful content
   - Sample blog posts using blocks with real, meaningful content
   - Navigation menu (`items_json`)
   - Footer (`columns_json`) — **must include `/sitemap.xml` link**
   - Categories and tags — use generic names relevant to the business, NOT the example names from the docs
2. Create `seed_cms_admin.py` management command.
3. Implement `/sitemap.xml` and `/robots.txt` views.
4. Run `python manage.py migrate && python manage.py seed_cms_admin && python manage.py seed_cms_content`.
5. Verify the full flow works.

---

### Non-Negotiable Rules

These are the most common mistakes. Do NOT make them.

1. **Design fidelity is paramount.** The homepage must look like the provided design, not "inspired by" it. Match fonts, colors, spacing, radii, and layout precisely.
2. **No nested `<form>` tags.** The delete confirmation form must be OUTSIDE the main edit form. Nested forms are invalid HTML and silently break submissions.
3. **`blocks_json` canonical shape is `{ "blocks": [...] }`.** Not a raw array. The builder must normalize `null`, `"null"`, empty strings, and raw arrays to this shape.
4. **"Add block" bars at BOTH top AND bottom** of the block builder. Without both, users can't easily insert blocks at the beginning of content.
5. **Form submit hook saves EditorJS instances FIRST.** EditorJS saves are async. If you submit the form without awaiting saves, rich text content is lost.
6. **Footer must include `/sitemap.xml` link.** This is an SEO requirement.
7. **The `<slug>/` URL pattern must be LAST** in `urlpatterns` so it doesn't shadow `/blog/`, `/cms/`, etc.
8. **`seo.py` must check `blocks_html` when `body_html` is empty.** All content lives in blocks now — the SEO builder must fall back to `blocks_html` for description and image extraction.
9. **Never commit real credentials.** Use environment variables via `.env` files that are gitignored.
10. **The CMS `code` block is a trusted embed.** It allows raw HTML/JS. Only grant CMS access to trusted users.
11. **Always include the trailing slash for POST endpoints.** `APPEND_SLASH` can redirect POSTs to GETs, breaking form submissions.
12. **Use `update_or_create` in seed commands** for idempotency — running the command twice must not duplicate content.
13. **Do NOT use Django admin for the CMS.** Build a custom `/cms/` admin UI as described in the docs. The whole point is a branded, block-based editing experience.
14. **Categories and tags should appear on blog templates** — display them on post detail pages and as filter chips on the blog index.
15. **Every public view must load nav + footer** from the database and pass them to the template context.

---

### Verification Checklist

Before considering the implementation complete, verify:

- [ ] Homepage visually matches the provided design at desktop, tablet, and mobile breakpoints
- [ ] `/cms/` redirects to login for anonymous users
- [ ] CMS dashboard shows stats and recent content
- [ ] Can create a new page with blocks (rich text, CTA, feature grid) and it renders correctly on the public site
- [ ] Can create a new blog post with blocks, cover image, categories, and tags
- [ ] Publish/unpublish toggles work from list views and edit forms
- [ ] Delete works with confirmation from edit forms
- [ ] Cloudinary upload works for cover images, OG images, and gallery block images
- [ ] Block builder has drag-and-drop reordering
- [ ] Block builder has "Add block" bars at top AND bottom
- [ ] Rich text content persists after save (form submit hook works)
- [ ] Navigation editor saves changes that appear on all public pages
- [ ] Footer editor saves changes that appear on all public pages
- [ ] `/sitemap.xml` lists all published pages and posts
- [ ] `/robots.txt` disallows `/cms/` and `/admin/`, points to sitemap
- [ ] SEO meta tags render correctly on all public pages
- [ ] Mobile menu works on all public pages
- [ ] Scroll-reveal animations work on public pages
- [ ] Draft/unpublished content returns 404 on public URLs
- [ ] No hardcoded credentials anywhere in the codebase
- [ ] Run the security checklist from `09-finalsecuritycheck.md`
