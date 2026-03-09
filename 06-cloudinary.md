# 06 — Cloudinary Image Storage

> All CMS images are stored on Cloudinary and referenced via plain `URLField` columns.
> There is no `CloudinaryField`, no `django-cloudinary-storage` — just a thin utility
> module, one upload endpoint, and two JS integration points.

---

## Architecture Overview

```
                        ┌──────────────────┐
  CMS Form Upload  ──► │  POST /cms/upload/│ ──► cloudinary_utils.upload_image()
  (cover_image,         │  (cms_views.py)   │            │
   og_image fields)     └──────────────────┘            ▼
                                                ┌──────────────┐
  Gallery Block    ──► window.cmsUploadImage() ─►│  Cloudinary   │
  Upload Button         (cms/base.html IIFE)     │  Python SDK   │
  (cms-editor.js)       │                        └──────┬───────┘
                        ▼                               │
                   POST /cms/upload/  ◄─────────────────┘
                        │                          Returns JSON:
                        ▼                          {url, public_id,
                   Hidden <input>                   width, height}
                   gets Cloudinary URL
                   stored in URLField
```

Two upload paths, one endpoint:

| Path | Trigger | Folder | Result |
|------|---------|--------|--------|
| Cover/OG image fields | `<input type="file">` in `.cms-cloudinary-upload` widget | `cms` | URL written to hidden `<input>`, saved as URLField |
| Gallery block images | "Upload Image" button inside block editor | `cms/gallery` | URL inserted into block JSON `images[].src` |

---

## Django Settings

In `settings.py` (reads from `.env`):

```python
CLOUDINARY_CLOUD_NAME = env('CLOUDINARY_CLOUD_NAME', default='')
CLOUDINARY_API_KEY = env('CLOUDINARY_API_KEY', default='')
CLOUDINARY_API_SECRET = env('CLOUDINARY_API_SECRET', default='')
```

Never commit real credentials. Use placeholders in docs and configure values via env/secrets:

```
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

---

## Getting Started (Cloudinary Signup)

This CMS uses Cloudinary to store images. A free Cloudinary account is sufficient for development and small marketing sites.

1) Create a Cloudinary account

- Sign up at `https://cloudinary.com/`
- Choose the free tier to start

2) Create a cloud (project)

- In the Cloudinary console, create/select a cloud
- Copy the following values from the dashboard:
  - `Cloud name`
  - `API Key`
  - `API Secret`

3) Configure environment variables

Add the values to your local `.env` (gitignored) or to your deployment secrets:

```
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

4) Verify in the CMS

- Start the server
- Log into `/cms/` as a CMS Admin
- Edit a Page or Post
- Use the image upload widget

Expected result:

- The preview updates immediately
- The hidden input is populated with an HTTPS URL like:
  - `https://res.cloudinary.com/<cloud>/image/upload/...`

If uploads fail:

- Confirm the env vars are loaded by Django (restart server after editing `.env`)
- Confirm `/cms/upload/` returns 200 JSON for an authenticated CMS admin
- Check Cloudinary dashboard for the uploaded asset

`requirements.txt` entry:

```
cloudinary==1.44.1
```

---

## Complete Source: `marketing/cloudinary_utils.py`

```python
import cloudinary
import cloudinary.uploader
from django.conf import settings


def _configure():
    """Ensure cloudinary is configured from Django settings."""
    cloudinary.config(
        cloud_name=settings.CLOUDINARY_CLOUD_NAME,
        api_key=settings.CLOUDINARY_API_KEY,
        api_secret=settings.CLOUDINARY_API_SECRET,
        secure=True,
    )


def upload_image(file, folder='cms', public_id=None):
    """
    Upload an image file to Cloudinary.

    Args:
        file: A file-like object (e.g. request.FILES['image']) or a file path string.
        folder: Cloudinary folder to upload into.
        public_id: Optional public_id override.

    Returns:
        dict with 'url', 'secure_url', 'public_id', 'width', 'height'.
    """
    _configure()
    options = {
        'folder': folder,
        'resource_type': 'image',
        'overwrite': True,
        'quality': 'auto',
        'fetch_format': 'auto',
    }
    if public_id:
        options['public_id'] = public_id
    result = cloudinary.uploader.upload(file, **options)
    return {
        'url': result['secure_url'],
        'public_id': result['public_id'],
        'width': result.get('width'),
        'height': result.get('height'),
    }


def upload_from_path(path, folder='cms', public_id=None):
    """Upload a local file by path."""
    with open(path, 'rb') as f:
        return upload_image(f, folder=folder, public_id=public_id)
```

### Key patterns

- **Lazy configuration** — `_configure()` is called on every upload, not at module import. This avoids crashes if env vars aren't set (e.g. in tests or CI).
- **`quality: 'auto'` + `fetch_format: 'auto'`** — Cloudinary optimises format (WebP/AVIF) and quality per-client automatically.
- **Returns `secure_url`** — Always HTTPS. The returned dict key is `url` (not `secure_url`) for simplicity downstream.
- **`upload_from_path()`** — Convenience for scripts (used when batch-uploading existing local images to Cloudinary).

---

## Upload Endpoint: `cms_views.py` — `upload_image`

```python
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

URL registration in `marketing/urls.py`:

```python
path('cms/upload/', cms_views.upload_image, name='cms-upload-image'),
```

### Key patterns

- **`@cms_admin_required`** — Only CMS Admins group members can upload.
- **`@require_POST`** — Rejects non-POST requests.
- **Lazy import** — `from .cloudinary_utils import upload_image as do_upload` inside the function to avoid circular imports and to keep cloudinary optional at module level.
- **Folder from request** — The JS can pass `folder=cms/gallery` to organise gallery images separately.
- **Returns JSON** — `{url, public_id, width, height}` on success; `{error}` on failure.

---

## Frontend JS: Upload Widget IIFE

This script lives inline in `marketing/templates/marketing/cms/base.html`. It runs on every CMS admin page.

```javascript
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
```

### Two integration points

#### 1. `.cms-cloudinary-upload` widget (cover_image, og_image fields)

HTML structure expected per widget:

```html
<div class="cms-cloudinary-upload" data-target="id_cover_image">
  <input type="hidden" name="cover_image" id="id_cover_image" value="{{ form.cover_image.value|default_if_none:'' }}">
  <div class="cms-upload-preview">
    {% if post and post.cover_image %}
      <img src="{{ post.cover_image }}" alt="Cover image">
    {% endif %}
  </div>
  <label class="cms-btn cms-btn-ghost cms-upload-btn">
    <input type="file" accept="image/*" style="display:none" data-upload-trigger>
    Upload Image
  </label>
  <span class="cms-upload-status"></span>
</div>
```

Flow:
1. User picks a file via `<input type="file">`
2. `change` event fires, POSTs to `/cms/upload/` with `folder=cms`
3. On success, writes Cloudinary URL into the hidden `<input>`
4. Shows preview thumbnail and "Uploaded" status in teal
5. When form submits, the hidden input's value (a Cloudinary URL) is saved to the URLField

#### 2. `window.cmsUploadImage(file, callback)` (gallery blocks)

Used by `cms-editor.js` in the `renderImageGalleryFields` function.

```javascript
// Gallery image row upload handler (each row has its own Upload button)
fileInput.addEventListener('change', function () {
  var file = fileInput.files[0];
  if (!file) return;
  status.textContent = 'Uploading...';
  window.cmsUploadImage(file, function (url, err) {
    if (err) {
      status.textContent = 'Upload failed';
      return;
    }
    item.src = url;
    sync();
    rerender();
  });
});
```

Flow:
1. User clicks "Upload Image" button inside a gallery block in the editor
2. Hidden file input opens the OS file picker
3. On file selection, calls `window.cmsUploadImage(file, callback)`
4. Global function POSTs to `/cms/upload/` with `folder=cms/gallery`
5. Callback receives URL, pushes it into `block.images[]`
6. Gallery fields re-render showing the new image

---

## Migration Story: ImageField to URLField

### Why

Originally, `cover_image` and `og_image` on `Page` and `Post` were `ImageField` storing files in `MEDIA_ROOT`. Problems:
- Local filesystem doesn't work on multi-server or containerised deployments
- No CDN, no auto-optimisation
- Large file uploads went through Django

### What changed

1. **Model fields changed** from `ImageField` to `URLField(max_length=500, blank=True, default='')` — stores the full Cloudinary `https://res.cloudinary.com/...` URL
2. **Migration** changed column type and set NULLs to empty string:

```sql
-- Applied via RunSQL in migration
ALTER TABLE marketing_page ALTER COLUMN cover_image TYPE varchar(500);
ALTER TABLE marketing_page ALTER COLUMN og_image TYPE varchar(500);
ALTER TABLE marketing_post ALTER COLUMN cover_image TYPE varchar(500);
ALTER TABLE marketing_post ALTER COLUMN og_image TYPE varchar(500);
UPDATE marketing_page SET cover_image = '' WHERE cover_image IS NULL;
UPDATE marketing_page SET og_image = '' WHERE og_image IS NULL;
UPDATE marketing_post SET cover_image = '' WHERE cover_image IS NULL;
UPDATE marketing_post SET og_image = '' WHERE og_image IS NULL;
```

3. **Templates updated** — changed from `{{ page.cover_image.url }}` to `{{ page.cover_image }}` (the field IS the URL now)
4. **SEO updated** — `seo.py` changed from building media URLs to using the field value directly
5. **Forms updated** — replaced file input widgets with the Cloudinary upload widget

### Applying to a new project

1. Start with URLField from day one (skip the migration)
2. Copy `cloudinary_utils.py` and the upload view
3. Add the IIFE script to your CMS base template
4. Add `window.cmsUploadImage()` for any JS-driven upload flows
5. Set Cloudinary env vars

---

## Uploading Existing Images to Cloudinary

When we migrated, we had existing local images that needed uploading. Here's the procedure:

### Using Django shell

```bash
python manage.py shell
# (conda example)
conda run -n <env> python manage.py shell
```

```python
from marketing.cloudinary_utils import upload_from_path
from marketing.models import Post

# Upload a blog cover
result = upload_from_path(
    'media/cms/blog/ai-contract-review-2026-cover.png',
    folder='cms/blog',
    public_id='ai-contract-review-2026-cover'
)
print(result['url'])
# https://res.cloudinary.com/<cloud>/image/upload/v123/cms/blog/ai-contract-review-2026-cover.jpg

# Update the post
post = Post.objects.get(slug='ai-contract-review-2026')
post.cover_image = result['url']
post.save()
```

### Updating the remote database

After uploading images to Cloudinary and updating local DB, you may need to sync to remote.
Do not commit or paste live database credentials in docs. Use placeholders:

```bash
psql "postgres://USER:PASSWORD@HOST:PORT/DBNAME" \
  -c "UPDATE marketing_post SET cover_image = 'https://res.cloudinary.com/<cloud>/image/upload/...jpg' WHERE slug = 'ai-contract-review-2026';"
```

---

## Current Cloudinary URLs

### Blog covers

| Post slug | Cloudinary URL |
|-----------|---------------|
| `ai-contract-review-2026` | `https://res.cloudinary.com/<cloud>/image/upload/v123/cms/blog/ai-contract-review-2026-cover.jpg` |
| `contract-workflow-automations` | `https://res.cloudinary.com/<cloud>/image/upload/v123/cms/blog/contract-workflow-automations-cover.jpg` |
| `compliance-first-culture` | `https://res.cloudinary.com/<cloud>/image/upload/v123/cms/blog/compliance-first-culture-cover.jpg` |
| `product-roadmap-2026` | `https://res.cloudinary.com/<cloud>/image/upload/v123/cms/blog/product-roadmap-2026-cover.jpg` |

### Gallery images

| Name | Cloudinary URL |
|------|---------------|
| `gallery-ai-dashboard` | `https://res.cloudinary.com/<cloud>/image/upload/v123/cms/gallery/gallery-ai-dashboard.jpg` |
| `gallery-collaboration` | `https://res.cloudinary.com/<cloud>/image/upload/v123/cms/gallery/gallery-collaboration.jpg` |
| `gallery-integrations` | `https://res.cloudinary.com/<cloud>/image/upload/v123/cms/gallery/gallery-integrations.jpg` |

---

## Replicating in a New Project — Checklist

1. `pip install cloudinary`
2. Add `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` to settings/env
3. Copy `cloudinary_utils.py` (51 lines)
4. Add the upload view to your CMS views (10 lines)
5. Add the URL: `path('cms/upload/', views.upload_image, name='cms-upload-image')`
6. Add the IIFE script to your CMS base template (lines 72-144 of `cms/base.html`)
7. Use URLField for image columns, not ImageField
8. Build upload widgets with the `.cms-cloudinary-upload` / `data-target` pattern
9. For JS-driven uploads, call `window.cmsUploadImage(file, callback)`
