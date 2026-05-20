# Scent.M — WordPress Implementation Handoff

This document captures everything Claude Code (or any developer) needs to know to implement the static HTML designs in WordPress. Read alongside the design files.

---

## 1. Tech Stack

| Layer | Choice |
|---|---|
| CMS | WordPress core |
| Base theme | Blocksy + Custom Child Theme |
| Page content | Hand-coded Tailwind HTML/PHP templates (no Page Builders, no Elementor) |
| Custom Post Types | CPT UI |
| Fields & blocks | ACF Pro (Flexible Content as primary layout engine) |
| Booking + commerce | **SureCart** (Headless, no WooCommerce) |
| Booking UX | Native slide-out cart, Stripe (incl. Apple/Google Pay, AlipayHK, WeChat Pay HK, FPS offline) |
| Forms | Forminator or Fluent Forms (Raw HTML output for full CSS control) |
| Smooth scroll | Lenis 1.1.14 |
| Animation | GSAP 3.12.5 + ScrollTrigger |
| Fonts | Google Fonts (Baskervville, Cactus Classical Serif, Mea Culpa, Montserrat, Noto Sans TC) |
| Map embed | Google Maps iframe (ACF Map field stores coords) |
| Analytics | Google Analytics (requires cookie consent banner — PDPO/GDPR) |
| Newsletter | Double opt-in (Mailchimp/Klaviyo) → `newsletter-confirm.html` is the post-submit landing |

---

## 2. Custom Post Types (CPT UI)

```
workshops          → Workshop sessions (one CPT per date/event)
journal            → Journal articles
odm_inquiry        → ODM lead capture (form submissions)
marketing_inquiry  → Scent Marketing lead capture
```

Workshops and Journal are public, looped on archive pages. Inquiries are admin-only.

---

## 3. ACF Flexible Content — Workshop Detail page

Right column body in `workshop-detail.html` is rendered from a single ACF Flexible Content field. Admin can **add / remove / reorder** any of these block types per workshop:

| Block name | ACF field group | Demonstrated in HTML |
|---|---|---|
| `block_lead_paragraph` | wysiwyg (single line emphasis) | "一場關於記憶嘅練習。" |
| `block_rich_text` | wysiwyg | Multi-paragraph body |
| `block_single_image` | image (returns array) | 16:9 inline figure |
| `block_pull_quote` | textarea + optional attribution | Italic terracotta with left border |
| `block_image_album` | gallery (1–N images) | 3-column grid |
| `block_video_embed` | text (oEmbed URL) or video upload | Aspect-video with play overlay |
| `block_session_timeline` | repeater (time + label) | 5-row hairline timeline ("Session Flow") |

Each block is independent — admin freely composes any combination.

### Journal Detail — ACF Flexible Content blocks

Body uses the same Flexible Content pattern as Workshop Detail, but with a smaller block subset (no session timeline / what's included / map):

| Block name | ACF field group |
|---|---|
| `block_rich_text` | wysiwyg + optional checkbox "first paragraph drop cap" |
| `block_section_heading` | text (h2 sub-section title) |
| `block_single_image` | image |
| `block_photo_album` | gallery (3-col layout) |
| `block_video_embed` | text (oEmbed URL) or video upload |
| `block_pull_quote` | textarea |

Admin freely adds / removes / reorders. Body width fixed at `max-w-[680px]`. Drop cap only renders on first paragraph of a rich_text block when checkbox enabled (use `.essay-dropcap` class).

---

## 4. ACF Flexible Content — Page-level (for ACF-driven sections)

For sections that should be editable on the homepage and other pages, the same Flexible Content model applies. Suggested block names (already mocked in the static HTML):

- `block_hero_split` (left text + right image)
- `block_hero_centered` (typography-led, no image)
- `block_image_full_bleed`
- `block_image_caption_pair`
- `block_section_quote` (theme-switch background variant)
- `block_three_columns_index` (service ToC)
- `block_left_sticky_right_stack` (ODM-style scroll-story)
- `block_horizontal_carousel` (Workshops-style card row)
- `block_journal_grid`
- `block_inquiry_form` (Forminator render)

---

## 5. Workshop Detail — sidebar metadata fields

The left sidebar is **not** Flexible Content. It's a fixed field group on the Workshop CPT:

```
workshop_date       → date picker
workshop_time_start → time picker
workshop_time_end   → time picker
workshop_place      → text (e.g. "上環 Sheung Wan")
workshop_class_size → number (e.g. 8)
workshop_duration   → text (e.g. "3 小時")
workshop_fee        → number (HKD)
workshop_map        → Google Map (lat/lng — render as iframe in template)
surecart_product_id → text (linked SureCart product slug; see §6)
```

---

## 6. SureCart Integration

- One workshop session = one SureCart Product
- Product inventory = class capacity (e.g. 8). Auto sold-out at 0.
- Each Book/Reserve button uses:
  ```html
  data-sc-product="<%= surecart_product_id %>" data-sc-add-to-cart
  ```
- Slide-out checkout fires automatically — **no page redirect**
- Apple Pay / Google Pay enabled via Stripe Express Checkout
- Offline payment (FPS / 銀行轉帳) routes to admin manual confirmation

### Auto-sync option (recommended for one-stop admin)
Hook `save_post_workshops` → call SureCart REST API to create/update the linked product. Store product ID in `surecart_product_id` ACF field automatically. Admin only ever touches the Workshop CPT, never SureCart admin.

### Card states (already styled in `workshops.html`)
- Default: normal Book button
- `class="is-sold-out"` on article: opacity 0.55, button replaced with disabled "Fully Booked"
- Low stock (≤2 remaining): italic terracotta "Only N spots left" badge above price row

---

## 7. Forminator / Fluent Forms — Inquiry Form Markup

Forms in `contact.html` (and any inquiry section) use **Raw HTML mode**. The existing form markup is the design — plugin should not inject its own CSS classes.

Required form fields (contact.html):
- name (text, required)
- email (email, required)
- topic (select: odm / marketing / workshop / press / other)
- message (textarea, required)
- newsletter (checkbox — triggers double opt-in flow)

Form posts → triggers admin email + (if newsletter checked) → Mailchimp/Klaviyo subscribe → user lands on `newsletter-confirm.html`. Otherwise → `thanks.html`.

URL `contact.html#form?topic=odm` (or `?topic=marketing`) — Forminator should prefill the topic select based on query param.

---

## 8. Cross-page Navigation

### Top nav (all pages)
- Logo (homepage link)
- Burger menu → fullscreen overlay

### Fullscreen menu items (in order)
1. About → `about.html`
2. Boutique ODM → homepage `#odm` (no dedicated page; cross-page anchor on non-homepage)
3. Workshops → `workshops.html`
4. Scent Marketing → homepage `#services` (no dedicated page; cross-page anchor)
5. Journal → `journal.html`
6. Contact → `contact.html`
7. Hebe Lab → `https://hibilab.com` (external, `target="_blank"`)

### Footer (3 columns)
- Services: ODM / Workshops / Scent Marketing
- Brand: About / Journal
- Connect: Contact / Hebe Lab

### Sub-footer
© 2026 Scent.M · Privacy Policy · Terms of Service

---

## 9. CTA Wording — Permanent Rules

| Action | Wording |
|---|---|
| Any inquiry-style CTA (ODM / Marketing / general) | **聯絡我們** (do not use 品牌專屬方案 / 聯絡研發團隊 / 索取商業方案) |
| Workshop booking | **立即預約** |
| Hebe Lab external | **Explore Store** |
| Newsletter | **訂閱 Scent.M Journal** |

Floating WhatsApp button is on every page — **never add redundant "Contact us" buttons** elsewhere; WhatsApp is the primary direct channel.

---

## 10. Design Language — Permanent Rules

- **No small italic right-side helper text** in register rows (e.g. "Late 2026 — Spring 2027", "Notes from the studio"). Approved design language does not include this pattern.
- **No image captions** in italic small Baskervville below figures.
- **No "Last updated · date" italic meta** on legal pages.
- **Headings**: bilingual pattern is `中文 <italic terracotta>English</italic terracotta>.` — this is the homepage signature, **only use sparingly** on other pages (avoid making every heading bilingual two-tone).
- **Width**: maintain consistent `max-w-screen-2xl` gutter throughout one page. Never narrow → wide → narrow.
- **Hero image**: not every page needs a hero image. Workshops / Workshop Detail use pure typography hero.

---

## 11. Theme Switch (Background Color Sections)

Sections marked `class="theme-switch" data-switchto="brown|terracotta"` trigger a JS observer that toggles `<body class="in-brown">` / `in-terracotta`. All `text-current` / `border-current` elements adapt via CSS.

Used on:
- Homepage: Brown intermission quote, Terracotta Hebe Lab CTA

---

## 12. Floating WhatsApp Button

Persistent across all pages. Bottom-right. Target:
```
https://wa.me/85212345678?text=...
```
ACF site-wide setting for phone number + optional prefilled text.

---

## 13. SEO Basics (must implement)

- `sitemap.xml` (auto-generated by Yoast SEO or RankMath)
- `robots.txt` allowing all
- `<title>` per page already set in HTML
- `<meta description>` per page (TODO — add ACF field on each CPT)
- Open Graph image (ACF site-wide default + per-post override)
- Cookie Consent Banner (Cookiebot / Iubenda / native) — required for GA + PDPO/GDPR

---

## 14. Pages Index

| File | Purpose |
|---|---|
| `Scent.M Responsive.html` | Homepage |
| `about.html` | Brand story / philosophy |
| `workshops.html` | Workshop listings (CPT archive) |
| `workshop-detail.html` | Single workshop (CPT single) |
| `journal.html` | Journal listings (CPT archive) |
| `journal-detail.html` | Single journal article (CPT single) |
| `contact.html` | Contact + general inquiry form |
| `thanks.html` | Form submission success page |
| `newsletter-confirm.html` | Double opt-in landing |
| `404.html` | Not found |
| `privacy.html` | Privacy Policy |
| `terms.html` | Terms of Service |

---

_Last updated by design phase: May 19, 2026_
