# 03 — Implementation Plan (6 Minggu)

Lebih panjang dari BalasChat karena: video pipeline kompleks, API approval external dependencies, Remotion learning curve.

## Critical Path: Apply API Approval HARI 1

Sebelum coding apapun, hari pertama:

- [ ] **Apply Meta App** untuk Instagram Graph API (`instagram_content_publish`)
- [ ] **Apply TikTok Developer** untuk Content Posting API
- [ ] **Setup AWS account** untuk Remotion Lambda
- [ ] **Setup Cloudflare R2 bucket**
- [ ] **Apply ElevenLabs API**
- [ ] **Apply Pexels API key**

Approval Meta/TikTok bisa 2-8 minggu — paralel dengan development.

---

## Minggu 1: Foundation + Video Pipeline Prototype

### Hari 1 — Project Setup + API Applications
- [ ] `npx create-next-app@latest kontenkilat`
- [ ] Setup Git repo, push GitHub
- [ ] Install deps (lihat `02-architecture.md`)
- [ ] Setup Tailwind + shadcn/ui
- [ ] **Apply semua API approval (Meta, TikTok, AWS, etc)**

### Hari 2 — Supabase + Auth
- [ ] Create Supabase project (region Singapore)
- [ ] Run full DB schema SQL
- [ ] Setup auth: email + Google OAuth
- [ ] Build login/signup pages
- [ ] Onboarding flow (industry selection)

### Hari 3 — AI Script Generator
- [ ] Setup Anthropic client
- [ ] Implement script prompt per template (6 templates)
- [ ] `/api/generate/script` endpoint
- [ ] Test output structure: `{scenes, voiceover_text, caption, hashtags}`
- [ ] Validate JSON output dengan Zod

### Hari 4 — ElevenLabs Integration
- [ ] Setup ElevenLabs client
- [ ] Test voice quality untuk Bahasa Indonesia (4 voice options)
- [ ] Implement generate-voice helper
- [ ] Cache voice files di R2 (avoid re-gen)

### Hari 5 — Pexels Integration
- [ ] Setup Pexels API client
- [ ] Implement fetch-footage helper dengan keyword search
- [ ] Pixabay backup integration
- [ ] Curate "approved keywords" untuk consistency

### Hari 6 — Remotion Setup
- [ ] Install Remotion + Lambda
- [ ] Buat composition pertama: PromoFlash (template paling sederhana)
- [ ] Test render locally
- [ ] Deploy ke Remotion Lambda (AWS)

### Hari 7 — End-to-End Test
- [ ] Wire up: script → voice → footage → Remotion compose → output URL
- [ ] Test 5 generate dengan input berbeda
- [ ] Measure: total time, cost, quality
- [ ] Iterate prompt & composition based on output
- [ ] Commit: "Week 1: E2E video pipeline prototype works"

---

## Minggu 2: 6 Templates + Generator UI

### Hari 8-9 — Build All 6 Remotion Compositions
- [ ] PromoFlash (done minggu 1, polish)
- [ ] ProductShowcase
- [ ] Testimonial
- [ ] TipsTricks
- [ ] BehindScenes
- [ ] Quote/Inspiration
- [ ] Setup music library (curate 10 royalty-free tracks per vibe)
- [ ] Add intro/outro animations per template

### Hari 10 — Generator UI (Form)
- [ ] Template card selector dengan thumbnail preview
- [ ] Form: prompt + industry + tone
- [ ] Voice & music selectors dengan inline preview
- [ ] Product image uploader (max 3, drag & drop)
- [ ] Form validation

### Hari 11 — Generator UI (Output)
- [ ] Loading state dengan progress indicator
- [ ] Video preview player
- [ ] Caption display dengan edit mode
- [ ] Hashtag chips dengan add/remove
- [ ] Action buttons: Download, Schedule, Regenerate

### Hari 12 — Job Queue
- [ ] Supabase Edge Function untuk background job processing
- [ ] Insert video record dengan status='queued'
- [ ] Worker picks up queued → rendering → completed
- [ ] Retry mechanism (max 2 retries)
- [ ] Notify user via real-time channel (Supabase Realtime)

### Hari 13 — Quota Management
- [ ] Tier-based quota check before generate
- [ ] Monthly reset cron job
- [ ] Storage quota tracking + enforce
- [ ] Watermark logic (free = yes, pro = no)

### Hari 14 — Buffer & Polish
- [ ] Fix bugs dari week 2 testing
- [ ] Mobile responsive (generator paling critical)
- [ ] Commit: "Week 2: Generator full UX working with 6 templates"

---

## Minggu 3: Scheduler + Platform Integration

### Hari 15 — Scheduler UI
- [ ] Calendar component (react-big-calendar atau full-calendar)
- [ ] Drag-and-drop reschedule
- [ ] Smart Time slot recommendation
- [ ] Per-platform caption editor

### Hari 16-17 — Instagram Integration
- [ ] OAuth flow dengan Meta (kalau approved; pakai test mode kalau belum)
- [ ] Save token ke `social_accounts`
- [ ] Implement `lib/platforms/instagram.ts`: upload + publish
- [ ] Token refresh cron (Meta token expire ~60 hari)
- [ ] Test posting end-to-end

### Hari 18-19 — TikTok Integration
- [ ] OAuth flow dengan TikTok
- [ ] Save token
- [ ] Implement `lib/platforms/tiktok.ts`
- [ ] Test inbox post (kalau approval delay, fallback ke direct sandbox)
- [ ] Handle music copyright check error case

### Hari 20 — Scheduler Worker
- [ ] Vercel Cron job tiap 5 menit: check `scheduled_posts` due
- [ ] Trigger post via platform API
- [ ] Handle success: update status, save post_url
- [ ] Handle failure: retry 1x, notify user
- [ ] Cancel logic untuk Free tier yang downgrade

### Hari 21 — Buffer
- [ ] Test scheduling end-to-end
- [ ] Edge cases: token expired, network failure, rate limited
- [ ] Mobile responsive untuk calendar
- [ ] Commit: "Week 3: Multi-platform scheduling works"

---

## Minggu 4: Analytics + Library + Templates SEO

### Hari 22-23 — Analytics Sync
- [ ] Cron job sync metrics dari IG + TikTok daily
- [ ] Store di `post_analytics`
- [ ] Build aggregation views (weekly, monthly)
- [ ] AI insights generator (Claude reads metrics, output 3 tips)

### Hari 24 — Analytics Dashboard UI
- [ ] Overview cards (total reach, engagement rate, best post)
- [ ] Chart: views over time
- [ ] Top 3 video card
- [ ] Best posting time analysis chart
- [ ] Filter by platform + date range

### Hari 25 — Auto-Report Email (Pro)
- [ ] Weekly Monday cron job
- [ ] AI-generated insight per user
- [ ] Email template via Resend
- [ ] Unsubscribe handling

### Hari 26 — Video Library
- [ ] Grid view all videos
- [ ] Filter & search
- [ ] Bulk actions (download, delete, re-schedule)
- [ ] Storage indicator (used vs quota)
- [ ] Auto-delete cron (free tier 30 days, pro 90 days)

### Hari 27-28 — Template SEO Pages
- [ ] Build 60 SEO pages (6 templates × 10 industri)
- [ ] AI-assist content generation, manual edit top 10
- [ ] Sample video per page (pre-generated)
- [ ] Schema.org markup, OG images
- [ ] Sitemap + Search Console
- [ ] Commit: "Week 4: Analytics + library + 60 SEO pages"

---

## Minggu 5: Pricing + Payment + Onboarding Polish

### Hari 29 — Stripe Setup
- [ ] Create Stripe account (Indonesia atau Singapore)
- [ ] Create products: Pro Rp 99K, Bundle Rp 249K
- [ ] Setup payment methods: card, GoPay, OVO, QRIS (via Xendit kalau perlu)
- [ ] Webhook endpoint

### Hari 30 — Pricing Page
- [ ] Pricing comparison table
- [ ] Toggle monthly/yearly (15% discount yearly)
- [ ] FAQ section
- [ ] Testimonial section
- [ ] Trust badges

### Hari 31 — Checkout Flow
- [ ] Stripe Checkout integration
- [ ] Success/cancel pages
- [ ] Webhook handler: update profile.tier on subscription event
- [ ] Email confirmation

### Hari 32 — Aiyoo Bundle Cross-sell
- [ ] Aiyoo provisioning API integration
- [ ] On Bundle subscription: auto-provision Aiyoo account
- [ ] Cross-sell banners di high-intent locations
- [ ] Bundle FAQ + comparison

### Hari 33-34 — Onboarding Polish
- [ ] Guided tour first-time user (Driver.js atau Intro.js)
- [ ] Sample prompts per industry
- [ ] First video celebration animation
- [ ] Empty states everywhere

### Hari 35 — Buffer
- [ ] Test payment flow end-to-end
- [ ] Test downgrade flow
- [ ] Test cancel flow
- [ ] Commit: "Week 5: Payment + onboarding complete"

---

## Minggu 6: Polish + Launch

### Hari 36 — Performance Optimization
- [ ] Lighthouse audit, target 90+ mobile
- [ ] Code splitting (Remotion deps very heavy)
- [ ] Image optimization
- [ ] Video lazy loading
- [ ] Service worker untuk offline support

### Hari 37 — Landing Page
- [ ] Hero dengan demo video embed
- [ ] Features section
- [ ] How it works (3 steps)
- [ ] Social proof + testimonial
- [ ] Pricing teaser
- [ ] FAQ
- [ ] Footer

### Hari 38 — Analytics & Monitoring
- [ ] PostHog full event tracking
- [ ] Sentry alerts setup
- [ ] Uptime monitoring (Better Uptime)
- [ ] Cost monitoring dashboard (AWS + Anthropic + ElevenLabs)

### Hari 39-40 — Beta Testing
- [ ] Invite 30 beta tester
- [ ] Setup feedback channel (WA group + Typeform)
- [ ] Daily standup dengan beta data
- [ ] Critical bugs fix priority

### Hari 41-42 — Launch Prep
- [ ] ProductHunt assets
- [ ] Social media kit (8 carousels, 12 TikTok)
- [ ] Demo video 60s (extra polished)
- [ ] Cold email outreach list (300 prospects)

### Hari 43 — Soft Launch
- [ ] Deploy production
- [ ] DNS + SSL
- [ ] Invite Aiyoo existing users (cross-promote)
- [ ] Monitor 24h

### Hari 44-45 — Public Launch
- [ ] ProductHunt launch
- [ ] Content blast
- [ ] Press outreach
- [ ] Commit: "Week 6: Public launch"

---

## Dependencies Antar Tugas

```
Hari 1 (API Apply) ──── Hari 16-19 (Platform integrations) ── (block!)
                                                                   ↓
Hari 1 (Setup) ── Hari 2-7 (E2E Pipeline) ── Hari 8-14 (Templates+UI) ── Hari 20+ (Scheduler)
                                                                              ↓
                                                                      Hari 22-26 (Analytics)
                                                                              ↓
                                                                      Hari 29-35 (Payment)
                                                                              ↓
                                                                      Hari 36-45 (Launch)
```

## Risk Mitigation

| Risk | Impact | Mitigation |
|---|---|---|
| **Meta API approval lambat** | High | Build feature behind feature flag, fallback ke manual download |
| **TikTok API approval lambat** | High | Same, manual post fallback |
| **Remotion Lambda render slow** | Medium | Pre-render frequently-used compositions, scale Lambda concurrency |
| **ElevenLabs voice kualitas Indonesia jelek** | Medium | Test minggu 1, fallback ke Google TTS Indonesia kalau perlu |
| **Cost per video > target** | Medium | Aggressive caching, lower res output, tighter limits |
| **Video copyright complaints (music)** | High | Hanya pakai royalty-free, document license, takedown procedure |
| **Stock footage limited variety** | Low | Add Pixabay + Coverr as backup |
| **Server timeout pada generation** | High | Background job pattern (not sync API), realtime notification |

## Skip-If-Tight Features

- ❌ AI insights di analytics (manual chart cukup)
- ❌ Drag-and-drop scheduler reschedule (basic edit modal cukup)
- ❌ 60 SEO pages full (30 dulu)
- ❌ Aiyoo Bundle auto-provision (manual provisioning awalnya, automate later)
- ❌ Yearly pricing toggle (monthly only v1)
- ❌ Service worker offline (luxury)
- ❌ Custom brand voice (Pro+ future feature)

## Definition of Done untuk Launch

- [ ] Generate 1 video <60 detik (P95)
- [ ] 6 templates render properly
- [ ] Voice + footage + music + caption auto-compose
- [ ] Schedule + post ke IG works end-to-end
- [ ] Schedule + post ke TikTok works (atau manual fallback clear)
- [ ] Analytics show real metrics
- [ ] Free → Pro upgrade flow works dengan payment
- [ ] Pro user no watermark, free user has
- [ ] Storage quota enforced
- [ ] Monthly quota reset works
- [ ] 30 SEO pages indexed
- [ ] Cost per video < $0.30
- [ ] 30+ beta tester feedback collected
- [ ] 0 critical Sentry errors di 48 jam test
- [ ] Mobile Lighthouse >85
- [ ] At least one viral-worthy demo video tested
