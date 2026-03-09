# 08 — Deployment & Management Commands

> Three management commands handle seeding content, setting up CMS access, and generating
> AI images. Plus: database sync procedures for pushing local data to the remote server.

---

## Management Commands Overview

| Command | Purpose | Lines |
|---------|---------|-------|
| `seed_cms_content` | Create sample pages, blog posts, navigation, and footer | 441 |
| `seed_cms_admin` | Create the "CMS Admins" group and add an initial admin user | 18 |
| `generate_images` | Generate blog cover + gallery images via fal.ai | 230 |

Run any command with:

```bash
python manage.py <command_name>
# (conda example)
conda run -n <env> python manage.py <command_name>
```

---

## Complete Source: `seed_cms_content.py`

The content below is intentionally example content. Replace names, copy, and links with your own brand and site structure.

`marketing/management/commands/seed_cms_content.py` (441 lines)

```python
from django.core.management.base import BaseCommand
from django.utils import timezone

from marketing.models import Page, Post, NavMenu, Footer, Category, Tag


class Command(BaseCommand):
    help = 'Seed sample CMS content: pages, blog posts, navigation, and footer'

    def handle(self, *args, **options):
        self._seed_categories_tags()
        self._seed_pages()
        self._seed_posts()
        self._seed_navigation()
        self._seed_footer()
        self.stdout.write(self.style.SUCCESS('CMS content seeded successfully.'))

    # -- Categories & Tags --------------------------------------------------

    # IMPORTANT: Use generic category/tag names relevant to your business.
    # These examples are specific to the original project - replace with yours.
    # Good generic choices: "Product Updates", "Industry News", "How-To", "Case Studies"

    def _seed_categories_tags(self):
        for name, slug in [
            ('Business Tools', 'business-tools'),
            ('Project Management', 'project-management'),
            ('Compliance', 'compliance'),
            ('Product Updates', 'product-updates'),
        ]:
            Category.objects.get_or_create(slug=slug, defaults={'name': name})

        for name, slug in [
            ('AI', 'ai'),
            ('Automation', 'automation'),
            ('DMS', 'dms'),
            ('Enterprise', 'enterprise'),
            ('Security', 'security'),
            ('Workflows', 'workflows'),
        ]:
            Tag.objects.get_or_create(slug=slug, defaults={'name': name})
        self.stdout.write('  Categories & tags created.')

    # -- Pages --------------------------------------------------------------

    def _seed_pages(self):
        pages = [
            {
                'title': 'About Example Company',
                'slug': 'about',
                'status': 'published',
                'blocks_json': {
                    'blocks': [
                        {
                            'type': 'rich_text',
                            'content': _editorjs_body([
                                {'type': 'header', 'data': {'text': 'Our Mission', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Example Company was founded to remove manual work from teams. We build tools that let people focus on strategy instead of paperwork.'}},
                                {'type': 'paragraph', 'data': {'text': 'Our platform combines automation, workflows, and collaboration to transform how teams operate.'}},
                                {'type': 'header', 'data': {'text': 'Our Story', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'We started with firsthand frustration with outdated tools. We set out to build the platform we always wished we had.'}},
                            ]),
                        },
                        {
                            'type': 'callout',
                            'title': 'Trusted by 500+ Teams',
                            'body': 'From small teams to large enterprises, we power workflows at every scale.',
                        },
                        {
                            'type': 'feature_grid',
                            'items': [
                                {'title': 'AI Document Review', 'body': 'Instantly flag issues, missing sections, and non-standard language across thousands of documents.'},
                                {'title': 'Automated Workflows', 'body': 'Build approval chains, renewal reminders, and compliance checks that run on autopilot.'},
                                {'title': 'Real-Time Collaboration', 'body': 'Edit and review documents live with internal teams and external stakeholders.'},
                                {'title': 'Enterprise Security', 'body': 'SOC 2 Type II certified, end-to-end encryption, and role-based access controls.'},
                            ],
                        },
                        {
                            'type': 'cta',
                            'title': 'Ready to modernize your operations?',
                            'body': 'See the product in action with a personalized demo from our team.',
                            'button_label': 'Book a Demo',
                            'button_url': '/contact/',
                        },
                    ]
                },
                'seo_title': 'About Example Company',
                'seo_description': 'Learn how Example Company helps teams automate workflows and move faster.',
            },
            {
                'title': 'Contact Us',
                'slug': 'contact',
                'status': 'published',
                'blocks_json': {
                    'blocks': [
                        {
                            'type': 'rich_text',
                            'content': _editorjs_body([
                                {'type': 'header', 'data': {'text': 'Get in Touch', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Have questions? Want to see a demo? Our team typically responds within one business day.'}},
                                {'type': 'paragraph', 'data': {'text': 'Email us at hello@example.com or use the form below. For enterprise inquiries, reach out to sales@example.com.'}},
                            ]),
                        },
                        {
                            'type': 'callout',
                            'title': 'Enterprise & Government',
                            'body': 'For organizations with 100+ users, dedicated onboarding, or specific compliance requirements, contact our enterprise team directly at enterprise@example.com.',
                        },
                    ]
                },
                'seo_title': 'Contact Us — Get a Demo',
                'seo_description': 'Contact the team for demos, support, and enterprise inquiries.',
            },
            {
                'title': 'Pricing',
                'slug': 'pricing',
                'status': 'published',
                'blocks_json': {
                    'blocks': [
                        {
                            'type': 'rich_text',
                            'content': _editorjs_body([
                                {'type': 'header', 'data': {'text': 'Simple, transparent pricing', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'No hidden fees. No long-term contracts. Start with a free trial and scale as you grow.'}},
                            ]),
                        },
                        {
                            'type': 'pricing_table',
                            'plans': [
                                {
                                    'title': 'Starter',
                                    'price': '$49/mo',
                                    'features': ['Up to 50 projects', '1 workspace', 'Basic AI review', 'Email support'],
                                },
                                {
                                    'title': 'Professional',
                                    'price': '$149/mo',
                                    'features': ['Unlimited projects', '5 workspaces', 'Advanced AI + redlining', 'Priority support', 'Custom workflows'],
                                },
                                {
                                    'title': 'Enterprise',
                                    'price': 'Custom',
                                    'features': ['Unlimited everything', 'SSO & SCIM', 'Dedicated CSM', 'SLA guarantees', 'On-prem deployment option', 'Custom integrations'],
                                },
                            ],
                        },
                        {
                            'type': 'faq',
                            'items': [
                                {'question': 'Can I switch plans anytime?', 'answer': 'Yes. Upgrade or downgrade at any time. Changes take effect on your next billing cycle.'},
                                {'question': 'Is there a free trial?', 'answer': 'Absolutely. Every plan includes a 14-day free trial with full feature access. No credit card required.'},
                                {'question': 'What payment methods do you accept?', 'answer': 'We accept all major credit cards and ACH. Enterprise customers can pay by invoice with NET-30 terms.'},
                                {'question': 'Do you offer discounts for non-profits?', 'answer': 'Yes, we offer 30% off for registered non-profit organizations. Contact us at hello@example.com for details.'},
                            ],
                        },
                        {
                            'type': 'cta',
                            'title': 'Not sure which plan is right for you?',
                            'body': 'Our team can help you find the perfect fit for your workflow.',
                            'button_label': 'Talk to Sales',
                            'button_url': '/contact/',
                        },
                    ]
                },
                'seo_title': 'Pricing',
                'seo_description': 'Explore plans and pricing. Start free, scale as you grow.',
            },
        ]

        for data in pages:
            page, created = Page.objects.update_or_create(
                slug=data['slug'],
                defaults={
                    'title': data['title'],
                    'status': data['status'],
                    'body_json': None,
                    'blocks_json': data['blocks_json'],
                    'seo_title': data.get('seo_title', ''),
                    'seo_description': data.get('seo_description', ''),
                },
            )
            action = 'created' if created else 'updated'
            self.stdout.write(f'  Page "{page.title}" {action}.')

    # -- Blog Posts ---------------------------------------------------------

    def _seed_posts(self):
        now = timezone.now()
        cat_legaltech = Category.objects.filter(slug='business-tools').first()
        cat_clm = Category.objects.filter(slug='project-management').first()
        cat_compliance = Category.objects.filter(slug='compliance').first()

        tag_ai = Tag.objects.filter(slug='ai').first()
        tag_automation = Tag.objects.filter(slug='automation').first()
        tag_clm = Tag.objects.filter(slug='dms').first()
        tag_enterprise = Tag.objects.filter(slug='enterprise').first()
        tag_security = Tag.objects.filter(slug='security').first()
        tag_workflows = Tag.objects.filter(slug='workflows').first()

        posts_data = [
            {
                'title': 'How AI Is Transforming Document Review in 2026',
                'slug': 'ai-document-review-2026',
                'status': 'published',
                'author_name': 'Sarah Chen',
                'excerpt': 'Large language models have finally crossed the accuracy threshold that makes fully automated document review viable for enterprise teams.',
                'publish_at': now - timezone.timedelta(days=5),
                'categories': [cat_legaltech, cat_clm],
                'tags': [tag_ai, tag_clm],
                'blocks_json': {
                    'blocks': [
                        {
                            'type': 'rich_text',
                            'content': _editorjs_body([
                                {'type': 'paragraph', 'data': {'text': 'The industry has talked about AI-powered document review for years, but 2026 marks a genuine inflection point. Accuracy rates for section extraction and risk scoring have crossed 95%, making these tools reliable enough for production use without human double-checking on every item.'}},
                                {'type': 'header', 'data': {'text': 'What Changed?', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Three converging trends made this possible: domain-specific fine-tuning of large language models on millions of real documents, retrieval-augmented generation (RAG) that grounds AI outputs in your organization\'s own playbook, and better calibration techniques that make models say "I don\'t know" instead of hallucinating.'}},
                                {'type': 'header', 'data': {'text': 'Practical Impact', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Teams using AI review report 60-80% reductions in first-pass review time. For high-volume operations processing hundreds of NDAs, vendor agreements, and SOWs each month, this translates to millions in saved outsourcing spend.'}},
                                {'type': 'header', 'data': {'text': 'What to Look For', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'When evaluating AI document review tools, focus on: accuracy metrics on your specific document types, the ability to train on your playbook and preferred positions, integration with your existing workflow and document management systems, and transparent audit trails that show exactly why the AI flagged something.'}},
                            ]),
                        },
                        {
                            'type': 'comparison_table',
                            'headers': ['Capability', 'Traditional Review', 'AI-Assisted Review'],
                            'rows': [
                                ['NDA first-pass review', '45 minutes', '3 minutes'],
                                ['Risk identification', 'Manual checklist', 'Automated + scored'],
                                ['Playbook compliance', 'Memory-dependent', 'Systematically enforced'],
                                ['Volume capacity', '5-10 documents/day', '100+ documents/day'],
                            ],
                        },
                        {
                            'type': 'cta',
                            'title': 'See AI document review in action',
                            'body': 'Upload a sample document and get an instant risk analysis.',
                            'button_label': 'Try It Free',
                            'button_url': '/contact/',
                        },
                    ]
                },
            },
            {
                'title': '5 Workflow Automations Every Team Should Deploy',
                'slug': 'workflow-automations',
                'status': 'published',
                'author_name': 'Marcus Wright',
                'excerpt': 'Stop reinventing the wheel. These five workflow automations will save your team hours every week with minimal setup effort.',
                'publish_at': now - timezone.timedelta(days=12),
                'categories': [cat_clm],
                'tags': [tag_automation, tag_workflows, tag_clm],
                'blocks_json': {
                    'blocks': [
                        {
                            'type': 'rich_text',
                            'content': _editorjs_body([
                                {'type': 'paragraph', 'data': {'text': 'Most teams know they should automate more, but choosing where to start is paralyzing. After working with hundreds of organizations, we\'ve identified the five automations that deliver the highest ROI with the least implementation effort.'}},
                                {'type': 'header', 'data': {'text': '1. Renewal Reminders', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Auto-renew clauses are revenue leakers. Set up automated alerts 90, 60, and 30 days before any agreement renewal date. Link these to your calendar and Slack so the right stakeholder gets notified at the right time.'}},
                                {'type': 'header', 'data': {'text': '2. Approval Routing', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Route documents for approval based on value, type, and risk level. Projects under $50K go straight to the business owner. Over $50K triggers manager review. Over $500K adds finance and executive sign-off. No more email chains asking "who needs to approve this?"'}},
                                {'type': 'header', 'data': {'text': '3. Template Selection', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'When a team member needs a document, a short intake form should automatically select the right template, pre-fill known data from your CRM, and route it to the correct workflow. This alone eliminates 70% of "can you send me the right template?" requests.'}},
                                {'type': 'header', 'data': {'text': '4. Obligation Tracking', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Extract key obligations from executed agreements and create automated tasks with deadlines. Payment milestones, delivery dates, reporting requirements — these should all flow into a central tracker with automated reminders.'}},
                                {'type': 'header', 'data': {'text': '5. Expiration & Compliance Alerts', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Certificates of insurance, regulatory filings, license agreements — all have expiration dates. Automate monitoring and alert the responsible party well before deadlines hit.'}},
                            ]),
                        },
                        {
                            'type': 'feature_grid',
                            'items': [
                                {'title': 'Renewal Reminders', 'body': 'Never miss a renewal or auto-renew date again.'},
                                {'title': 'Approval Routing', 'body': 'Route documents to the right approvers automatically.'},
                                {'title': 'Template Selection', 'body': 'Auto-select and pre-fill the right document template.'},
                                {'title': 'Obligation Tracking', 'body': 'Extract and track obligations from agreements.'},
                            ],
                        },
                        {
                            'type': 'callout',
                            'title': 'Start Small',
                            'body': 'You don\'t need to automate everything at once. Pick the one automation that addresses your biggest pain point and build from there.',
                        },
                    ]
                },
            },
            {
                'title': 'Building a Compliance-First Culture in Your Organization',
                'slug': 'compliance-first-culture',
                'status': 'published',
                'author_name': 'Priya Nair',
                'excerpt': 'Compliance isn\'t just about checking boxes. Here\'s how forward-thinking teams are embedding compliance into their daily workflows.',
                'publish_at': now - timezone.timedelta(days=20),
                'categories': [cat_compliance],
                'tags': [tag_enterprise, tag_security, tag_workflows],
                'blocks_json': {
                    'blocks': [
                        {
                            'type': 'rich_text',
                            'content': _editorjs_body([
                                {'type': 'paragraph', 'data': {'text': 'Regulatory landscapes are getting more complex every year. GDPR, CCPA, the EU AI Act, evolving SEC disclosure rules — teams can\'t afford to treat compliance as an annual audit exercise. It needs to be woven into everyday operations.'}},
                                {'type': 'header', 'data': {'text': 'From Reactive to Proactive', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'The traditional model is reactive: something goes wrong, the team investigates, policies get updated. A compliance-first culture flips this by building guardrails into processes before issues arise.'}},
                                {'type': 'header', 'data': {'text': 'Key Principles', 'level': 2}},
                                {'type': 'list', 'data': {'style': 'unordered', 'items': ['Make the compliant path the easiest path', 'Automate compliance checks at the point of action', 'Provide real-time feedback, not after-the-fact audits', 'Invest in training that\'s contextual, not annual slide decks']}},
                                {'type': 'header', 'data': {'text': 'Technology\'s Role', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Modern workflow platforms can enforce compliance automatically: required sections that can\'t be removed, prohibited terms that trigger warnings, mandatory approval workflows for regulated document types, and automated regulatory reporting.'}},
                            ]),
                        },
                        {
                            'type': 'quote',
                            'quote': 'The best compliance program is one where people don\'t even realize they\'re being compliant — because the systems make it effortless.',
                             'author': 'Example Author',
                        },
                        {
                            'type': 'faq',
                            'items': [
                                {'question': 'How do I get buy-in from leadership for compliance tooling?', 'answer': 'Frame it in terms of risk reduction and cost avoidance. A single compliance violation can cost more than years of tooling investment. Quantify your current exposure.'},
                                {'question': 'Should compliance be centralized or distributed?', 'answer': 'The answer is both. Set centralized policies and standards, but embed compliance champions and automated checks into each business unit\'s workflow.'},
                                {'question': 'How often should compliance policies be reviewed?', 'answer': 'At minimum quarterly, but the real answer is continuously. Regulatory changes should trigger immediate policy reviews in affected areas.'},
                            ],
                        },
                    ]
                },
            },
            {
                'title': 'Product Roadmap: What\'s Coming Next',
                'slug': 'product-roadmap-2026',
                'status': 'published',
                'author_name': 'Team',
                'excerpt': 'A look at the features and improvements shipping in the next two quarters, including enhanced AI capabilities and new integrations.',
                'publish_at': now - timezone.timedelta(days=2),
                'categories': [Category.objects.filter(slug='product-updates').first()],
                'tags': [tag_ai, tag_automation, tag_enterprise],
                'blocks_json': {
                    'blocks': [
                        {
                            'type': 'rich_text',
                            'content': _editorjs_body([
                                {'type': 'paragraph', 'data': {'text': 'We\'ve been heads-down building, and we\'re excited to share what\'s coming next. These updates are driven directly by feedback from our customers and our vision for where the product is headed.'}},
                                {'type': 'header', 'data': {'text': 'Q1 2026: Intelligence Layer', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'Our new intelligence layer brings AI-powered insights to every part of the platform. Document risk scores, section recommendations based on your history, and predictive analytics for project cycle times.'}},
                                {'type': 'header', 'data': {'text': 'Q2 2026: Integration Ecosystem', 'level': 2}},
                                {'type': 'paragraph', 'data': {'text': 'We\'re launching new integrations with popular tools, plus an API gateway that makes it trivial to connect the product to any system in your stack.'}},
                            ]),
                        },
                        {
                            'type': 'feature_grid',
                            'items': [
                                {'title': 'AI Risk Scoring', 'body': 'Automatic risk scores for every document based on content analysis and historical data.'},
                                {'title': 'Template Library', 'body': 'A searchable library of approved templates with AI-powered recommendations.'},
                                 {'title': 'CRM Integration', 'body': 'Native bi-directional sync with your CRM.'},
                                 {'title': 'API Gateway', 'body': 'Connect the product to any system with a developer-friendly API.'},
                            ],
                        },
                        {
                            'type': 'cta',
                            'title': 'Want early access?',
                            'body': 'Join our beta program to test new features before they launch.',
                            'button_label': 'Join the Beta',
                            'button_url': '/contact/',
                        },
                    ]
                },
            },
        ]

        for data in posts_data:
            categories = data.pop('categories', [])
            tags = data.pop('tags', [])
            post, created = Post.objects.update_or_create(
                slug=data['slug'],
                defaults={
                    'title': data['title'],
                    'status': data['status'],
                    'author_name': data['author_name'],
                    'excerpt': data['excerpt'],
                    'publish_at': data.get('publish_at'),
                    'body_json': None,
                    'blocks_json': data['blocks_json'],
                },
            )
            post.categories.set([c for c in categories if c])
            post.tags.set([t for t in tags if t])
            action = 'created' if created else 'updated'
            self.stdout.write(f'  Post "{post.title}" {action}.')

    # -- Navigation ---------------------------------------------------------

    def _seed_navigation(self):
        nav, _ = NavMenu.objects.update_or_create(
            name='Primary',
            defaults={
                'items_json': [
                    {'label': 'About', 'url': '/about/'},
                    {'label': 'Pricing', 'url': '/pricing/'},
                    {'label': 'Blog', 'url': '/blog/'},
                    {'label': 'Contact', 'url': '/contact/'},
                ],
            },
        )
        self.stdout.write(f'  Navigation menu seeded.')

    # -- Footer -------------------------------------------------------------

    def _seed_footer(self):
        footer, _ = Footer.objects.update_or_create(
            label='Default',
            defaults={
                'columns_json': [
                    {
                        'title': 'Links',
                        'links': [
                            {'label': 'Privacy Notice', 'url': '/privacy/'},
                            {'label': 'Terms of Use', 'url': '/terms/'},
                            {'label': 'Sitemap', 'url': '/sitemap.xml'},
                            {'label': 'Contact Us', 'url': '/contact/'},
                        ],
                    },
                ],
                'cta_title': 'Ready to streamline your workflows?',
                'cta_body': 'Start your free 14-day trial. No credit card required.',
                'cta_button_label': 'Get Started Free',
                'cta_button_url': '/contact/',
                'legal_text': (
                    'Acme Corp is a technology platform. '
                    'This is example disclaimer text for your application.'
                ),
            },
        )
        self.stdout.write(f'  Footer seeded.')


# -- Helpers ----------------------------------------------------------------

def _editorjs_body(blocks):
    """Wrap a list of EditorJS block dicts in the standard EditorJS envelope."""
    return {
        'time': 1700000000000,
        'version': '2.30.2',
        'blocks': blocks,
    }
```

### Key patterns

- **`update_or_create`** — Idempotent. Run it repeatedly without duplicating content.
- **`_editorjs_body()` helper** — Wraps EditorJS block arrays in the standard `{time, version, blocks}` envelope that EditorJS expects.
- **Categories & tags created first** — Posts reference them via M2M `.set()` after creation.
- **`blocks_json` not `body_json`** — The old `body_json` field is set to `None`. All content lives in the blocks system.
- **Navigation uses `items_json`** — A simple list of `{label, url}` dicts.
- **Footer uses `columns_json`** — Each column has `{title, links: [{label, url}]}`.

---

## Complete Source: `seed_cms_admin.py`

`marketing/management/commands/seed_cms_admin.py` (18 lines)

```python
from django.core.management.base import BaseCommand
from django.contrib.auth import get_user_model
from django.contrib.auth.models import Group


class Command(BaseCommand):
    help = 'Seed CMS Admins group and initial admin membership'

    def handle(self, *args, **options):
        group, _ = Group.objects.get_or_create(name='CMS Admins')
        User = get_user_model()
        user = User.objects.filter(email=os.getenv('CMS_ADMIN_EMAIL', '')).first()
        if not user:
            self.stdout.write(self.style.WARNING('CMS admin user not found'))
            return
        user.groups.add(group)
        self.stdout.write(self.style.SUCCESS('Seeded CMS Admins group'))
```

### Notes

- Creates the `CMS Admins` group if it doesn't exist
- Adds the configured initial admin user to the group
- Fails gracefully if the user doesn't exist
- The `@cms_admin_required` decorator in `permissions.py` checks for this group membership

---

## Complete Source: `generate_images.py`

`marketing/management/commands/generate_images.py` (230 lines)

```python
"""
Generate images for blog posts and gallery blocks using fal.ai Nano Banana Pro.

Usage:
    python manage.py generate_images
    python manage.py generate_images --blog-only
    python manage.py generate_images --gallery-only
"""
import os
import time
from pathlib import Path
from urllib.request import urlretrieve

import fal_client
from django.conf import settings
from django.core.management.base import BaseCommand

from marketing.models import Post, Page


# Blog post cover image prompts -- tailored to each post topic
BLOG_PROMPTS = {
    'ai-document-review-2026': (
        'Dark moody professional photograph of a futuristic workspace, '
        'holographic documents floating above a sleek obsidian desk, '
        'amber and violet light accents, AI neural network patterns subtly '
        'overlaid, ultra-wide 16:9 cinematic composition, dark background, '
        'no text, no people, editorial photography style'
    ),
    'workflow-automations': (
        'Dark moody professional photograph of an automated workflow system, '
        'glowing amber flowchart connections between document icons on a dark '
        'glass surface, interconnected nodes and arrows with warm golden light '
        'trails, futuristic business technology concept, dark obsidian background, '
        'no text, no people, editorial photography style'
    ),
    'compliance-first-culture': (
        'Dark moody professional photograph of a shield-shaped holographic '
        'compliance dashboard, regulatory checkmarks glowing in teal and amber '
        'on a dark surface, data streams flowing through protective barriers, '
        'dark obsidian office environment, no text, no people, '
        'editorial photography style'
    ),
    'product-roadmap-2026': (
        'Dark moody professional photograph of a futuristic product roadmap '
        'visualization, timeline with glowing amber milestone markers on a '
        'dark glass surface, subtle violet accent highlights for AI features, '
        'holographic UI elements floating, dark obsidian background, '
        'no text, no people, editorial photography style'
    ),
}

# Gallery images for the About page -- showcase the product
GALLERY_PROMPTS = [
    (
        'gallery-ai-dashboard',
        'Dark moody screenshot-style image of a futuristic AI-powered '
        'dashboard interface, analytics cards with amber accent charts '
        'and graphs on obsidian dark background, clean modern UI design, '
        'professional business software, no text, editorial product photography'
    ),
    (
        'gallery-collaboration',
        'Dark moody professional photograph of a collaborative document editing '
        'interface floating holographically, multiple cursor indicators in amber '
        'and violet, redline annotations glowing softly, dark obsidian workspace, '
        'futuristic collaboration tool concept, no text, no people'
    ),
    (
        'gallery-integrations',
        'Dark moody professional photograph of interconnected glowing nodes '
        'representing software integrations, API connections visualized as '
        'amber light streams between floating platform icons, dark obsidian '
        'background with subtle grid pattern, futuristic enterprise tech concept, '
        'no text, no people'
    ),
]


class Command(BaseCommand):
    help = 'Generate images using fal.ai Nano Banana Pro for blog covers and gallery blocks'

    def add_arguments(self, parser):
        parser.add_argument('--blog-only', action='store_true',
                            help='Only generate blog cover images')
        parser.add_argument('--gallery-only', action='store_true',
                            help='Only generate gallery images')

    def handle(self, *args, **options):
        api_key = os.getenv('FAL_KEY') or os.getenv('FAL_API_KEY')
        if not api_key:
            self.stderr.write(self.style.ERROR(
                'FAL_KEY or FAL_API_KEY not set. Add it to .env'))
            return

        # fal_client reads FAL_KEY from env
        os.environ['FAL_KEY'] = api_key

        do_blog = not options.get('gallery_only', False)
        do_gallery = not options.get('blog_only', False)

        if do_blog:
            self._generate_blog_covers()
        if do_gallery:
            self._generate_gallery_images()

        self.stdout.write(self.style.SUCCESS('Image generation complete.'))

    def _generate_image(self, prompt, output_path):
        """Generate a single image via Nano Banana Pro and save to disk."""
        self.stdout.write(f'  Generating: {output_path.name}...')

        result = fal_client.subscribe('fal-ai/nano-banana-pro', arguments={
            'prompt': prompt,
            'aspect_ratio': '16:9',
            'num_images': 1,
            'output_format': 'png',
        })

        image_url = result['images'][0]['url']
        output_path.parent.mkdir(parents=True, exist_ok=True)
        urlretrieve(image_url, str(output_path))

        self.stdout.write(self.style.SUCCESS(
            f'    Saved: {output_path} ({output_path.stat().st_size // 1024} KB)'))
        # Be polite to the API
        time.sleep(1)
        return output_path

    def _generate_blog_covers(self):
        self.stdout.write(self.style.MIGRATE_HEADING('Generating blog cover images...'))
        media_root = Path(settings.MEDIA_ROOT)
        blog_dir = media_root / 'cms' / 'blog'

        for slug, prompt in BLOG_PROMPTS.items():
            filename = f'{slug}-cover.png'
            output_path = blog_dir / filename

            # Skip if already exists
            if output_path.exists():
                self.stdout.write(f'  Skipping {filename} (already exists)')
                post = Post.objects.filter(slug=slug).first()
                if post and not post.cover_image:
                    post.cover_image = f'cms/blog/{filename}'
                    post.save()
                    self.stdout.write(f'    Linked to post: {post.title}')
                continue

            self._generate_image(prompt, output_path)

            # Link to the blog post
            post = Post.objects.filter(slug=slug).first()
            if post:
                post.cover_image = f'cms/blog/{filename}'
                post.save()
                self.stdout.write(f'    Linked to post: {post.title}')
            else:
                self.stdout.write(self.style.WARNING(
                    f'    Post with slug "{slug}" not found. Run seed_cms_content first.'))

    def _generate_gallery_images(self):
        self.stdout.write(self.style.MIGRATE_HEADING('Generating gallery images...'))
        media_root = Path(settings.MEDIA_ROOT)
        gallery_dir = media_root / 'cms' / 'gallery'

        gallery_paths = []
        for name, prompt in GALLERY_PROMPTS:
            filename = f'{name}.png'
            output_path = gallery_dir / filename

            if output_path.exists():
                self.stdout.write(f'  Skipping {filename} (already exists)')
                gallery_paths.append(f'/media/cms/gallery/{filename}')
                continue

            self._generate_image(prompt, output_path)
            gallery_paths.append(f'/media/cms/gallery/{filename}')

        # Add an image_gallery block to the About page
        about = Page.objects.filter(slug='about').first()
        if not about:
            self.stdout.write(self.style.WARNING(
                '  About page not found. Run seed_cms_content first.'))
            return

        blocks_data = about.blocks_json
        if not blocks_data:
            blocks_data = {'blocks': []}
        if isinstance(blocks_data, list):
            blocks_data = {'blocks': blocks_data}

        # Check if gallery block already exists
        has_gallery = any(
            b.get('type') == 'image_gallery'
            for b in blocks_data.get('blocks', [])
        )
        if has_gallery:
            self.stdout.write('  About page already has an image_gallery block.')
            return

        # Insert gallery block after the rich_text intro (position 1)
        gallery_block = {
            'type': 'image_gallery',
            'title': 'The Platform',
            'layout': 'grid',
            'images': [
                {
                    'src': gallery_paths[0],
                    'alt': 'AI-powered analytics dashboard',
                    'caption': 'AI-powered analytics dashboard',
                },
                {
                    'src': gallery_paths[1],
                    'alt': 'Real-time collaborative document editing',
                    'caption': 'Real-time collaborative editing',
                },
                {
                    'src': gallery_paths[2],
                    'alt': 'Enterprise integration ecosystem',
                    'caption': 'Native integrations with your tech stack',
                },
            ],
        }

        blocks_data['blocks'].insert(1, gallery_block)
        about.blocks_json = blocks_data
        about.save()
        self.stdout.write(self.style.SUCCESS(
            '  Added image_gallery block to About page.'))
```

### Key patterns

- **fal.ai Nano Banana Pro** — Fast, cheap image generation model. Uses `fal_client.subscribe()` for synchronous generation.
- **Prompts match the design system** — Tailor prompts to match your brand look and feel.
- **16:9 aspect ratio** — Blog covers and gallery images are wide format.
- **Idempotent** — Skips images that already exist on disk.
- **Auto-links to models** — Blog covers are linked to Post objects. Gallery images are inserted into the About page's `blocks_json`.
- **`--blog-only` / `--gallery-only`** flags for selective generation.

### Note on current state

After the Cloudinary migration, `generate_images.py` would need updating if reused — it still references local `MEDIA_ROOT` paths and `post.cover_image = f'cms/blog/{filename}'` (which was an ImageField path). In the current system, you'd instead:
1. Generate the image to a temp file
2. Upload to Cloudinary via `cloudinary_utils.upload_from_path()`
3. Save the Cloudinary URL to `post.cover_image`

---

## Database Sync Procedures

### Dumping marketing tables from local to remote

The marketing tables that need syncing:

```
marketing_category
marketing_tag
marketing_page
marketing_post
marketing_post_categories    (M2M join table)
marketing_post_tags          (M2M join table)
marketing_navmenu
marketing_footer
```

#### Method: pg_dump + psql

```bash
# Dump marketing tables from local
podman exec app-postgres pg_dump -U postgres -d myapp \
  --data-only --inserts \
  -t marketing_category \
  -t marketing_tag \
  -t marketing_page \
  -t marketing_post \
  -t marketing_post_categories \
  -t marketing_post_tags \
  -t marketing_navmenu \
  -t marketing_footer \
  > /tmp/marketing_data.sql

# Load into remote (after clearing existing data)
podman exec app-postgres psql \
  "postgres://USER:PASSWORD@HOST:PORT/DBNAME" \
  -c "TRUNCATE marketing_post_categories, marketing_post_tags, marketing_post, marketing_page, marketing_category, marketing_tag, marketing_navmenu, marketing_footer CASCADE;"

podman exec -i app-postgres psql \
  "postgres://USER:PASSWORD@HOST:PORT/DBNAME" \
  < /tmp/marketing_data.sql
```

#### Method: Individual UPDATE for specific fields

```bash
# Update a single post's cover image on remote
podman exec app-postgres psql \
  "postgres://USER:PASSWORD@HOST:PORT/DBNAME" \
  -c "UPDATE marketing_post SET cover_image = 'https://res.cloudinary.com/<cloud>/image/upload/v123/cms/blog/ai-document-review-2026-cover.jpg' WHERE slug = 'ai-document-review-2026';"
```

---

## Setting Up CMS Admins on a New Environment

### Option 1: Management command

```bash
python manage.py seed_cms_admin
```

### Option 2: Manual via Django shell

```bash
python manage.py shell
```

```python
from django.contrib.auth.models import Group
from django.contrib.auth import get_user_model

User = get_user_model()
group, _ = Group.objects.get_or_create(name='CMS Admins')

user = User.objects.get(email='<admin-email>')
user.groups.add(group)
```

### Option 3: Direct SQL on remote

```bash
# Note: remote uses users_user table, not accounts_user
podman exec app-postgres psql \
  "postgres://USER:PASSWORD@HOST:PORT/DBNAME" \
  -c "
    INSERT INTO auth_group (name) VALUES ('CMS Admins') ON CONFLICT DO NOTHING;
    INSERT INTO users_user_groups (user_id, group_id)
    SELECT u.id, g.id
    FROM users_user u, auth_group g
    WHERE u.email = '<admin-email>' AND g.name = 'CMS Admins'
    ON CONFLICT DO NOTHING;
  "
```

---

## Full Deployment Sequence (New Environment)

```bash
# 1. Run migrations
python manage.py migrate

# 1b. Collect static assets (manifest storage in production)
python manage.py collectstatic --noinput

# 2. Create CMS Admins group + add user
python manage.py seed_cms_admin

# 3. Seed sample content
python manage.py seed_cms_content

# 4. (Optional) Generate AI images and upload to Cloudinary
# Set FAL_KEY in .env first
python manage.py generate_images

# 5. Upload generated images to Cloudinary
python manage.py shell
# >>> from marketing.cloudinary_utils import upload_from_path
# >>> result = upload_from_path('media/cms/blog/ai-document-review-2026-cover.png', folder='cms/blog')
# >>> from marketing.models import Post
# >>> post = Post.objects.get(slug='ai-document-review-2026')
# >>> post.cover_image = result['url']
# >>> post.save()
# (repeat for each image)

# 6. Verify
python manage.py runserver
# Visit http://localhost:8000/ — marketing pages
# Visit http://localhost:8000/cms/ — CMS admin (must be logged in as CMS Admin)

# Also verify:
# - /robots.txt disallows /cms/ and points to /sitemap.xml
# - /sitemap.xml is served by the chosen sitemap implementation (avoid duplicates)
```
