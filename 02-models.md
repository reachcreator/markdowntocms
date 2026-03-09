# 02 — Models

All models live in `marketing/models.py`.

---

## Model Overview

| Model | Purpose |
|-------|---------|
| `SeoFieldsMixin` | Abstract mixin providing SEO/OG/Twitter fields |
| `MediaAsset` | Uploaded media (Cloudinary URL + metadata) |
| `Category` | Blog post categories |
| `Tag` | Blog post tags |
| `Page` | Marketing pages (About, Pricing, Contact, etc.) |
| `Post` | Blog posts |
| `NavMenu` | Navigation menu (stores items as JSON) |
| `NavItem` | Individual nav links (relational, currently unused — NavMenu.items_json is used instead) |
| `Footer` | Footer content (columns, CTA, legal text as JSON) |

---

## Complete Source Code

```python
import re
from django.db import models
from django.urls import reverse
from django.utils import timezone

from .renderers import render_lexical_json, render_editorjs, sanitize_html


class PublishedStatus(models.TextChoices):
    DRAFT = 'draft', 'Draft'
    PUBLISHED = 'published', 'Published'
    SCHEDULED = 'scheduled', 'Scheduled'


class SeoFieldsMixin(models.Model):
    seo_title = models.CharField(max_length=255, blank=True)
    seo_description = models.TextField(blank=True)
    og_title = models.CharField(max_length=255, blank=True)
    og_description = models.TextField(blank=True)
    og_image = models.URLField(max_length=500, blank=True)
    twitter_image = models.URLField(max_length=500, blank=True)


    class Meta:
        abstract = True


class MediaAsset(models.Model):
    file = models.URLField(max_length=500, blank=True)
    alt_text = models.CharField(max_length=255, blank=True)
    caption = models.CharField(max_length=255, blank=True)
    width = models.PositiveIntegerField(blank=True, null=True)
    height = models.PositiveIntegerField(blank=True, null=True)
    content_type = models.CharField(max_length=100, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.alt_text or self.file


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(unique=True)

    class Meta:
        ordering = ['name']

    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(unique=True)

    class Meta:
        ordering = ['name']

    def __str__(self):
        return self.name

> **Important**: When creating categories and tags for your blog, use generic names relevant to your industry and content strategy. Avoid hardcoding examples from this documentation (like "Legal Tech" or "AI") — these were specific to the original project. Choose categories like "Industry News", "Product Updates", "How-To", "Case Studies" that can work for any business.


class Page(SeoFieldsMixin):
    title = models.CharField(max_length=255)
    slug = models.SlugField(unique=True)
    status = models.CharField(
        max_length=20,
        choices=PublishedStatus.choices,
        default=PublishedStatus.DRAFT,
    )
    publish_at = models.DateTimeField(blank=True, null=True)
    unpublish_at = models.DateTimeField(blank=True, null=True)
    body_json = models.JSONField(blank=True, null=True)
    body_html = models.TextField(blank=True)
    blocks_json = models.JSONField(blank=True, null=True)
    blocks_html = models.TextField(blank=True)
    primary_image = models.URLField(max_length=500, blank=True)
    updated_at = models.DateTimeField(auto_now=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        abstract = False
        ordering = ['title']

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('marketing-page-detail', kwargs={'slug': self.slug})

    def save(self, *args, **kwargs):
        if self.body_json:
            rendered = render_editorjs(self.body_json)
            self.body_html = sanitize_html(rendered)
        else:
            self.body_html = ''
        if self.blocks_json:
            from .blocks import render_blocks

            self.blocks_html = sanitize_html(render_blocks(self.blocks_json))
        super().save(*args, **kwargs)

    @property
    def is_published(self):
        now = timezone.now()
        if self.status == PublishedStatus.DRAFT:
            return False
        if self.status == PublishedStatus.PUBLISHED:
            if self.unpublish_at and self.unpublish_at <= now:
                return False
            return True
        if self.status == PublishedStatus.SCHEDULED:
            return bool(self.publish_at and self.publish_at <= now)
        return False


class Post(SeoFieldsMixin):
    title = models.CharField(max_length=255)
    slug = models.SlugField(unique=True)
    status = models.CharField(
        max_length=20,
        choices=PublishedStatus.choices,
        default=PublishedStatus.DRAFT,
    )
    publish_at = models.DateTimeField(blank=True, null=True)
    author_name = models.CharField(max_length=255, blank=True)
    excerpt = models.TextField(blank=True)
    categories = models.ManyToManyField(Category, blank=True, related_name='posts')
    tags = models.ManyToManyField(Tag, blank=True, related_name='posts')
    body_json = models.JSONField(blank=True, null=True)
    body_html = models.TextField(blank=True)
    blocks_json = models.JSONField(blank=True, null=True)
    blocks_html = models.TextField(blank=True)
    cover_image = models.URLField(max_length=500, blank=True)
    primary_image = models.URLField(max_length=500, blank=True)
    updated_at = models.DateTimeField(auto_now=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        abstract = False
        ordering = ['-publish_at', '-created_at']

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('marketing-blog-post', kwargs={'slug': self.slug})

    def save(self, *args, **kwargs):
        if self.body_json:
            rendered = render_editorjs(self.body_json)
            self.body_html = sanitize_html(rendered)
        else:
            self.body_html = ''
        if self.blocks_json:
            from .blocks import render_blocks

            self.blocks_html = sanitize_html(render_blocks(self.blocks_json))
        super().save(*args, **kwargs)

    @property
    def is_published(self):
        now = timezone.now()
        if self.status == PublishedStatus.DRAFT:
            return False
        if self.status == PublishedStatus.PUBLISHED:
            return True
        if self.status == PublishedStatus.SCHEDULED:
            return bool(self.publish_at and self.publish_at <= now)
        return False

    @property
    def computed_excerpt(self):
        """Return the explicit excerpt, or auto-generate from content."""
        if self.excerpt:
            return self.excerpt
        # Fall back to first paragraph from body_html or blocks_html
        html = self.body_html or self.blocks_html or ''
        match = re.search(r'<p[^>]*>(.*?)</p>', html, flags=re.I | re.S)
        if match:
            raw = re.sub(r'<[^>]+>', '', match.group(1)).strip()
            from django.utils.text import Truncator
            return Truncator(raw).chars(200)
        return ''


class NavMenu(models.Model):
    name = models.CharField(max_length=100, unique=True)
    items_json = models.JSONField(blank=True, null=True)

    def __str__(self):
        return self.name


class NavItem(models.Model):
    menu = models.ForeignKey(NavMenu, on_delete=models.CASCADE, related_name='items')
    label = models.CharField(max_length=100)
    url = models.CharField(max_length=255, blank=True)
    page = models.ForeignKey(Page, on_delete=models.SET_NULL, blank=True, null=True)
    external = models.BooleanField(default=False)
    parent = models.ForeignKey('self', on_delete=models.CASCADE, blank=True, null=True)
    order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['order']

    def __str__(self):
        return self.label


class Footer(models.Model):
    label = models.CharField(max_length=100, default='Default')
    columns_json = models.JSONField(blank=True, null=True)
    cta_title = models.CharField(max_length=255, blank=True)
    cta_body = models.TextField(blank=True)
    cta_button_label = models.CharField(max_length=100, blank=True)
    cta_button_url = models.CharField(max_length=255, blank=True)
    legal_text = models.TextField(blank=True)

    def __str__(self):
        return self.label
```

---

## Key Patterns Explained

### The `save()` Override

Both `Page` and `Post` override `save()` to render content:

```python
def save(self, *args, **kwargs):
    # Render body_json (legacy EditorJS content) → body_html
    if self.body_json:
        rendered = render_editorjs(self.body_json)
        self.body_html = sanitize_html(rendered)
    else:
        self.body_html = ''

    # Render blocks_json → blocks_html
    if self.blocks_json:
        from .blocks import render_blocks
        self.blocks_html = sanitize_html(render_blocks(self.blocks_json))

    super().save(*args, **kwargs)
```

This means every time content is saved (via CMS form, admin, or management command), the HTML is regenerated. The import of `render_blocks` is deferred to avoid circular imports.

Sanitization is layered:

- `render_editorjs()` sanitizes inline HTML via BeautifulSoup allowlists (tags/attrs).
- `sanitize_html()` removes `<script>` and inline `on*=` handlers for normal rendered output.
- Content inside `<!--TRUSTED_CODE_START-->...<!--TRUSTED_CODE_END-->` markers (from CMS `code` blocks) is treated as trusted and preserved.

### The `is_published` Property

Provides runtime publish logic without a cron job:

- **Draft**: never published
- **Published**: always published (unless `unpublish_at` has passed — Page only)
- **Scheduled**: published only if `publish_at` is in the past

Note: `Post.is_published` does not check `unpublish_at` (posts don't have that field).

Note on scheduling in the CMS UI:

- The model supports `status='scheduled'` + `publish_at`.
- The CMS “Publish” button forces `status='published'` immediately (ignores `publish_at`).
- To schedule, set Status=Scheduled and `publish_at`, then click “Save Changes” (not Publish).
- List “Publish” toggles also force scheduled → published.

### Image Fields as URLField

All image fields (`og_image`, `twitter_image`, `primary_image`, `cover_image`, `MediaAsset.file`) are `URLField(max_length=500, blank=True)`. They store Cloudinary URLs like:

```
https://res.cloudinary.com/<cloud>/image/upload/v123/cms/blog/ai-contract-review-2026-cover.jpg
```

No `.url` accessor needed — the field value IS the URL.

### NavMenu.items_json

Stores navigation as a JSON array:

```json
[
    {"label": "About", "url": "/about/"},
    {"label": "Pricing", "url": "/pricing/"},
    {"label": "Blog", "url": "/blog/"},
    {"label": "Contact", "url": "/contact/"}
]
```

The `NavItem` relational model exists but is unused — `items_json` is simpler and sufficient.

---

## Blocks JSON Shapes

### Canonical `blocks_json` envelope

The block builder stores a JSON object:

```json
{ "blocks": [ { "type": "rich_text", "content": { "blocks": [] } } ] }
```

For backward compatibility, the server and builder also tolerate:

- `null` / `"null"` / empty strings for new pages (normalized to `{ "blocks": [] }`)
- A raw array (`[ ... ]`) (normalized to `{ "blocks": [ ... ] }`)

### `code` block (trusted embeds)

```json
{ "type": "code", "html": "<script src=\"...\"></script>" }
```

This is intended for trusted third-party embeds. It is wrapped in markers and preserved through sanitization.

### Footer.columns_json

Stores footer columns as a JSON array:

```json
[
    {
        "title": "Links",
        "links": [
            {"label": "Privacy Notice", "url": "/privacy/"},
            {"label": "Terms of Use", "url": "/terms/"},
            {"label": "Contact Us", "url": "/contact/"}
        ]
    }
]
```

> **Important**: Always include a link to `/sitemap.xml` in your footer columns. This ensures search engines can discover all your content and is an SEO best practice. Add it under a "Links" or "Resources" column:

```json
{"label": "Sitemap", "url": "/sitemap.xml"}
```

---

## Forms Reference

The forms live in `marketing/forms.py` (59 lines):

```python
from django import forms

from .models import Page, Post, NavMenu, Footer


class PageForm(forms.ModelForm):
    class Meta:
        model = Page
        fields = [
            'title', 'slug', 'status', 'publish_at', 'unpublish_at',
            'seo_title', 'seo_description', 'og_title', 'og_description',
            'og_image', 'twitter_image', 'primary_image',
            'body_json', 'blocks_json',
        ]

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # body_json is kept in the form for backward compat but hidden;
        # all content now lives in blocks_json via rich_text blocks.
        self.fields['body_json'].required = False
        self.fields['body_json'].widget = forms.HiddenInput()


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = [
            'title', 'slug', 'status', 'publish_at', 'author_name', 'excerpt',
            'seo_title', 'seo_description', 'og_title', 'og_description',
            'og_image', 'twitter_image', 'primary_image', 'cover_image',
            'categories', 'tags', 'body_json', 'blocks_json',
        ]

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['body_json'].required = False
        self.fields['body_json'].widget = forms.HiddenInput()


class NavMenuForm(forms.ModelForm):
    class Meta:
        model = NavMenu
        fields = ['name', 'items_json']

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['name'].widget = forms.HiddenInput()
        if not self.instance or not self.instance.name:
            self.initial['name'] = 'Primary'


class FooterForm(forms.ModelForm):
    class Meta:
        model = Footer
        fields = [
            'label', 'columns_json', 'cta_title', 'cta_body',
            'cta_button_label', 'cta_button_url', 'legal_text',
        ]
```

Note: `body_json` is hidden because all content now uses `blocks_json` with `rich_text` blocks. The field remains for backward compatibility.

Important: the CMS views override `status` based on which submit button was clicked (`action=draft|publish|save|unpublish`). The `status` dropdown value is not always authoritative when using action buttons.
