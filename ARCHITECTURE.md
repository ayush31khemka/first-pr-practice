# Real Estate Portal — Architecture Plan & Execution Steps

**Owner:** Ayush Khemka
**Scope:** (1) A main portal for browsing client listings, (2) per-project lead-generation
landing pages on subdomains (e.g. `birla-arika.yourdomain.com`), (3) a lead pipeline and
admin back-office that serves both.

---

## 1. Product Vision & Personas

| Persona | What they do on the site | What success looks like |
|---|---|---|
| **You (owner/admin)** | Add/edit listings & projects, upload images/brochures, receive and work leads | Every enquiry lands in one inbox/CRM with source attribution |
| **End user (buyer/tenant)** | Browses listings, filters by budget/BHK/locality, views details & photos, sends enquiry | Finds relevant inventory in <3 clicks, contacts you with one tap |
| **Broker (B2B)** | Scans your inventory for a client requirement, checks specifics (floor, facing, price), connects to co-broke | Can shortlist and reach you on WhatsApp/call instantly |
| **Project buyer (via ads)** | Lands on a project subdomain from Google/Meta ads, wants price list, floor plan, brochure | Downloads collateral after leaving name + phone (gated lead magnet) |

Two distinct funnels, one system:

1. **Portal funnel** — organic/direct traffic → browse → listing detail → "Connect" → lead.
2. **Landing-page funnel** — paid traffic → project page → gated brochure/floor-plan
   download or "Get Best Price" form → lead. This is the primary lead engine; it ships first
   (see §10 Execution).

---

## 2. Learnings from 99acres / MagicBricks / RealBetter

Patterns worth copying (and what to skip at your scale):

### Search & listing structure (99acres, MagicBricks)
- **Faceted search results page**: left rail (desktop) / bottom-sheet (mobile) filters —
  purpose (buy/rent), property type (apartment/builder floor/plot/commercial), BHK, budget
  range slider, locality multi-select, area (sq ft), furnishing, possession status
  (ready / under construction / new launch), posted-by, age of property.
- **Listing card anatomy**: cover photo + photo count badge, price (₹ with per-sq-ft),
  title (`3 BHK Builder Floor in Sushant Lok 1`), 3–4 key specs (area, floor, facing,
  possession), locality, "Contact" + "WhatsApp" CTAs, `RERA` badge where applicable,
  posted date / verified tag.
- **Listing detail page (LDP)**: photo gallery first, price block with EMI hint, spec
  table (carpet/built-up area, floor X of Y, facing, parking, age, furnishing), amenities
  grid, location map + nearby landmarks, description, similar listings carousel, and a
  **sticky contact card** (persistent on scroll — the single highest-converting element
  on these portals).
- **SEO locality pages**: `/property-in-gurgaon/sushant-lok-1` style landing pages that
  aggregate matching listings — this is how 99acres/MagicBricks own long-tail search.
- Full-detail listings on these portals carry 40+ fields including lat/long, RERA ID,
  amenities, floor plans, virtual tour links — model your schema generously up front
  (§4) even if you only fill 15 fields on day one.

### Broker-side patterns (RealBetter)
- RealBetter is a **verified B2B inventory exchange**: brokers browse each other's
  curated stock and connect to co-broke; trust comes from KYC/RERA verification badges.
- Takeaway for you: brokers don't want a pretty consumer UX, they want **density and
  speed** — a compact table/list view, exact inventory attributes (tower, floor, facing,
  demand price), and instant WhatsApp connect. Add a "Broker view" toggle or a
  `/inventory` dense list rather than building a separate app.
- Optional later: a lightweight "share as PDF/image card" per listing so brokers can
  forward your inventory to their clients (this is how inventory actually circulates
  on WhatsApp in NCR).

### What to deliberately skip
- User accounts/saved searches, chat systems, paid listing tiers, map-draw search —
  portal-scale features that don't pay off for a single-broker inventory site.
- You are the only supply source, so **no seller onboarding flows** — the admin panel is
  the only write path.

---

## 3. System Architecture

```
                    ┌────────────────────────────────────────────┐
                    │                DNS (wildcard)              │
                    │  yourdomain.com          → portal          │
                    │  *.yourdomain.com        → landing pages   │
                    │  admin.yourdomain.com    → admin panel     │
                    └───────────────┬────────────────────────────┘
                                    │
                     ┌──────────────▼───────────────┐
                     │   Next.js app (single repo)   │
                     │  Middleware reads Host header │
                     │  ├─ apex  → /(portal)/*       │
                     │  ├─ sub   → /(projects)/[slug]│
                     │  └─ admin → /(admin)/*        │
                     └──────┬───────────────┬───────┘
                            │               │
              ┌─────────────▼──┐      ┌─────▼──────────────┐
              │ Postgres +     │      │ Object storage/CDN │
              │ Auth + REST    │      │ images, brochures, │
              │ (Supabase)     │      │ floor plans (PDF)  │
              └─────────┬──────┘      └────────────────────┘
                        │
          ┌─────────────▼─────────────────────────────┐
          │ Lead pipeline (DB insert triggers):        │
          │  • WhatsApp/email notification to you      │
          │  • Google Sheet / CRM append (n8n or       │
          │    Supabase webhook)                       │
          │  • UTM + source stamped on every lead      │
          └────────────────────────────────────────────┘
```

### 3.1 Recommended stack

| Layer | Choice | Why |
|---|---|---|
| Framework | **Next.js 15 (App Router)** on **Vercel** | One codebase serves portal + all subdomains via middleware host-based rewrites; wildcard domains are first-class on Vercel; SSG/ISR gives fast, SEO-friendly pages |
| Database & auth | **Supabase** (Postgres + Row Level Security + Storage) | Free tier covers you for a long time; instant REST API; auth for the admin panel only |
| Media | Supabase Storage or **Cloudinary** for images (auto WebP/AVIF, resizing); PDFs (brochures/floor plans) in Supabase Storage behind a **signed URL issued only after lead capture** | Gating the download is the whole point of the landing pages |
| Admin panel | Next.js `/admin` routes with Supabase auth, or **off-the-shelf: Directus / Payload CMS** if you'd rather not build CRUD | You are the only writer; keep it simple |
| Lead automation | Supabase **Database Webhook → n8n** (you already use n8n) → WhatsApp Cloud API / email + Google Sheets/CRM | Zero-code changes when you swap CRM later |
| Analytics | GA4 sitewide + **Meta Pixel & Google Ads tag on landing pages only**, with server-side Conversions API events on lead submit | Landing pages exist to feed ad optimization; conversion events must fire reliably |
| Maps | Google Maps embed (free tier) per listing/project | Location is a top-3 decision factor |

**Alternative (lower-code):** WordPress + Houzez/RealHomes theme for the portal and a
landing-page builder for subdomains. Faster week-1, but you'll fight it on: gated
downloads, lead webhooks, dense broker views, and multi-subdomain management. The
Next.js/Supabase route costs more up front and near-zero after, and everything above is
on free/hobby tiers until traffic justifies paying. **Recommendation: Next.js path.**

### 3.2 Subdomain strategy for project landing pages

- **One app, many hosts.** DNS: `A/CNAME` wildcard `*.yourdomain.com` → Vercel.
  Next.js middleware maps `birla-arika.yourdomain.com` → renders `projects/birla-arika`
  from the DB. Adding "Oberoi 360 North" = one row in the `projects` table + assets;
  **no deploy needed**.
- Each project row stores its own: theme accent color, hero media, SEO meta, form
  config, pixel IDs (if you run separate ad accounts per project), and RERA number.
- Also expose the same page at `yourdomain.com/projects/birla-arika` (canonical to the
  subdomain) so projects benefit from and contribute to the main domain's SEO.
- Landing pages are **statically generated with ISR** (revalidate on admin edit) — they
  must score 90+ on mobile PageSpeed because ~80% of ad traffic is mobile.

---

## 4. Data Model (Postgres)

```sql
-- Core inventory
listings (
  id, slug, status,             -- draft | active | sold | rented | withdrawn
  intent,                       -- sale | rent | both
  property_type,                -- apartment | builder_floor | villa | plot | office | shop
  title, description,
  price, price_negotiable, price_per_sqft, maintenance_monthly,
  bhk, bathrooms, balconies,
  area_carpet, area_builtup, area_super, area_unit,
  floor_no, total_floors, facing, furnishing, parking_covered, parking_open,
  possession_status,            -- ready | under_construction | new_launch
  age_years, rera_id,
  locality_id → localities, society_name, address_text, lat, lng,
  is_featured, is_broker_visible_only, owner_notes,   -- private field, never rendered
  created_at, updated_at
)

localities  ( id, name, city, slug, seo_description )        -- powers SEO pages
amenities   ( id, name, icon )  +  listing_amenities (m2m)
media       ( id, parent_type,  -- listing | project
              parent_id, kind,  -- photo | floor_plan | brochure | video | payment_plan
              url, sort_order, alt_text, is_gated )

-- Project landing pages (subdomains)
projects (
  id, slug,                     -- slug == subdomain, e.g. 'birla-arika'
  name, developer, status,      -- new_launch | under_construction | ready
  city, micro_market,           -- 'Gurgaon', 'Sector 31'
  price_min, price_max, configurations,   -- jsonb: [{type:'3BHK', size:2100, price:...}]
  possession_date, rera_no, land_area, towers, units,
  highlights jsonb, amenities jsonb, location_advantages jsonb,
  hero_media_id, theme jsonb,   -- accent color, logo
  seo jsonb, pixels jsonb,      -- {meta_pixel_id, gads_id, gtm_id}
  is_live, created_at, updated_at
)

-- Leads (single table for BOTH funnels)
leads (
  id, name, phone, email,
  source_type,                  -- portal_listing | project_page | whatsapp | call_click
  listing_id NULL, project_id NULL,
  message, requirement jsonb,   -- budget, config, timeline (optional fields)
  asset_requested,              -- brochure | floor_plan | price_list | callback | site_visit
  utm jsonb,                    -- source, medium, campaign, term, content
  page_url, referrer, ip_hash, user_agent,
  status,                       -- new | contacted | qualified | site_visit | closed | junk
  created_at
)
```

Design notes:
- **One `leads` table for everything.** Attribution lives in `source_type` + FK +
  `utm` — you'll thank yourself when comparing ad campaigns to organic portal leads.
- `media.is_gated=true` for brochures/floor plans: the API returns a signed URL only
  after a lead row is inserted (server action validates phone format first).
- `is_broker_visible_only` lets you keep off-market/co-broke inventory out of the
  public portal but inside the dense broker view.
- Phone is the primary dedupe key: on insert, if the same phone hit the same
  project/listing within 30 days, mark the lead `duplicate` in the pipeline but still
  serve the download (never block a user to protect a metric).

---

## 5. Main Portal — Sitemap & Page Anatomy

```
yourdomain.com/
├── /                          Home
├── /listings                  Search results (filterable, ISR + client filters)
│     ?intent=buy&bhk=3&locality=...&budget_min=...
├── /listings/[slug]           Listing detail page
├── /localities/[slug]         SEO locality pages ("Property in Sushant Lok 1")
├── /projects                  Index of all project landing pages (cards → subdomains)
├── /inventory                 Dense broker view (table layout, WhatsApp-first)
├── /about  /contact           Trust pages (your RERA no., experience, testimonials)
└── /privacy  /terms           Required for running Meta/Google ads
```

### Home
1. Hero: one-line positioning ("Curated resale & new-launch inventory in Gurgaon") +
   search bar (intent toggle, locality, budget, BHK).
2. Featured listings carousel (admin-flagged `is_featured`).
3. New-launch projects strip → links to subdomain pages (cross-funnel traffic is free).
4. "Why work with me" trust block: RERA registration, years, deals closed, testimonials.
5. Localities served grid → locality SEO pages.
6. Footer: contact, WhatsApp, RERA disclaimer, privacy.

### Search results (`/listings`)
- Filters as in §2; **chips for applied filters**; sort by price/newest/area.
- Card = photo, price, title, 4 specs, locality, posted date, `Contact` + `WhatsApp`.
- Mobile: filter bottom-sheet + sticky "Filter / Sort" bar. URL-driven state
  (shareable/SEO-crawlable filter combos for top localities).

### Listing detail (`/listings/[slug]`)
- Gallery (swipeable) → price block → spec table → amenities → map + landmarks →
  description → similar listings.
- **Sticky contact card** (desktop right rail / mobile bottom bar): your name + photo,
  `Call`, `WhatsApp` (deep link with pre-filled message containing the listing title/URL),
  and a 3-field enquiry form (name, phone, optional message).
- Every CTA writes a `leads` row (including click-to-call/WhatsApp taps, logged
  client-side) so you can see which listings generate interest.
- `schema.org/RealEstateListing` JSON-LD + OpenGraph (WhatsApp link previews matter
  more than Google here).

### Broker view (`/inventory`)
- Password-less but noindex; dense table: type, society, tower/floor, size, facing,
  demand, status; row-level WhatsApp button. Include `is_broker_visible_only` stock.
- Later: per-listing "Download share card" (branded image/PDF).

---

## 6. Project Landing Pages (subdomains) — Anatomy

Every project (`birla-arika.yourdomain.com`) renders the same battle-tested template,
content-driven from the `projects` row. Section order, top to bottom:

1. **Hero** — project render, name, developer logo, micro-market ("Sector 31, Gurgaon"),
   starting price ("₹5.2 Cr* onwards"), possession date, RERA no. in small print, and the
   primary CTA: **"Download Brochure"** + secondary "Enquire Now". On mobile, a **sticky
   bottom bar**: `Call` | `WhatsApp` | `Enquire`.
2. **Key highlights strip** — 4–6 icon stats (land area, towers, units, club size,
   possession).
3. **Price & configuration table** — one row per config (3 BHK / 3100 sq ft / ₹X Cr) with
   a per-row **"Get Price Breakup"** CTA (opens lead form).
4. **Floor plans** — blurred/watermarked thumbnails; clicking any = lead form →
   `asset_requested='floor_plan'` → signed URL / on-screen unlock. Same for **brochure**
   and **payment plan**.
5. **Gallery / renders**, optional walkthrough video.
6. **Amenities grid.**
7. **Location & connectivity** — map embed + "5 min to NH-48"-style distance chips.
8. **About developer** — 2 lines + credibility stats (Birla/Oberoi names sell themselves;
   borrow their trust).
9. **FAQ accordion** (also targets "Birla Arika price" long-tail queries).
10. **Final CTA block** + footer with the mandatory fine print: RERA number & QR,
    *"Authorized channel partner. This is not the official developer website."*,
    privacy policy link.

**Lead form fields:** name, phone (validated, +91 default), and *one* optional select
("Interested in: 3 BHK / 4 BHK / Just exploring"). Nothing else — every extra field
drops conversion measurably. OTP verification is optional later; start without it and
add only if junk-lead rate hurts.

**Conversion plumbing:** on submit → insert lead → fire Meta Pixel `Lead` +
Google Ads conversion (browser) → Supabase webhook → n8n → (a) WhatsApp template message
*to you* with lead details, (b) append to Google Sheet/CRM, (c) optional auto-WhatsApp
*to the lead* with the brochure PDF link — instant gratification and it moves the
conversation into WhatsApp where deals actually happen.

---

## 7. Admin Panel

- Auth: single Supabase email login (you), on `admin.yourdomain.com`.
- **Listings CRUD** with image multi-upload, drag-sort, status changes (marking `sold`
  keeps the page live with a "Sold" ribbon → social proof + SEO retention).
- **Projects CRUD**: create a project → subdomain is live in minutes (ISR revalidate).
- **Leads inbox**: filter by source/status/date, one-tap call/WhatsApp, status pipeline
  (new → contacted → qualified → site visit → closed/junk), CSV export.
- Nice-to-have later: per-campaign lead counts (group by `utm.campaign`).

---

## 8. SEO, Performance & Compliance

- **SEO:** locality pages + listing pages with JSON-LD; XML sitemap auto-generated from
  DB; canonical from subdomain → apex `/projects/[slug]` (or the reverse — pick one and
  stay consistent); `noindex` on `/inventory` and admin.
- **Performance budget:** LCP < 2.5s on 4G for landing pages (ads Quality Score and
  bounce depend on it). Next/Image everywhere, hero preloaded, no heavy sliders.
- **Compliance (India-specific):**
  - Your RERA agent registration number in the footer of every page.
  - Project pages: project RERA number + "not the official developer site" disclaimer —
    developers issue takedowns against channel-partner pages that skip this.
  - Privacy policy + consent line under lead forms (required by Meta/Google ad policies,
    and by DPDP Act since you're storing phone numbers).
  - `tel:` and WhatsApp links must use your registered business number (WhatsApp
    Business API if you automate sends).

---

## 9. Risks & Decisions to Lock Early

| Decision | Recommendation |
|---|---|
| Build vs WordPress | Build (Next.js + Supabase) — gating, subdomains, and lead webhooks are first-class; WP fights you on all three |
| Subdomain vs subfolder for projects | **Subdomains** for ad landing pages (clean brand per project, per-project pixels), mirrored under `/projects/*` for SEO |
| OTP-verify leads? | Not at launch; add if junk >20% |
| CRM now? | No — Google Sheet via n8n first; graduate to TeleCRM/Privyr/HubSpot once >100 leads/month |
| Who hosts brochures? | Your storage with signed URLs — never link the developer's PDF (they move/expire and you lose the gate) |

---

## 10. Execution Plan

### Phase 0 — Foundation (Week 1)
1. Buy/confirm domain; set up Vercel + wildcard DNS (`*` CNAME) and Supabase project.
2. Scaffold Next.js app with host-based middleware routing (apex / `*.` / `admin.`).
3. Create the schema from §4 (SQL migration), storage buckets (`photos`, `collateral`
   with gated policy), and seed one dummy listing + one dummy project.
4. Set up GA4, GTM container, and the n8n lead webhook skeleton (webhook → WhatsApp
   notification to you + Google Sheet append).

**Exit criteria:** `test-project.yourdomain.com` renders from a DB row; a form submit
creates a lead row and pings your WhatsApp.

### Phase 1 — Project landing pages MVP (Weeks 2–3) ← ships first: it makes money first
1. Build the landing-page template (§6) with all sections content-driven.
2. Implement gated downloads (lead → signed URL) + pixel/conversion events.
3. Load **Birla Arika** and **Oberoi 360 North** with real content, renders, floor
   plans, brochures; QA on mobile (PageSpeed ≥ 90).
4. Legal pass: RERA numbers, disclaimers, privacy policy.
5. Point ad campaigns at the subdomains; verify conversions fire in Meta/Google.

**Exit criteria:** two live project pages collecting attributed leads end-to-end.

### Phase 2 — Portal core (Weeks 4–6)
1. Admin panel: listings CRUD + image upload + leads inbox.
2. Public portal: home, `/listings` with filters, listing detail with sticky contact
   card, WhatsApp deep links, similar listings.
3. Enter your real inventory (aim for every active listing with ≥5 photos).
4. JSON-LD, OpenGraph, sitemap; submit to Search Console.

**Exit criteria:** full browse → filter → detail → enquire loop live with real inventory.

### Phase 3 — Broker & SEO layer (Weeks 7–8)
1. `/inventory` dense broker view incl. broker-only stock.
2. Locality SEO pages for your top 8–10 micro-markets.
3. Lead pipeline statuses in admin; duplicate detection; CSV export.
4. Auto-WhatsApp brochure delivery to leads (WhatsApp Business API via n8n).

### Phase 4 — Growth & polish (ongoing)
- New project subdomains as launches happen (content-only, no code).
- Share-card generator for brokers; testimonials; campaign-level lead reporting;
  CRM migration when volume justifies it; optional OTP verification.

### Rough effort/cost
- Domain ~₹1k/yr; Vercel hobby + Supabase free tier: ₹0 to start; WhatsApp Cloud API
  ~free at your volumes; Cloudinary free tier. **Real cost is build time:** Phases 0–2
  ≈ 4–6 weeks of focused part-time work (or a small freelance engagement), after which
  adding a project page is a 30-minute content task.

---

## 11. Reference Sources

- Portal field/filter structure: 99acres & MagicBricks data-anatomy write-ups
  ([Actowiz overview](https://www.actowizsolutions.com/indian-real-estate-data-extraction-99acres-magicbricks-housing.php),
  [Apify 99acres scraper docs](https://apify.com/easyapi/99acres-com-scraper/api),
  [MagicBricks details-page fields](https://apify.com/ecomscrape/magicbricks-property-details-page-scraper/api))
- Broker B2B patterns: [RealBetter on Google Play](https://play.google.com/store/apps/details?id=com.presideatech.realbetter&hl=en)
- Landing-page conversion practices:
  [MNKY landing page guide](https://mnky.agency/real-estate-landing-page-guide/),
  [Landingi real-estate templates](https://landingi.com/blog/real-estate-landing-pages/),
  [HousingWire high-converting pages](https://www.housingwire.com/articles/real-estate-landing-pages/),
  [Placester landing page types](https://placester.com/real-estate-marketing-academy/custom-landing-pages-real-estate-lead-generation)
