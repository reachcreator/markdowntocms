# 04 -- Templates

> Every template in the CMS, from the public-facing `base.html` to the CMS
> admin shell to the page/post editors. Includes the standalone homepage,
> the template tags module, and explains how the Cloudinary upload widgets
> and blocks builder are wired into forms.

---

## Template Map

### Public Templates (`marketing/templates/marketing/`)

| Template | Extends | Lines | Purpose |
|---|---|---|---|
| `base.html` | -- | 94 | Public site shell: nav, footer, SEO meta, mobile menu, scroll-reveal |
| `home.html` | Standalone | 176 | Homepage -- does NOT extend base.html. Has its own `<html>` |
| `page.html` | `base.html` | 15 | Generic CMS page detail |
| `blog_index.html` | `base.html` | 34 | Blog listing with post cards |
| `blog_post.html` | `base.html` | 45 | Single blog post with cover image, categories, tags |

### CMS Admin Templates (`marketing/templates/marketing/cms/`)

| Template | Extends | Lines | Purpose |
|---|---|---|---|
| `base.html` | -- | 149 | CMS admin shell: sidebar nav, EditorJS CDN, upload widget JS |
| `dashboard.html` | `cms/base.html` | 141 | Stats cards + recent pages/posts with inline actions |
| `page_list.html` | `cms/base.html` | 81 | Page listing table with inline toggle/delete |
| `page_form.html` | `cms/base.html` | 216 | Page create/edit form with action bars + blocks builder |
| `post_list.html` | `cms/base.html` | 83 | Post listing table with inline toggle/delete |
| `post_form.html` | `cms/base.html` | 239 | Post create/edit form with action bars + blocks builder |
| `navigation.html` | `cms/base.html` | 225 | Nav items + footer columns editor |

### Supporting Code

| File | Lines | Purpose |
|---|---|---|
| `marketing/templatetags/form_extras.py` | 46 | `add_class`, `add_attr`, `to_json` template filters |

---

## Key Patterns

### 1. Two Separate Base Templates

The public site uses `marketing/base.html` and the CMS admin uses
`marketing/cms/base.html`. They share the same design-system.css tokens
but load different stylesheets:

- **Public:** `design-system.css` + `marketing.css`
- **CMS Admin:** `design-system.css` + `cms.css`
- **Homepage:** `design-system.css` + `homepage.css` (standalone, not extending base)

### 2. The Homepage is Standalone

`home.html` has its own `<!DOCTYPE html>`, `<head>`, and `<body>`. It does
NOT use `{% extends %}`. This is intentional -- the homepage has a completely
different layout (hero section, chat widget mockup, feature blocks, trust bar)
that doesn't fit the `base.html` content wrapper.

The homepage has hardcoded fallback nav links in case `nav_menu` is empty:
```html
{% if nav_menu and nav_menu.items_json %}
    {% for item in nav_menu.items_json %}
        <li><a href="{{ item.url }}">{{ item.label }}</a></li>
    {% endfor %}
{% else %}
    <li><a href="/blog/">Resources</a></li>
    <li><a href="#">Platform</a></li>
    ...
{% endif %}
```

### 3. SEO Meta Block

Both `base.html` and `home.html` render the same SEO meta structure from the
`seo` context dict:

```html
<title>{{ seo.title }}</title>
<meta name="description" content="{{ seo.description }}">
<meta property="og:title" content="{{ seo.og_title }}">
<meta property="og:description" content="{{ seo.og_description }}">
{% if seo.image %}
<meta property="og:image" content="{{ seo.image }}">
<meta name="twitter:image" content="{{ seo.image }}">
{% endif %}
```

### 4. The `data-blocks-input` / `data-blocks-builder` Contract

In both `page_form.html` and `post_form.html`, the blocks system is wired
with two elements:

```html
<input type="hidden" name="blocks_json" id="id_blocks_json"
       value='{{ form.blocks_json.value|default_if_none:""|to_json }}'
       data-blocks-input>
<div class="cms-blocks-builder" data-blocks-builder></div>
```

- `data-blocks-input`: Hidden input holding the JSON array of blocks. This
  is what gets POSTed to the server.
- `data-blocks-builder`: Container div where `cms-editor.js` renders the
  interactive block UI.
- `cms-editor.js` reads from the hidden input on init, builds the UI, and
  writes back to the hidden input before form submit.

Note: the canonical shape is a JSON object: `{ "blocks": [...] }` (not a raw array). The builder tolerates arrays and `null`/`"null"` for backward compatibility and normalizes to the object.

### 5. Cloudinary Upload Widget Pattern

Image fields use a reusable widget structure:

```html
<div class="cms-cloudinary-upload" data-target="id_primary_image">
    <input type="hidden" name="primary_image" id="id_primary_image" value="...">
    <div class="cms-upload-preview">
        {% if page and page.primary_image %}
        <img src="{{ page.primary_image }}" alt="Primary image">
        {% endif %}
    </div>
    <label class="cms-btn cms-btn-ghost cms-upload-btn">
        <input type="file" accept="image/*" style="display:none" data-upload-trigger>
        Upload Image
    </label>
    <span class="cms-upload-status"></span>
</div>
```

The JS in `cms/base.html` finds all `.cms-cloudinary-upload` elements and
wires up the file input to POST to `/cms/upload/`, then sets the hidden input
value to the returned Cloudinary URL.

### 6. The `to_json` Template Filter

The `blocks_json` field is a Django `JSONField` which means Python might
give it to the template as a dict/list (not a string). The `to_json` filter
ensures it's always a valid JSON string for use in an HTML attribute:

```html
value='{{ form.blocks_json.value|default_if_none:""|to_json }}'
```

### 7. Mobile Menu

Both `base.html` and `home.html` implement the same mobile menu pattern:
- A hidden `#mobile-menu` div with nav links
- A `#menu-backdrop` overlay
- `toggleMobileMenu()` / `closeMobileMenu()` JS functions
- Escape key closes the menu

### 8. Action Bars & Inline Actions

**Form action bars** (`.cms-form-actions-bar`): Both `page_form.html` and
`post_form.html` have top and bottom action bars. In edit mode, the left side includes:

- `Save Changes` (`action=save`) — saves edits without forcing status.
- `Publish` (`action=publish`) — forces published.
- `Unpublish` (`action=unpublish`) — fast-path flip to draft (does not run full validation).

In create mode, the actions are typically `Save Draft` (`action=draft`) and `Publish` (`action=publish`).

**Delete confirmation banner** (`.cms-confirm-delete`): Clicking Delete shows a hidden inline banner.
Important: the delete confirmation uses a *separate* `<form>` outside the main edit form.
Nested forms are invalid HTML and can break submissions.

**Inline actions in lists/dashboard**: `page_list.html`, `post_list.html`,
and `dashboard.html` use inline `<form method="post">` elements (`.cms-inline-form`)
inside each row's actions cell for Publish/Unpublish toggle and Delete. The toggle
form includes a hidden `next` field so the redirect goes back to the originating
page. The delete form uses `onsubmit="return confirm(...)"` for a quick JS
confirmation dialog.

> **Recommended: Add drag-and-drop ordering to page/post lists**. When you have many pages or blog posts, a drag-and-drop reorder interface in the list view is essential for controlling display order. Add an `order` field to your Page/Post models and implement drag-and-drop reordering via AJAX in the list templates.

---

## Complete Source Files

### `marketing/templates/marketing/base.html` (94 lines) -- Public Base

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ seo.title }}</title>
    <meta name="description" content="{{ seo.description }}">
    <meta property="og:title" content="{{ seo.og_title }}">
    <meta property="og:description" content="{{ seo.og_description }}">
    <meta property="og:type" content="website">
    {% if seo.url %}
    <meta property="og:url" content="{{ seo.url }}">
    {% endif %}
    {% if seo.image %}
    <meta property="og:image" content="{{ seo.image }}">
    <meta name="twitter:image" content="{{ seo.image }}">
    {% endif %}
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:title" content="{{ seo.og_title }}">
    <meta name="twitter:description" content="{{ seo.og_description }}">
    {% load static %}
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="{% static 'css/design-system.css' %}">
    <link rel="stylesheet" href="{% static 'css/marketing.css' %}">
    <link rel="icon" href="{% static 'images/favicon.ico' %}">
    {% if seo.url %}<link rel="canonical" href="{{ seo.url }}">{% endif %}
</head>
<body>
    <div id="menu-backdrop" class="menu-backdrop" hidden onclick="closeMobileMenu()"></div>
    <div class="marketing-shell">
        <nav class="d-nav"><div class="nav-inner"><div class="d-logo"><a href="/"><img src="{% static 'images/logo.png' %}" alt="Site"></a></div><ul class="d-nav-links">
            {% if nav_menu and nav_menu.items_json %}
                {% for item in nav_menu.items_json %}
                    <li><a href="{{ item.url }}">{{ item.label }}</a></li>
                {% endfor %}
            {% endif %}
        </ul><button class="nav-burger" aria-label="Open menu" onclick="toggleMobileMenu()">&#9776;</button><a class="d-nav-cta" href="/login">Log in</a></div></nav>
        <div id="mobile-menu" class="mobile-menu" hidden><div class="mm-close">Menu<button aria-label="Close menu" onclick="closeMobileMenu()">&#10005;</button></div><div class="mm-body"><ul class="mm-links">
            {% if nav_menu and nav_menu.items_json %}
                {% for item in nav_menu.items_json %}
                    <li><a href="{{ item.url }}">{{ item.label }}</a></li>
                {% endfor %}
            {% endif %}
        </ul><div class="mm-cta"><a class="d-btn-primary" style="width:100%;display:inline-flex;justify-content:center;text-decoration:none" href="/login">Log in</a></div></div></div>

        <main class="marketing-main">
            {% block content %}{% endblock %}
        </main>

        {% if footer and footer.legal_text %}
        <p class="d-footer-disc">{{ footer.legal_text }}</p>
        {% endif %}
        <footer class="d-footer">
            <div class="d-logo">
                <img src="{% static 'images/logo.png' %}" alt="Site">
            </div>
            <div class="d-footer-links">
                {% if footer and footer.columns_json %}
                    {% for column in footer.columns_json %}
                        {% for link in column.links %}
                            <a href="{{ link.url }}">{{ link.label }}</a>
                        {% endfor %}
                    {% endfor %}
                {% endif %}
            </div>
        </footer>
    </div>
    <script>
    function closeMobileMenu(){
      document.getElementById('menu-backdrop').setAttribute('hidden', true);
      document.getElementById('mobile-menu').hidden = true;
      document.body.classList.remove('menu-open');
    }
    function toggleMobileMenu(){
      var menu = document.getElementById('mobile-menu');
      if (!menu) return;
      var isOpen = !menu.hidden;
      closeMobileMenu();
      if (!isOpen){
        menu.hidden = false;
        document.getElementById('menu-backdrop').removeAttribute('hidden');
        document.body.classList.add('menu-open');
      }
    }
    document.addEventListener('keydown', function(e){
      if (e.key === 'Escape') closeMobileMenu();
    });
    </script>
    <script src="{% static 'js/scroll-reveal.js' %}"></script>
</body>
</html>
```

### `marketing/templates/marketing/home.html` (176 lines) -- Standalone Homepage

```html
{% load static %}
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{ seo.title }}</title>
<meta name="description" content="{{ seo.description }}">
<meta property="og:title" content="{{ seo.og_title }}">
<meta property="og:description" content="{{ seo.og_description }}">
<meta property="og:type" content="website">
{% if seo.url %}<meta property="og:url" content="{{ seo.url }}">{% endif %}
{% if seo.image %}
<meta property="og:image" content="{{ seo.image }}">
<meta name="twitter:image" content="{{ seo.image }}">
{% endif %}
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="{{ seo.og_title }}">
<meta name="twitter:description" content="{{ seo.og_description }}">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=DM+Serif+Display:ital@0;1&family=General+Sans:wght@400;500;600;700&family=Instrument+Serif:ital@0;1&family=Inter:wght@400;500;600;700;800;900&family=Satoshi:wght@400;500;700;900&family=Space+Mono:wght@400;700&family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap" rel="stylesheet">
<link rel="stylesheet" href="{% static 'css/design-system.css' %}">
<link rel="stylesheet" href="{% static 'css/homepage.css' %}">
</head>
<body>
<div id="menu-backdrop" class="menu-backdrop" hidden onclick="closeMobileMenu()"></div>
<div class="version version-a" id="version-a">
<nav class="d-nav"><div class="nav-inner"><div class="d-logo"><a href="/"><img src="{% static 'images/logo.png' %}?v=1" alt="Acme Corp"></a></div><ul class="d-nav-links">
    {% if nav_menu and nav_menu.items_json %}
        {% for item in nav_menu.items_json %}
            <li><a href="{{ item.url }}">{{ item.label }}</a></li>
        {% endfor %}
    {% else %}
        <li><a href="/blog/">Resources</a></li>
        <li><a href="#">Platform</a></li>
        <li><a href="#">Why Acme Corp</a></li>
        <li><a href="#">Pricing</a></li>
    {% endif %}
</ul><button class="nav-burger" aria-label="Open menu" onclick="toggleMobileMenu('a')">&#9776;</button><a class="d-nav-cta" href="/login">Log in</a></div></nav>
<div id="mobile-menu-a" class="mobile-menu" hidden style="background:var(--d-obsidian-light);color:var(--d-stone);border-color:var(--d-bdr)"><div class="mm-close">Menu<button aria-label="Close menu" onclick="closeMobileMenu()" style="color:var(--d-stone)">&#10005;</button></div><div class="mm-body"><ul class="mm-links" style="color:var(--d-stone)">
    {% if nav_menu and nav_menu.items_json %}
        {% for item in nav_menu.items_json %}
            <li><a href="{{ item.url }}">{{ item.label }}</a></li>
        {% endfor %}
    {% else %}
        <li><a href="/blog/">Resources</a></li>
        <li><a href="#">Platform</a></li>
        <li><a href="#">Why Acme Corp</a></li>
        <li><a href="#">Pricing</a></li>
    {% endif %}
</ul><div class="mm-cta"><a class="d-btn-primary" style="width:100%;display:inline-flex;justify-content:center;text-decoration:none" href="/login">Log in</a></div></div></div>

<section class="d-hero">
<div class="d-hero-copy">
<div class="d-hero-tag"><span class="d-ai-badge">AI-Powered</span> Business infrastructure for startups</div>
<h1>Scale with <em>Confidence</em></h1>
<p>Acme Corp streamlines projects, workflows and operations, empowering SaaS, Fintech, and Marketplace companies to unlock growth and scale with confidence.</p>
<div class="d-hero-btns"><a class="d-btn-primary" href="/login">Request a demo</a><button class="d-btn-ghost">Take a tour &#8594;</button></div>
</div>
<div>
<div class="d-chat-widget">
<div class="d-chat-head">
<div class="d-chat-avi"><img src="{% static 'images/app-icon.png' %}?v=1" alt="Acme Corp"></div>
<div class="d-chat-head-text"><h4>Acme Corp</h4><span>Getting Started</span></div>
</div>
<div class="d-chat-body">
<div class="d-chat-msgs">
<div class="d-msg-row d-msg-row-bot">
  <div class="d-msg-avi"><img src="{% static 'images/app-icon.png' %}?v=1" alt="Acme Corp"></div>
  <div class="d-cmsg d-cmsg-bot">Welcome to Acme Corp. Tell me about your company and I'll recommend the right setup.</div>
</div>
<div class="d-msg-row d-msg-row-user">
  <div class="d-cmsg d-cmsg-user">B2B SaaS startup, 12 countries, processing payments.</div>
</div>
<div class="d-msg-row d-msg-row-bot">
  <div class="d-msg-avi"><img src="{% static 'images/app-icon.png' %}?v=1" alt="Acme Corp"></div>
  <div class="d-cmsg d-cmsg-bot">I'd recommend a Project Plan, Onboarding Checklist, and Report template. Want me to generate these?</div>
</div>
<div class="d-msg-row d-msg-row-bot" aria-label="Assistant is typing">
  <div class="d-msg-avi"><img src="{% static 'images/app-icon.png' %}?v=1" alt="Acme Corp"></div>
  <div class="d-cmsg-thinking"><span></span><span></span><span></span></div>
</div>
</div>
<div class="d-chat-input-row"><input type="text" placeholder="Type your message..."><button>&#8594;</button></div>
</div>
</div>
</div>
</section>

<div class="d-trust"><span>Anthropic</span><span>Robinhood</span><span>Loom</span><span>Duolingo</span><span>Discord</span><span>Brooklinen</span><span>NPR</span><span>Gusto</span></div>

<div class="d-sh"><p class="tag">How It Works</p><h2>Three Steps to Confidence</h2></div>
<div class="d-how">
<div class="d-how-steps">
<div class="d-step"><div class="d-step-num">Step 01</div><h3>Drafting on Autopilot</h3><p>Creating your first document is as simple as answering a few questions. After a short intake, Acme Corp generates documents and templates tailored to your product or service.</p></div>
<div class="d-step"><div class="d-step-num">Step 02</div><h3>Your One Stop Shop</h3><p>Acme Corp keeps all your templates and documents in one place &mdash; no more lost files! Getting documents shared or published is quick and easy.</p></div>
<div class="d-step"><div class="d-step-num">Step 03</div><h3>In Sync with Your Roadmap</h3><p>Product evolved? Just tell Acme Corp and your templates will be automatically updated to reflect your latest changes.</p></div>
</div>
<div class="mock-widget wcb"><div class="mock-topbar"><div class="mock-dots"><span style="background:#ff5f57"></span><span style="background:#F59E0B"></span><span style="background:#0D9488"></span></div><span class="mock-topbar-title">Document Builder</span></div><div class="mock-body"><div class="field-row"><div><div class="field-label">Company Name</div><div class="field">Acme SaaS Inc.</div></div><div><div class="field-label">Agreement Type</div><div class="field">Service Agreement</div></div></div><div class="field-row"><div><div class="field-label">Jurisdiction</div><div class="field">Delaware, USA</div></div><div><div class="field-label">Term</div><div class="field">12 months, auto-renew</div></div></div><div class="gen-bar"><div class="gen-label" style="color:var(--d-stone-mid)">Generating sections...</div><div class="gen-progress"><div class="gen-fill" style="background:linear-gradient(90deg,#F59E0B,#D97706)"></div></div></div><div class="clause"><div class="clause-title">1. Definitions</div>"Service" means the cloud-based software platform provided by Acme SaaS Inc...</div><div class="clause"><div class="clause-title">2. Scope of Work</div>Subject to the terms of this Agreement, Company grants Customer access to the platform and associated services...</div></div></div>
</div>

<div class="d-contracts"><div class="d-sh"><p class="tag">Templates</p><h2>Documents &amp; Templates</h2></div>
<div class="d-contracts-grid">
<div class="d-contract"><h4>Proposals</h4><p>Create polished proposals that win clients and close deals faster.</p></div>
<div class="d-contract"><h4>Service Agreements</h4><p>Industry-standard terms to shorten your sales cycle and protect your business.</p></div>
<div class="d-contract"><h4>Onboarding Docs</h4><p>Streamline client and team onboarding with clear, consistent documentation.</p></div>
<div class="d-contract"><h4>Project Plans</h4><p>Set clear goals, milestones, and deliverables for every engagement.</p></div>
<div class="d-contract"><h4>Reports</h4><p>Generate professional reports for stakeholders, clients, and internal teams.</p></div>
<div class="d-contract"><h4>Scope of Work</h4><p>Outline roles, deliverables, and payment terms clearly.</p></div>
</div></div>

<div class="d-cta-band"><h2>Try Acme Corp Free</h2><p>Get started in minutes. No credit card required.</p><button class="d-btn-primary">Start for free &#8594;</button></div>

<section class="d-quality">
<div class="d-quality-copy">
<p class="tag">Quality</p>
<h2>Professional Quality. Startup Speed.</h2>
<p>Don't leave your business foundations to chance. Acme Corp is developed by a team of industry experts, product managers and developers that understand the needs of a growing tech company.</p>
<div class="d-qi"><div class="d-qi-icon">&#9997;</div><div><h4>Expert-Authored</h4><p>Our team builds the rigorous logic that powers every document.</p></div></div>
<div class="d-qi"><div class="d-qi-icon">&#127919;</div><div><h4>Zero Hallucinations</h4><p>We use precision engineering to ensure a reliable output &mdash; nothing is left to chance.</p></div></div>
</div>
<div class="mock-widget wes"><!-- E-signature widget mockup --></div>
</section>

<section class="d-features"><div class="d-sh" style="padding-top:0"><p class="tag">Features</p><h2>Unlock Efficiency</h2></div>
<div class="d-fg">
<!-- Four feature blocks with mock widget visualisations (Clickwrap, E-Signatures, Document Lifecycle, Approval Workflows) -->
<!-- See full source for complete HTML -->
</div></section>

{% if footer and footer.legal_text %}
<p class="d-footer-disc">{{ footer.legal_text }}</p>
{% else %}
<p class="d-footer-disc">Acme Corp is a technology platform. This is example disclaimer text for your application.</p>
{% endif %}
<footer class="d-footer"><div class="d-logo"><img src="{% static 'images/logo.png' %}?v=1" alt="Acme Corp"></div><div class="d-footer-links">
    {% if footer and footer.columns_json %}
        {% for column in footer.columns_json %}
            {% for link in column.links %}
                <a href="{{ link.url }}">{{ link.label }}</a>
            {% endfor %}
        {% endfor %}
    {% else %}
        <a href="#">Privacy Notice</a><a href="#">Terms of Use</a><a href="#">Contact Us</a>
    {% endif %}
</div></footer>
</div>

<script>
function closeMobileMenu(){
  document.getElementById('menu-backdrop')?.setAttribute('hidden', true);
  document.querySelectorAll('.mobile-menu').forEach(el=>{ el.hidden = true; });
  document.body.classList.remove('menu-open');
}

function toggleMobileMenu(v){
  const menu = document.getElementById('mobile-menu-' + v);
  if (!menu) return;
  const isOpen = !menu.hidden;
  closeMobileMenu();
  if (!isOpen){
    menu.hidden = false;
    document.body.classList.add('menu-open');
  }
}

document.addEventListener('keydown', (e)=>{
  if (e.key === 'Escape') closeMobileMenu();
});
</script>
<script src="{% static 'js/scroll-reveal.js' %}"></script>
</body></html>
```

> **Note:** The homepage source above has the mock widget HTML for the feature
> blocks abbreviated. The full source is in the repo at
> `marketing/templates/marketing/home.html`. The feature blocks are purely
> decorative HTML/CSS mockups (Document Builder, E-Signature, Clickwrap,
> Approval Workflow, Document Lifecycle dashboard).

### `marketing/templates/marketing/page.html` (15 lines)

```html
{% extends "marketing/base.html" %}

{% block content %}
<article class="cms-page">
    <header class="cms-page-hero">
        <h1>{{ page.title }}</h1>
    </header>
    {% if page.blocks_html %}
    <div class="cms-blocks">
        {{ page.blocks_html|safe }}
    </div>
    {% endif %}
</article>
{% endblock %}
```

The `blocks_html` is pre-rendered on model save. It's marked `|safe` because
rich text rendering sanitizes inline HTML. Note that the CMS `code` block is a trusted embed and is preserved inside markers.

### `marketing/templates/marketing/blog_index.html` (34 lines)

```html
{% extends "marketing/base.html" %}

{% block content %}
<section class="cms-blog-index">
    <header class="cms-blog-hero">
        <h1>Blog</h1>
        <p>Research, insights, and product updates from the team.</p>
    </header>
    {% if categories %}
    <div class="cms-blog-filters">
        {% for cat in categories %}
        <a href="/blog/?category={{ cat.slug }}" class="cms-filter-chip">{{ cat.name }}</a>
        {% endfor %}
    </div>
    {% endif %}
    <div class="cms-post-grid">
        {% for post in posts %}
            <article class="cms-post-card">
                {% if post.cover_image %}
                    <div class="cms-post-card-img">
                        <img src="{{ post.cover_image }}" alt="{{ post.title }}" loading="lazy">
                    </div>
                {% endif %}
                <div class="cms-post-card-body">
                    <h2><a href="/blog/{{ post.slug }}/">{{ post.title }}</a></h2>
                    {% if post.computed_excerpt %}<p>{{ post.computed_excerpt }}</p>{% endif %}
                    <div class="cms-post-card-meta">
                        {% if post.categories.exists %}
                            <span class="cms-post-category">{{ post.categories.first.name }}</span>
                        {% endif %}
                        {% if post.author_name %}<span class="cms-post-author">{{ post.author_name }}</span>{% endif %}
                        {% if post.publish_at %}<time>{{ post.publish_at|date:"M d, Y" }}</time>{% endif %}
                    </div>
                </div>
            </article>
        {% empty %}
            <div class="cms-empty-state">
                <p>No posts published yet. Check back soon.</p>
            </div>
        {% endfor %}
    </div>
</section>
{% endblock %}
```

> **Tip**: Include category filter chips on the blog index page. Pass categories to the template context in `views.blog_index()`:
> ```python
> categories = Category.objects.all()
> return render(request, 'marketing/blog_index.html', {
>     'posts': posts,
>     'categories': categories,
>     ...
> })
> ```

### `marketing/templates/marketing/blog_post.html` (45 lines)

```html
{% extends "marketing/base.html" %}

{% block content %}
<article class="cms-post-detail">
    <header class="cms-post-hero">
        <div class="cms-post-hero-meta">
            {% if post.author_name %}<span class="cms-post-author">{{ post.author_name }}</span>{% endif %}
            {% if post.publish_at %}<time>{{ post.publish_at|date:"F d, Y" }}</time>{% endif %}
        </div>
        <h1>{{ post.title }}</h1>
        {% if post.computed_excerpt %}<p class="cms-post-excerpt">{{ post.computed_excerpt }}</p>{% endif %}
        {% if post.cover_image %}
            <div class="cms-post-cover">
                <img src="{{ post.cover_image }}" alt="{{ post.title }}" loading="lazy">
            </div>
        {% endif %}
    </header>

    {% if post.blocks_html %}
    <div class="cms-blocks">
        {{ post.blocks_html|safe }}
    </div>
    {% endif %}

    {% if post.categories.exists or post.tags.exists %}
    <footer class="cms-post-footer">
        {% if post.categories.exists %}
        <div class="cms-post-categories">
            {% for cat in post.categories.all %}
            <span class="cms-tag">{{ cat.name }}</span>
            {% endfor %}
        </div>
        {% endif %}
        {% if post.tags.exists %}
        <div class="cms-post-tags">
            {% for tag in post.tags.all %}
            <span class="cms-tag cms-tag-muted">{{ tag.name }}</span>
            {% endfor %}
        </div>
        {% endif %}
    </footer>
    {% endif %}
</article>
{% endblock %}
```

> **Tip**: Use `post.computed_excerpt` instead of `post.excerpt` in templates. This property automatically falls back to extracting the first paragraph from the post's content if no explicit excerpt was provided. This ensures every blog post has an excerpt for SEO and display without manual entry.

> **Important - Display categories and tags**: Blog categories and tags are stored in the database but often forgotten in templates. Always display them on the blog post detail page (as shown above) and consider adding them as OpenGraph meta tags:
> ```html
> {% for cat in post.categories.all %}
> <meta property="article:section" content="{{ cat.name }}">
> {% endfor %}
> ```

### `marketing/templates/marketing/cms/base.html` (149 lines) -- CMS Admin Shell

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}CMS{% endblock %}</title>
    {% load static %}
    <link rel="stylesheet" href="{% static 'css/design-system.css' %}">
    <link rel="stylesheet" href="{% static 'css/cms.css' %}">
    <link rel="icon" href="{% static 'images/favicon.ico' %}">
</head>
<body class="cms-body">
    <div class="cms-shell">
        <aside class="cms-sidebar">
            <div class="cms-sidebar-brand">
                <a href="/cms/">
                    <img src="{% static 'images/logo.png' %}" alt="Site">
                </a>
                <span>CMS</span>
            </div>
            <nav class="cms-nav">
                <div class="cms-nav-section">Content</div>
                <a href="{% url 'cms-dashboard' %}" {% if request.resolver_match.url_name == 'cms-dashboard' %}class="active"{% endif %}>
                    <!-- Dashboard SVG icon -->
                    Dashboard
                </a>
                <a href="{% url 'cms-page-list' %}" {% if 'page' in request.resolver_match.url_name %}class="active"{% endif %}>
                    <!-- Pages SVG icon -->
                    Pages
                </a>
                <a href="{% url 'cms-post-list' %}" {% if 'post' in request.resolver_match.url_name %}class="active"{% endif %}>
                    <!-- Posts SVG icon -->
                    Blog Posts
                </a>
                <div class="cms-nav-section">Settings</div>
                <a href="{% url 'cms-navigation' %}" {% if 'navigation' in request.resolver_match.url_name %}class="active"{% endif %}>
                    <!-- Navigation SVG icon -->
                    Navigation
                </a>
            </nav>
            <div class="cms-sidebar-footer">
                <a href="/" target="_blank" rel="noopener">
                    <!-- External link SVG icon -->
                    View Live Site
                </a>
            </div>
        </aside>
        <main class="cms-main">
            {% if messages %}
            <div class="cms-toast">
                {% for message in messages %}
                <div class="cms-toast-item cms-toast-{{ message.tags }}">{{ message }}</div>
                {% endfor %}
            </div>
            {% endif %}
            {% block content %}{% endblock %}
        </main>
    </div>

    {% block extra_head %}{% endblock %}
    <script src="{% static 'js/cms-json.js' %}"></script>
    <script src="https://cdn.jsdelivr.net/npm/@editorjs/editorjs@2.30.2"></script>
    <script src="https://cdn.jsdelivr.net/npm/@editorjs/header@2.8.1"></script>
    <script src="https://cdn.jsdelivr.net/npm/@editorjs/list@1.9.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@editorjs/quote@2.7.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@editorjs/table@2.3.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@editorjs/code@2.9.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@editorjs/delimiter@1.3.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@editorjs/warning@1.3.0"></script>
    <script src="{% static 'js/cms-editor.js' %}"></script>
    <script>
    /* Cloudinary upload widget for CMS image fields */
    (function() {
      'use strict';
      var csrfToken = document.querySelector('[name=csrfmiddlewaretoken]');
      if (!csrfToken) return;

      function initUploadWidgets() {
        document.querySelectorAll('.cms-cloudinary-upload').forEach(function(widget) {
          var targetId = widget.dataset.target;
          var hiddenInput = document.getElementById(targetId);
          var fileInput = widget.querySelector('[data-upload-trigger]');
          var preview = widget.querySelector('.cms-upload-preview');
          var status = widget.querySelector('.cms-upload-status');
          if (!hiddenInput || !fileInput) return;

          fileInput.addEventListener('change', function() {
            var file = fileInput.files[0];
            if (!file) return;
            status.textContent = 'Uploading...';
            status.style.color = 'var(--d-amber)';

            var fd = new FormData();
            fd.append('file', file);
            fd.append('folder', 'cms');

            fetch('{% url "cms-upload-image" %}', {
              method: 'POST',
              headers: { 'X-CSRFToken': csrfToken.value },
              body: fd,
            })
            .then(function(r) { return r.json(); })
            .then(function(data) {
              if (data.error) {
                status.textContent = 'Error: ' + data.error;
                status.style.color = 'var(--color-danger)';
                return;
              }
              hiddenInput.value = data.url;
              status.textContent = 'Uploaded';
              status.style.color = 'var(--d-teal)';
              preview.innerHTML = '<img src="' + data.url + '" alt="Preview">';
            })
            .catch(function(err) {
              status.textContent = 'Upload failed';
              status.style.color = 'var(--color-danger)';
            });
          });
        });
      }

      /* Expose for cms-editor.js gallery blocks */
      window.cmsUploadImage = function(file, callback) {
        var fd = new FormData();
        fd.append('file', file);
        fd.append('folder', 'cms/gallery');
        var csrf = document.querySelector('[name=csrfmiddlewaretoken]');

        fetch('{% url "cms-upload-image" %}', {
          method: 'POST',
          headers: { 'X-CSRFToken': csrf ? csrf.value : '' },
          body: fd,
        })
        .then(function(r) { return r.json(); })
        .then(function(data) {
          callback(data.error ? null : data.url, data.error || null);
        })
        .catch(function(err) {
          callback(null, 'Upload failed');
        });
      };

      document.addEventListener('DOMContentLoaded', initUploadWidgets);
    })();
    </script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

**Key things in the CMS base:**

1. **Sidebar nav** uses `request.resolver_match.url_name` to highlight the active link
2. **EditorJS CDN scripts** (7 plugins) are loaded on every CMS page
3. **`cms-editor.js`** and **`cms-json.js`** are loaded as static files
4. **Upload widget IIFE** wires all `.cms-cloudinary-upload` widgets
5. **`window.cmsUploadImage()`** global function for gallery block uploads
6. **`{% block extra_js %}`** lets child templates add inline JS (used by navigation.html)

### `marketing/templates/marketing/cms/dashboard.html` (141 lines)

```html
{% extends "marketing/cms/base.html" %}

{% block title %}Dashboard{% endblock %}

{% block content %}
<div class="cms-page-header">
    <h1>Dashboard</h1>
    <div class="cms-header-actions">
        <a href="{% url 'cms-page-create' %}" class="cms-btn cms-btn-secondary">
            <!-- Plus SVG icon -->
            New Page
        </a>
        <a href="{% url 'cms-post-create' %}" class="cms-btn cms-btn-primary">
            <!-- Plus SVG icon -->
            New Post
        </a>
    </div>
</div>

<div class="cms-stats">
    <div class="cms-stat-card">
        <div class="cms-stat-label">Total Pages</div>
        <div class="cms-stat-value accent">{{ page_count }}</div>
    </div>
    <div class="cms-stat-card">
        <div class="cms-stat-label">Total Posts</div>
        <div class="cms-stat-value accent">{{ post_count }}</div>
    </div>
    <div class="cms-stat-card">
        <div class="cms-stat-label">Published</div>
        <div class="cms-stat-value">{{ published_count }}</div>
    </div>
    <div class="cms-stat-card">
        <div class="cms-stat-label">Drafts</div>
        <div class="cms-stat-value">{{ draft_count }}</div>
    </div>
</div>

<section class="cms-section">
    <div class="cms-section-header">
        <h2>Recent Pages</h2>
        <a href="{% url 'cms-page-list' %}">View all</a>
    </div>
    <div class="cms-card">
        {% if pages %}
        <ul class="cms-recent-list">
            {% for page in pages %}
            <li>
                <div class="item-info">
                    <span class="item-title">{{ page.title }}</span>
                    <span class="item-slug">/{{ page.slug }}/</span>
                    {% if page.status == 'published' %}
                        <span class="cms-badge cms-badge-published">Published</span>
                    {% elif page.status == 'scheduled' %}
                        <span class="cms-badge cms-badge-scheduled">Scheduled</span>
                    {% else %}
                        <span class="cms-badge cms-badge-draft">Draft</span>
                    {% endif %}
                </div>
                <div class="item-actions">
                    <a href="{% url 'cms-page-edit' page_id=page.id %}">Edit</a>
                    <form method="post" action="{% url 'cms-page-toggle-status' page_id=page.id %}" class="cms-inline-form">
                        {% csrf_token %}
                        <input type="hidden" name="next" value="{% url 'cms-dashboard' %}">
                        {% if page.status == 'published' %}
                        <button type="submit" class="cms-btn-inline cms-btn-inline-warning" title="Unpublish">Unpublish</button>
                        {% else %}
                        <button type="submit" class="cms-btn-inline cms-btn-inline-success" title="Publish">Publish</button>
                        {% endif %}
                    </form>
                    <form method="post" action="{% url 'cms-page-delete' page_id=page.id %}" class="cms-inline-form" onsubmit="return confirm('Delete page?')">
                        {% csrf_token %}
                        <button type="submit" class="cms-btn-inline cms-btn-inline-danger" title="Delete">Delete</button>
                    </form>
                </div>
            </li>
            {% endfor %}
        </ul>
        {% else %}
        <div class="cms-empty">
            <h3>No pages yet</h3>
            <p>Create your first page to get started.</p>
            <a href="{% url 'cms-page-create' %}" class="cms-btn cms-btn-primary cms-btn-sm">Create Page</a>
        </div>
        {% endif %}
    </div>
</section>

<!-- Recent Posts section follows the same pattern with post toggle/delete inline forms -->
{% endblock %}
```

### `marketing/templates/marketing/cms/page_list.html` (81 lines)

```html
{% extends "marketing/cms/base.html" %}

{% block title %}Pages{% endblock %}

{% block content %}
<div class="cms-page-header">
    <h1>Pages</h1>
    <div class="cms-header-actions">
        <a href="{% url 'cms-page-create' %}" class="cms-btn cms-btn-primary">
            New Page
        </a>
    </div>
</div>

<div class="cms-card">
    {% if pages %}
    <div class="cms-table-wrap">
        <table class="cms-data-table">
            <thead>
                <tr>
                    <th>Title</th>
                    <th>Slug</th>
                    <th>Status</th>
                    <th>Updated</th>
                    <th></th>
                </tr>
            </thead>
            <tbody>
                {% for page in pages %}
                <tr>
                    <td class="title-cell">{{ page.title }}</td>
                    <td class="slug-cell">/{{ page.slug }}/</td>
                    <td>
                        {% if page.status == 'published' %}
                            <span class="cms-badge cms-badge-published">Published</span>
                        {% elif page.status == 'scheduled' %}
                            <span class="cms-badge cms-badge-scheduled">Scheduled</span>
                        {% else %}
                            <span class="cms-badge cms-badge-draft">Draft</span>
                        {% endif %}
                    </td>
                    <td class="date-cell">{{ page.updated_at|date:"M j, Y" }}</td>
                    <td class="actions-cell">
                        {% if page.status == 'published' %}
                        <a href="/{{ page.slug }}/" target="_blank" class="cms-preview-link">View</a>
                        {% endif %}
                        <a href="{% url 'cms-page-edit' page_id=page.id %}">Edit</a>
                        <form method="post" action="{% url 'cms-page-toggle-status' page_id=page.id %}" class="cms-inline-form">
                            {% csrf_token %}
                            <input type="hidden" name="next" value="{% url 'cms-page-list' %}">
                            {% if page.status == 'published' %}
                            <button type="submit" class="cms-btn-inline cms-btn-inline-warning">Unpublish</button>
                            {% else %}
                            <button type="submit" class="cms-btn-inline cms-btn-inline-success">Publish</button>
                            {% endif %}
                        </form>
                        <form method="post" action="{% url 'cms-page-delete' page_id=page.id %}" class="cms-inline-form" onsubmit="return confirm('Delete page?')">
                            {% csrf_token %}
                            <button type="submit" class="cms-btn-inline cms-btn-inline-danger">Delete</button>
                        </form>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
    {% else %}
    <div class="cms-empty">
        <h3>No pages yet</h3>
        <p>Create your first page to get started.</p>
        <a href="{% url 'cms-page-create' %}" class="cms-btn cms-btn-primary cms-btn-sm">Create Page</a>
    </div>
    {% endif %}
</div>
{% endblock %}
```

### `marketing/templates/marketing/cms/page_form.html` (216 lines)

```html
{% extends "marketing/cms/base.html" %}
{% load form_extras %}

{% block title %}{% if mode == 'create' %}New Page{% else %}Edit: {{ page.title }}{% endif %}{% endblock %}

{% block content %}
<div class="cms-page-header">
    <h1>{% if mode == 'create' %}New Page{% else %}Edit Page{% endif %}</h1>
    <div class="cms-header-actions">
        {% if page and page.status == 'published' %}
        <a href="/{{ page.slug }}/" target="_blank" class="cms-btn cms-btn-ghost">View Page</a>
        {% endif %}
        <a href="{% url 'cms-page-list' %}" class="cms-btn cms-btn-secondary">Back to Pages</a>
    </div>
</div>

<form method="post" enctype="multipart/form-data" class="cms-form" id="page-form">
    {% csrf_token %}

    <!-- Top Action Bar -->
    <div class="cms-form-actions-bar">
        <div class="cms-form-actions-left">
            {% if mode == 'edit' %}
                <button type="submit" name="action" value="save" class="cms-btn cms-btn-secondary">Save Changes</button>
                {% if page.status == 'published' %}
                    <button type="submit" name="action" value="unpublish" class="cms-btn cms-btn-warning" formnovalidate>Unpublish</button>
                {% else %}
                    <button type="submit" name="action" value="publish" class="cms-btn cms-btn-primary">Publish</button>
                {% endif %}
            {% else %}
                <button type="submit" name="action" value="draft" class="cms-btn cms-btn-secondary">Save Draft</button>
                <button type="submit" name="action" value="publish" class="cms-btn cms-btn-primary">Publish</button>
            {% endif %}
        </div>
        <div class="cms-form-actions-right">
            {% if mode == 'edit' %}
            <button type="button" class="cms-btn cms-btn-danger" onclick="document.getElementById('delete-confirm').style.display='flex'">Delete</button>
            {% endif %}
        </div>
    </div>

    <!-- Delete confirmation lives OUTSIDE the main form (no nested forms) -->

    <!-- Core Fields -->
    <div class="cms-card">
        <div class="cms-card-header"><h3>Page Details</h3></div>
        <div class="cms-card-body">
            <div class="cms-form-grid-3">
                <div class="cms-field">
                    <label for="id_title">Title</label>
                    {{ form.title }}
                    {{ form.title.errors }}
                </div>
                <div class="cms-field">
                    <label for="id_slug">Slug</label>
                    {{ form.slug }}
                    <span class="cms-help">URL path: /your-slug/</span>
                </div>
                <div class="cms-field">
                    <label for="id_status">Status</label>
                    {{ form.status }}
                </div>
            </div>
            <!-- publish_at, unpublish_at, primary_image upload widget -->
        </div>
    </div>

    <!-- Blocks Builder -->
    <div class="cms-card">
        <div class="cms-card-header">
            <h3>Content Blocks</h3>
            <span class="cms-help">Drag to reorder. Add rich text, CTAs, callouts, features, pricing, and more.</span>
        </div>
        <div class="cms-card-body">
            <input type="hidden" name="blocks_json" id="id_blocks_json"
                   value='{{ form.blocks_json.value|default_if_none:""|to_json }}' data-blocks-input>
            <div class="cms-blocks-builder" data-blocks-builder></div>
        </div>
    </div>

    <!-- SEO (collapsed by default) -->
    <div class="cms-collapse" id="seo-section">
        <div class="cms-collapse-header" onclick="this.parentElement.classList.toggle('open')">
            <h3>SEO & Social Sharing</h3>
            <svg class="chevron"><!-- chevron icon --></svg>
        </div>
        <div class="cms-collapse-body">
            <!-- seo_title, seo_description, og_title, og_description -->
            <!-- og_image + twitter_image Cloudinary upload widgets -->
        </div>
    </div>

    <!-- Bottom Action Bar (same buttons as top) -->
    <div class="cms-form-actions-bar">
        <div class="cms-form-actions-left">
            <a href="{% url 'cms-page-list' %}" class="cms-btn cms-btn-ghost">Cancel</a>
            {% if mode == 'edit' %}
                <button type="submit" name="action" value="save" class="cms-btn cms-btn-secondary">Save Changes</button>
                {% if page.status == 'published' %}
                    <button type="submit" name="action" value="unpublish" class="cms-btn cms-btn-warning" formnovalidate>Unpublish</button>
                {% else %}
                    <button type="submit" name="action" value="publish" class="cms-btn cms-btn-primary">Publish</button>
                {% endif %}
            {% else %}
                <button type="submit" name="action" value="draft" class="cms-btn cms-btn-secondary">Save Draft</button>
                <button type="submit" name="action" value="publish" class="cms-btn cms-btn-primary">Publish</button>
            {% endif %}
        </div>
        <div class="cms-form-actions-right">
            {% if mode == 'edit' %}
            <button type="button" class="cms-btn cms-btn-danger" onclick="document.getElementById('delete-confirm').style.display='flex'">Delete</button>
            {% endif %}
        </div>
    </div>
</form>

{% if mode == 'edit' %}
<div class="cms-confirm-delete" id="delete-confirm" style="display:none">
    <span>Are you sure you want to delete "<strong>{{ page.title }}</strong>"? This cannot be undone.</span>
    <div class="cms-confirm-actions">
        <button type="button" class="cms-btn cms-btn-ghost cms-btn-sm" onclick="document.getElementById('delete-confirm').style.display='none'">Cancel</button>
        <form method="post" action="{% url 'cms-page-delete' page_id=page.id %}" style="display:inline">
            {% csrf_token %}
            <button type="submit" class="cms-btn cms-btn-danger cms-btn-sm">Yes, Delete</button>
        </form>
    </div>
</div>
{% endif %}
{% endblock %}
```

### `marketing/templates/marketing/cms/post_list.html` (83 lines)

Same pattern as `page_list.html` but with an extra "Author" column. Each row
has the same inline action forms: View (if published), Edit link,
Publish/Unpublish toggle form (`.cms-inline-form` posting to `cms-post-toggle-status`
with hidden `next` field), and Delete form with `confirm()` dialog. See full
source in repo.

### `marketing/templates/marketing/cms/post_form.html` (239 lines)

Same action bar pattern as `page_form.html` (top + bottom `.cms-form-actions-bar`
with Save Draft / Publish / Unpublish / Delete, plus `.cms-confirm-delete`
banner) with these additions:
- **Author name** field
- **Cover image** Cloudinary upload widget (instead of primary_image)
- **Excerpt** textarea
- **Categories & Tags** collapsible section (open by default)
- Same blocks builder and SEO section

### `marketing/templates/marketing/cms/navigation.html` (225 lines)

This template is unique -- it doesn't use the blocks builder. Instead it has
two custom inline JS editors:

1. **Nav Items Editor** -- Manages `items_json` (array of `{label, url}`)
2. **Footer Columns Editor** -- Manages `columns_json` (array of `{title, links[]}`)

Both use the same pattern:
- Parse the JSON from a hidden `<textarea>`
- Render an editable list of input fields
- Sync back to the textarea on every change
- Add/remove buttons

```html
{% block extra_js %}
<script>
(function() {
    // ---- Nav Items Editor ----
    const navInput = document.getElementById('id_items_json');
    const navList = document.getElementById('nav-items-list');
    let navItems = [];
    try { navItems = JSON.parse(navInput.value || '[]'); } catch(e) { navItems = []; }

    function renderNav() {
        navList.innerHTML = '';
        navItems.forEach((item, i) => {
            // Create row with label input, URL input, remove button
        });
        syncNav();
    }

    function syncNav() {
        navInput.value = JSON.stringify(navItems);
    }

    // Event delegation for input changes and remove clicks
    // "Add Link" button handler
    renderNav();

    // ---- Footer Columns Editor ----
    // Same pattern: parse JSON, render columns with nested links,
    // add/remove columns, add/remove links within columns
})();
</script>
{% endblock %}
```

---

### `marketing/templatetags/form_extras.py` (46 lines)

```python
import json

from django import template


register = template.Library()


@register.filter
def add_class(field, css_class):
    attrs = field.field.widget.attrs.copy()
    existing = attrs.get('class', '')
    merged = f"{existing} {css_class}".strip()
    attrs['class'] = merged
    field.field.widget.attrs.update(attrs)
    return field.as_widget(attrs=attrs)


@register.filter
def add_attr(field, attr):
    key, _, value = attr.partition('=')
    attrs = field.field.widget.attrs.copy()
    attrs[key] = value
    field.field.widget.attrs.update(attrs)
    return field.as_widget(attrs=attrs)


@register.filter(is_safe=True)
def to_json(value):
    """Convert a Python object (dict, list, etc.) to a JSON string.

    Handles the case where Django's JSONField returns Python objects
    that need to be serialized for use in HTML attributes or JS.
    """
    if value is None or value == '':
        return ''
    if isinstance(value, str):
        # Already a string -- try parsing to verify it's valid JSON,
        # otherwise treat as raw and re-serialize
        try:
            parsed = json.loads(value)
            return json.dumps(parsed)
        except (json.JSONDecodeError, TypeError):
            return value
    return json.dumps(value)
```

**Three filters:**

1. **`add_class`** -- Add CSS classes to a Django form field widget:
   `{{ form.title|add_class:"cms-input" }}`
2. **`add_attr`** -- Add any HTML attribute:
   `{{ form.title|add_attr:"placeholder=Enter title" }}`
3. **`to_json`** -- Safely serialize Python objects to JSON strings for
   use in HTML attributes. Critical for the `blocks_json` hidden input.

---

## How to Reuse These Templates

1. Copy the `marketing/templates/marketing/` directory
2. Copy `marketing/templatetags/form_extras.py`
3. Create `__init__.py` in the `templatetags/` directory
4. Update `INSTALLED_APPS` to include your marketing app
5. Make sure static files exist: `design-system.css`, `marketing.css`, `cms.css`
6. The CMS templates expect EditorJS CDN scripts and `cms-editor.js`
7. The upload widgets expect the `/cms/upload/` endpoint and CSRF token
