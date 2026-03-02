# BurgerBuddy MVP Execution Blueprint (FlutterFlow-Ready)

## 1) Final Data Model (tables, fields, relationships)

### 1.1 Core design choices (with options)

- **Choice: Database backend for FlutterFlow**
  - Option A: **Firebase Firestore** (Recommended default)
    - Pros: Native FlutterFlow support, easy auth/storage, fast MVP setup.
    - Cons: Aggregations require Cloud Functions/scheduled jobs.
  - Option B: Supabase (Postgres)
    - Pros: SQL constraints and analytics easier.
    - Cons: Slightly more setup complexity in FlutterFlow.

- **Choice: Venue/city consistency enforcement**
  - Option A: free text only
    - Pros: Fastest to build.
    - Cons: Weak integrity (NOT recommended).
  - Option B: **canonical tables + user text input mapped to canonical records** (Recommended default)
    - Pros: Keeps discovery and averages clean.
    - Cons: Needs normalization workflow.

### 1.2 Tables

#### A) `users`
- `id` (string/uuid, PK, required)
- `email` (string, required, unique)
- `callsign` (string, required, unique, 3–20 chars)
- `home_city_id` (fk -> `cities.id`, required)
- `xp_total` (int, required, default 0)
- `rank_code` (enum, required, computed+stored)
- `last_audit_date` (datetime, nullable)
- `is_active` (bool, computed+stored)
- `profile_photo_url` (string, nullable)
- `role` (enum: `user|admin`, required, default `user`)
- `status` (enum: `active|blocked`, required, default `active`)
- `callsign_changed_at` (datetime, nullable)
- `created_at` (datetime, required)
- `updated_at` (datetime, required)

#### B) `cities`
- `id` (string/uuid, PK)
- `name` (string, required, unique)
- `country_code` (string, required, default `NL`)
- `is_seed_city` (bool, required; true for Alkmaar/Schagen)
- `is_active` (bool, required)

#### C) `venues`
- `id` (string/uuid, PK)
- `canonical_name` (string, required)
- `city_id` (fk -> `cities.id`, required)
- `normalized_key` (string, required, unique per city)
- `address_line` (string, nullable)
- `lat` (number, nullable)
- `lng` (number, nullable)
- `created_by_user_id` (fk -> `users.id`, nullable)
- `is_verified` (bool, default false)
- `created_at`, `updated_at`

#### D) `burgers`
- `id` (string/uuid, PK)
- `burger_name` (string, required)
- `normalized_key` (string, required)
- `venue_id` (fk -> `venues.id`, required)
- `city_id` (fk -> `cities.id`, required)
- `is_active` (bool, required, default true)
- `created_at`, `updated_at`
- Unique composite index: (`normalized_key`, `venue_id`)

#### E) `audits`
- `id` (string/uuid, PK)
- `user_id` (fk -> `users.id`, required)
- `burger_id` (fk -> `burgers.id`, required)
- `venue_id` (fk -> `venues.id`, required)
- `city_id` (fk -> `cities.id`, required)
- `photo_url` (string, required)
- `burger_name_snapshot` (string, required)
- `venue_name_snapshot` (string, required)
- `quick_score` (decimal(2,1), required, 1.0–5.0)
- `patty_score` (decimal(2,1), nullable, 0.5–5.0)
- `bun_score` (decimal(2,1), nullable, 0.5–5.0)
- `balance_score` (decimal(2,1), nullable, 0.5–5.0)
- `value_score` (decimal(2,1), nullable, 0.5–5.0)
- `has_deep_scores` (bool, computed)
- `buddy_score` (decimal(3,2), computed+stored)
- `short_note` (string, nullable, max 120 chars)
- `xp_awarded` (int, required)
- `is_hidden` (bool, default false)
- `moderation_status` (enum: `visible|pending|removed`, required)
- `report_count_open` (int, required, default 0)
- `created_at`, `updated_at`

#### F) `reports`
- `id` (string/uuid, PK)
- `audit_id` (fk -> `audits.id`, required)
- `reporter_user_id` (fk -> `users.id`, required)
- `reason` (enum: `fake_photo|spam|abusive_content|other`, required)
- `details` (string, nullable, max 300)
- `status` (enum: `open|resolved_restore|resolved_remove`, required)
- `created_at`, `resolved_at`, `resolved_by_admin_id` (fk -> `users.id`, nullable)
- Unique index: (`audit_id`, `reporter_user_id`) (one report per user per audit)

#### G) `xp_ledger`
- `id` (string/uuid, PK)
- `user_id` (fk -> `users.id`, required)
- `audit_id` (fk -> `audits.id`, required)
- `xp_amount` (int, required)
- `reason` (enum: `quick_audit|deep_audit|cap_reached`)
- `window_start_utc` (datetime, required)
- `created_at` (datetime, required)

#### H) `daily_city_stats` (materialized for fast discovery)
- `id` (string/uuid)
- `city_id` (fk)
- `date_utc` (date)
- `audit_count` (int)
- `avg_buddy_score` (decimal)
- unique: (`city_id`, `date_utc`)

#### I) `burger_monthly_stats`
- `id`
- `burger_id` (fk)
- `city_id` (fk)
- `year_month` (string `YYYY-MM`)
- `audit_count_30d` (int)
- `audit_count_month` (int)
- `avg_buddy_score_month` (decimal)
- `city_avg_buddy_score_month` (decimal)
- `above_city_avg_delta` (decimal)
- unique: (`burger_id`, `city_id`, `year_month`)

### 1.3 Relationships checklist
- [ ] `users.home_city_id -> cities.id`
- [ ] `venues.city_id -> cities.id`
- [ ] `burgers.venue_id -> venues.id`
- [ ] `audits.user_id -> users.id`
- [ ] `audits.burger_id -> burgers.id`
- [ ] `reports.audit_id -> audits.id`
- [ ] `xp_ledger.audit_id -> audits.id`

---

## 2) Screen-by-screen user flows

### 2.1 Onboarding
1. Launch screen -> CTA: `Get Started`.
2. Sign up (email/password or Apple/Google).
3. Choose unique callsign (live availability check).
4. Choose home city (default list includes Alkmaar/Schagen first).
5. Optional profile photo.
6. Confirm community rules.
7. Land on Feed.

Checklist:
- [ ] Block completion if callsign not unique.
- [ ] Save role=`user`, xp=0, rank=`Recruit`.
- [ ] Show tooltip: “You become Off Duty after 60 days of no audits.”

### 2.2 Feed (Home)
1. Show mixed cards: recent visible audits (city-aware).
2. Card tap -> Audit detail.
3. FAB `+ Add Audit`.
4. Overflow menu on card -> Report.

### 2.3 Add Audit
1. Upload/take photo (required).
2. Enter burger name (required).
3. Select/create venue (required) + city (required).
4. Quick score slider (required).
5. Optional “Deep scores” toggle -> patty/bun/balance/value.
6. Optional short note (max 120).
7. Submit -> backend computes buddy score + XP (cap-aware) -> success screen.

### 2.4 Discovery
Tabs/sections (city filter pinned):
1. Trending (30d audits desc).
2. Top Rated This Month (min 3 audits).
3. Above City Average.
4. Recent.
Tap item -> burger detail with venue context.

### 2.5 Leaderboard
1. Select city.
2. Show only Active users by XP desc.
3. Display callsign, rank badge, XP.
4. No negative ranking language.

### 2.6 Profile
1. Show callsign, rank, XP, active/off duty state.
2. My audits grid/list.
3. Edit profile photo.
4. Callsign change action (policy-limited).

### 2.7 Report flow
1. User taps `Report` on audit.
2. Select reason + optional details.
3. Submit -> confirmation.
4. If report count reaches 3 unique users: audit auto set `pending` and hidden from public feeds.

### 2.8 Admin moderation
1. Queue screen: pending/flagged audits.
2. Open case -> view audit + reports.
3. Actions: Restore / Remove permanently / Block user.
4. System logs admin action.

---

## 3) UX copy (language keys)

### 3.1 Global
- `app.name`: "BurgerBuddy"
- `common.save`: "Save"
- `common.cancel`: "Cancel"
- `common.submit`: "Submit"

### 3.2 Onboarding
- `onboarding.welcome.title`: "Welcome to BurgerBuddy"
- `onboarding.welcome.subtitle`: "Track burgers. Level up your burger game."
- `onboarding.callsign.label`: "Choose your callsign"
- `onboarding.callsign.helper`: "3–20 characters, unique"
- `onboarding.city.label`: "Home city"
- `onboarding.rules.accept`: "I agree to the community rules"

### 3.3 Feed / Audit cards
- `feed.title`: "Home"
- `feed.empty`: "No audits yet. Be the first in your city."
- `audit.card.score`: "Buddy Score"
- `audit.card.report`: "Report"

### 3.4 Add audit
- `audit.add.title`: "Add Audit"
- `audit.add.photo.required`: "Photo is required"
- `audit.add.burger_name.label`: "Burger name"
- `audit.add.venue_name.label`: "Venue"
- `audit.add.city.label`: "City"
- `audit.add.quick_score.label`: "Quick score"
- `audit.add.deep.toggle`: "Add deep scores"
- `audit.add.note.label`: "Short note (optional)"
- `audit.add.note.helper`: "Max 120 characters"
- `audit.add.success`: "Audit posted"
- `audit.add.xp_cap_notice`: "You reached today’s XP cap. Audit posted with 0 XP."

### 3.5 Discovery
- `discovery.title`: "Discover"
- `discovery.trending`: "Trending"
- `discovery.top_rated_month`: "Top Rated This Month"
- `discovery.above_city_avg`: "Above City Average"
- `discovery.recent`: "Recent"
- `discovery.top_rated.rule`: "Minimum 3 audits this month"

### 3.6 Leaderboard
- `leaderboard.title`: "City Leaderboard"
- `leaderboard.filter.city`: "Select city"
- `leaderboard.off_duty_hidden`: "Only active users are shown"

### 3.7 Profile
- `profile.title`: "Profile"
- `profile.rank`: "Rank"
- `profile.xp_total`: "Total XP"
- `profile.status.active`: "Active"
- `profile.status.off_duty`: "Off Duty"
- `profile.callsign.change`: "Change callsign"

### 3.8 Reports & moderation
- `report.title`: "Report audit"
- `report.reason.fake_photo`: "Fake photo"
- `report.reason.spam`: "Spam"
- `report.reason.abusive_content`: "Abusive content"
- `report.reason.other`: "Other"
- `report.success`: "Thanks, your report was sent"
- `moderation.pending`: "Pending review"
- `moderation.removed`: "Removed"

---

## 4) Exact calculation specs

### 4.1 Buddy score
- If all deep fields present:
  - `buddy_score = round( patty*0.40 + bun*0.20 + balance*0.25 + value*0.15 , 2)`
- Else:
  - `buddy_score = quick_score`
- Validation: deep mode requires all 4 deep fields (no partial deep).

### 4.2 City average (monthly)
- Scope: visible audits only (`moderation_status=visible`) for city + current calendar month UTC.
- `city_average = avg(buddy_score)` across scoped audits.

### 4.3 Trending
- Scope: last 30 days, visible audits, grouped by burger in city.
- Primary sort: `audit_count_30d DESC`
- Tie-breakers: `avg_buddy_score_30d DESC`, then `most_recent_audit_at DESC`.

### 4.4 Top Rated This Month
- Eligibility: burger has `audit_count_month >= 3` in selected city.
- Ranking: `avg_buddy_score_month DESC`.
- Tie-breaker: higher `audit_count_month`, then recent audit.

### 4.5 Above City Average
- Eligibility: burger has `audit_count_month >= 3`.
- Compute: `delta = avg_buddy_score_month - city_average_month`.
- Show burgers where `delta > 0`, sorted by `delta DESC`.

### 4.6 XP and rank updates
- Base XP:
  - quick-only audit: +20
  - deep audit: +100 total
- Daily cap: max 200 XP per rolling 24h.
- Award algorithm per submit:
  1. `window_xp = sum(xp_ledger.xp_amount where user_id and created_at >= now-24h)`
  2. `remaining = max(0, 200 - window_xp)`
  3. `base = (has_deep_scores ? 100 : 20)`
  4. `xp_awarded = min(base, remaining)`
  5. write `xp_ledger`, update `users.xp_total += xp_awarded`
- Rank mapping from `xp_total`:
  - 0–199 Recruit
  - 200–599 Grill Starter
  - 600–1199 Smash Scout
  - 1200–2499 Patty Analyst
  - 2500–4999 Flavor Hunter
  - 5000+ Street Judge

### 4.7 Active / Off Duty
- `is_active = (last_audit_date >= now - 60 days)`
- Off Duty users excluded from leaderboard queries.
- Profile still shows stored rank + XP.
- Any new audit sets `last_audit_date=now` and reactivates.

---

## 5) Edge cases + operational rules

### 5.1 Duplicate burgers/venues
- Use normalized keys:
  - lowercased, trimmed, single-space, punctuation stripped.
- On submit, run exact normalized lookup in same city.
- If candidate match found, user must pick existing entry.
- If no match, create new as `is_verified=false` for admin review queue.

### 5.2 Venue naming normalization
- Mandatory canonical venue picker with autosuggest.
- New venue requires city assignment and normalized key generation.
- Admin tool: merge duplicate venues (repoint audits/burgers).

### 5.3 Callsign change policy
Options:
1. Free unlimited changes
   - Pros: user freedom
   - Cons: impersonation/confusion risk
2. **1 change every 90 days (Recommended default)**
   - Pros: balance of identity stability and flexibility
   - Cons: support tickets for exceptions
3. Paid change token (future monetization)
   - Pros: revenue
   - Cons: not MVP-friendly

### 5.4 Deletion requests (GDPR-ready)
- In-app “Delete account” request.
- Soft-delete immediately (login blocked).
- Hard-delete personal data within 30 days.
- Keep audits as anonymized records (`user_id` replaced with tombstone) OR fully delete by policy choice.
- Recommended default: anonymize audits to preserve city stats integrity.

### 5.5 Abuse prevention
- Rate limits:
  - max 10 audit posts/hour/user
  - max 5 reports/hour/user
- Photo requirement + MIME/size check.
- One report per user per audit.
- Auto-hide at 3 unique open reports.
- Repeat offenders: 3 removed audits in 30 days -> auto `status=blocked` pending admin confirmation.

---

## 6) Store readiness checklist (Apple + Google)

### 6.1 Product/legal essentials
- [ ] Privacy Policy URL (public, final)
- [ ] Terms of Use URL
- [ ] Community guidelines URL
- [ ] Contact/support email + web form
- [ ] Content moderation policy documented
- [ ] Data deletion process documented

### 6.2 Apple App Store (iOS)
- [ ] App name/subtitle
- [ ] Bundle ID + versioning
- [ ] App icon (1024x1024, no alpha)
- [ ] iPhone screenshots (6.7" and 6.1" minimum sets)
- [ ] App Privacy nutrition labels completed
- [ ] Sign in with Apple configured (if any social login)
- [ ] Age rating questionnaire completed
- [ ] Review notes + demo account for reviewer

### 6.3 Google Play (Android)
- [ ] App title + short/full description
- [ ] High-res icon (512x512)
- [ ] Feature graphic (1024x500)
- [ ] Phone screenshots (minimum 2)
- [ ] Data safety form completed
- [ ] Content rating questionnaire
- [ ] Privacy policy URL set
- [ ] Internal testing track with 10–20 testers

### 6.4 Assets you must prepare
- [ ] Brand logo + icon source files (SVG/PNG)
- [ ] 8–12 launch screenshots (real app data)
- [ ] 30-second promo video (optional but useful)
- [ ] Privacy policy text (plain + legal)
- [ ] Terms text
- [ ] In-app disclaimer copy (user-generated opinions)

Recommended disclaimer key:
- `legal.disclaimer.ugc`: "Ratings are user-generated opinions and do not represent verified facts about venues."

---

## 7) 30-day launch plan (Alkmaar + Schagen)

### Targets
- Seed auditors: 40 users (Alkmaar 25, Schagen 15)
- Seed audits: 300 total in 30 days
- Minimum quality: >= 85% audits with valid photo + city/venue mapping

### Week-by-week

#### Week 1 (Foundation)
- [ ] Recruit 10 “Founding Buddies” in Alkmaar, 6 in Schagen.
- [ ] Preload top 40 venues across both cities into canonical `venues`.
- [ ] Publish social teaser + waiting list.
- [ ] QA every flow on 2 iOS + 2 Android devices.

#### Week 2 (Private beta)
- [ ] Invite-only beta (20–25 users).
- [ ] Goal: 100 audits, 0 critical crashes, <5% moderation false positives.
- [ ] Daily review of pending reports and duplicate venues.

#### Week 3 (City challenge)
- [ ] Launch “Burger Sprint” challenge: 3 audits/week badge push.
- [ ] Goal: +120 audits, 70% WAU retention from Week 2 cohort.
- [ ] Highlight Trending and Top Rated stories on Instagram/TikTok (3 posts/week).

#### Week 4 (Public soft launch)
- [ ] Open app publicly with city-first homepage banners.
- [ ] Goal: total 300+ audits, 150 MAU, 35% D7 retention.
- [ ] Run leaderboard spotlight (positive framing only).

### Go / No-Go metrics at day 30
- **Go** if all true:
  - [ ] Crash-free sessions >= 99.3%
  - [ ] 300+ visible audits
  - [ ] >= 120 active users (last 60 days naturally true in first month)
  - [ ] >= 30 burgers with 3+ audits
  - [ ] Report auto-hide precision >= 80% after admin review
- **No-Go (delay scale-up)** if any:
  - [ ] Duplicate venue rate > 15%
  - [ ] D7 retention < 20%
  - [ ] Moderation backlog > 72h

---

## 8) Phase 2 roadmap (post-MVP)

### 8.1 Places API / venue validation
Options:
1. Google Places
2. Foursquare/Yelp equivalents
3. **Hybrid (Recommended): API match + admin override**

Actions:
- [ ] Add place_id on `venues`
- [ ] Auto-suggest verified venues
- [ ] Confidence score for matching

### 8.2 Multilingual expansion
- [ ] Add i18n pipeline for EN/NL keys.
- [ ] Localize store listings.
- [ ] City launch packs for Haarlem/Amsterdam.

### 8.3 Premium
- [ ] Premium profile flair
- [ ] Advanced personal stats dashboard
- [ ] Draft list/bookmarks

### 8.4 Merch unlocks
- [ ] Cosmetic badge-to-merch milestones (no gambling mechanics)
- [ ] Redeem code infrastructure

### 8.5 Venue insights (B2B-light)
- [ ] Anonymous trend dashboards for venues
- [ ] Monthly snapshot PDF
- [ ] Strict privacy aggregation thresholds (k-anonymity)

---

## 9) FlutterFlow build order (execution sequence)

1. [ ] Configure Auth + `users` table and onboarding.
2. [ ] Build Add Audit flow with validation and image upload.
3. [ ] Implement XP/Rank/Active calculations via backend actions/functions.
4. [ ] Build Feed + Discovery queries using materialized stats collections.
5. [ ] Build leaderboard city filter + active-only logic.
6. [ ] Implement reports + auto-hide + admin moderation screens.
7. [ ] Add badges and profile polish.
8. [ ] Run app store compliance pass and submit test builds.

---

## 10) MVP badges (10 cosmetic badges)

1. `badge.first_bite` — First audit posted.
2. `badge.quick_starter` — 5 quick audits posted.
3. `badge.deep_diver` — 5 deep audits posted.
4. `badge.city_scout_alkmaar` — 10 audits in Alkmaar.
5. `badge.city_scout_schagen` — 10 audits in Schagen.
6. `badge.week_streak_1` — 7-day audit streak.
7. `badge.photo_proof` — 20 audits with valid photos.
8. `badge.trend_spotter` — 10 audits on burgers later marked Trending.
9. `badge.community_guard` — 5 valid reports confirmed by admin.
10. `badge.street_judge` — Reach rank Street Judge.

Badge implementation checklist:
- [ ] Award asynchronously after each audit/report/admin resolution event.
- [ ] Store in `user_badges` join table (`user_id`, `badge_code`, `awarded_at`).
- [ ] Cosmetic only: no XP bonus in MVP.
