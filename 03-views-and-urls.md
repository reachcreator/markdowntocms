# 03 -- Views, URLs & Permissions

> Every URL in the marketing app, the views that power them, the permission
> decorator that locks down the CMS, and the SEO payload builder that feeds
> meta tags to every template.

---

## URL Map at a Glance

| URL pattern | View | Auth | Purpose |
|---|---|---|---|
| `/` | `views.marketing_home` | Public | Homepage (standalone template) |
| `/sitemap.xml` | `django.contrib.sitemaps.views.sitemap` (project-level) | Public | XML sitemap for Google |
| `/robots.txt` | `views.robots_txt` | Public | Robots.txt with sitemap ref |
| `/blog/` | `views.blog_index` | Public | Blog listing |
| `/blog/<slug>/` | `views.blog_post_detail` | Public | Single blog post |
| `/cms/` | `cms_views.dashboard` | CMS Admin | Dashboard with stats |
| `/cms/pages/` | `cms_views.page_list` | CMS Admin | Page listing |
| `/cms/pages/new/` | `cms_views.page_create` | CMS Admin | Create page |
| `/cms/pages/<id>/` | `cms_views.page_edit` | CMS Admin | Edit page |
| `/cms/pages/<id>/delete/` | `cms_views.page_delete` | CMS Admin | Delete page (POST only) |
| `/cms/pages/<id>/toggle-status/` | `cms_views.page_toggle_status` | CMS Admin | Publish/unpublish page (POST only) |
| `/cms/posts/` | `cms_views.post_list` | CMS Admin | Post listing |
| `/cms/posts/new/` | `cms_views.post_create` | CMS Admin | Create post |
| `/cms/posts/<id>/` | `cms_views.post_edit` | CMS Admin | Edit post |
| `/cms/posts/<id>/delete/` | `cms_views.post_delete` | CMS Admin | Delete post (POST only) |
| `/cms/posts/<id>/toggle-status/` | `cms_views.post_toggle_status` | CMS Admin | Publish/unpublish post (POST only) |
| `/cms/navigation/` | `cms_views.navigation_edit` | CMS Admin | Nav + footer editor |
| `/cms/upload/` | `cms_views.upload_image` | CMS Admin | Cloudinary image upload (AJAX) |
| `/<slug>/` | `views.page_detail` | Public | Catch-all page detail |

**Important:** The `<slug>/` catch-all is the LAST urlpattern so it doesn't
shadow `/blog/`, `/cms/`, etc.

`/sitemap.xml` should be served by a single sitemap implementation (recommended: Django's sitemap framework), using sitemap classes for pages/posts and a static sitemap for key non-model URLs.

---

## Key Patterns

### 1. Nav + Footer Loading

Every public view loads the same nav and footer objects:

```python
nav = NavMenu.objects.filter(name='Primary').first()
footer = Footer.objects.first()
```

These are passed to the template context so `base.html` can render the
site-wide navigation and footer. The nav is filtered by `name='Primary'`
which means you could have multiple nav menus in the DB, but only one is
used on the public site.

### 2. Publish Gate

Public views check `page.is_published` / `post.is_published` and raise
`Http404` if the content isn't live. The `is_published` property on the
model checks both `status == 'published'` and `publish_at <= now`.

```python
if not page.is_published:
    raise Http404()
```

The blog index also filters in Python after the queryset:

```python
posts = Post.objects.filter(status__in=['published', 'scheduled']).order_by('-publish_at')
posts = [post for post in posts if post.is_published]
```

This catches scheduled posts whose `publish_at` is still in the future.

### 3. CMS Create/Edit Pattern

All CMS CRUD views follow the same Django pattern:

```python
@cms_admin_required
def page_edit(request, page_id):
    page = get_object_or_404(Page, id=page_id)
    if request.method == 'POST':
        action = request.POST.get('action')
        if action == 'unpublish':
            page.status = 'draft'
            page.save(update_fields=['status', 'updated_at'])
            return redirect('cms-page-edit', page_id=page.id)

        form = PageForm(request.POST, request.FILES, instance=page)
        if form.is_valid():
            _apply_action_status(form, request)
            form.save()
            return redirect('cms-page-edit', page_id=page.id)
    else:
        form = PageForm(instance=page)
    return render(request, 'marketing/cms/page_form.html', {
        'form': form, 'mode': 'edit', 'page': page
    })
```

On successful save, it redirects back to the same edit page (not to a list).
The `mode` variable (`'create'` or `'edit'`) lets the template show the
appropriate heading and button text. The `_apply_action_status()` call
(see Pattern 5) overrides the status field based on which submit button
the user clicked.

### 4. SEO Payload

Every public view passes an `seo` dict to the template. For pages/posts with
model instances, it calls `build_seo_payload(obj, request=request)`. For the
homepage and blog index, it builds a manual dict. The SEO payload feeds the
`<title>`, `<meta>`, and Open Graph tags in `base.html`.

---

## Complete Source Files

### `marketing/urls.py` (28 lines)

```python
from django.urls import path

from . import views
from . import cms_views


urlpatterns = [
    path('', views.marketing_home, name='marketing-home'),
    path('sitemap.xml', views.sitemap_xml, name='sitemap'),
    path('robots.txt', views.robots_txt, name='robots-txt'),
    path('blog/', views.blog_index, name='marketing-blog-index'),
    path('blog/<slug:slug>/', views.blog_post_detail, name='marketing-blog-post'),
    path('cms/', cms_views.dashboard, name='cms-dashboard'),
    path('cms/pages/', cms_views.page_list, name='cms-page-list'),
    path('cms/pages/new/', cms_views.page_create, name='cms-page-create'),
    path('cms/pages/<int:page_id>/', cms_views.page_edit, name='cms-page-edit'),
    path('cms/pages/<int:page_id>/delete/', cms_views.page_delete, name='cms-page-delete'),
    path('cms/pages/<int:page_id>/toggle-status/', cms_views.page_toggle_status, name='cms-page-toggle-status'),
    path('cms/posts/', cms_views.post_list, name='cms-post-list'),
    path('cms/posts/new/', cms_views.post_create, name='cms-post-create'),
    path('cms/posts/<int:post_id>/', cms_views.post_edit, name='cms-post-edit'),
    path('cms/posts/<int:post_id>/delete/', cms_views.post_delete, name='cms-post-delete'),
    path('cms/posts/<int:post_id>/toggle-status/', cms_views.post_toggle_status, name='cms-post-toggle-status'),
    path('cms/navigation/', cms_views.navigation_edit, name='cms-navigation'),
    path('cms/upload/', cms_views.upload_image, name='cms-upload-image'),
    path('<slug:slug>/', views.page_detail, name='marketing-page-detail'),
]
```

### `marketing/permissions.py` (13 lines)

```python
from django.contrib.auth.decorators import login_required
from django.core.exceptions import PermissionDenied


def cms_admin_required(view_func):
    @login_required
    def _wrapped(request, *args, **kwargs):
        if request.user.groups.filter(name='CMS Admins').exists():
            return view_func(request, *args, **kwargs)
        raise PermissionDenied('CMS access restricted')

    return _wrapped
```

**How it works:**

1. `@login_required` redirects anonymous users to the login page.
2. If logged in, it checks if the user belongs to the `CMS Admins` group.
3. If not in the group, it raises `PermissionDenied` (Django returns 403).

Every CMS view is decorated with `@cms_admin_required`. The upload endpoint
also stacks `@require_POST` on top for HTTP method enforcement.

### `marketing/views.py` (152 lines) -- Public Views

```python
from django.http import Http404, HttpResponse
from django.shortcuts import render, get_object_or_404
from django.utils import timezone

from .models import Page, Post, NavMenu, Footer, Category
from .seo import build_seo_payload


def marketing_home(request):
    nav = NavMenu.objects.filter(name='Primary').first()
    footer = Footer.objects.first()
    return render(request, 'marketing/home.html', {
        'nav_menu': nav,
        'footer': footer,
        'seo': {
            'title': 'Example Company',
            'description': 'Example marketing site',
            'og_title': 'Example Company',
            'og_description': 'Example marketing site',
            'image': None,
            'url': request.build_absolute_uri('/'),
        },
    })


def page_detail(request, slug):
    page = get_object_or_404(Page, slug=slug)
    if not page.is_published:
        raise Http404()
    nav = NavMenu.objects.filter(name='Primary').first()
    footer = Footer.objects.first()
    return render(request, 'marketing/page.html', {
        'page': page,
        'nav_menu': nav,
        'footer': footer,
        'seo': build_seo_payload(page, request=request),
    })


def blog_index(request):
    posts = Post.objects.filter(status__in=['published', 'scheduled']).order_by('-publish_at')
    posts = [post for post in posts if post.is_published]
    categories = Category.objects.all()  # For category filter chips
    nav = NavMenu.objects.filter(name='Primary').first()
    footer = Footer.objects.first()
    return render(request, 'marketing/blog_index.html', {
        'posts': posts,
        'categories': categories,
        'nav_menu': nav,
        'footer': footer,
        'seo': {
            'title': 'Company Blog',
            'description': 'Insights from the team',
            'og_title': 'Company Blog',
            'og_description': 'Insights from the team',
            'image': None,
        },
    })


def blog_post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug)
    if not post.is_published:
        raise Http404()
    nav = NavMenu.objects.filter(name='Primary').first()
    footer = Footer.objects.first()
    return render(request, 'marketing/blog_post.html', {
        'post': post,
        'nav_menu': nav,
        'footer': footer,
        'seo': build_seo_payload(post, request=request),
    })


def sitemap_xml(request):
    """Google-compliant XML sitemap listing all published pages and posts."""
    now = timezone.now()
    base = request.build_absolute_uri('/').rstrip('/')

    urls = []

    # Homepage
    urls.append({
        'loc': f'{base}/',
        'changefreq': 'weekly',
        'priority': '1.0',
    })

    # Blog index
    urls.append({
        'loc': f'{base}/blog/',
        'changefreq': 'daily',
        'priority': '0.8',
    })

    # Published pages
    pages = Page.objects.filter(
        status__in=['published', 'scheduled'],
    ).order_by('title')
    for page in pages:
        if not page.is_published:
            continue
        urls.append({
            'loc': f'{base}/{page.slug}/',
            'lastmod': page.updated_at.strftime('%Y-%m-%d'),
            'changefreq': 'monthly',
            'priority': '0.7',
        })

    # Published posts
    posts = Post.objects.filter(
        status__in=['published', 'scheduled'],
    ).order_by('-publish_at')
    for post in posts:
        if not post.is_published:
            continue
        urls.append({
            'loc': f'{base}/blog/{post.slug}/',
            'lastmod': post.updated_at.strftime('%Y-%m-%d'),
            'changefreq': 'monthly',
            'priority': '0.6',
        })

    # Build XML
    lines = [
        '<?xml version="1.0" encoding="UTF-8"?>',
        '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">',
    ]
    for entry in urls:
        lines.append('  <url>')
        lines.append(f'    <loc>{entry["loc"]}</loc>')
        if 'lastmod' in entry:
            lines.append(f'    <lastmod>{entry["lastmod"]}</lastmod>')
        lines.append(f'    <changefreq>{entry["changefreq"]}</changefreq>')
        lines.append(f'    <priority>{entry["priority"]}</priority>')
        lines.append('  </url>')
    lines.append('</urlset>')

    xml = '\n'.join(lines)
    return HttpResponse(xml, content_type='application/xml')


def robots_txt(request):
    """Serve robots.txt with sitemap reference."""
    base = request.build_absolute_uri('/').rstrip('/')
    content = (
        'User-agent: *\n'
        'Allow: /\n'
        'Disallow: /cms/\n'
        'Disallow: /admin/\n'
        f'\nSitemap: {base}/sitemap.xml\n'
    )
    return HttpResponse(content, content_type='text/plain')
```

### `marketing/cms_views.py` (195 lines) -- CMS Admin Views

```python
from django.shortcuts import render, get_object_or_404, redirect
from django.http import JsonResponse
from django.views.decorators.http import require_POST

from .forms import PageForm, PostForm, NavMenuForm, FooterForm
from .models import Page, Post, NavMenu, Footer
from .permissions import cms_admin_required


def _apply_action_status(form, request):
    """Override form status based on submit-button action (draft/publish)."""
    action = request.POST.get('action')
    if action == 'draft':
        form.instance.status = 'draft'
    elif action == 'publish':
        form.instance.status = 'published'


@cms_admin_required
def dashboard(request):
    pages = Page.objects.all().order_by('-updated_at')[:5]
    posts = Post.objects.all().order_by('-updated_at')[:5]

    page_count = Page.objects.count()
    post_count = Post.objects.count()
    published_count = (
        Page.objects.filter(status='published').count()
        + Post.objects.filter(status='published').count()
    )
    draft_count = (
        Page.objects.filter(status='draft').count()
        + Post.objects.filter(status='draft').count()
    )

    return render(request, 'marketing/cms/dashboard.html', {
        'pages': pages,
        'posts': posts,
        'page_count': page_count,
        'post_count': post_count,
        'published_count': published_count,
        'draft_count': draft_count,
    })


@cms_admin_required
def page_list(request):
    pages = Page.objects.all().order_by('-updated_at')
    return render(request, 'marketing/cms/page_list.html', {'pages': pages})


@cms_admin_required
def page_create(request):
    if request.method == 'POST':
        form = PageForm(request.POST, request.FILES)
        if form.is_valid():
            _apply_action_status(form, request)
            page = form.save()
            return redirect('cms-page-edit', page_id=page.id)
    else:
        form = PageForm()
    return render(request, 'marketing/cms/page_form.html', {'form': form, 'mode': 'create'})


@cms_admin_required
def page_edit(request, page_id):
    page = get_object_or_404(Page, id=page_id)
    if request.method == 'POST':
        form = PageForm(request.POST, request.FILES, instance=page)
        if form.is_valid():
            _apply_action_status(form, request)
            form.save()
            return redirect('cms-page-edit', page_id=page.id)
    else:
        form = PageForm(instance=page)
    return render(request, 'marketing/cms/page_form.html', {'form': form, 'mode': 'edit', 'page': page})


@cms_admin_required
def post_list(request):
    posts = Post.objects.all().order_by('-updated_at')
    return render(request, 'marketing/cms/post_list.html', {'posts': posts})


@cms_admin_required
def post_create(request):
    if request.method == 'POST':
        form = PostForm(request.POST, request.FILES)
        if form.is_valid():
            _apply_action_status(form, request)
            post = form.save()
            return redirect('cms-post-edit', post_id=post.id)
    else:
        form = PostForm()
    return render(request, 'marketing/cms/post_form.html', {'form': form, 'mode': 'create'})


@cms_admin_required
def post_edit(request, post_id):
    post = get_object_or_404(Post, id=post_id)
    if request.method == 'POST':
        action = request.POST.get('action')
        if action == 'unpublish':
            post.status = 'draft'
            post.save(update_fields=['status', 'updated_at'])
            return redirect('cms-post-edit', post_id=post.id)

        form = PostForm(request.POST, request.FILES, instance=post)
        if form.is_valid():
            _apply_action_status(form, request)
            form.save()
            return redirect('cms-post-edit', post_id=post.id)
    else:
        form = PostForm(instance=post)
    return render(request, 'marketing/cms/post_form.html', {'form': form, 'mode': 'edit', 'post': post})


@cms_admin_required
def navigation_edit(request):
    nav = NavMenu.objects.filter(name='Primary').first()
    footer = Footer.objects.first()

    if request.method == 'POST':
        nav_form = NavMenuForm(request.POST, instance=nav)
        footer_form = FooterForm(request.POST, instance=footer)
        if nav_form.is_valid() and footer_form.is_valid():
            saved_nav = nav_form.save(commit=False)
            if not saved_nav.name:
                saved_nav.name = 'Primary'
            saved_nav.save()
            footer_form.save()
            return redirect('cms-navigation')
    else:
        nav_form = NavMenuForm(instance=nav)
        footer_form = FooterForm(instance=footer)

    return render(request, 'marketing/cms/navigation.html', {
        'nav_form': nav_form,
        'footer_form': footer_form,
    })


@cms_admin_required
@require_POST
def page_delete(request, page_id):
    page = get_object_or_404(Page, id=page_id)
    page.delete()
    return redirect('cms-page-list')


@cms_admin_required
@require_POST
def post_delete(request, post_id):
    post = get_object_or_404(Post, id=post_id)
    post.delete()
    return redirect('cms-post-list')


@cms_admin_required
@require_POST
def page_toggle_status(request, page_id):
    page = get_object_or_404(Page, id=page_id)
    if page.status == 'published':
        page.status = 'draft'
    else:
        page.status = 'published'
    page.save()
    next_url = request.POST.get('next') or request.META.get('HTTP_REFERER') or 'cms-page-list'
    return redirect(next_url)


@cms_admin_required
@require_POST
def post_toggle_status(request, post_id):
    post = get_object_or_404(Post, id=post_id)
    if post.status == 'published':
        post.status = 'draft'
    else:
        post.status = 'published'
    post.save()
    next_url = request.POST.get('next') or request.META.get('HTTP_REFERER') or 'cms-post-list'
    return redirect(next_url)


@cms_admin_required
@require_POST
def upload_image(request):
    """AJAX endpoint: upload an image to Cloudinary, return its URL."""
    file = request.FILES.get('file')
    if not file:
        return JsonResponse({'error': 'No file provided'}, status=400)

    folder = request.POST.get('folder', 'cms')

    from .cloudinary_utils import upload_image as do_upload
    try:
        result = do_upload(file, folder=folder)
    except Exception as e:
        return JsonResponse({'error': str(e)}, status=500)

    return JsonResponse(result)
```

### `marketing/seo.py` (54 lines) -- SEO Payload Builder

```python
import re
from django.utils.text import Truncator


def extract_first_image(html):
    if not html:
        return None
    match = re.search(r'<img[^>]+src="([^"]+)"', html)
    if not match:
        return None
    return match.group(1).strip()


def extract_first_paragraph(html, limit=160):
    if not html:
        return ''
    match = re.search(r'<p[^>]*>(.*?)</p>', html, flags=re.I | re.S)
    if not match:
        return ''
    raw = re.sub(r'<[^>]+>', '', match.group(1)).strip()
    return Truncator(raw).chars(limit)


def build_seo_payload(obj, request=None):
    html = getattr(obj, 'body_html', '') or ''
    title = obj.seo_title or obj.title
    description = obj.seo_description or extract_first_paragraph(html)
    og_title = obj.og_title or title
    og_description = obj.og_description or description
    image = None
    if obj.og_image:
        image = obj.og_image
    elif obj.twitter_image:
        image = obj.twitter_image
    elif getattr(obj, 'primary_image', None):
        image = obj.primary_image
    elif getattr(obj, 'cover_image', None):
        image = obj.cover_image
    else:
        image = extract_first_image(html)

    payload = {
        'title': title,
        'description': description,
        'og_title': og_title,
        'og_description': og_description,
        'image': image,
    }
    if request:
        payload['url'] = request.build_absolute_uri(obj.get_absolute_url())
        if payload.get('image') and not payload['image'].startswith('http'):
            payload['image'] = request.build_absolute_uri(payload['image'])
    return payload
```

**SEO Fallback Chain:**

1. **Title:** `seo_title` -> `title`
2. **Description:** `seo_description` -> first `<p>` from rendered HTML (truncated to 160 chars)
3. **OG Title:** `og_title` -> computed title
4. **OG Description:** `og_description` -> computed description
5. **Image:** `og_image` -> `twitter_image` -> `primary_image` -> `cover_image` -> first `<img>` in HTML
6. **URL:** Built from `obj.get_absolute_url()` via `request.build_absolute_uri()`

If an image URL doesn't start with `http`, it's made absolute using the request.

> **Important - Make SEO work with blocks**: The SEO builder above looks at `body_html`, but if you're using the blocks system (`blocks_json` -> `blocks_html`), this will be empty! Update `build_seo_payload()` to also check `blocks_html`:
>
> ```python
> html = getattr(obj, 'body_html', '') or ''
> if not html:
>     html = getattr(obj, 'blocks_html', '') or ''
> ```
>
> Similarly, the image extraction only looks at `body_html`. If using blocks, you need to extract images from `blocks_json` as well. Add a helper that parses `blocks_json` to find the first image in `rich_text` blocks or `image_gallery` blocks.

> **Important - Categories should do something**: Blog categories are stored but often not displayed or used for SEO. At minimum:
> - Display categories on the blog post detail page (below the content)
> - Add category links to the blog index page (as filter chips)
> - Consider adding `article:tag` meta tags for each category (helps with social sharing)
> - Use categories to generate `<meta name="keywords">` or `article:section` OpenGraph tags

> **Extracting images from blocks_json**: The current `extract_first_image()` only searches `body_html`. If using blocks, you'll need a separate helper to extract images from `blocks_json`:
>
> ```python
> def extract_first_image_from_blocks(blocks_json):
>     if not blocks_json:
>         return None
>     blocks = blocks_json.get('blocks', []) if isinstance(blocks_json, dict) else blocks_json
>     for block in blocks:
>         # Check rich_text blocks for embedded images
>         if block.get('type') == 'rich_text':
>             content = block.get('content', {})
>             if isinstance(content, dict):
>                 for b in content.get('blocks', []):
>                     if b.get('type') == 'image':
>                         return b.get('data', {}).get('file', {}).get('url')
>         # Check image_gallery blocks
>         if block.get('type') == 'image_gallery':
>             images = block.get('images', [])
>             if images and images[0].get('src'):
>                 return images[0]['src']
>     return None
> ```

### 5. Save Draft / Publish Actions

The form templates use named submit buttons (`name="action"`) so one form can support multiple outcomes.

```html
<button type="submit" name="action" value="draft">Save Draft</button>
<button type="submit" name="action" value="publish">Publish</button>
<button type="submit" name="action" value="save">Save Changes</button>
<button type="submit" name="action" value="unpublish" formnovalidate>Unpublish</button>
```

The `_apply_action_status()` helper in `cms_views.py` reads `request.POST['action']`
and overrides `form.instance.status` before the form saves (draft/publish only):

```python
def _apply_action_status(form, request):
    action = request.POST.get('action')
    if action == 'draft':
        form.instance.status = 'draft'
    elif action == 'publish':
        form.instance.status = 'published'
```

Edit-mode unpublish is handled as a fast-path in `page_edit()` / `post_edit()`:

```python
action = request.POST.get('action')
if action == 'unpublish':
    obj.status = 'draft'
    obj.save(update_fields=['status', 'updated_at'])
    return redirect(...)
```

This is called from the create views and the standard save/publish edit flow after `form.is_valid()` but before `form.save()`.

Pitfall: `action=publish` forces `status='published'` and will override a selected `scheduled` status.
Use `action=save` if you want to keep `scheduled`.

### 6. Delete & Toggle Status

Four POST-only views handle deletion and status toggling without requiring
a separate confirmation page:

- **`page_delete` / `post_delete`**: Decorated with `@require_POST`, fetches
  the object via `get_object_or_404()`, calls `.delete()`, and redirects to
  the list view. The form templates show an inline confirmation banner
  (`.cms-confirm-delete`) that the user must click through before the POST
  is submitted.

- **`page_toggle_status` / `post_toggle_status`**: Decorated with
  `@require_POST`, flips `status` between `'published'` and `'draft'`,
  saves, and redirects. The redirect target is determined by a `next`
  hidden field in the form, falling back to `HTTP_REFERER`, then to the
  list URL. This allows toggle buttons on the dashboard, list views, and
  edit forms to all redirect back to wherever the user came from.

```python
next_url = request.POST.get('next') or request.META.get('HTTP_REFERER') or 'cms-page-list'
return redirect(next_url)
```

---

## Sitemap & Robots

**`/sitemap.xml`** -- Served by Django's sitemap framework. Ensure it includes:

- Homepage (`/`)
- Blog index (`/blog/`)
- All published pages
- All published posts

**`/robots.txt`** -- Allows all crawlers, blocks `/cms/` and `/admin/`,
references the sitemap URL.

---

## How to Reuse This

To drop this view layer into a new project:

1. Copy `views.py`, `cms_views.py`, `urls.py`, `permissions.py`, `seo.py`
2. Wire `marketing.urls` into the root `urls.py` (typically at `path('', include('marketing.urls'))`)
3. Make sure the `CMS Admins` group exists in the DB
4. Make sure your models have the expected fields (`seo_title`, `og_image`, `get_absolute_url()`, etc.)
5. Create the templates referenced by each view
