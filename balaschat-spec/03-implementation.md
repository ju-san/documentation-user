# 03 — Implementation Plan (4 Minggu)

Target: production-ready, soft-launch ke 100 user, ready scale.

## Minggu 1: Foundation & Core AI

### Hari 1 (Senin) — Setup Project
- [ ] `npx create-next-app@latest balaschat --typescript --tailwind --app`
- [ ] Setup repo Git, push ke GitHub private
- [ ] Install dependencies (lihat 02-architecture.md `package.json`)
- [ ] Setup shadcn/ui: `npx shadcn-ui@latest init`
- [ ] Install komponen yang dibutuhkan: `button`, `input`, `select`, `textarea`, `dialog`, `card`, `slider`, `toast`
- [ ] Setup Tailwind theme matching Aiyoo brand color (teal `#00BFA6`)
- [ ] Setup `.env.local` & `.env.example`

### Hari 2 — Supabase Setup
- [ ] Create Supabase project (region Singapore untuk latency Indonesia)
- [ ] Run SQL schema dari `02-architecture.md`
- [ ] Setup auth providers: email + Google OAuth
- [ ] Test signup flow via Supabase dashboard
- [ ] Bikin client/server Supabase helpers di `lib/supabase/`
- [ ] Setup middleware auth refresh

### Hari 3 — AI Reply Generator API
- [ ] Bikin Anthropic client di `lib/anthropic/client.ts`
- [ ] Implement prompt builder di `lib/anthropic/prompts.ts` (lihat `04-code-critical.md`)
- [ ] Bikin `app/api/generate/route.ts` dengan:
  - Input validation (Zod)
  - Rate limit check
  - Call Claude API
  - Parse JSON response
  - Save ke `generations` table
  - Return structured response
- [ ] Test via Postman/curl
- [ ] Handle error cases: API timeout, invalid output, quota exceeded

### Hari 4 — Generator UI
- [ ] Bikin `components/generator/GeneratorForm.tsx`
  - Industry dropdown dari `content/industries.json`
  - Message textarea (character counter)
  - Tone radio group
  - Goal dropdown
  - Submit button dengan loading state
- [ ] Bikin `components/generator/ReplyCard.tsx`
  - Display reply text
  - Copy button (dengan toast feedback)
  - WhatsApp share button (`wa.me/?text=`)
  - Thumbs up/down feedback
  - Regenerate button
- [ ] Integrate ke homepage `app/(marketing)/page.tsx`
- [ ] Test end-to-end flow

### Hari 5 — Rate Limiting & Anonymous Tracking
- [ ] Setup Upstash Redis
- [ ] Implement `lib/ratelimit.ts` dengan tier-based limits
- [ ] Bikin anonymous fingerprint via cookie + IP hash
- [ ] Integrate ke `/api/generate` middleware
- [ ] Bikin `components/generator/EmailGateModal.tsx`
- [ ] Test quota flow: anonymous 3 → email 10 → user 30

### Hari 6 — Lead Capture
- [ ] Bikin `app/api/leads/route.ts`
- [ ] Email validation + disposable email check
- [ ] Save ke `leads` table dengan UTM tracking
- [ ] Trigger Resend welcome email
- [ ] Bikin welcome email template (HTML + plain text)

### Hari 7 — Buffer & Polish
- [ ] Bug fixing dari testing minggu 1
- [ ] Error boundaries
- [ ] Loading skeletons
- [ ] Mobile responsive check (Chrome devtools + real Android device)
- [ ] Commit: "Week 1: Core AI Generator working end-to-end"

---

## Minggu 2: ROI Calculator + User Account

### Hari 8 — ROI Calculator Logic
- [ ] Implement calculation logic di `lib/calculator.ts`
- [ ] Bikin `app/api/calculate/route.ts`
- [ ] Validate input ranges
- [ ] Save calculator results to DB
- [ ] Unit tests untuk edge cases (0 chats, huge revenue, etc)

### Hari 9 — Calculator UI
- [ ] Bikin `components/calculator/CalculatorForm.tsx`
  - Slider untuk chats/day dengan label dinamis
  - Number input untuk avg order value (with Rp formatter)
  - Dropdown untuk response time
  - Range slider untuk operating hours
  - Real-time calculation (debounced 500ms)
- [ ] Bikin `components/calculator/ResultDisplay.tsx`
  - Big number animation
  - Visual donut chart (recharts atau visx)
  - ROI summary card
  - Aiyoo CTA box
- [ ] Page `app/(marketing)/kalkulator/page.tsx`

### Hari 10 — User Dashboard
- [ ] Bikin auth layout `app/(app)/layout.tsx`
- [ ] Dashboard page dengan tabs: History | Settings | Templates
- [ ] History tab: list generate sebelumnya, filter by date/industry
- [ ] Settings tab: profile, business info, WA number
- [ ] Templates tab (Pro tease): "Saved templates akan jadi fitur Aiyoo →"

### Hari 11 — Auth Flows
- [ ] Login page dengan Google + email
- [ ] Signup flow dengan onboarding (business info)
- [ ] Magic link option
- [ ] Forgot password flow
- [ ] Test all auth edge cases

### Hari 12 — Email Sequences
- [ ] Setup Resend templates atau pakai React Email
- [ ] Welcome email (immediate)
- [ ] Day 1 email: "Berikut 5 tips balas WA lebih cepat"
- [ ] Day 3 email: "Lihat case study: toko ini hemat 6 jam/hari pakai Aiyoo"
- [ ] Day 7 email: "Aiyoo trial gratis 14 hari + diskon early adopter"
- [ ] Setup queue via Supabase Edge Function atau cron job

### Hari 13 — Aiyoo CTA Integration
- [ ] Bikin `components/shared/AiyooCTA.tsx` (reusable banner)
- [ ] Variants: inline (in result), modal (after 5 generates), exit-intent
- [ ] Track click via PostHog
- [ ] Setup webhook dari Aiyoo untuk track conversion
- [ ] UTM parameters untuk attribution

### Hari 14 — Buffer & Test
- [ ] End-to-end test: anonymous → email → account → upsell click
- [ ] Performance check: Lighthouse audit
- [ ] Mobile real-device test
- [ ] Commit: "Week 2: Calculator + accounts + email sequences"

---

## Minggu 3: SEO + Polish

### Hari 15 — Landing Page Polish
- [ ] Hero section: headline kuat + demo video embed
- [ ] Features section: 3 benefits (AI smart, gratis, Bahasa Indonesia)
- [ ] How it works: 3 steps illustrated
- [ ] Social proof: rating + testimonial (kalau ada)
- [ ] FAQ accordion
- [ ] Footer dengan link ke Aiyoo

### Hari 16 — SEO Infrastructure
- [ ] `app/sitemap.ts` — dynamic sitemap untuk SEO pages
- [ ] `app/robots.ts`
- [ ] OG image generator pakai `@vercel/og`
- [ ] Schema.org structured data (Organization, WebApplication)
- [ ] Meta tags strategy per page
- [ ] Setup Google Search Console + submit sitemap

### Hari 17 — Template Library v1 (20 Industri)
- [ ] Bikin static data `content/templates/` per industri
- [ ] AI-assisted: generate konten awal pakai Claude, edit manual
- [ ] Template index page `/template/page.tsx`
- [ ] Industry pages `/template/[industri]/page.tsx`
- [ ] Use case detail pages `/template/[industri]/[usecase]/page.tsx`
- [ ] Inter-linking strategy (related templates, related industries)

### Hari 18 — Template Library v2 (30 Industri)
- [ ] Generate konten untuk 30 industri lagi (total 50)
- [ ] Polish copy, add real examples
- [ ] Embed CTA box di tiap page

### Hari 19 — Analytics Setup
- [ ] PostHog: initialize client + server tracking
- [ ] Track semua events dari `01-product-spec.md`
- [ ] Bikin dashboard PostHog: funnel + retention + key metrics
- [ ] GA4 backup untuk SEO traffic analysis
- [ ] Setup Hotjar atau PostHog session replay (sampling 10%)

### Hari 20 — Performance Optimization
- [ ] Image optimization: WebP, lazy load, proper sizing
- [ ] Code splitting: dynamic imports for heavy components
- [ ] Font optimization: subset + preload
- [ ] Cache strategy: static pages, ISR untuk templates
- [ ] Bundle analysis: kill unused deps

### Hari 21 — Buffer
- [ ] Lighthouse audit 90+
- [ ] Cross-browser test (Chrome, Safari, Firefox, mobile)
- [ ] Accessibility audit (axe DevTools)
- [ ] Commit: "Week 3: SEO + 50 template pages + polish"

---

## Minggu 4: Launch Prep + Soft Launch

### Hari 22 — Launch Assets
- [ ] Demo video 60 detik (pakai HeyGen sesuai storyboard sebelumnya)
- [ ] 5 screenshots untuk ProductHunt
- [ ] OG image custom
- [ ] Social media kit (3 IG carousels, 5 TikTok video drafts)

### Hari 23 — ProductHunt & Launch Pages
- [ ] Setup ProductHunt listing draft
- [ ] Setup BetaList listing
- [ ] Setup IndieHackers post draft
- [ ] Internal Indonesia: Dailysocial.id, Tech in Asia outreach email
- [ ] LinkedIn announcement draft

### Hari 24 — Email Outreach List
- [ ] Compile list 500 UMKM Indonesia (LinkedIn Sales Navigator atau scrap manual)
- [ ] Bikin sequence email outreach 3-step
- [ ] Setup Apollo/Lemlist atau manual via Gmail
- [ ] Influencer micro list: 30 creator UMKM/jualan online di IG/TikTok

### Hari 25 — Soft Launch
- [ ] Deploy ke production (Vercel)
- [ ] DNS setup: `balaschat.aiyoo.id` → Vercel
- [ ] Test SSL, redirects, all flows in prod
- [ ] Invite 20 beta tester (komunitas Aiyoo existing)
- [ ] Monitor Sentry + analytics setup hari
- [ ] Collect feedback via WA chat

### Hari 26 — Iterate dari Feedback
- [ ] Fix critical bugs dari beta
- [ ] Iterate copy/UX yang membingungkan
- [ ] Tambah feature request kecil yang impactful
- [ ] Update FAQ dengan pertanyaan beta tester

### Hari 27 — Pre-Launch Content
- [ ] Post 5 TikTok bertahap selama 5 hari (jadwal Buffer/Later)
- [ ] Post LinkedIn launch announcement
- [ ] Email blast ke existing Aiyoo users
- [ ] Schedule ProductHunt launch (Tuesday/Wednesday 00:01 PST)

### Hari 28 — Public Launch Day
- [ ] ProductHunt launch (asks for upvote di komunitas dev Indonesia)
- [ ] Post di Hacker News (Show HN: BalasChat)
- [ ] Post di r/indonesia subreddit
- [ ] Live posting di TikTok + IG Stories tiap 2 jam
- [ ] Monitor traffic, respond to comments
- [ ] Track funnel conversion real-time

### Hari 29 — Outreach Push
- [ ] Influencer DM (sudah list dari hari 24)
- [ ] Cold email batch 1 (100 prospects)
- [ ] Engage di komunitas FB/Telegram UMKM
- [ ] Post case study di LinkedIn

### Hari 30 — Review & Plan
- [ ] Analyze launch metrics
- [ ] Document learnings
- [ ] Plan bulan 2 priorities
- [ ] Plan A/B tests pertama (hero copy, CTA placement, dll)
- [ ] Commit: "Week 4: Public launch + iteration"

---

## Dependencies Antar Tugas

```
Hari 1 (Setup) ─┬─ Hari 2 (Supabase) ──── Hari 5 (Rate limit) ─── Hari 6 (Lead capture)
                 │
                 └─ Hari 3 (AI API) ──── Hari 4 (Generator UI) ────┘

Hari 8-9 (Calculator) ────── independent dari hari 10-11 (Auth) tapi sama-sama prep buat hari 12-13

Minggu 3 (SEO) requires minggu 1-2 selesai

Minggu 4 (Launch) requires semua minggu 1-3 selesai
```

## Risk Mitigation

| Risk | Impact | Mitigation |
|---|---|---|
| AI output kualitas jelek | High | Iterate prompt minggu 1 dengan 20+ test case nyata |
| API cost balloon | Medium | Rate limit ketat + monitoring cost di Anthropic dashboard |
| Supabase quota habis | Medium | Upgrade ke Pro $25 kalau dekat limit, alert di 80% |
| Launch flat | High | Pre-engage komunitas Aiyoo, soft launch dapat feedback dulu |
| SEO lambat | Low | Template library jadi long-term play, OK kalau bulan 2-3 baru rank |
| Prompt injection attack | Medium | Sanitize input, system prompt defense, monitor logs |

## Skip-If-Tight Features

Kalau timeline mepet, skip dulu fitur ini (move ke v2 pasca-launch):

- ❌ Template library full 50 industri (do 20 dulu)
- ❌ Magic link login (Google OAuth cukup)
- ❌ Session replay (cukup PostHog event)
- ❌ A/B testing infrastructure (manual eksperimen dulu)
- ❌ Custom industries (50 preset cukup)
- ❌ Multi-language (English version v2)
- ❌ Brand voice training (Pro feature, sell as upsell)

## Definition of Done untuk Launch

- [ ] AI generate <3s response time
- [ ] Rate limit working: anon 3/day, email 10/day, account 30/day
- [ ] Email gate triggers correctly
- [ ] Calculator outputs correct numbers (verify formula)
- [ ] Welcome email arrives <30s
- [ ] Lighthouse mobile >85
- [ ] 0 critical errors di Sentry selama 24 jam test
- [ ] Funnel tracking accurate di PostHog
- [ ] All 50 industry pages indexed
- [ ] SSL + custom domain working
- [ ] At least 20 beta tester feedback collected
