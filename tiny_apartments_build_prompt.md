# Build Prompt: Tiny Apartments (Next.js + Supabase)

Paste this into Claude Code or Cursor, with `tiny_apartments_schema.sql` and the logo file in the repo root. Work phase by phase and stop after each phase for review.

## 1. Goal
Build a direct-booking website for **Tiny Apartments**, a boutique two-bedroom guesthouse apartment on **Sh. Goidhoo, Shaviyani Atoll, Maldives** (approx. 6.429 N, 72.923 E; replace with the exact pin). Guests check live availability, browse a non-alcoholic minibar menu and a laundry service, submit a booking request, pay by manual bank transfer (BML / MIB) and upload a receipt. Staff verify receipts and approve or reject. No third-party commissions.

This is a **local-island guesthouse**. Copy and imagery should feel quiet, personal and coastal. Avoid resort-style language.

## 2. Stack
- Next.js (App Router, Server Actions, TypeScript strict), Tailwind CSS, shadcn/ui
- Supabase (Free tier): Postgres + RLS, Auth, Storage
- Resend for email, Vercel for hosting
- `browser-image-compression` for client-side image compression
- Zod for all input validation

## 3. Source of truth: the database
`tiny_apartments_schema.sql` is final. **Do not redesign it.** If you think a change is needed, propose it and wait. Key points:
- Double-booking is prevented by a DB exclusion constraint; handle error code `23P01` as "dates no longer available".
- Prices come only from `quote_booking()`. Never calculate totals in the client.
- Guests never touch `bookings` or `payment_slips` directly. Server Actions call `create_booking()` and `register_slip_upload()` using the **service role key**, which must exist only in server code and never reach the client bundle.
- Staff actions use the logged-in user's client and call `change_booking_status()` (approve / reject / cancel / complete). A rejection requires a reason.
- Availability calendar uses `get_unavailable_ranges()`; ranges are half-open, so check-out day is free.
- Anonymous reads of `apartment_settings` must list columns explicitly (no `select *`); bank details are served server-side only after a booking exists.
- All "today" logic uses the `Indian/Maldives` time zone (UTC+5).
- **One add-ons catalog:** minibar and laundry share `addon_categories`, `addon_products` (with `addon_type` = `minibar` | `laundry`) and `addon_variants` (`track_stock = false` for laundry). Build one catalog UI and one CRUD, filtered by type. Tax: TGST applies to add-ons and laundry; Green Tax does not.
- **Laundry:** pre-booked laundry is priced in the booking invoice (same slip) and becomes a queue job automatically on approval. In-stay requests use `create_laundry_request()` (approved booking, during the stay) and are billed separately: pay at checkout or upload a slip via `register_laundry_slip()`; Front Desk marks them paid with `mark_laundry_paid()`. Queue status changes use `set_laundry_status()`: requested -> picked_up -> in_progress -> delivered (cancel is Front Desk / Super Admin only).

## 4. Brand
- Primary: burgundy `#7B1113`. Backgrounds: white, `#EBF6FF`, `#E3F2FD`. Accents: `#BCE0FD`, `#90CAF9`. Text: `#1E293B`, muted `#64748B`.
- Fonts: Cormorant Garamond or Playfair Display for headings; Plus Jakarta Sans or Inter for body; a wide-tracked uppercase light sans for small labels, nav and buttons (like the "APARTMENTS" wordmark).
- Motifs from the logo: **arched doorway** (use `rounded-t-full` as the crop for hero, gallery and minibar images) and **soft layered wave dividers** in the pale blues between sections.
- Logo: use the supplied file for now. Ask for a transparent SVG, a horizontal lockup and a door-only icon (favicon, email header, loading mark).
- Accessibility: WCAG AA contrast, visible focus states, mobile-first (most guests will book on a phone).

## 5. Pages
**Public:** Home (hero, highlights, availability bar, gallery), Minibar (search + filters by category and dietary tag, variants, stock badges), a **Laundry & Pressing** section directly below the minibar using the same arched cards (per piece / per bag, speed variants, no stock badges), Getting Here (see section 7), Booking flow, Booking confirmation, Manage Booking (reference + email lookup, re-upload slip, and during the stay a "Request laundry pickup" form with a payment choice), House Rules, Contact.
**Admin (`/admin`, role-gated middleware):** Dashboard, Bookings queue with slip preview (zoom/pan) and expected-vs-paid side by side, Calendar and blocked dates, Pricing and seasonal rates, Settings (FX rate, TGST %, Green Tax, hold hours, bank details, location, getting-here text), Add-ons catalog CRUD (minibar + laundry), Laundry orders (payment status, slips), Users and roles.
**Housekeeping:** reads only the `housekeeping_schedule` view (minibar restock checklist) and the `housekeeping_laundry_queue` view (a Laundry Queue tab with Requested / Picked Up / In Progress / Delivered columns), and calls `adjust_stock()` and `set_laundry_status()`. They never see money, slips, ID notes or contact details.

## 6. Booking flow
1. Dates + guests (+ local-resident toggle with ID type and number) + optional minibar items. Live price from `quote_booking()`; show USD/MVR toggle and the breakdown: nightly rate x nights + minibar + TGST + Green Tax = total.
2. Guest details: name, email, phone/WhatsApp, passport/ID note, special requests, **arrival details** (ferry / speedboat / flight and expected time).
3. Server Action calls `create_booking()`, returns the reference (`TA-YYYY-NNNN`), shows bank instructions and sends the "Booking Created" email.
4. Slip upload: server issues a **signed upload URL** to the private `payment-slips` bucket; the client compresses first (receipts: WebP/JPEG ~70% quality, max 1200px; photos: WebP 85-90%, max 1600px, under ~250 KB) and uploads; then the server calls `register_slip_upload()`.
5. Staff review in admin: approve or reject via `change_booking_status()`; send the matching Resend email (Approved voucher, or Rejected with reason and re-upload link).
6. Holds expire automatically through the `pg_cron` job; show the guest a countdown on pending bookings.

Rate-limit the public Server Actions (booking creation, slip upload, lookup) and verify booking reference + email on every guest lookup.

## 7. Getting Here (placeholder)
Render `apartment_settings.getting_here` (editable in admin) plus a map link built from latitude/longitude. **[TO CONFIRM WITH OWNER: usual route to Sh. Goidhoo, travel times, who arranges pickup, days when transfers don't run.]** Do not invent transport information. Show a short "Contact us to arrange your transfer" fallback until the text is filled in.

## 8. Emails (Resend)
Laundry Request Received (with payment instructions if a slip is needed), Booking Created, Slip Received (verification usually within 1-2 hours; confirm this with the owner), Booking Approved (voucher, check-in info, Wi-Fi, getting-here summary, minibar pre-order), Slip Rejected (specific reason + re-upload link). Branded layout using the burgundy/pale-blue palette.

## 9. Environment variables
`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` (server only), `RESEND_API_KEY`, `NEXT_PUBLIC_SITE_URL`. Provide `.env.example`; never commit secrets.

## 10. Phases and acceptance criteria
1. **Foundation:** project, theme tokens, Supabase clients (anon / service), auth, role middleware, generated DB types. *Done when:* a seeded Super Admin logs in and each role sees only its own routes.
2. **Admin backend:** settings editor, image uploader with compression, add-ons catalog CRUD (minibar and laundry types), blocked dates, seasonal rates, user management. *Done when:* a Front Desk user cannot change tax, pricing or users (verified by RLS, not just by hiding the UI).
3. **Public storefront:** landing, gallery, calendar, minibar, currency toggle, live quote, Getting Here. *Done when:* the calendar never offers a taken night and prices match `quote_booking()` exactly.
4. **Booking, receipts, laundry, emails, analytics:** full flow, in-stay laundry requests and queue, admin verification modal, Resend, KPI dashboard (revenue USD/MVR, occupancy, pending verifications, laundry revenue). *Done when:* two simultaneous bookings for the same dates result in exactly one success.

## 11. Working agreements
- Ask before adding dependencies beyond section 2.
- No `localStorage` for anything security-relevant. Validate all inputs with Zod, server-side.
- Write tests for the booking Server Actions and the tax / local-resident logic.
- Tax rates are admin settings, not constants. The owner will confirm current TGST, Green Tax (including the under-2 rule) and any service charge with MIRA before launch.
