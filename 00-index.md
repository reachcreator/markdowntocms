# CMSPack — Django Marketing CMS Documentation

A complete reference for a custom SSR marketing CMS. This is a from-scratch Django CMS with a blocks-based content system, EditorJS rich text editing, Cloudinary image storage, and a themeable admin UI.

No django-CMS. No Wagtail. No headless CMS. Just Django models, views, templates, and vanilla JS.

---

## Quick Start

```bash
# 1. Activate your environment
# (conda example)
conda activate <env>

# 2. Run migrations
python manage.py migrate

# 3. Seed sample content (pages, posts, nav, footer)
python manage.py seed_cms_content

# 4. Create the CMS Admins group and assign your user
python manage.py seed_cms_admin

# 5. Run the dev server
python manage.py runserver

# 6. Visit the CMS at http://localhost:8000/cms/
#    (requires login as a user in the "CMS Admins" group)
```

---

## Table of Contents

| Doc | Title | What's Inside |
|-----|-------|---------------|
| [01-architecture.md](./01-architecture.md) | Architecture | Design decisions, tech stack, file map, data flow |
| [02-models.md](./02-models.md) | Models | Full model reference with complete source code |
| [03-views-and-urls.md](./03-views-and-urls.md) | Views & URLs | All views, URL patterns, access control |
| [04-templates.md](./04-templates.md) | Templates | Template hierarchy, base templates, form templates, public templates |
| [05-blocks-system.md](./05-blocks-system.md) | Blocks System | Block types, server-side renderer, EditorJS renderer, cms-editor.js |
| [06-cloudinary.md](./06-cloudinary.md) | Cloudinary | Image upload system, migration story, upload widgets |
| [07-frontend.md](./07-frontend.md) | Frontend | CSS architecture, design tokens, scroll-reveal, mobile menu |
| [08-deployment.md](./08-deployment.md) | Deployment | Seed commands, DB sync, remote setup, image generation |

---

## Key Design Decisions

1. **No django-CMS** — Custom SSR CMS from scratch. Simpler, no plugin dependencies, full control over every aspect.

2. **Blocks-based content** — Pages and posts store content as `blocks_json` (JSONField). On model save, `blocks.py` renders the JSON to `blocks_html` (TextField). Public templates just do `{{ page.blocks_html|safe }}`.

3. **EditorJS inside blocks** — Rich text blocks use EditorJS instances loaded via CDN. Each `rich_text` block gets its own EditorJS instance, created dynamically by `cms-editor.js`.

4. **12 block types** — `rich_text`, `code`, `callout`, `cta`, `feature_grid`, `comparison_table`, `table`, `faq`, `quote`, `logo_cloud`, `pricing_table`, `image_gallery`.

   Note: EditorJS “code” (inside a `rich_text` block) is for formatting code snippets.
   The CMS `code` block type is for embedding trusted raw HTML/JS snippets on the public site.

5. **Cloudinary URLs stored directly** — Image fields are plain `URLField` storing Cloudinary URLs. No django-cloudinary-storage, no `CloudinaryField`. A single AJAX upload endpoint returns the URL.

6. **Gallery images in blocks_json** — Stored as URL strings in `images[].src` inside the block JSON data.

7. **DB-driven nav & footer** — `NavMenu` and `Footer` models loaded in every view context. Changes are instant across the entire site.

8. **SEO automatic** — `seo.py` builds OpenGraph/Twitter meta from model fields with intelligent fallbacks (title -> seo_title, first paragraph -> description, etc.).

9. **CMS access restricted** — Only users in the `CMS Admins` Django group can access `/cms/` routes. Uses a custom `@cms_admin_required` decorator.

10. **SSR only on marketing side** — No Vue, no React, no build step for the marketing/CMS pages. The main app uses Vue but the marketing side is pure Django templates + vanilla JS + CSS.

---

## File Map

```
marketing/
├── __init__.py
├── models.py              # Page, Post, NavMenu, Footer, MediaAsset, etc.
├── views.py               # Public views (home, page, blog, sitemap, robots)
├── cms_views.py           # CMS admin views (dashboard, CRUD, upload)
├── urls.py                # All URL patterns (public + CMS)
├── forms.py               # ModelForms for Page, Post, NavMenu, Footer
├── permissions.py          # @cms_admin_required decorator
├── blocks.py              # Server-side block renderer (JSON -> HTML)
├── renderers.py           # EditorJS/Lexical HTML renderer + sanitizer
├── seo.py                 # SEO payload builder
├── cloudinary_utils.py    # Cloudinary upload helpers
├── templatetags/
│   └── form_extras.py     # Template filters: add_class, add_attr, to_json
├── templates/marketing/
│   ├── base.html          # Public base (nav, footer, SEO meta, mobile menu)
│   ├── home.html          # Standalone homepage (separate CSS)
│   ├── page.html          # Public page detail
│   ├── blog_index.html    # Blog listing
│   ├── blog_post.html     # Blog post detail
│   └── cms/
│       ├── base.html      # CMS admin shell (sidebar, EditorJS CDN, upload JS)
│       ├── dashboard.html # CMS dashboard with stats
│       ├── page_list.html # Page listing table
│       ├── page_form.html # Page create/edit form
│       ├── post_list.html # Post listing table
│       ├── post_form.html # Post create/edit form
│       └── navigation.html # Nav + footer editor
├── management/commands/
│   ├── seed_cms_content.py   # Seeds pages, posts, nav, footer
│   ├── seed_cms_admin.py     # Creates CMS Admins group
│   └── generate_images.py   # AI image generation via fal.ai
└── migrations/
    ├── 0001_initial.py
    └── 0002_alter_mediaasset_file_...  # Cloudinary migration

static/
├── css/
│   ├── design-system.css    # Design tokens (~1370 lines)
│   ├── marketing.css        # Public marketing styles
│   ├── cms.css              # CMS admin styles (~1360 lines)
│   └── homepage.css  # Standalone homepage styles
└── js/
    ├── cms-editor.js        # Block builder + embedded EditorJS integration
    ├── cms-json.js          # JSON field helpers (39 lines)
    └── scroll-reveal.js     # Scroll-reveal + image fade (164 lines)
```

---

## Environment & Credentials

Do not hardcode or commit credentials in docs. Configure via environment variables (or a local `.env` that is gitignored).

```
Conda env:       <env>  (optional)
Django project:  project/
Marketing app:   marketing/

Cloudinary:
  CLOUDINARY_CLOUD_NAME=your_cloud_name
  CLOUDINARY_API_KEY=your_api_key
  CLOUDINARY_API_SECRET=your_api_secret

Database:
  DATABASE_URL=postgres://<user>:<password>@<host>:<port>/<db>
```
