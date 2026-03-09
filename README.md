# CMSPack

**Turn any homepage design into a full website with a CMS and block builder — in one prompt.**

CMSPack is an open-source collection of markdown files that give AI coding agents everything they need to build a complete Django website with a custom content management system, a drag-and-drop block builder, a blog, SEO, Cloudinary image hosting, and a branded admin panel. Hand it a homepage design and the CMSPack docs. One prompt. Full site.

No Wagtail. No django-CMS. No headless CMS. No WordPress. Just your design, your Django project, and a set of markdown instructions that an AI agent follows to build it all from scratch.

**License: MIT** — use it for client work, SaaS projects, personal sites, whatever you want.

---

## What You Get

You give your AI agent two things: a homepage design and the CMSPack docs. The agent builds:

- **A pixel-perfect homepage** ported from your design (static HTML, v0 export, Figma export, Next.js mock — whatever you have)
- **A custom `/cms/` admin panel** — not Django admin, a branded block-based editing experience with sidebar navigation, stats dashboard, and a modern UI
- **12 content block types** — rich text (EditorJS), CTA, callout, feature grid, FAQ, quote, comparison table, pricing table, logo cloud, image gallery, code embed, and generic table
- **A drag-and-drop block builder** — vanilla JS, no framework, no build step, with EditorJS rich text editing embedded inside blocks
- **A full blog** with categories, tags, cover images, author attribution, and auto-generated excerpts
- **DB-driven navigation and footer** — edit site nav and footer from the CMS, changes appear instantly across every page
- **Automatic SEO** — OpenGraph, Twitter Cards, canonical URLs, XML sitemap, robots.txt, meta descriptions with intelligent fallbacks
- **Cloudinary image hosting** — upload images from the CMS admin, URLs stored directly in the database, CDN-optimized delivery
- **Scroll-reveal animations** — IntersectionObserver-based reveal animations that auto-apply to all content blocks
- **Management commands** to seed content, set up CMS admins, and optionally generate AI images

The entire CMS is a single Django app. Server-side rendered. No React. No Vue. No build step on the marketing side. Content is stored as JSON, rendered to static HTML on save, and served with zero runtime rendering cost.

---

## How It Works

CMSPack is a markdown CMS blueprint. The documentation files contain complete source code, architecture decisions, data flow diagrams, template hierarchies, and implementation details for every component of the system. When fed to a frontier-level AI coding agent alongside your design, the agent has everything it needs to build the complete system without guessing.

The `initialprompt.md` file is the key. It's a structured build prompt that walks the agent through seven phases — from project setup to deployment — with non-negotiable rules that prevent the most common implementation mistakes.

### The Flow

```
Your homepage design (HTML/CSS, v0, Figma, etc.)
         +
CMSPack documentation (these markdown files)
         +
initialprompt.md (the build prompt)
         ↓
   AI coding agent
         ↓
Complete Django site with CMS, block builder,
blog, SEO, image hosting, and admin panel
```

---

## Recommended Models

**Claude Opus 4.6** is the recommended model — it handles the full build with the highest fidelity to both the design and the CMS architecture.

Any frontier-level model will do a great job:

- Claude Opus 4.6 (Anthropic)
- GPT-5.4 (OpenAI)
- GLM-4.7 (Zhipu AI)

Use whichever model you have access to. The documentation is detailed enough that any strong model can follow it.

---

## Quick Start

1. Clone or download this repo
2. Open your AI coding agent (Cursor, Windsurf, OpenCode, Aider, Claude Code, etc.)
3. Provide the agent with:
   - Your homepage design files
   - The CMSPack markdown docs (`00-index.md` through `09-finalsecuritycheck.md`)
   - The contents of `initialprompt.md` as the prompt
4. Let the agent build
5. Run migrations, seed content, and you're live

```bash
python manage.py migrate
python manage.py seed_cms_admin
python manage.py seed_cms_content
python manage.py runserver
# Visit http://localhost:8000/ — your site
# Visit http://localhost:8000/cms/ — the CMS admin
```

---

## Documentation Files

| File | What's Inside |
|------|---------------|
| `initialprompt.md` | The AI agent build prompt — paste this into your agent to kick off the build |
| `00-index.md` | Quick start, table of contents, file map, key design decisions |
| `01-architecture.md` | Tech stack, data flow diagrams, routing, security model, key patterns |
| `02-models.md` | Complete model reference with full source code and forms |
| `03-views-and-urls.md` | All views, URL patterns, permissions, SEO builder with source code |
| `04-templates.md` | Template hierarchy, complete template source code, form patterns |
| `05-blocks-system.md` | All 12 block types, server renderer, EditorJS integration, block builder JS |
| `06-cloudinary.md` | Image upload system, frontend integration, Cloudinary setup guide |
| `07-frontend.md` | CSS architecture, design tokens, scroll-reveal system, mobile menu |
| `08-deployment.md` | Seed commands with source code, database sync, deployment checklist |
| `09-finalsecuritycheck.md` | Pre-deployment security checklist for the CMS and marketing site |

---

## The Block Builder

The heart of the CMS. Content is structured as typed JSON blocks in the editor and pre-rendered to static HTML on save. Zero runtime cost. The editor UI is built in vanilla JavaScript with embedded EditorJS instances for rich text editing.

### 12 Block Types

| Block | What It Does |
|-------|-------------|
| **Rich Text** | Full EditorJS editor — headings, paragraphs, lists, quotes, tables, code snippets |
| **CTA** | Call-to-action section with title, body, and button |
| **Callout** | Highlighted information box |
| **Feature Grid** | Grid of feature cards with titles and descriptions |
| **FAQ** | Question and answer pairs |
| **Quote** | Styled blockquote with author attribution |
| **Comparison Table** | Side-by-side comparison with headers and rows |
| **Pricing Table** | Pricing plan cards with feature lists |
| **Logo Cloud** | Row of logos (clients, partners, integrations) |
| **Image Gallery** | Image grid with captions and multiple layout options |
| **Code Embed** | Trusted raw HTML/JS for third-party widgets |
| **Table** | Generic data table with headers and rows |

Adding new block types is straightforward — define the JSON shape, build the editor UI, write the server renderer, add the CSS class. The docs explain exactly how.

---

## Architecture at a Glance

```
marketing/
├── models.py            # Page, Post, NavMenu, Footer + save() renders blocks
├── views.py             # Public views — homepage, pages, blog, sitemap, robots
├── cms_views.py         # CMS admin — dashboard, CRUD, upload, toggle status
├── urls.py              # All URL patterns (public + /cms/)
├── blocks.py            # Server-side block renderer (JSON → HTML)
├── renderers.py         # EditorJS HTML renderer + sanitizer
├── seo.py               # SEO payload builder with fallback chain
├── permissions.py       # @cms_admin_required decorator
├── cloudinary_utils.py  # Cloudinary upload helper
├── forms.py             # ModelForms for all CMS content types
├── templatetags/        # add_class, add_attr, to_json filters
├── templates/marketing/
│   ├── base.html        # Public base — nav, footer, SEO, mobile menu
│   ├── home.html        # Standalone homepage (separate CSS)
│   ├── page.html        # CMS page detail
│   ├── blog_index.html  # Blog listing with category filters
│   ├── blog_post.html   # Blog post detail
│   └── cms/             # CMS admin templates (dashboard, forms, lists, nav editor)
├── management/commands/ # seed_cms_content, seed_cms_admin, generate_images
└── migrations/
```

**Key pattern**: Content is authored as JSON in the block builder, rendered to static HTML on model save, and served to public templates as pre-rendered HTML. The JSON is the source of truth. The HTML is derived and can be regenerated at any time.

---

## Why Markdown-Driven CMS Architecture?

Traditional CMS platforms ship code. CMSPack ships knowledge.

Instead of installing a CMS package with opinions you'll fight against, CMSPack gives your AI agent a complete architectural specification in markdown. The agent builds the CMS from scratch, tailored to your design, in your project, under your full control. No plugin conflicts. No upgrade headaches. No fighting framework opinions.

The markdown CMS documentation approach means:

- **Full control** — every line of code is yours, written for your project
- **No dependencies** — no Wagtail, no django-CMS, no WordPress, no Contentful
- **Design fidelity** — the agent ports your exact design, not a theme approximation
- **One Django app** — the entire CMS is a single app you can drop into any Django project
- **Transparent architecture** — every decision is documented and justified in the markdown files

---

## Use Cases

- **Agency sites** — take a client's approved homepage design and build the full site with CMS in hours
- **SaaS marketing sites** — build the marketing site for your product with a CMS your team can actually use
- **Portfolio sites** — pages + blog + SEO, all editable without touching code
- **Startup landing pages** — move fast from design to live site with content management built in
- **Content marketing** — blog with categories, tags, SEO meta, and a block-based editor for rich content

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Django 5.x |
| Templates | Django SSR (no frontend framework) |
| Rich text | EditorJS 2.30 via CDN |
| Block builder | Vanilla JS |
| Images | Cloudinary |
| CSS | Custom design system with CSS custom properties |
| Database | PostgreSQL |
| Static files | WhiteNoise (production) |
| Animations | IntersectionObserver |

---

## FAQ

**Is this a Django package I install?**
No. CMSPack is documentation — markdown files that contain the complete specification and source code for a custom Django CMS. An AI coding agent reads these files and builds the CMS in your project.

**What AI agents work with this?**
Any agent that can read files and write code. Cursor, Windsurf, OpenCode, Aider, Claude Code, ChatGPT with file upload, Cline — all work.

**Can I customize the block types?**
Yes. The docs include a step-by-step guide for adding new block types. Define the JSON shape, build the editor UI, write the renderer, add CSS.

**Does it work with existing Django projects?**
Yes. The CMS is a single Django app (`marketing/`). Add it to `INSTALLED_APPS`, include the URLs, run migrations, and you're set.

**What about TypeScript / React / Vue?**
The CMS and marketing side are intentionally SSR-only with vanilla JS. No build step. If your main app uses React/Vue, the marketing app sits alongside it without conflict.

**Is the CMS admin panel ugly?**
No. It's a custom-styled admin with a sidebar, dashboard with stats, data tables, and a block builder. It inherits your design system's tokens. It looks like a product, not Django admin.

---

## Contributing

Contributions welcome. Open an issue or PR.

---

## License

MIT — do whatever you want with it.

---

Made with love by linkbuilding agency [ReachCreator.com](https://reachcreator.com)
