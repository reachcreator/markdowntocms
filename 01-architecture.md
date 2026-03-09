# 01 — Architecture

## Why a Custom CMS?

Teams often need marketing pages (About, Pricing, Contact, Blog) with:
- Full design control matching a provided design system
- A blocks-based content editor for non-developers
- Cloudinary image hosting
- SEO meta tags with intelligent fallbacks
- DB-driven navigation and footer
- No additional framework dependencies (no Wagtail, no django-CMS)

The solution: a single Django app (commonly named `marketing/`) with a small set of Python files and a few small JS files that delivers a complete CMS.

---

## Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend | Django 5.x | Models, views, forms, template tags |
| Templates | Django templates (SSR) | No Vue/React on marketing side |
| Rich text editor | EditorJS 2.30 | Loaded via CDN, no build step |
| Block builder | Vanilla JS (`cms-editor.js`) | IIFE pattern, embedded EditorJS instances |
| Image storage | Cloudinary | URLField stores URLs directly |
| CSS | Custom design system | CSS custom properties, no Tailwind |
| Fonts | Google Fonts (Inter, etc.) | Preconnected in templates |
| Scroll effects | IntersectionObserver | `scroll-reveal.js`, auto-marks elements |
| Database | PostgreSQL | Via Podman container |

Note: the database can be local Postgres, Docker/Podman, or a managed Postgres instance.

---

## Data Flow

### Content Creation (CMS Admin)

```
User fills form in CMS
    ↓
cms-editor.js builds blocks_json in hidden input
    ↓
Form POST → cms_views.py (page_edit / post_edit)
    ↓
PageForm / PostForm validates
    ↓
Page.save() / Post.save() triggered
    ↓
save() override calls blocks.py → render_blocks(blocks_json)
    ↓
render_blocks() calls renderers.py for rich_text blocks
    ↓
Result stored in blocks_html (TextField)
    ↓
Database commit
```

### Content Display (Public)

```
Request → views.py (page_detail / blog_post_detail)
    ↓
Query Page/Post model
    ↓
Check is_published property
    ↓
Load NavMenu + Footer (every view)
    ↓
Build SEO payload via seo.py
    ↓
Render template with {{ page.blocks_html|safe }}
    ↓
scroll-reveal.js auto-marks elements for animation
    ↓
IntersectionObserver triggers reveal animations
```

### Image Upload

```
User clicks upload in CMS form
    ↓
File selected → JS reads file
    ↓
AJAX POST to /cms/upload/ (via fetch)
    ↓
cms_views.upload_image() receives file
    ↓
cloudinary_utils.upload_image() uploads to Cloudinary
    ↓
Returns JSON {url, public_id, width, height}
    ↓
JS sets hidden input value = url
    ↓
On form save, URL string stored in URLField
```

---

## Key Architecture Patterns

### 1. The blocks_json → blocks_html Pattern

This is the core pattern. Content is authored as structured JSON but rendered to static HTML on save. This means:
- **Zero runtime rendering cost** — public pages just output pre-rendered HTML
- **No JS required on public side** for content rendering
- **JSON is the source of truth** — HTML is derived, can be re-generated
- **Block types are extensible** — add a new type to `blocks.py`, `cms-editor.js`, and you're done

### 2. The EditorJS-inside-Blocks Pattern

Rich text blocks contain EditorJS data. Each rich_text block gets its own EditorJS instance:

```javascript
// In cms-editor.js
var editor = new EditorJS({
    holder: holderId,        // Unique per block
    data: block.content,     // EditorJS JSON from blocks_json
    tools: { header, list, quote, table, code, delimiter, warning },
    onChange: function() {
        editor.save().then(function(output) {
            block.content = output;   // Update blocks_json
            sync();                    // Write to hidden input
        });
    },
});
```

### 3. The `data-blocks-input` / `data-blocks-builder` Contract

Templates declare where blocks data lives and where the builder UI renders:

```html
<!-- Hidden input holds the JSON string -->
<input type="hidden" name="blocks_json" value='...' data-blocks-input>

<!-- Builder UI renders here -->
<div class="cms-blocks-builder" data-blocks-builder></div>
```

`cms-editor.js` finds these elements on DOMContentLoaded and initializes.

Important implementation detail: the builder defensively normalizes the hidden JSON value.
For new pages the field can be empty or the literal string `"null"` (which parses to `null`).
The builder normalizes these to `{ "blocks": [] }` so the editor doesn't crash.

### 4. The Upload Widget Pattern

Image upload fields use a consistent widget pattern:

```html
<div class="cms-cloudinary-upload" data-target="id_fieldname">
    <input type="hidden" name="fieldname" id="id_fieldname" value="...">
    <div class="cms-upload-preview">
        {% if obj.fieldname %}<img src="{{ obj.fieldname }}">{% endif %}
    </div>
    <label class="cms-btn cms-btn-ghost cms-upload-btn">
        <input type="file" accept="image/*" style="display:none" data-upload-trigger>
        Upload Image
    </label>
    <span class="cms-upload-status"></span>
</div>
```

The JS in `cms/base.html` finds all `.cms-cloudinary-upload` widgets and wires up the upload flow.

### 5. Every View Loads Nav + Footer

```python
def page_detail(request, slug):
    page = get_object_or_404(Page, slug=slug)
    nav = NavMenu.objects.filter(name='Primary').first()
    footer = Footer.objects.first()
    return render(request, 'marketing/page.html', {
        'page': page,
        'nav_menu': nav,
        'footer': footer,
        'seo': build_seo_payload(page, request=request),
    })
```

This ensures consistent navigation and footer across all pages without template context processors.

---

## CMS Routing (/cms)

CMS is served by the marketing app (`marketing/urls.py`) under:

- `/cms/` dashboard
- `/cms/pages/` list, `/cms/pages/new/`, `/cms/pages/<id>/`
- `/cms/posts/` list, `/cms/posts/new/`, `/cms/posts/<id>/`
- `/cms/navigation/` nav + footer editor
- `/cms/upload/` authenticated POST endpoint for Cloudinary uploads
- `/cms/pages/<id>/toggle-status/` and `/cms/posts/<id>/toggle-status/` quick publish/unpublish toggles

Pitfall: always include the trailing slash for POST endpoints (e.g. `/cms/upload/`). With `APPEND_SLASH`, missing slashes can redirect, and redirects can break POST.

---

## Sitemap & Robots (One Source Of Truth)

This repo serves `/sitemap.xml` via Django's sitemap framework (project-level URL).
The sitemap includes a static sitemap class for `/` and `/blog/` plus model sitemaps for pages and posts.

`/robots.txt` points to `/sitemap.xml` and disallows `/cms/` and `/admin/`.

---

## Publish Controls & Button Semantics

CMS forms submit a `name="action"` value. The views use it to decide whether to override status:

- `Save Draft` (`action=draft`): forces `status='draft'`
- `Publish` (`action=publish`): forces `status='published'` (overrides the Status dropdown)
- `Save Changes` (`action=save`): saves edits without changing status (use this for `scheduled`)
- `Unpublish` (`action=unpublish`): flips to draft without running full form validation

Pitfall: list/dashboard “Publish/Unpublish” is a simple toggle. If an item is `scheduled`, clicking “Publish” will set it to `published` immediately (schedule is lost).

---

## Security Model

- **CMS access**: `@cms_admin_required` decorator wraps `@login_required` + checks `CMS Admins` group membership
- **Upload endpoint**: Also protected by `@cms_admin_required` + `@require_POST`
- **HTML sanitization**:
  - EditorJS rich text uses BeautifulSoup allowlists (tags/attrs) in `_sanitize_inline_html()`.
  - `sanitize_html()` is a lightweight regex pass that removes `<script>` and inline `on*=` handlers outside trusted markers.
  - The CMS `code` block is a trusted embed and is preserved through sanitization inside `<!--TRUSTED_CODE_START-->...<!--TRUSTED_CODE_END-->`.
- **CSRF**: All forms use `{% csrf_token %}`, AJAX uploads send `X-CSRFToken` header
- **robots.txt**: Disallows `/cms/` and `/admin/` from crawlers

---

## Common CMS Pitfalls (Read This First)

- **Never nest `<form>` tags** inside the main edit form. Nested forms are invalid HTML and can break submissions (including blocks JSON updates).
- **Do not paste `<script>` into `rich_text`** blocks. Use the CMS `code` block for trusted embeds.
- **Builder JSON must be stable**: treat `blocks_json` as `{ "blocks": [...] }`. The builder tolerates arrays and nulls for backward compat, but the canonical shape should be the object.

---

## Adapting This CMS For Your Project

This documentation contains examples specific to the original project (company names, branding, sample content). When adapting this CMS for your own use:

1. **Replace all company-specific content** — Update seed commands, sample pages, and example content to reflect your brand
2. **Use generic category/tag names** — Avoid industry-specific examples; use broad categories like "Product Updates", "Industry News", "How-To"
3. **Include sitemap in footer** — Always add `/sitemap.xml` link to your footer for SEO
4. **Match your design system** — Update CSS tokens to match your provided design, not the examples here

---

## What This CMS Does NOT Do

- No revision history / drafts comparison
- No media library browser (images are uploaded inline)
- No user roles beyond "is in CMS Admins group"
- No scheduled publishing automation (the `is_published` property checks dates, but there's no cron job to flip status)
- No inline preview / live preview
- No multilingual support
- No comments system
- No search within CMS
