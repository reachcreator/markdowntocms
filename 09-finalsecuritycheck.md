# 09 — Final Security Check (CMS + Marketing)

Use this checklist before deploying changes to the marketing site + CMS.

This is intentionally “boring Django hygiene” that prevents most incidents.

---

## 1) Access Control & Permissions

### CMS routes

- All `/cms/` routes require authentication.
- All `/cms/` routes require an explicit authorization check (e.g. “CMS Admins” group).
- No CMS view should be reachable by a logged-in but unauthorized user.

Verify:

- `@login_required` is enforced (directly or via a wrapper like `@cms_admin_required`).
- Authorization checks happen server-side, not only in templates.

### Admin / staff-only routes

- `/admin/` is protected by Django admin auth (staff).
- Any other “admin-only” endpoints should use explicit decorators.

### Public routes

- Public pages/posts are accessible without login.
- Draft/unpublished content returns 404 publicly.
- CMS content is not leaked via public JSON endpoints.

---

## 2) Method Safety (GET vs POST)

- Any endpoint that mutates state must be POST-only.
- State-changing endpoints should use `@require_POST` (or equivalent).

Examples:

- Delete endpoints: POST only
- Publish/unpublish toggles: POST only
- Upload endpoints: POST only

---

## 3) CSRF Protection

- All HTML forms include `{% csrf_token %}`.
- All fetch/AJAX POST requests send `X-CSRFToken`.
- No production CMS endpoints should be `@csrf_exempt` unless there is a strong reason.

---

## 4) File Upload Safety (Cloudinary)

Minimum:

- Upload endpoint gated by CMS admin permission.
- Only accepts POST.
- Does not leak stack traces / raw exception messages to the client.

Recommended hardening:

- Enforce max file size (server-side)
- Validate content type / extension allowlist (e.g. images only)
- Restrict `folder` parameter to an allowlist (don’t trust client input)
- Add rate limiting if exposed beyond trusted internal users

---

## 5) HTML Safety / XSS

### Rich text sanitization

- Rich text renderer must sanitize user-provided HTML (allowlist tags/attrs).
- Avoid relying on regex-only sanitizers for arbitrary HTML.

### Trusted embeds (`code` block)

- `code` blocks intentionally embed raw HTML/JS.
- Ensure only trusted CMS admins can create/edit content.
- Clearly document that this bypasses strict sanitization inside markers.

---

## 6) Data Validation & Form Integrity

- Avoid nested `<form>` tags (invalid HTML breaks submissions and can drop fields).
- Hidden JSON fields (`blocks_json`) must be normalized (`null`/`"null"`/empty should not crash the editor).
- Ensure there is a “Save Changes” action in edit mode that doesn’t accidentally force publish.

---

## 7) Indexing & Crawlers

- `robots.txt` disallows `/cms/` and `/admin/`.
- `sitemap.xml` includes homepage + blog index + all published pages/posts.
- Don’t ship duplicate/conflicting sitemaps. Pick one implementation.

---

## 8) Static Assets

- If production uses hashed manifest static files, deploy pipeline must run `collectstatic`.
- Verify CMS still loads `cms-editor.js` and EditorJS tool scripts after deployment.

---

## 9) Quick Manual Smoke

Unauthenticated:

- Visit `/cms/` → redirect to login (or 403)
- Visit a published page/post → 200
- Visit an unpublished draft page/post → 404

Authenticated CMS admin:

- Create page → save draft → reload → fields persist
- Edit page → save changes → reload → fields persist
- Publish/unpublish → public visibility matches expectation
- Upload image → URL saved
