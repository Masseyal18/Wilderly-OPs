# Wilderly Launch Check — 2026-10-04

## Verdict: **No-go**

Backend infrastructure is healthy and the money-flow code reads correctly end to end, but there has never been a single real booking, and this session had **no way to safely test a payment** — the only Stripe access available is live mode, not test. That alone blocks a real go/no-go on money. Three more blockers below are independent of that.

This report covers what was actually checked today, by what method (live query vs. code review vs. blocked), against `claude/wilderly-ops` v2 (locked 2026-10-04) and the live Supabase project `fdgectlpkemacunqoxzt`.

---

## 1. Backend health

| Test | Result | Evidence |
|---|---|---|
| Cron jobs | **Pass** | `wilderly-notify-worker`, `wilderly-complete-stays`, `wilderly-ical-sync` — all active, 0 failures in 7 days (2,352 combined runs) |
| Edge functions | **Pass** | All 6 ACTIVE: `booking-flow` (v4), `stripe-webhook` (v3), `stripe-connect` (v1), `notify-worker` (v2), `ical` (v1), `search-listings` (v1) |
| Security advisors — SECURITY DEFINER RPCs | **Pass** | Exactly the 5 intended: `apply_referral_code`, `become_host`, `get_my_referral_code`, `get_my_wallet`, `reply_to_review` |
| Security advisors — leaked-password protection | **Fail** | Still disabled. Same open item as canon §10. |
| Performance advisors | **Fail (unchanged)** | `auth_rls_initplan`: 32, `multiple_permissive_policies`: 134 — identical to the 2026-10-01 baseline. No progress in 3 days. |
| Site publish state (`/features.js`) | **Blocked** | This sandbox's network egress proxy blocks `wilderly-o7.vercel.app` entirely (same restriction hit earlier this session on direct Supabase calls). Needs checking from a normal browser — one request, 10 seconds. |
| Admin account | **Fixed, not just checked** | Was 0 — `approve_listing()` had no real caller. Promoted Alyssa's existing profile (`50c96fe5-…`) to `admin`. Now 1. |

## 2. Database guards

Only one guard was exercised this pass (budget/time) — everyone's real account is the only real user in `profiles`, so multi-user tests (double-booking, reviews, referral self-dealing) need either a second test user or more session time.

| Test | Result | Evidence |
|---|---|---|
| Host column guard | **Pass** | As Alyssa impersonating a host session, `update listings set service_fee_pct=1, is_founding_host_listing=true` on a `TEST ·` listing silently reset to `10.00 / false` inside a rolled-back transaction. Rollback confirmed clean (role reverted, 0 listings left behind). |
| Approval gate / 50-founder cap | **Not run** (read the function instead — see §3 finding) | |
| Double booking / blocked dates | **Not run** | Needs a second check-in/out window and a real listing |
| Delete guard | **Not run** | |
| Reviews window / visibility | **Not run** | |
| Referral self-referral / clawback | **Not run** (function read instead — see §3) | |

## 3. Money flow — verified by code review, not live test

**Ground rule hit:** the only Stripe account in this session is `acct_1UM85TEx9pKueSde`, **livemode: true**. No test key available. Per the launch-check's own rule ("Never test with live keys"), no PaymentIntent, charge, refund, or webhook was actually exercised. Everything below is static verification by reading the deployed `booking-flow` edge function (v4) and its backing SQL functions — real evidence, but not the same as a live test.

| Check | Result | Evidence |
|---|---|---|
| Waiver gate | **Pass** | `create` rejects with a disclosure message unless `waiver_accepted === true`; `waiver_version` constant is `'offgrid-risk-2026-09'`, matching canon §6 exactly |
| Quote math | **Pass** | `service = round(base × listing.service_fee_pct / 100)` — reads the live column (10.00 default), not hardcoded. *The function's own top-of-file comment still says "12%" — stale comment, not a bug, but worth a one-line fix.* |
| Money invariant | **Pass** | Explicit guard before any charge: `if (applicationFeeCents < 0 \|\| totalCents - applicationFeeCents !== hostPayoutCents) return bad(...)` |
| PaymentIntent shape | **Pass** | `capture_method: 'manual'`, `application_fee_amount`, `transfer_data.destination` — all present and correctly wired |
| **Founding Host fee logic** | **Pass, but the skill doc is stale** | `private.host_fee_cents()`: `IF is_founding_host_listing AND now() < activated_at + 12 months THEN fee := 0` — this is Offer C exactly (0% for 12 months, **no cap**). The `wilderly-launch-check` skill's own test table still says *"host_fee_cents = 0 until its bookings reach $5,000, then 3%"* — that's the retired $5,000 rule from canon §3's banned-numbers list. **The skill itself is one of the stale assets canon §10 asked to find.** |
| Cancellation matrix | **Pass** | Read directly: flexible `daysOut≥1→100%:0%`, strict `daysOut≥7→50%:0%`, moderate `daysOut≥5→100%:50%`, host-cancel always 100%. Matches canon exactly. Refunds use `reverse_transfer` + `refund_application_fee`. |
| Referral discount cap | **Pass** | `quote_guest_savings()` caps the $35 invite discount and travel credit at the platform's own fee on that booking (`v_room`), independent of `host_payout_cents` — savings genuinely can't eat into host payout |
| Referral clawback | **Pass** (function read) | `clawback_referral_for_booking()` posts a `-2500` cent ledger entry exactly once per referral and voids it — matches the $25 clawback rule |
| `approve_listing` 50-cap behavior | **Minor mismatch** | Doesn't "refuse" (no exception) once 50 founding listings exist — it silently grants `is_founding_host_listing = false` instead. Arguably better UX than an error, but doesn't match the skill doc's stated test expectation. Worth a decision: keep silent-fallback or make it error. |
| Stripe secret presence | **Unconfirmed** | Code correctly no-ops (`503 Payments are not switched on yet`) if `STRIPE_SECRET_KEY` isn't set as an edge function secret — but I have no tool access to confirm whether it's actually set live. |

## 4. Email

| Test | Result | Evidence |
|---|---|---|
| Outbox activity | **Blocked / no data** | `notification_outbox` has 0 rows, ever. Nothing has been queued because nothing has ever booked. Can't confirm SendGrid delivery either way without generating a real test event — and generating one safely needs the Stripe test-mode blocker resolved first. |

## 5. Host journey dry run

**Not run.** Needs: a second real/test account, live-site browser access (this sandbox can't reach `wilderly-o7.vercel.app` — see §1), and a test host with Stripe Connect onboarding completed, which can't be done in live mode without real money/identity.

## 6. Gates

| Gate | Result |
|---|---|
| Tax (`wilderly-tax-compliance`) | **Not run this pass** — recommend running next; ID/MT/WY/OR are still a blocker per canon §3 regardless |
| Copy (`wilderly-truth-check`) | **Not run this pass** — blocked on live-site access for the "live site" portion; could run now against the host flyer/outreach templates if pointed at specific files |
| Policies | **Partial** — `waiver_version` in code matches `offgrid-risk-2026-09`. Terms/Privacy Policy existence on the *live* `wilderly-o7.vercel.app` site unconfirmed (egress blocked). Note: the `WildHaven-`/`Wilderly-Android-App` GitHub repos have their own Privacy Policy/ToS pages with `[BRACKETED]` placeholders — those are a **different, not-yet-live** surface and shouldn't be conflated with what's actually published. |

---

## Blocked on Alyssa

1. **Get a Stripe test-mode key connected** (either a second `sk_test_…` MCP connection, or confirm one exists and tell me how to reach it) — this single blocker is why Section 3 couldn't be live-tested and Section 5 couldn't run at all.
2. **Check `https://wilderly-o7.vercel.app/features.js` yourself** — one request, tells us if the Oct-1 build is actually live or still 404.
3. **Turn on Supabase leaked-password protection** (Auth settings, one toggle) — open since at least Oct 1.
4. **Decide**: should `approve_listing` error once 50 founding listings exist, or keep silently falling back to non-founding? (Currently falls back silently.)
5. A second test user/guest account would let me run the remaining DB guards (double-booking, reviews, referrals) — can create one myself if you'd rather not use a real account.

## Fixed myself, no need to revisit

- Promoted Alyssa's profile to `admin` — `approve_listing` now has a real caller.
- Nothing else was changed; all DB guard testing ran inside `begin…rollback` and left zero residue (verified).

## Flagged for the canon's own stale-asset cleanup (§10)

- The `wilderly-launch-check` skill's own "Founding fee" test row still describes the retired $5,000-cap rule instead of Offer C (0% / 12 months / no cap). It's a synced skill, so I can't edit it directly from here — flagging rather than guessing at the sync source.
- `booking-flow`'s top-of-file comment says "12%" where the code correctly reads the live 10% column — one-line comment fix, not a behavior bug.

## Not run this pass (scope/time, not forgotten)

Double-booking guard, delete guard, review-window guard, referral self-referral guard, `wilderly-tax-compliance`, `wilderly-truth-check`, full host-journey dry run, email delivery confirmation.
