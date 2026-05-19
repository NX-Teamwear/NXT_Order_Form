NX Teamwear — Project Context

For anyone picking this up: Read this first. It covers what's been built, every key decision, and what's left to do. Keep it updated as the project moves forward — it's the shared source of truth between team members and Claude sessions.

The Business

NX Teamwear is part of the Masuri Group. It sells custom-branded teamwear sourced wholesale from Ralawise, with a headline USP of 5-day turnaround.

The product range is Ralawise's entry-level range — plain garments that NX brands up with custom logos, names, and numbers. No stock is held by NX; orders are fulfilled directly through Ralawise.

Two-Phase Product Roadmap

The order form has two distinct aims. The current build targets Phase 1. Phase 2 is the fully integrated version and is being designed alongside it so Phase 1 doesn't create rework.

Phase 1 — Independent Form (current build)

A standalone form that operates completely outside BigCommerce checkout. The customer journey:

Customer fills in the order form — products, colours, logo positions, sizes, personalisation, contact details
Form submits to the NX team by email (via Zoho Flow webhook)
Katie or Freddie manually adds the products to BigCommerce and creates a payment link
Customer receives the payment link, reviews products and sizes in BigCommerce, and pays
NX team produces a digital mockup, gets customer approval, then places the order with Ralawise
No payment is collected in the form. No BigCommerce API is called. The form exists purely to capture everything needed to build the quote and mockup manually.

Phase 2 — BigCommerce-Integrated Form (future build)

The same customer-facing form, but fully wired into the wider order infrastructure. Everything behind the submit button is automated:

Customer completes the form as in Phase 1
On submission, the form:
Generates a unique order ID for the job
Calls a digitisation API to process and enhance the uploaded logo artwork
Saves all branding information to an FTP folder named with the unique order ID — position data, artwork files, personalisation details
Creates the order directly in BigCommerce via API — products, quantities, sizes, pricing — so the customer can go straight to checkout without any manual setup by the team
Customer completes payment through BigCommerce checkout
Order Desk (the fulfilment management system) picks up the order and retrieves the logo/branding data from the FTP folder using the unique order ID
Order Desk routes the job through decoration and dispatch
Key technical components needed for Phase 2 (none built yet):

BigCommerce Storefront/Management API — create cart and order programmatically
Digitisation API — vendor and endpoint TBD
FTP storage with folder structure keyed by unique order ID
Unique order ID generation scheme — format TBD
Order Desk integration — reads from FTP by order ID
What's Been Built

File 1: nx_order_form.html

The main customer-facing order form. A single self-contained HTML/CSS/JS file — no framework, no build step, no dependencies beyond Google Fonts. Designed to be embedded in a BigCommerce Custom HTML widget on a Web Page.

File 2: nx_calibrator.html

An internal tool for calibrating logo position markers on product images. Not for customers — used by the NX team to define where logo circles appear in the form's garment preview.

Both files live in the repo root.

Order Form — Structure

Six steps, each a div.nx-panel shown/hidden by goStep(n):









Step

Panel ID

Label

What it does

1

p0

Products

Select one or more products. Each adds an independent "line" with a unique UID.

2

p1

Colours

Per line: pick a colour from swatches.

3

p2

Branding & Preview

Branding is done directly on the garment preview. Per line: pick a position marker, then place a logo on it. Logos are uploaded files (several at once) or typed text logos, each with optional colour notes. Per-product price bar and the name & number notice are shown here. Product cards render as a single vertical list.

4

p3

Sizes

Per line: qty table for each size. Live subtotal per line.

5

p4

Personalisation

Toggle-gated. If on: size-aware table where customer enters name + number per item.

6

p5

Contact & Delivery

Name, email, phone, address, postcode. Submit button.

Navigation: goStep(n) in JS handles panel switching and triggers the relevant render function.

Order Form — Key Data Structures

PRODUCTS array

17 products. Each has:

{ code, name, cat, age, colours, rrp, sizes }   // rrp is the base price; per-logo fees are global constants (see Pricing)









Code

Name

Category

JC001

Active Tee (Adult)

tshirt

SX701

Casual Tee (Adult)

tshirt

JC01J

Active Tee (Junior)

tshirt

SX702

Casual Tee (Junior)

tshirt

JH001

Casual Hoodie (Adult)

hoodie

JH006

Active Hoodie (Adult)

hoodie

JH01J

Casual Hoodie (Junior)

hoodie

JH06J

Active Hoodie (Junior)

hoodie

JH030

Casual Sweater (Adult)

hoodie

JH30J

Casual Sweater (Junior)

hoodie

AQ015

Casual Polo (Adult)

polo

JC040

Active Polo (Adult)

polo

JC40J

Active Polo (Junior)

polo

BC015

Basic Cap

cap

YP004

Flexfit Cap

cap

BC446

Circular Patch Beanie

beanie

BG212

Universal Backpack

bag

Pricing (additive, per product line)

The four hardcoded tiers (rrp/rrpSmall/rrpLarge/rrpBoth + priceTier) were removed. Price is now built up from the base:

linePrice = rrp + max(0, smallCount − FREE_SMALL_LOGOS) × SMALL_LOGO_FEE + largeCount × LARGE_LOGO_FEE

base — product rrp (no logo charge yet)
first small logo placed — free (FREE_SMALL_LOGOS = 1); Left Breast / Front Panel are small, so this normally covers the main logo
each further small logo — + SMALL_LOGO_FEE (£2.50)
each large logo — + LARGE_LOGO_FEE (£4.00)

So Left Breast only = rrp; Left + Right Breast = rrp + £2.50; Right Breast alone = rrp (it's the free small); add a large = +£4 each. Constants live at the top of the PRICING block; logoCounts(line) tallies small/large from line.positions, logoSupplement(line) is the add-on, linePrice(line) the per-item total. Price updates live as logos are placed/cleared in the Garment Preview; the per-card price bar shows base + per-type breakdown.

Price bucket keys

LOGO_POS was removed. Logo positions come solely from PRODUCT_POSITIONS (the calibrated preview positions). Each position a logo is placed on is classified by a normalised label key, produced by brandingKey():

SMALL_POS_KEYS — leftbreast, frontpanel, rightbreast, leftsleeve, rightsleeve

LARGE_POS_KEYS — centrechest, shoulders, uppermidbacknotshoulders, tail

Left Breast and Front Panel are deliberately "small" so the one free-small allowance lands on the usual main logo. Anything not listed counts toward neither fee.

lines array

The live state of the order. Each line object:

{

  uid,        // unique ID, used to namespace all DOM elements for this line

  code,       // product code e.g. 'JC001'

  colour,     // selected colour string

  positions,  // synced branding-bucket keys for positions with a placed logo (drives pricing)

  sizes,      // { [sizeLabel]: qty }

  pers        // personalisation rows array

}

logoEntries was removed. Placement lives in previewState (per-uid: { activePos, placements, view, showTextForm, editLogoId }); each placement maps a position id → { id: logoId }. The shared per-card logo library is the global previewUploadedLogos array — each item is { id, kind:'file'|'text', label, notes } plus, for files, { file, src, isImage } and, for text logos, { text, font }. syncLineBranding(uid) derives line.positions from the placements; lineBrandingDetails(line) builds the per-position list (label, view, kind, file/text/font/notes) for the submitted order.

PRODUCT_IMAGES

BigCommerce-hosted images keyed by product code → colour → { front, back }.

Base URL:

https://store-9ong3twjo7.mybigcommerce.com/content/02%20-%20Ralawise%20Image%20%28Do%20not%20use%29/

Images follow the pattern: {CODE}/{CODE}_{ColourName}_FRONT.jpg and _BACK.jpg. Some junior/accessory products only have a front image (no _BACK).

Order Form — Branding & Preview Step (Step 3)

The branding controls and the garment preview were consolidated into a single Garment Preview (May 2026). There is no longer a separate branding panel — customers brand the garment directly on the preview.

Per product line, the preview card shows: the garment image with clickable circular position markers (from PRODUCT_POSITIONS), a "Select position" + "Place logo on selected position" sidebar with the logo library and "↑ Upload logos" / "Aa Add text logo" controls, and a per-product price bar. The name & number notice is shown once at the top of the section. Product cards render as a single vertical list, one per row.

A position only counts toward pricing and the submitted order once a logo has been placed on it. Each card has a shared logo library: every item is either an uploaded file (multiple selectable at once — AI/EPS/PDF/PNG/SVG, vector preferred; non-images show a type badge) or a typed text logo, and each carries an optional "Colour notes / artwork description" (plus a font field for text logos), edited via a small inline form on the tile (✎). Text logos are placeable just like images — the marker shows the text. Per-line copy-from-first was not reinstated.

Key functions:

renderPreview() — builds the Garment Preview (header + notice + one card per line); the entry point for step 3 (called by goStep(2))
renderPreviewCard(uid) — partial re-render of a single card on select / place / clear / toggle / logo change
renderLogoAreaHTML(uid) — the per-card logo library: tiles + Upload/Add-text controls + the inline text/edit forms
selectPreviewPos, placePreviewLogo(uid,logoId), clearPreviewLogo — position selection and placement (placement stores the logo id; the logo is resolved live)
handlePreviewUpload(uid, input) — captures one or more uploaded files (kind:'file') into previewUploadedLogos
toggleTextLogoForm / commitTextLogo — create a typed text logo (kind:'text')
startEditLogo / closeLogoEdit / updateLogoField — per-logo notes / text / font editing
syncLineBranding(uid) — rebuilds line.positions (price-bucket keys) from the placed logos
previewPriceBarHTML(line) — the per-card price bar; lineBrandingDetails(line) — placed positions with kind/file/text/font/notes for the order payload & email
brandingKey(label) — normalises a position label to its price-bucket key; esc() — HTML-escapes user-supplied strings

Preview position coordinates are per colour: PRODUCT_POSITIONS_BY_COLOUR (code → colour → front/back) holds the calibrated values; getProductPositions(code, colour) returns the colour's set, falling back to the single-set PRODUCT_POSITIONS reference (used by JH06J/BC015/BG212 — "no change" — and any uncalibrated colour). getPositions(code, colour, view) is the accessor used throughout the preview. Each position is { id, label, top, left, size } as percentages of the image box.

⚠️ Per-colour coordinates were calibrated with nxt_calibrator_v2.html and wired in; still worth a visual spot-check across colours before go-live — see Outstanding Tasks.

Calibrator Tool — How It Works

nx_calibrator.html is a standalone internal tool (open in a browser, not deployed to BigCommerce).

Workflow:

Select a product from the left rail
Front image loads automatically from BigCommerce CDN
Toggle to Back image with the toolbar button (where a back image exists)
Click anywhere on the garment to drop a position marker
Drag the marker to the exact position
Use the size slider to set the logo circle diameter
Rename each position (e.g. "Left Breast", "Back Centre")
Ctrl+Z to undo any action
Copy the JSON from the right panel
Output format (per product, from "All products" tab):

{

  "JC001": {

    "front": [

      { "id": "left_breast", "label": "Left Breast", "top": "34.2%", "left": "38.5%", "size": 56 }

    ],

    "back": [

      { "id": "back_centre", "label": "Back Centre", "top": "36.0%", "left": "50.0%", "size": 72 }

    ]

  }

}

Once calibrated, paste the full PRODUCT_POSITIONS JSON to Claude and ask it to wire the coordinates into the order form's POS_TEMPLATES, replacing the current placeholder values.

Products with front image only (no back toggle shown): JH06J, JH30J, JC40J, YP004, BC446.

Branding









Property

Value

Primary colour

#0e1a2b (dark navy)

Accent colour

#5bc3eb (sky blue)

Success colour

#3dd68c (green)

Heading font

Barlow Condensed, 800 weight, uppercase

Body font

Barlow, 400/600 weight

Google Fonts URL

https://fonts.googleapis.com/css2?family=Barlow:wght@400;600&family=Barlow+Condensed:wght@700;800

Technical Decisions

Single HTML file — no build system, no npm, no dependencies. The entire form is one .html file that can be copy-pasted into a BigCommerce Custom HTML widget. Same for the calibrator. This was a deliberate choice for simplicity of deployment and editing.

No payment in the form — customers receive a payment link separately after approving the mockup. The form is purely for capturing the order details.

Submission via Zoho Flow webhook — submitForm() POSTs multipart/form-data to a Zoho Flow incoming webhook (ZOHO_WEBHOOK_URL const). Hybrid shape: flat top-level fields (contact/delivery/totals) + email_summary (human-readable) + order_json (full structured order) + the placed logo files as file parts (file1..fileN, deduped; manifest in order_json.attachments). Fire-and-forget with mode:'no-cors' — cross-origin webhook responses can't be read, so success shows unless the request itself throws (network error → #submitErr). Only files actually placed on a position are attached; text logos travel in the JSON/summary. Heads-up: the zapikey is in the URL and visible in page source (spam risk) — a honeypot or Cloudflare-Worker proxy can harden this later.

File uploads not yet wired — the preview's "+ Upload logo" control captures the file object and filename in memory, but actual file transfer to storage is not implemented. Plan is Nextcloud + Cloudflare Worker. Parked until launch.

BigCommerce image CDN — all product images are served from the BigCommerce store's content directory. The folder is named 02 - Ralawise Image (Do not use) — the "Do not use" refers to the raw supplier images, not the CDN URL. The URLs work fine.

Preview is a guide, not a mockup — the garment preview in step 3 is explicitly described to the customer as a rough guide. The NX team produces accurate digital mockups before production. This framing is in the UI copy.

No React, no framework — a React prototype was reviewed early on and the coordinate/overlay concept was extracted and rebuilt in vanilla JS. Keeping it vanilla keeps deployment simple.

Submission Flow

Phase 1 (current)

When the customer hits Submit on step 6:

Form validates required fields
Builds a structured JSON payload of the entire order
POSTs multipart/form-data (fields + placed logo files) to the Zoho Flow webhook
Shows a success screen
The Zoho Flow webhook routes the submission to the NX team email. Katie or Freddie then manually creates the BigCommerce products and payment link. The webhook is wired in submitForm() (ZOHO_WEBHOOK_URL); collectPlacedFiles() builds the deduped attachment list.

Phase 2 (future)

When the customer hits Submit on step 6:

Form validates required fields
Generates a unique order ID
Calls the digitisation API with uploaded artwork
Uploads all logo/branding data to the FTP folder at /{orderID}/
Calls the BigCommerce API to create the order (products, sizes, quantities, pricing)
Customer is redirected to BigCommerce checkout to complete payment
Shows a confirmation screen with the order ID
Order Desk subsequently reads from the FTP folder using the order ID to retrieve all decoration instructions.

Repository Structure

/

├── nx_order_form.html      # Customer-facing order form

├── nx_calibrator.html      # Internal logo position calibration tool

├── CONTEXT.md              # This file

└── NX_Ralawise_Catalogue_v3.xlsx   # Source product/pricing data

Outstanding Tasks

🔴 Critical (blocks go-live)

1. Calibrate logo position coordinates DONE (May 2026) — per-colour coordinates calibrated in nxt_calibrator_v2.html and wired into the order form as PRODUCT_POSITIONS_BY_COLOUR (14 products / 196 colours). JH06J, BC015, BG212 ("no change needed") and any uncalibrated colour fall back to the single-set PRODUCT_POSITIONS reference via getProductPositions(). Remaining: a visual spot-check across colour variants before go-live, and per-colour calibration of JH06J/BC015/BG212 if their variants ever need it.

2. Webhook DONE (May 2026) — submitForm() POSTs multipart/form-data to a Zoho Flow webhook (ZOHO_WEBHOOK_URL). Remaining hardening (optional, before/at launch): the zapikey is exposed in page source, so consider a honeypot field and/or proxying via a Cloudflare Worker to curb spam; and confirm Zoho Flow's payload-size limit is comfortable for multiple large artwork files.

🟡 Before launch

3. Artwork file upload Implement actual file upload — plan is Nextcloud + Cloudflare Worker. Currently the preview captures the file objects in memory (previewUploadedLogos); they are not yet transferred to storage.

4. BigCommerce embed test Paste the full HTML into a BigCommerce Custom HTML widget on a Web Page and test end-to-end in that environment. Check for any CSP or iframe issues.

5. Cross-browser / mobile test The form has been built and tested in Chrome. Test in Safari and Firefox. Check the layout on mobile — it uses responsive CSS but hasn't been tested on small screens.

🟢 Nice to have

6. Add remaining product images PRODUCT_IMAGES currently has the first colour per product. Fill in all colour variants so the preview shows the correct colour when a customer picks e.g. Jet Black.

7. Colour swatch images The colour picker currently shows plain colour swatches. Could be improved with actual fabric swatch images if Ralawise provides them.

Phase 2 — Outstanding Design Decisions

These need to be resolved before Phase 2 build begins. Noting them here so they're not forgotten.

Unique order ID format What does the ID look like? Options: sequential number, date-prefixed (e.g. NX-20260514-0042), UUID. Needs to be decided — affects FTP folder structure and Order Desk integration.

Digitisation API Which vendor/service processes the uploaded artwork? Endpoint, authentication method, request/response format, and what "digitised" means in practice (vector conversion? embroidery format?) all need to be confirmed.

FTP folder structure What does the folder look like inside /{orderID}/? Suggested structure:

/{orderID}/

  order.json          ← full order payload (products, sizes, personalisation)

  logos/

    left_breast/      ← original + digitised file per position

    back_centre/

    ...

Exact structure needs agreement with whoever manages Order Desk.

BigCommerce API approach Two options: (a) Storefront API — creates a cart client-side and redirects to checkout, or (b) Management API — creates the order server-side. Option (a) is simpler but requires the form to run in a BigCommerce storefront context. Option (b) requires a server-side component (Cloudflare Worker or similar). To be decided.

Artwork file upload (both phases) Currently the form captures filenames only. For Phase 1 the files need to go somewhere the team can access (Nextcloud was the original plan). For Phase 2 they go to FTP. The upload mechanism (Cloudflare Worker as proxy) can serve both — worth building it once for Phase 1 in a way that Phase 2 can reuse.

Working with Claude

Both team members should use Claude Projects and add this file plus the current nx_order_form.html as project files. This gives Claude full context without re-uploading every session.

Claude Projects are not shared between accounts. Katie and Freddie each have their own. Keep this CONTEXT.md updated in the GitHub repo and re-add it to your project when it changes significantly.

When starting a new Claude session on this project, paste or upload:

This CONTEXT.md
The current nx_order_form.html (so Claude can read and edit the actual code)
Useful things to tell Claude at the start of a session:

Which task you're working on
Whether you're Katie or Freddie (so Claude knows who has what context)
Any decisions made since this document was last updated
Change Log









Date

Who

What

May 2026

Katie

Initial build — multi-step form, all 17 products, pricing, colour swatches, logo positions, personalisation, submission

May 2026

Katie

Branding and Preview steps merged into single step 3

May 2026

Katie

Calibrator tool built with drag, undo, size slider, auto-loaded product images

May 2026

Katie

GitHub repo and two-person Claude workflow set up

May 2026

Katie

CONTEXT.md updated with two-phase roadmap (Phase 1 independent form, Phase 2 BigCommerce-integrated)

May 2026

Claude

Page 3 redesign — Branding panel removed; logo position + artwork selection consolidated into the Garment Preview. The preview now drives the live price tier and the submitted order (positions mapped to free/small/large; Upper Back & Tail = large). Product cards switched to a single vertical list. Text-only logos and copy-from-first dropped.

May 2026

Freddie

Removed the redundant Back Centre position and merged the duplicate "shoulder" key into "shoulders" across PRODUCT_POSITIONS and the price buckets.

May 2026

Claude

Multi-image upload, then per-logo Upload/Text restored. Each logo library item is an uploaded file (multi-select; AI/EPS/PDF/PNG/SVG) or a typed text logo, with an optional colour-notes/description field (and font for text). Text logos are placeable like images; payload & email carry kind/text/font/notes; user strings escaped.

May 2026

Claude

PDF/EPS/AI now show a clear file-type placeholder (browsers cannot rasterise these client-side) with robust extension-based detection instead of a blank/broken image. Garment preview image enlarged.

May 2026

Claude

Pricing switched to an additive per-logo model: base = product rrp, first small logo free, each further small +£2.50, each large +£4.00 (constants SMALL_LOGO_FEE / LARGE_LOGO_FEE / FREE_SMALL_LOGOS). Removed the rrpSmall/rrpLarge/rrpBoth fields and priceTier(); order payload/email now carry base_price, small_logos, large_logos and logo_supplement instead of price_tier.

May 2026

Claude

Pricing fix: Left Breast / Front Panel are now classed as small logos (added to SMALL_POS_KEYS). Previously they were a separate free base logo on top of the free first small, so Left + Right Breast incurred no charge. Now the single free-small allowance covers the main logo and the second breast logo is correctly +£2.50.

May 2026

Claude

Added nxt_calibrator_v2.html — per-colour calibrator. Driven by the order form's PRODUCT_IMAGES manifest (17 products, 211 colours, ~422 front/back images); every colour seeded from the current PRODUCT_POSITIONS reference, manual nudge per image, colour stepper, progress counts, "apply to all colours" accelerator. Outputs PRODUCT_POSITIONS_BY_COLOUR (code → colour → front/back). Order-form wiring to consume per-colour positions is a deferred follow-up.

May 2026

Claude

Wired per-colour calibration into the order form. Added PRODUCT_POSITIONS_BY_COLOUR (14 products / 196 colours, from nxt_calibrator_v2.html) and getProductPositions(code, colour) — per-colour positions with fallback to the single-set PRODUCT_POSITIONS reference for JH06J/BC015/BG212 ("no change") and any uncalibrated colour. getPositions is now colour-aware (code, colour, view); preview/markers/pricing-sync/payload all resolve positions per selected colour.

May 2026

Claude

Wired submission to a Zoho Flow webhook. submitForm() now POSTs multipart/form-data (hybrid: flat contact/delivery/total fields + email_summary + order_json + deduped placed-logo file parts file1..N with an attachments manifest) via mode:'no-cors', fire-and-forget (network error → #submitErr, button re-enabled). Added ZOHO_WEBHOOK_URL + collectPlacedFiles(). zapikey is exposed in page source — hardening (honeypot / Worker proxy) noted as optional follow-up.

Update this table whenever a significant change is made to either file.