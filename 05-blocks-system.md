# 05 -- The Blocks System

> The heart of the CMS. Content is stored as a JSON object containing a typed blocks array in
> `blocks_json`. On save, the model's `save()` override renders the blocks to
> HTML via `blocks.py` and stores the result in `blocks_html`. The CMS editor
> UI is built by `cms-editor.js` which renders interactive forms for each
> block type, including embedded EditorJS instances for rich text.

---

## Architecture Overview

```
                     CMS Editor (browser)
                     ┌────────────────────────────────────────┐
                     │  cms-editor.js                         │
                     │  ┌──────────────────────────────────┐  │
                     │  │ Block Builder UI                 │  │
                     │  │  ┌─────────┐ ┌─────────┐ ┌────┐ │  │
                     │  │  │Rich Text│ │   CTA   │ │FAQ │ │  │
                     │  │  │(EditorJS│ │(inputs) │ │(...│ │  │
                     │  │  │instance)│ │         │ │    │ │  │
                     │  │  └─────────┘ └─────────┘ └────┘ │  │
                     │  └──────────────────────────────────┘  │
                     │              │ on form submit          │
                     │              ▼                         │
                     │  <input type="hidden"                  │
                     │   name="blocks_json"                   │
                     │   data-blocks-input>                   │
                     └────────────────────────────────────────┘
                                    │ POST
                                    ▼
                     Django Model.save()
                     ┌────────────────────────────────────────┐
                     │  blocks_json (JSONField)               │
                     │         │                              │
                     │         ▼                              │
                     │  blocks.render_blocks(payload)         │
                     │         │                              │
                     │         ▼                              │
                     │  blocks_html (TextField)               │
                     │         │                              │
                     │  (stored pre-rendered, served to       │
                     │   templates via {{ blocks_html|safe }})│
                     └────────────────────────────────────────┘
```

## The 12 Block Types

| Type | Editor UI | JSON Shape | Server CSS Class |
|---|---|---|---|
| `rich_text` | Embedded EditorJS | `{type, content: {blocks: [...]}}` | `.cms-prose` |
| `code` | Raw HTML/JS textarea | `{type, html}` | (no default class; renders as-is) |
| `cta` | Title + Body + Button fields | `{type, title, body, button_label, button_url}` | `.cms-cta` |
| `callout` | Title + Body fields | `{type, title, body}` | `.cms-callout` |
| `feature_grid` | List of title+body pairs | `{type, items: [{title, body}]}` | `.cms-feature-grid` |
| `faq` | List of Q+A pairs | `{type, items: [{question, answer}]}` | `.cms-faq` |
| `quote` | Quote + Author fields | `{type, quote, author}` | `.cms-quote` |
| `comparison_table` | Headers + Rows matrix | `{type, headers: [], rows: [[]]}` | `.cms-comparison` |
| `pricing_table` | Plans with features | `{type, plans: [{title, price, features: []}]}` | `.cms-pricing` |
| `logo_cloud` | List of src+alt pairs | `{type, logos: [{src, alt}]}` | `.cms-logo-cloud` |
| `image_gallery` | Title + Layout + Images | `{type, title, layout, images: [{src, alt, caption}]}` | `.cms-gallery-wrap` |

Notes:

- `blocks_json` is canonically `{ "blocks": [...] }`. The builder/server tolerate arrays and `null`/`"null"` for backward compatibility.
- `code` is a trusted embed block. It is wrapped in markers and preserved through sanitization.

---

## Data Flow

### 1. Creating Content (Editor → Server)

1. User clicks "Add block" in the CMS editor
2. `cms-editor.js` pushes a default block object into `state.blocks[]`
3. The block builder renders the appropriate form fields
4. For `rich_text` blocks, an EditorJS instance is created inside the block
5. On every field change, `sync()` writes `JSON.stringify(state)` to the
   hidden `<input data-blocks-input>`
6. On form submit, the submit handler first saves all EditorJS instances
   (async), then syncs, then submits the form
7. Django receives `blocks_json` as a JSON string in POST data
8. The model's `save()` override calls `render_blocks(self.blocks_json)`
9. The rendered HTML is stored in `self.blocks_html`

### 2. Rendering Content (Server → Browser)

1. View loads the page/post from the database
2. Template outputs `{{ page.blocks_html|safe }}`
3. The pre-rendered HTML includes semantic CSS classes (`.cms-prose`,
   `.cms-cta`, `.cms-feature-grid`, etc.)
4. CSS in `marketing.css` styles the rendered blocks

---

## Complete Source Files

### `marketing/blocks.py` (142 lines) -- Server-Side Block Renderer

```python
from html import escape

from .renderers import render_editorjs


def render_blocks(payload):
    if not payload:
        return ''
    blocks = payload.get('blocks', []) if isinstance(payload, dict) else payload
    html = ''
    for block in blocks:
        block_type = block.get('type')
        if block_type == 'rich_text':
            html += _rich_text(block)
        elif block_type == 'code':
            html += _code(block)
        elif block_type == 'callout':
            html += _callout(block)
        elif block_type == 'cta':
            html += _cta(block)
        elif block_type == 'feature_grid':
            html += _feature_grid(block)
        elif block_type == 'comparison_table':
            html += _comparison_table(block)
        elif block_type == 'table':
            html += _table(block)
        elif block_type == 'faq':
            html += _faq(block)
        elif block_type == 'quote':
            html += _quote(block)
        elif block_type == 'logo_cloud':
            html += _logo_cloud(block)
        elif block_type == 'pricing_table':
            html += _pricing_table(block)
        elif block_type == 'image_gallery':
            html += _image_gallery(block)
    return html


def _code(block):
    """Trusted raw HTML/JS block.

    This is intended for CMS admins to embed snippets (e.g. clickwrap embed).
    It is wrapped in a marker that the sanitizer preserves.
    """
    raw = block.get('html', '')
    if not raw:
        return ''
    return f"<!--TRUSTED_CODE_START-->{raw}<!--TRUSTED_CODE_END-->"


def _rich_text(block):
    content = block.get('content', {})
    if not content:
        return ''
    rendered = render_editorjs(content)
    return _wrap('div', rendered, 'cms-prose')


def _callout(block):
    title = escape(block.get('title', ''))
    body = escape(block.get('body', ''))
    return _wrap('div', _wrap('h3', title, 'cms-callout-title') + _wrap('p', body, 'cms-callout-body'), 'cms-callout')


def _cta(block):
    title = escape(block.get('title', ''))
    body = escape(block.get('body', ''))
    label = escape(block.get('button_label', ''))
    url = escape(block.get('button_url', ''))
    button = f"<a class=\"cms-cta-button\" href=\"{url}\">{label}</a>" if label and url else ''
    return _wrap('div', _wrap('h3', title, 'cms-cta-title') + _wrap('p', body, 'cms-cta-body') + button, 'cms-cta')


def _feature_grid(block):
    items = block.get('items', [])
    cards = ''.join(_wrap('div', _wrap('h4', escape(item.get('title', ''))) + _wrap('p', escape(item.get('body', ''))), 'cms-feature-card') for item in items)
    return _wrap('div', cards, 'cms-feature-grid')


def _comparison_table(block):
    headers = block.get('headers', [])
    rows = block.get('rows', [])
    head_cells = ''.join(_wrap('th', escape(header)) for header in headers)
    head = _wrap('thead', _wrap('tr', head_cells))
    body_rows = ''.join(_wrap('tr', ''.join(_wrap('td', escape(cell)) for cell in row)) for row in rows)
    body = _wrap('tbody', body_rows)
    return _wrap('table', head + body, 'cms-comparison')


def _table(block):
    headers = block.get('headers', [])
    rows = block.get('rows', [])
    head_cells = ''.join(_wrap('th', escape(header)) for header in headers)
    head = _wrap('thead', _wrap('tr', head_cells)) if headers else ''
    body_rows = ''.join(_wrap('tr', ''.join(_wrap('td', escape(cell)) for cell in row)) for row in rows)
    body = _wrap('tbody', body_rows)
    return _wrap('table', head + body, 'cms-table')


def _faq(block):
    items = block.get('items', [])
    entries = ''.join(_wrap('div', _wrap('h4', escape(item.get('question', ''))) + _wrap('p', escape(item.get('answer', ''))), 'cms-faq-item') for item in items)
    return _wrap('div', entries, 'cms-faq')


def _quote(block):
    quote = escape(block.get('quote', ''))
    author = escape(block.get('author', ''))
    inner = _wrap('blockquote', quote)
    if author:
        inner += _wrap('p', author, 'cms-quote-author')
    return _wrap('div', inner, 'cms-quote')


def _logo_cloud(block):
    logos = block.get('logos', [])
    items = ''.join(_wrap('div', f"<img src=\"{escape(logo.get('src', ''))}\" alt=\"{escape(logo.get('alt', ''))}\">", 'cms-logo-item') for logo in logos)
    return _wrap('div', items, 'cms-logo-cloud')


def _pricing_table(block):
    plans = block.get('plans', [])
    cards = ''
    for plan in plans:
        features = ''.join(_wrap('li', escape(feature)) for feature in plan.get('features', []))
        card = _wrap('h4', escape(plan.get('title', ''))) + _wrap('p', escape(plan.get('price', '')), 'cms-price') + _wrap('ul', features)
        cards += _wrap('div', card, 'cms-pricing-card')
    return _wrap('div', cards, 'cms-pricing')


def _image_gallery(block):
    title = escape(block.get('title', ''))
    layout = block.get('layout', 'grid')
    images = block.get('images', [])

    header = _wrap('h3', title, 'cms-gallery-title') if title else ''

    items = ''
    for img in images:
        src = escape(img.get('src', ''))
        alt = escape(img.get('alt', ''))
        caption = escape(img.get('caption', ''))
        inner = f'<img src="{src}" alt="{alt}" loading="lazy">'
        if caption:
            inner += _wrap('span', caption, 'cms-gallery-caption')
        items += _wrap('figure', inner, 'cms-gallery-item')

    layout_class = f'cms-gallery cms-gallery-{layout}'
    return _wrap('div', header + _wrap('div', items, layout_class), 'cms-gallery-wrap')


def _wrap(tag, content, class_name=None):
    attrs = f' class="{class_name}"' if class_name else ''
    return f'<{tag}{attrs}>{content}</{tag}>'
```

**Key points:**
- `render_blocks()` accepts either `{blocks: [...]}` or a raw list
- All text content is escaped with `html.escape()` to prevent XSS
- Rich text blocks delegate to `render_editorjs()` from `renderers.py`
- The `_wrap()` helper builds HTML tags with optional class attributes
- Image gallery supports a `layout` field (`grid`, `masonry`, `carousel`)
  which sets `cms-gallery-grid`, `cms-gallery-masonry`, etc.

---

### `marketing/renderers.py` (293 lines) -- EditorJS HTML Renderer + Sanitizer

```python
import re
from html import escape

from bs4 import BeautifulSoup, Comment


ALLOWED_TAGS = {
    'p', 'strong', 'em', 'b', 'i', 'u', 's', 'a', 'ul', 'ol', 'li', 'br', 'hr',
    'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'blockquote', 'code', 'pre', 'figure',
    'figcaption', 'img', 'table', 'thead', 'tbody', 'tr', 'th', 'td', 'span', 'div'
}

ALLOWED_ATTRS = {
    'a': {'href', 'title', 'target', 'rel'},
    'img': {'src', 'alt', 'title', 'width', 'height', 'loading'},
    'div': {'class'},
    'span': {'class'},
    'table': {'class'},
    'thead': {'class'},
    'tbody': {'class'},
    'tr': {'class'},
    'th': {'class'},
    'td': {'class'},
    'figure': {'class'},
    'figcaption': {'class'},
    'p': {'class'},
    'blockquote': {'class'},
    'h1': {'class'}, 'h2': {'class'}, 'h3': {'class'},
    'h4': {'class'}, 'h5': {'class'}, 'h6': {'class'},
}


def render_lexical_json(payload):
    if not payload:
        return ''
    root = payload.get('root', {})
    children = root.get('children', [])
    return ''.join(_render_node(child) for child in children)


def render_editorjs(payload):
    if not payload:
        return ''
    blocks = payload.get('blocks', []) if isinstance(payload, dict) else []
    html = ''
    for block in blocks:
        block_type = block.get('type')
        data = block.get('data', {})
        if block_type == 'paragraph':
            html += _wrap('p', _sanitize_inline_html(data.get('text', '')))
        elif block_type == 'header':
            level = str(data.get('level', 2))
            tag = f"h{level}"
            html += _wrap(tag, _sanitize_inline_html(data.get('text', '')))
        elif block_type == 'list':
            html += _render_editorjs_list(data)
        elif block_type == 'quote':
            html += _wrap('blockquote', _sanitize_inline_html(data.get('text', '')))
        elif block_type == 'table':
            html += _render_editorjs_table(data)
        elif block_type == 'code':
            html += _wrap('pre', _wrap('code', escape(data.get('code', ''))))
        elif block_type == 'delimiter':
            html += '<hr />'
        elif block_type == 'warning':
            title = _sanitize_inline_html(data.get('title', ''))
            message = _sanitize_inline_html(data.get('message', ''))
            html += _wrap('div', _wrap('h4', title) + _wrap('p', message), class_name='cms-callout')
        elif block_type == 'cta':
            html += _render_cta(data)
        elif block_type == 'callout':
            html += _render_callout(data)
        elif block_type == 'comparison':
            html += _render_comparison_table(data)
    return html


# ... (Lexical node renderers for forward compatibility)
# ... (Callout, CTA, Feature Grid, Comparison Table, FAQ, Quote, Logo Cloud, Pricing Table)


def _wrap(tag, content, class_name=None):
    if tag not in ALLOWED_TAGS:
        return ''
    attrs = ''
    if class_name:
        attrs = f' class="{class_name}"'
    return f'<{tag}{attrs}>{content}</{tag}>'


def sanitize_html(html):
    if not html:
        return ''

    # Preserve trusted code blocks (used by CMS admin "code" widget).
    # Everything outside these markers is still sanitized.
    code_blocks = []

    def _stash(m):
        code_blocks.append(m.group(0))
        return f"__TRUSTED_CODE_BLOCK_{len(code_blocks) - 1}__"

    html = re.sub(
        r'<!--TRUSTED_CODE_START-->.*?<!--TRUSTED_CODE_END-->',
        _stash,
        html,
        flags=re.S,
    )

    html = re.sub(r'<\s*script[^>]*>.*?<\s*/\s*script\s*>', '', html, flags=re.I | re.S)
    html = re.sub(r'on\w+\s*=\s*"[^"]*"', '', html, flags=re.I)

    for i, block in enumerate(code_blocks):
        html = html.replace(f"__TRUSTED_CODE_BLOCK_{i}__", block)

    return html


def _sanitize_inline_html(html):
    if not html:
        return ''
    soup = BeautifulSoup(html, 'html.parser')
    for comment in soup.find_all(string=lambda text: isinstance(text, Comment)):
        comment.extract()
    for tag in soup.find_all(True):
        if tag.name not in ALLOWED_TAGS:
            tag.unwrap()
            continue
        allowed = ALLOWED_ATTRS.get(tag.name, set())
        tag.attrs = {key: value for key, value in tag.attrs.items() if key in allowed}
    return str(soup)


def _render_editorjs_list(data):
    style = data.get('style', 'unordered')
    tag = 'ol' if style == 'ordered' else 'ul'
    items = ''.join(_wrap('li', _sanitize_inline_html(item)) for item in data.get('items', []))
    return _wrap(tag, items)


def _render_editorjs_table(data):
    rows = data.get('content', [])
    body_rows = ''.join(_wrap('tr', ''.join(_wrap('td', _sanitize_inline_html(cell)) for cell in row)) for row in rows)
    return _wrap('table', _wrap('tbody', body_rows), class_name='cms-table')
```

**Key points:**
- **Two renderer systems**: `render_editorjs()` (used) and `render_lexical_json()` (forward compat)
- **BeautifulSoup sanitizer**: `_sanitize_inline_html()` strips disallowed tags/attrs
- **Allowlist approach**: Only tags in `ALLOWED_TAGS` pass through. Only attributes in `ALLOWED_ATTRS` are kept
- **Script removal**: `sanitize_html()` strips `<script>` and `on*=` handlers via regex
- **EditorJS block types handled**: paragraph, header, list, quote, table, code, delimiter, warning, cta, callout, comparison

---

### `static/js/cms-editor.js` (919 lines) -- The Block Builder

This is the complete client-side block builder. Key sections:

#### Helpers (lines 1-51)
- `safeJsonParse()` -- Parse JSON with fallback
- `el()` -- DOM element factory (`el('div', {className: 'foo'}, children)`)
- `ICONS` -- SVG icon strings for drag, collapse, remove, plus
- `BLOCK_LABELS` -- Human labels for the block types (including `code`)

#### Block Defaults (lines 68-93)
```javascript
function blockDefaults(type) {
  switch (type) {
    case 'rich_text':
      return { type: type, content: { blocks: [] } };
    case 'code':
      return { type: type, html: '' };
    case 'cta':
      return { type: type, title: '', body: '', button_label: '', button_url: '' };
    case 'callout':
      return { type: type, title: '', body: '' };
    case 'feature_grid':
      return { type: type, items: [{ title: '', body: '' }] };
    case 'faq':
      return { type: type, items: [{ question: '', answer: '' }] };
    case 'quote':
      return { type: type, quote: '', author: '' };
    case 'comparison_table':
      return { type: type, headers: ['Feature', 'Plan A', 'Plan B'], rows: [['', '', '']] };
    case 'pricing_table':
      return { type: type, plans: [{ title: '', price: '', features: [''] }] };
    case 'logo_cloud':
      return { type: type, logos: [{ src: '', alt: '' }] };
    case 'image_gallery':
      return { type: type, title: '', layout: 'grid', images: [{ src: '', alt: '', caption: '' }] };
    default:
      return { type: type };
  }
}
```

#### Field Builders (lines 95-127)
- `fieldInput()` -- Creates `<div class="cms-field"><label><input>` with onChange
- `fieldTextarea()` -- Same but with `<textarea>`

#### Block Type Renderers (lines 129-592)
Each block type has a renderer function that returns a DOM node:
- `renderCtaFields()` -- Title, body, button label, button URL
- `renderCalloutFields()` -- Title, body
- `renderQuoteFields()` -- Quote, author
- `renderListBlock()` -- Generic list renderer (used by feature_grid, faq, logo_cloud)
- `renderFeatureGridFields()` -- Delegates to `renderListBlock`
- `renderFaqFields()` -- Delegates to `renderListBlock`
- `renderLogoCloudFields()` -- Delegates to `renderListBlock`
- `renderImageGalleryFields()` -- Custom: title, layout select, image list with upload buttons
- `renderComparisonTableFields()` -- Headers row + data rows with add/remove col/row
- `renderPricingTableFields()` -- Plans with nested features lists

#### Rich Text / EditorJS Integration (lines 594-656)
```javascript
var editorInstances = {};  // Track per block index

function renderRichTextFields(block, sync, _rerender, blockIndex) {
  // Creates a holder div
  // Defers EditorJS init with setTimeout(50ms)
  // Configures EditorJS with: Header, List, Quote, Table, Code, Delimiter, Warning
  // On every EditorJS change, saves content back to block.content
  // Stores instance in editorInstances[blockIndex]
}
```

#### Block Builder / Main Loop (lines 676-896)
`initBlocksBuilder()` is the entry point:
1. Finds `[data-blocks-builder]` and `[data-blocks-input]`
2. Parses state from the hidden input
3. `render()` function rebuilds the entire block list UI:
   - Each block gets a header (drag handle, type badge, collapse/remove buttons)
   - Each block body gets the appropriate field renderer
   - Collapsed blocks only show the header
4. Drag-and-drop: native HTML5 drag events with visual indicators
5. "Add block" bars at top AND bottom -- top bar inserts at position 0, bottom bar appends. Created by a shared `createAddBar(insertIndex)` helper function
6. **Form submit hook**: intercepts submit, saves all EditorJS instances async,
   syncs state to hidden input, then submits

#### Form Submit Hook (lines 861-896)
```javascript
form.addEventListener('submit', function handleBlocksSubmit(e) {
  // Collect all active EditorJS instances
  // Save them all in parallel with Promise.all
  // Sync state to hidden input
  // Remove the listener and submit the form
});
```

This is critical -- without this hook, EditorJS content would be lost because
EditorJS saves are async and the form would submit before they complete.

---

## Known UX Gaps / Follow-ups

- The builder supports "Add block at top" and "Add block at bottom" plus drag-and-drop.
  It does not currently provide per-block "Add above / Add below" buttons.
  Expected workflow: add, then drag to position.

> **Important - Always include drag-and-drop bars at both TOP and BOTTOM**: Without drag-and-drop bars at both the top and bottom of the block list, users cannot easily insert new blocks in the middle of existing content. They would have to add blocks at the top/bottom and then drag them into position, which is unintuitive. Ensure both "Add block" bars are always visible.

---

## Embedding Clickwrap Contracts (No Iframes)

If your CMS also generates clickwrap contracts, prefer a JS embed that injects content into the host DOM (no iframe).

Recommended snippet to paste into a CMS `code` block:

```html
<div id="yourapp-clickwrap-<PUBLIC_ID>"></div>
<script async
  src="https://YOUR_APP_HOST/embed/contracts/<PUBLIC_ID>/embed.js"
  data-yourapp-target="yourapp-clickwrap-<PUBLIC_ID>"></script>
```

Notes:

- Avoid iframes for clickwrap display: the goal is to render as native content on the host page.
- Keep embed CSS minimal and inheriting from the host site so it feels native.

---

### `static/js/cms-json.js` (39 lines) -- JSON Field Helpers

```javascript
function attachJsonHelpers() {
  const fields = document.querySelectorAll('[data-json-field]');
  fields.forEach((field) => {
    const formatBtn = field.parentElement.querySelector('[data-json-format]');
    const sampleBtn = field.parentElement.querySelector('[data-json-sample]');
    if (formatBtn) {
      formatBtn.addEventListener('click', () => {
        try {
          const parsed = JSON.parse(field.value || '{}');
          field.value = JSON.stringify(parsed, null, 2);
        } catch (err) {
          alert('Invalid JSON');
        }
      });
    }
    if (sampleBtn) {
      sampleBtn.addEventListener('click', () => {
        field.value = sampleBtn.dataset.sample;
      });
    }
  });

  const blocksFields = document.querySelectorAll('[data-blocks-field]');
  blocksFields.forEach((field) => {
    if (!field.value) {
      field.value = '{"blocks": []}';
    }
  });

  const bodyFields = document.querySelectorAll('[data-body-field]');
  bodyFields.forEach((field) => {
    if (!field.value) {
      field.value = '{"root":{"children":[]}}';
    }
  });
}

document.addEventListener('DOMContentLoaded', attachJsonHelpers);
```

Used by the navigation editor for the `[data-json-field]` textareas.
Also initializes empty JSON fields for blocks and body fields.

---

## How to Add a New Block Type

1. **Define the JSON schema** -- Add a case in `blockDefaults()` in `cms-editor.js`
2. **Build the editor UI** -- Create a `renderXxxFields(block, sync, rerender)` function
3. **Register in dispatcher** -- Add a case in `renderBlockFields()`
4. **Add label** -- Add to `BLOCK_LABELS` object
5. **Server renderer** -- Add a `_xxx(block)` function in `blocks.py`
6. **Register in `render_blocks()`** -- Add an `elif block_type == 'xxx'` branch
7. **CSS** -- Add styles for `.cms-xxx` in `marketing.css`

---

## How to Reuse This System

1. Copy `blocks.py`, `renderers.py`, `cms-editor.js`, `cms-json.js`
2. Install BeautifulSoup: `pip install beautifulsoup4`
3. Add `blocks_json = JSONField(blank=True, null=True)` and
   `blocks_html = TextField(blank=True)` to your model
4. Override `save()` to call `render_blocks(self.blocks_json)`
5. Load EditorJS via CDN in your CMS base template
6. Add `data-blocks-input` and `data-blocks-builder` to your form
7. Style the rendered blocks with your own CSS
