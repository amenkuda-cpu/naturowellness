# Naturo Wellness: website & online shop

A premium, animated e-commerce site for **Naturo Wellness**: natural wellness shots, traditional Zimbabwean
favourites and herbal blends. Built with Next.js 16 (App Router, Cache Components), React 19, TypeScript,
Tailwind CSS 4, GSAP + ScrollTrigger, Motion for React, Lenis and Supabase.

> **Status in one line:** the storefront, cart, checkout, order tracking and admin are built and tested
> end-to-end in development mode. **Before taking real orders** you must connect Supabase, set real prices,
> enable at least one payment method, fill in contact details and have the policy pages reviewed (see
> [Launch checklist](#launch-checklist)).

---

## Contents

1. [Quick start](#quick-start)
2. [Scripts](#scripts)
3. [Project structure](#project-structure)
4. [Environment variables](#environment-variables)
5. [Editing products & prices](#editing-products--prices)
6. [Replacing placeholder images](#replacing-placeholder-images)
7. [Database (Supabase)](#database-supabase)
8. [Creating the first administrator](#creating-the-first-administrator)
9. [Payments](#payments)
10. [Deployment](#deployment)
11. [Testing & measured results](#testing--measured-results)
12. [Feature status](#feature-status)
13. [Launch checklist](#launch-checklist)

---

## Quick start

Requirements: Node.js 20.9+ (developed on Node 24) and npm.

```bash
npm install
cp .env.example .env.local
```

For a full local demo **without any external services**, set these in `.env.local`:

```bash
NATURO_DEV_SAMPLE_PRICES=true   # clearly labelled SAMPLE prices (never used in production builds)
NATURO_DEV_ADMIN=true           # open /admin without login (dev only)
PAYMENT_COD_ENABLED=true
PAYMENT_MANUAL_ENABLED=true
```

```bash
npm run dev        # http://localhost:3000
```

In this mode, orders are saved to `.naturo-dev/store.json` (git-ignored) and a small "Dev preview" badge is
shown. The sample discount code `DEVTEST10` works only in this mode.

All development conveniences are **forced off when `NODE_ENV=production`**. A production build without
Supabase runs as a browse-only catalogue with "Price coming soon" on every product and checkout disabled.

## Scripts

| Command | What it does |
| --- | --- |
| `npm run dev` | Development server |
| `npm run build` / `npm start` | Production build / serve |
| `npm run lint` · `npm run typecheck` | ESLint · TypeScript |
| `npm test` | Unit tests (Vitest): pricing, promotions, tax, validation, cart, Paynow hashing, order tokens |
| `npm run test:db` | Applies the Supabase migration + seed to an in-memory Postgres (PGlite) and verifies `place_order` and RLS |
| `npm run test:e2e` | Browser tests (Playwright, desktop + mobile) against the dev server |
| `npm run placeholders` | Regenerates the placeholder bottle & ingredient artwork |
| `npm run db:seed-file` | Regenerates `supabase/seed.sql` from `src/data/products.ts` |

First-time Playwright setup: `npx playwright install chromium`.

## Project structure

```
src/
  app/
    (site)/            storefront pages (home, shop, product, cart, checkout, order, account, content, policies)
    admin/             staff dashboard (overview, orders, products, settings) + server actions
    api/               checkout, cart quote, order status, Paynow result webhook
    auth/callback/     Supabase magic-link callback
  components/
    home/              hero, cinematic showcase (server markup + client GSAP controller), moments, story…
    product/ cart/ checkout/ shop/ layout/ admin/ visual/ motion/ brand/ ui/
  config/              site.ts (brand, contact, nav) · commerce.ts (currency, tax, delivery, dev samples)
  data/                products.ts (THE product catalogue) · moments.ts
  lib/
    commerce/          pricing engine (pure), validation (Zod), cart ops, cached catalogue
    orders/            order service, access tokens, types
    payments/          provider registry + Paynow protocol
    repositories/      Supabase / dev JSON store / static catalogue
    supabase/          server clients (service role is server-only)
    auth.ts env.ts analytics.ts gsap.ts
  proxy.ts             refreshes Supabase session cookies (Next 16 "proxy", formerly middleware)
supabase/              migrations/ (schema + RLS + place_order) · seed.sql
public/images/         products/ · ingredients/ · brand/ · placeholder/
scripts/               placeholder generator · seed generator · DB validation
tests/                 unit/ · e2e/
```

### Creative & motion architecture

- **Concept: "Botanical Plates".** Products are presented like suspended specimens in a sunlit botanical
  archive: bottles hang from fine lines, numbered plates, editorial type (Bricolage Grotesque + Instrument
  Serif + Outfit), saturated colour fields balanced with cream.
- **Hero:** a cluster of seven SVG vine stems bursts from the bottom-right corner (~5.5 s), staggered
  and growing at different speeds: some shoot to the top, some sweep wide to the left, one hooks back
  over the others. Branches, sub-branches, curling tendrils and ~590 leaves spawn in waves behind the
  growing tips (stepped-width stroked segments + stroke-dashoffset; timings precomputed per element).
  Stems sit in back / mid / front depth layers (background lighter and softly blurred once grown); each
  stem is its own composited layer swaying slightly out of sync. The bottles then drop in and hang from
  the main arch. Reduced motion shows it fully grown. Vine: `src/components/home/hero-vine.tsx`.
- **"Meet your daily goodness":** desktop pins one scrubbed master timeline; every scene reveals its colour
  field with a clip-path and enters with its own choreography (swing, soft focus, drop, tropical spin,
  rise, steam, two-bottle meet). Phones and reduced-motion visitors get stacked, fully readable scenes.
- Animated elements use nested transform layers (`[data-pendulum]` → `[data-sway]` → `[data-tilt]`) so
  GSAP never fights itself; every animation lives in `useGSAP` / `gsap.matchMedia` and is reverted on
  unmount; below-the-fold choreography is built in idle time; `prefers-reduced-motion` disables it all.
- Lenis smooth scrolling is driven by GSAP's ticker and is lazy-loaded on desktop only.
- Three.js was deliberately **not** used: the 2D layered compositions deliver the effect at a fraction of
  the cost on ordinary phones.

## Environment variables

See [`.env.example`](.env.example) for every variable with comments. Key rules:

- `SUPABASE_SERVICE_ROLE_KEY` and `PAYNOW_INTEGRATION_KEY` are **server-only**. Never prefix them with
  `NEXT_PUBLIC_`. They are read only in `src/lib/env.ts` (marked `server-only`).
- `NEXT_PUBLIC_SITE_URL` must be the real public HTTPS URL in production (payment return URLs, magic
  links, sitemap, structured data).
- Contact details and social links stay hidden until set. Nothing is invented.

## Editing products & prices

**Product content** (names, descriptions, art direction, FAQs) lives in **`src/data/products.ts`**, the single
source used by every page. Fields such as `ingredients`, `nutrition`, `allergens`, `size`,
`preparation` and `verifiedBenefits` are intentionally empty: fill them in **only from the approved label**.
`motifs` are decorative and never shown as ingredients.

**Prices & stock:**
- With Supabase connected: use **Admin → Products** (price, stock status, quantity, enable/disable).
  Changes go live immediately (`updateTag("catalog")`).
- A product is purchasable only when it is enabled **and** has a price > 0 **and** isn't out of stock. This is
  enforced in the pricing engine on the server, in the database (`price_cents > 0` constraint) and in the UI.
  Unpriced products show "Price coming soon".

Adding a product: add an entry to `seeds` in `products.ts`, add its image, then (with Supabase) run
`npm run db:seed-file` and apply the new rows, or insert them via the Supabase dashboard.

## Replacing placeholder images

All bottle/pouch images are **temporary artwork** with a visible "PLACEHOLDER LABEL"; they are not approved
packaging. To replace one:

1. Export the final photo (ideally a transparent-background WebP or PNG, at least 900 × 1350 px).
2. Save it over the matching file in `public/images/products/`:
   `ginger-shot-placeholder.webp`, `glow-shot-…`, `hibiscus-shot-…`, `pineapple-shot-…`,
   `mens-vitality-…`, `maheu-…`, `muvhee-…`, `herbal-tea-…`, `instant-powder-…`
   (or use a new filename and update `image()` in `products.ts`).
3. Set `placeholder: false` for that product (removes the "Placeholder artwork" notice).
4. If the product hangs from a line (hero, showcase, CTA), adjust `HANG_POINT` in
   `src/components/visual/hanging-bottle.tsx` so the line meets the cap.

Images are rendered with `next/image` inside fixed-ratio boxes using `object-contain`, so a differently
proportioned photo still fits. Ingredient art lives in `public/images/ingredients/`. The logo is a vector
redrawn from the supplied reference (`src/components/brand/logo.tsx`, `src/app/icon.svg`); replace it with
the official vector when available.

## Database (Supabase)

1. Create a Supabase project (choose the region closest to Zimbabwe, e.g. South Africa / Europe).
2. Apply the schema:
   - CLI: `supabase link --project-ref <ref>` then `supabase db push`, **or**
   - paste `supabase/migrations/20261014000000_initial_schema.sql` into the SQL editor.
3. Load the catalogue: run `supabase/seed.sql` (regenerate with `npm run db:seed-file`).
4. Set `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`.
5. Auth → URL configuration: set Site URL to `NEXT_PUBLIC_SITE_URL` and add
   `https://<your-domain>/auth/callback` to redirect URLs.
6. Add delivery zones (with fees) in the `delivery_zones` table; zones without a fee are not offered.

Security model (verified by `npm run test:db`):
- **RLS on every table.** Public can read the catalogue; customers read only their own orders; guests'
  orders are never readable through the API (guests use a private hashed-token link); staff can read/update
  orders; only admins manage roles, promotions and catalogue rows.
- Orders are created only through `place_order()`, a `security definer` function callable **only by the
  service role**, which locks variants, checks and decrements stock, records inventory movements and
  enforces promotion limits in one transaction.
- Admin writes run as the signed-in staff member (RLS applies), not with the service key.

## Creating the first administrator

1. Sign in once at `/account/login` with the administrator's email (magic link). This creates their
   `auth.users` row.
2. In the Supabase SQL editor (which runs with owner rights):
   ```sql
   insert into public.staff_members (user_id, role)
   select id, 'admin' from auth.users where email = 'owner@your-domain.co.zw';
   ```
3. Visit `/admin`. Further staff can be added the same way with role `'staff'` (can manage orders but not
   prices or roles).

Never grant roles from the browser, and never set `NATURO_DEV_ADMIN` in production (it is ignored there anyway).

## Payments

Payment providers are modular (`src/lib/payments/index.ts`); checkout shows whatever is enabled.

| Method | Status | Notes |
| --- | --- | --- |
| Cash on delivery / collection | ✅ built | `PAYMENT_COD_ENABLED=true`. Order starts *unpaid*; staff mark it paid. |
| Manual (mobile money / bank transfer) | ✅ built | `PAYMENT_MANUAL_ENABLED=true` + instructions. Staff verify before marking paid. |
| **Paynow** (EcoCash, OneMoney, InnBucks, Zimswitch, Visa/Mastercard) | ✅ built, ⚠️ **untested against live Paynow** | Needs merchant approval + integration ID/key. |

**Research summary (October 2026, verify before signing up):** Paynow is a Zimbabwean gateway listing EcoCash,
OneMoney, InnBucks, Zimswitch, O'mari, Visa and Mastercard, with official SDKs requiring an integration ID and
key. Card payments use Paynow's redirect checkout. Pesepay and ZB Bank's Smile&Pay are other Zimbabwean
options that could be added as providers. I could **not** confirm that Stripe or Flutterwave accept
Zimbabwean merchants; check their official country lists before considering them.

**Paynow setup**
1. Apply for a Paynow merchant account and create an integration (use test mode first).
2. Set `PAYMENT_PAYNOW_ENABLED=true`, `PAYNOW_INTEGRATION_ID`, `PAYNOW_INTEGRATION_KEY`, `PAYNOW_MERCHANT_EMAIL`.
3. The result URL is `https://<your-domain>/api/payments/paynow/result` (sent automatically per transaction;
   must be publicly reachable over HTTPS).
4. Place test orders and confirm statuses update. **Do this before going live.**

**How payment is verified:** an order is never marked paid because the browser returns to a "success" page.
The webhook verifies Paynow's SHA-512 hash, then the server re-polls Paynow using the poll URL *it stored*
at initiation, re-verifies the hash, checks the reference and that the **amount equals the order total**,
and only then records `paid`. Staff cannot manually mark Paynow orders paid. Card data never touches this site.

## Deployment

Any Node.js host that supports Next.js 16 works (e.g. Vercel, a VPS with `npm run build && npm start`).

1. Set all production environment variables (never `NATURO_DEV_*`).
2. `npm run build`. The build prints which routes are static / partially prerendered.
3. Point your domain at the host (HTTPS is required for payments and magic links).
4. Update Supabase Auth URLs and the Paynow integration with the production domain.
5. Submit `https://<domain>/sitemap.xml` in Google Search Console.

Security headers (HSTS, frame denial, no-sniff, strict referrer: `no-referrer` on order links) are set in
`next.config.ts`.

## Testing & measured results

Results from the final run in this repository's development environment:

- **Unit tests:** 28/28 passing (`npm test`).
- **Database checks:** all passing on PGlite (PostgreSQL compiled to WebAssembly) with stubbed Supabase auth
  (`npm run test:db`): migration + seed apply; `place_order` reserves stock, rejects overselling and
  exhausted promo codes, and rolls back atomically; RLS blocks anonymous/customer access to orders,
  promotions, price changes and self-promotion to staff.
- **End-to-end:** 19 passing + 1 intentionally skipped (admin flow runs on desktop only), on desktop
  (1440 × 900) and mobile (Pixel 7): navigation, filters/search, unpriced-product protection (UI and API),
  token-protected order access, cart drawer, cart page, checkout validation & focus management, discount
  codes, order placement, admin status update, reduced motion and horizontal-overflow checks.
- **Horizontal overflow:** none at 360 px on home, shop, product, checkout, story, ingredients, contact, FAQ.
- **Lighthouse 12** (production build, local machine, default simulated throttling):

  | Page | Mobile perf | Desktop perf | Accessibility | Best practices | SEO | CLS |
  | --- | --- | --- | --- | --- | --- | --- |
  | Home | 65 | 88 | 100 | 100 | 100 | 0 |
  | Shop | 65 | 98 | 100 | 100 | 100 | 0 |
  | Product | 71 | 97 | 100 | 100 | 100 | 0 |
  | Our story | 58 | 95 | 100 | 100 | 100 | 0 |

  Accessibility was re-audited after contrast/heading fixes: 100 on home, shop, product, checkout, cart,
  story, ingredients, contact and FAQ (checkout SEO is 66 by design: it is `noindex`). Mobile LCP is ~4.4–5 s under simulated slow 4G + 4× CPU
  (≈2.8 s with real DevTools throttling); the animation-rich homepage is the heaviest page. Re-measure on
  your production host/CDN, where results will differ.

Not tested (requires credentials/services): live Supabase project, Supabase Auth email delivery, live Paynow
transactions and webhooks, real device testing on iOS Safari.

## Feature status

| Area | Status |
| --- | --- |
| Homepage hero, cinematic showcase, moments, story, standards, range, CTA | ✅ Complete |
| Shop (categories, search, sort, availability, moment filters), product pages, SEO/JSON-LD | ✅ Complete |
| Cart drawer + page, server-validated quotes, checkout, order confirmation & status | ✅ Complete |
| Guest order access via private hashed token | ✅ Complete |
| Supabase schema, RLS, atomic order function, seed | ✅ Built & validated on Postgres (not on a live project) |
| Customer accounts (magic link, order history) | ✅ Built, requires Supabase |
| Admin: overview, orders (status, payment confirmation), products (price/stock/enable), settings | ✅ Built (dev mode tested; Supabase mode untested live) |
| Paynow integration | ⚠️ Built to Paynow's documented protocol, untested without credentials |
| Analytics hooks (`product_view`, `add_to_cart`, `begin_checkout`, `purchase`, …) | ✅ Built; choose a provider |
| Admin: discount-code editor, product content/image upload, delivery-zone & category editors, charts | ❌ Not built (use Supabase dashboard for now) |
| Email/SMS order notifications | ❌ Not built (needs a provider decision) |
| Contact form | ❌ Not built (contact details only) |
| Reviews / best-sellers | ❌ Intentionally not built until genuine data exists |

## Launch checklist

- [ ] Supabase connected, migration + seed applied, first admin created
- [ ] Real prices, sizes and stock set; ingredient lists, allergens and storage copied from approved labels
- [ ] Final product photography and official logo vector in place (`placeholder: false`)
- [ ] Delivery zones & fees and collection details confirmed
- [ ] At least one payment method enabled; Paynow tested end-to-end in test mode
- [ ] Contact details & social links set in env
- [ ] Privacy, terms and returns pages reviewed and approved (currently marked **draft**)
- [ ] Marketing copy reviewed by Naturo (all health wording is deliberately non-medical; keep it that way)
- [ ] Tax rate confirmed with an accountant (`NEXT_PUBLIC_TAX_RATE_BPS`)
- [ ] Analytics provider chosen; cookie/consent needs reviewed
- [ ] Lighthouse & real-device check on the production domain
