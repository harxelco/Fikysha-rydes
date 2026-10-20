# Fikysha Rydes — Product Requirements (Android beta)

## Product
**Fikysha Rydes** is an Android marketplace for moving goods in Nairobi, like Uber or Bolt for pickups, trucks and moving companies. Customers post a transport job (house move, construction materials, furniture, shop stock), receive quotes from nearby drivers, accept one, pay (simulated, held until delivery), and follow the job to completion. Launch area: Githurai, Kahawa Sukari, Kahawa West, Kahawa Wendani, Ruiru, Juja, Roysambu, Kasarani, Zimmerman, Membley, Mwiki.

This is a **functional beta** that one person tests alone on a Samsung phone. Everything must behave like the real world, even though money and some people are simulated. The data model must be ready for real customers and drivers later.

## Tech stack
- Kotlin, Jetpack Compose, Material 3, single-activity, Navigation Compose
- MVVM with repositories, Hilt for DI, Kotlin Coroutines and Flow, kotlinx.serialization
- Supabase via **supabase-kt** (Auth, Postgrest, Realtime, Storage). Check the latest stable version and the official setup docs before adding dependencies. Do not guess APIs.
- Coil for images, DataStore for session and last-used mode
- Supabase backend: SQL migrations in `supabase/migrations` are the single source of truth. Never invent columns or change the schema outside a migration.
- Tests: JUnit for business logic (pricing, status machine, vehicle recommendation), Compose UI tests for key flows

## Brand and design
- Name: **Fikysha Rydes**. Friendly, trustworthy, local. Build a simple wordmark and app icon placeholder.
- Palette direction: deep green primary, amber accent, warm off-white background, dark mode supported. Rounded cards, large tap targets, clear status chips. Optimise for a 6.1 to 6.8 inch Samsung screen.
- Currency format: `KSh 3,500`. Times in East Africa Time (EAT). Put all strings in `strings.xml` so Swahili can be added later.
- Every empty state, loading state and error state needs a designed screen, not a blank page.

## One app, three modes
One account can use several modes. A mode switcher lives in the profile menu and the last mode is remembered.
- **Customer** (default for everyone)
- **Driver** (user completes a driver profile, then waits for admin approval)
- **Admin** (only if `profiles.is_admin = true`, set manually in SQL)

### Demo mode
A "Demo mode" section on the welcome screen (visible only when `BuildConfig.DEMO_MODE` is true) gives one-tap login for **Demo Customer**, **Demo Driver**, **Demo Admin**. These are real Supabase auth users created by the seed script.

## Authentication
- Phone number sign-up and login with a **fake OTP: always 123456** (beta only).
- Supabase phone auth needs an SMS provider, so implement it as follows: normalise the phone to `2547XXXXXXXX`, verify the OTP locally against a constant behind a `DEMO_OTP_ENABLED` flag, then sign up or sign in with Supabase email/password using a synthetic email `{phone}@phone.fikysha.app` and a password derived from a server-side secret via an Edge Function (`demo-auth`). Isolate this in an `AuthRepository` so real SMS OTP can replace it later without touching the UI.
- Profile: name, phone, profile photo, default mode. Sign out. Session persists across app restarts.

## Customer features
1. **Service type picker**: House move, Construction materials, Furniture, Business/shop stock, Other.
2. **Job details**: pickup area and drop-off area (dropdown of seeded areas, map later), exact address note, date and time, description of what is being moved, rough size options that change per service type (House move: bedsitter, 1-bedroom, 2-bedroom, 3-bedroom+; Construction: 5 bags cement, 20 bags, 50 bags, 100+ bags, tiles pallet, ballast/sand load; Furniture: single item, few items, full room; Shop stock: small, medium, large load; Other: small, medium, large), up to 5 photos, "Need loaders?" yes/no, floors/stairs (ground, 1 to 5+, lift available).
3. **Suggested vehicle**: the app recommends Boda, Pickup, Small truck (3 to 5 ton) or Large truck (7 ton+) from service type plus size, with a plain-language reason. The customer can override.
4. **Instant price estimate**: show a range (for example KSh 3,200 to 4,100) before posting, calculated from the pricing rules.
5. **Quotes**: after posting, matched drivers send quotes. The customer sees a live list (Realtime) comparing price, driver rating, vehicle, plate, ETA, and verified or demo chip, then accepts one. Accepting declines the others.
6. **Fake M-Pesa payment**: after accepting, a simulated STK push screen (enter phone, "check your phone and enter PIN" with a 3 to 4 second wait and a confirm button). The result is a payment record with status **held**. Clearly labelled "Demo payment, no real money moves".
7. **Booking status tracker**: Requested → Quoted → Accepted → Driver on the way → Loading → In transit → Delivered → Paid. Show a timeline with timestamps and live updates.
8. **Confirm delivery**: when the driver marks Delivered, the customer taps Confirm (or it auto-confirms after 10 minutes via cron). Confirmation releases payment and moves the job to Paid.
9. **Rating and review** after Paid (1 to 5 stars, comment).
10. **Job history** with "Book again" (prefills a new job), cancel (allowed before Loading), and **report a problem** (creates a dispute).
11. **Call driver** button (phone dialler) and **WhatsApp driver** button with a prepared message. No in-app chat.
12. In-app notifications screen with unread badge.

## Driver features
1. **Driver profile and onboarding**: vehicle type, make/model, capacity, plate, photos (vehicle, plate, driver), areas served (multi-select), services offered (multi-select), short description. Submitting sets status `pending`.
2. **Verification status**: pending, approved (shows a Verified badge), rejected (shows the admin's reason and allows resubmission).
3. **Online/Offline toggle**: only approved and online drivers are matched.
4. **Job feed**: jobs matched to this driver by vehicle type, services and area, newest first, with the suggested price and details.
5. **Send quote**: accept the suggested price in one tap, or counter with their own price and ETA. A driver can edit or withdraw a quote until it is accepted.
6. **Active job screen** with step buttons that unlock in order: Start trip (driver on the way) → Arrived and loading → Picked up, in transit → Delivered. Include customer name, call and WhatsApp buttons, addresses and notes.
7. **Earnings screen**: completed jobs, gross amount, commission deducted, net, payout status (pending, paid out; simulated).
8. **Ratings and reviews** received, average rating.
9. **Performance**: listing views, quotes sent, quotes accepted, acceptance rate.

## Admin features
1. **Dashboard**: jobs today, active jobs, completed jobs, revenue (commission), gross volume, top drivers, drivers pending approval.
2. **Driver approvals**: approve or reject with reason, view documents and photos.
3. **All jobs (live)**: filter by status, open any job, view its status event history, cancel a job, move it forward manually.
4. **Pricing rules**: edit base fee, per-km, loaders fee, per-floor fee, minimum fare and surge multiplier per vehicle type. Edit commission % (default 10%).
5. **Disputes**: list, open, resolve (refund customer or release to driver), add notes.
6. **Simulate** tools (see Automation).
7. **Reset demo data**: restore the seeded state without deleting real users.
8. **Activity log**: timestamped record of important actions.

## Automation (implement in Postgres first, Edge Functions only where needed)
Use Postgres functions (RPC, `security definer`, with strict role checks), triggers and `pg_cron`. Every automation must be idempotent and logged to `job_status_events` or `activity_log`.
- **create_job RPC**: validates input, calculates the estimate, stores the job, runs auto-matching, writes notifications.
- **Auto-matching**: pick up to 5 drivers who are approved, online, serve the pickup area, offer the service, and have the recommended vehicle type. Order by rating then proximity of area. Store in `job_matches` with `wave = 1` and notify them.
- **Quote timeout (cron, every minute)**: if a job is still `requested` with no quotes 5 minutes after the current wave, create wave 2: widen to neighbouring areas and the next vehicle size up, notify the new drivers, and notify the customer ("Looking wider for drivers"). After wave 3 with no quotes, notify the customer to adjust the job.
- **Auto price estimate**: `estimate_price(vehicle_type, pickup_area, dropoff_area, loaders, floors, service_type)` = base + per_km × distance + loaders + floor charges, multiplied by surge, floored at minimum. Distance comes from area coordinates (haversine × 1.3 road factor), clearly marked approximate. Return low and high (about ±12%).
- **Status machine**: one `advance_job_status(job_id, new_status)` RPC enforces the allowed transitions and who may trigger them (driver or customer or system), writes a `job_status_events` row, and creates notifications for both sides. Illegal transitions must fail with a clear error.
- **Payment**: `pay_for_job` creates a `held` payment. `confirm_delivery` (customer or cron after 10 minutes) releases it, calculates commission using the commission % at that time, stores gross, commission and net, and sets the job to `paid`. Disputes freeze release.
- **Reminders (cron)**: upcoming job in 1 hour (customer and driver), rate your driver 1 hour after Paid, driver offline for 30 minutes while having a pending matched job.
- **Simulate tools (admin only)**:
  - *Simulate job*: create a random job as a random demo customer, auto-match, and insert quotes from 2 to 4 matched demo drivers within ±15% of the estimate.
  - *Simulate next step*: advance a chosen job one step as the driver would.
  - *Simulate full job*: run a job to Paid.
  - *Simulate customer*: create a job for a real driver account so you can test the driver feed alone.
  - Everything created is flagged `is_demo = true`.

## Data model
Core tables: `profiles`, `drivers`, `vehicles`, `service_types`, `jobs`, `quotes`, `job_status_events`, `payments`, `reviews`, `notifications`, `pricing_rules`.
Supporting tables: `areas` (name, lat, lng, neighbours), `job_matches`, `disputes`, `app_settings` (commission %, quote timeout minutes, auto-confirm minutes), `activity_log`, `saved_drivers` (optional).
- Every table has `id uuid`, `created_at`, `updated_at` where appropriate, and `is_demo boolean default false` on user-facing records.
- Enums for job status, quote status, payment status, vehicle type, driver verification status.
- Photos live in Supabase Storage buckets `vehicle-photos`, `job-photos`, `avatars`, with storage policies matching table ownership. Compress images on the device before upload.

## Security (Row Level Security on every table)
- Customers read and write only their own jobs, quotes on their jobs (read), payments, notifications and reviews they wrote.
- Drivers read jobs they are matched to, create and edit only their own quotes, read only their own earnings and notifications, and edit only their own driver profile and vehicles.
- Customers can read approved drivers' public fields only (name, photo, rating, vehicle, plate, verified status), never phone numbers until a quote is accepted.
- Admin policies depend on `profiles.is_admin`. Status, payment and commission changes happen only through RPCs, never by direct table updates from the app.
- Never ship the service-role key in the app.

## Seed data (script, repeatable)
Write `supabase/seed/seed.ts` (run locally with the service-role key) that creates auth users and data:
- 25 demo drivers across the areas, mixed vehicle types, ratings 3.6 to 5.0, services, areas, plate numbers in Kenyan format, and a few pending or rejected ones for admin testing
- 10 demo customers
- 30 past jobs in varied statuses with quotes, payments, reviews, and status events
- Pricing rules for all four vehicle types, service types, areas with approximate coordinates, app settings
- Demo Customer, Demo Driver, Demo Admin accounts with known demo credentials
- All seeded records have `is_demo = true` and show a **DEMO** chip in the UI. Never present demo ratings or verification as real.
- Seeding is idempotent, and "Reset demo data" removes and recreates only `is_demo` records.

## Out of scope for this beta
Live GPS tracking, real M-Pesa, in-app chat, insurance, B2B accounts and invoicing, multi-city, referrals, maps (use area dropdowns), push notifications (in-app first).

## Definition of done
- A release-quality debug APK runs on a Samsung phone.
- Every journey below works end to end with no hardcoded fake responses in the UI. All state lives in Supabase.
- No crashes on rotation, background/resume, airplane mode, or a cold start with an expired session.
