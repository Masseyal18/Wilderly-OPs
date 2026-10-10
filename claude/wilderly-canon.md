# Wilderly Canon — detail, changelog, stale assets

Companion to `CLAUDE.md` (the locked rules). This file holds the *why* and *history* behind those rules, a running changelog of what changed and when, and a live list of assets that still contradict the locked rules and need fixing. `CLAUDE.md` always wins on anything this file seems to disagree with — update both together, never just one.

## Live system inventory (as of 2026-10-04 launch-check)

**Supabase project:** `fdgectlpkemacunqoxzt` (`wilderly`, us-east-1, Postgres 17.6).

Tables: `profiles`, `listings`, `availability`, `blocked_dates`, `bookings`, `reviews`, `wishlists`, `wishlist_items`, `trips`, `waitlist`, `markets`, `market_property_assessments`, `host_interviews`, `guest_trip_requests`, `guest_trip_matches`, `market_score_snapshots`, `notification_outbox`, `listing_ical_feeds`, `referral_codes`, `referrals`, `credit_ledger`. 34 migrations applied, latest `offer_c_founding_host_fee_and_10pct_guest_fee` (2026-10-02).

Edge functions (all ACTIVE as of 2026-10-04): `search-listings`, `booking-flow` (v4), `stripe-webhook` (v3), `stripe-connect`, `notify-worker` (v2), `ical`.

Cron jobs: `wilderly-notify-worker` (*/5 min), `wilderly-complete-stays` (hourly :15), `wilderly-ical-sync` (hourly :40) — all green, 0 failures in the trailing 7 days as of 2026-10-04.

Key money-flow functions, read directly from the live DB:
- `private.host_fee_cents()` — Offer C logic: 0% for 12 months from `listings.activated_at` on a founding listing, **no revenue cap**, else 3% (or `commission_override_pct` if set).
- `public.quote_guest_savings()` — caps the $35 invite discount + travel credit at Wilderly's own platform fee on that booking, so savings never reduce `host_payout_cents`.
- `public.clawback_referral_for_booking()` — posts a one-time `-2500` cent ledger entry and voids the referral on chargeback.
- `public.approve_listing(id, founding)` — admin-only (`private.is_privileged()`); grants founding status up to 50 listings, then silently falls through to non-founding rather than raising an error. (Flagged as a mismatch vs. the `wilderly-launch-check` skill's text, which says it "refuses" — see stale assets below.)

## Changelog

- **2026-10-02** — Offer C locked: 10% guest service fee (down from an earlier value), 0% Founding Host fee for 12 months with no revenue cap (replacing an earlier "first $5,000" rule), max 50 founding listings. Migration `offer_c_founding_host_fee_and_10pct_guest_fee` applied.
- **2026-10-04** — First `wilderly-launch-check` run. Verdict: No-go. Found: 0 admin profiles (fixed — Alyssa's profile promoted to admin), leaked-password protection still off, RLS performance warnings unchanged since 2026-10-01 (32 `auth_rls_initplan`, 134 `multiple_permissive_policies`), no Stripe test-mode key available in-session (money flow verified by code review only, not a live transaction). Full report: `claude/launch-check-2026-10-04.md`.
- **2026-10-05** — `wilderly-ops` repo created to hold this canon, `CLAUDE.md`, the agent definition, and launch-check reports going forward, since none of the three previously existed as actual files.

## Stale assets — fix or retire

| Asset | What's wrong | Status |
|---|---|---|
| `wilderly-launch-check` skill, "Founding fee" test row | Still describes the retired "$5,000 cap" rule instead of Offer C (0% / 12 months / no cap) | Open — it's a synced skill, can't be edited directly from a session; needs a fix at its source |
| `booking-flow` edge function, top-of-file comment | Says "guest service fee (listing.service_fee_pct, 12%)" — the code correctly reads the live 10% column, only the comment is stale | Open — one-line fix, low priority since it doesn't affect behavior |
| `Wilderly_Host_Referral_Program_Terms.md` | Describes a host revenue-share referral program that is **not offered** per CLAUDE.md §3 | Shelved per CLAUDE.md — do not resurrect without an explicit decision |
| Old `wilderly.com` references in `WildHaven-` and `Wilderly-Android-App` | Both repos had `wilderly.com`/placeholder emails hardcoded instead of the real `wilderlystays.com` | Fixed 2026-09-27 session (`.env.example` defaults, `server.ts`, `HostDashboard.tsx`, `PackLeader.tsx`, `TripsManager.tsx`, Android `.env.example` + `MainViewModel.kt`) |
| Sept 12–13 host promises | Those hosts were told "0% for 6 months" host fee and a 13% guest fee — both retired numbers | Covered by current terms per CLAUDE.md §3 ("no per-listing override is needed") — not a code fix, just a fact to remember when talking to those specific hosts |

## Known gaps not yet resolved (carried from the 2026-10-04 launch check)

- No Stripe test-mode access confirmed anywhere — money flow has never been live-tested end to end, only verified by reading the deployed code.
- `wilderly-o7.vercel.app` site publish state (`/features.js` 200 vs. 404) unconfirmed — last known report (2026-10-01) was 404.
- **Lodging-tax collection unbuilt for all 5 launch markets (ID/MT/OR/WA/WY), not just ID/MT/WY/OR** — `bookings` has no tax column and `booking-flow` has no tax line. Re-verified 2026-10-10 against live state sources (Oregon DOR via HB 4134, Washington DOR marketplace-facilitator page): Wilderly is the legally required collector in OR, MT, WY and ID by statute, and almost certainly in WA too once the facilitator threshold is hit (the WA lodging-type hotel exclusion specifically does *not* cover "home, apartment, cabin, or other residential dwelling" — i.e. almost everything Wilderly lists). **All 32 current host prospects (WA 10, OR 8, MT 7, ID 4, WY 4) sit in states where Wilderly cannot legally take a real paid booking yet.** This is the single blocker that gates every other market from going live, not a per-state concern. See `wilderly-tax-compliance` skill for the full table and the build spec.
- Attorney review of the waiver/terms has not happened.
- 0 of 32 `market_property_assessments` rows are marked `is_qualified` as of 2026-10-10 — worth checking whether that's a real gap in the qualification pass or the flag just isn't being set.
