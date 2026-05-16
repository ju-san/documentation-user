# 02 — Architecture

## Tech Stack Decisions

| Layer | Pilihan | Alasan |
|---|---|---|
| **Framework** | Next.js 14 App Router + TypeScript | SSR untuk SEO, edge functions |
| **Styling** | Tailwind + shadcn/ui | Production-ready, fast |
| **Database** | Supabase Postgres | Auth + DB in one |
| **Auth** | Supabase Auth | Email + Google OAuth |
| **AI Script** | Anthropic Claude Haiku 4.5 | Bahasa Indonesia bagus, murah |
| **AI Voice** | ElevenLabs Multilingual v2 | Best Indonesian voice quality, ~$0.30/1K char |
| **Stock Footage** | Pexels API (free) + Pixabay backup | Free tier unlimited, good quality |
| **Music** | YouTube Audio Library + Pixabay Music | Royalty-free, no licensing risk |
| **Video Composition** | **Remotion** (server-side render) | React-based, programmatic, full control |
| **Video Rendering** | **AWS Lambda + Remotion Lambda** | Serverless rendering, scale on-demand |
| **Video Storage** | **Cloudflare R2** | $0.015/GB vs S3 $0.023, no egress cost |
| **Video CDN** | Cloudflare (gratis) | Fast global delivery |
| **Queue/Jobs** | Supabase Edge Functions + pg_cron | Native, no extra infra |
| **Scheduler** | Vercel Cron + Supabase cron | Trigger scheduled posts |
| **Rate Limit** | Upstash Redis | Serverless |
| **Email** | Resend | 3K/bln free |
| **Analytics** | PostHog | Free 1M events |
| **Hosting** | Vercel Pro | $20/bln, longer function timeout |
| **Monitoring** | Sentry | Free 5K errors |

## Video Generation Pipeline

**End-to-end flow per generate:**

```
1. User submit form (template + prompt + industri + tone)
       ↓
2. /api/generate → enqueue job
       ↓
3. Job worker: AI Script Generation
   - Claude Haiku generate script JSON:
     {scenes: [{text, duration, b_roll_keywords}, ...], caption, hashtags}
       ↓
4. Parallel:
   ├─ ElevenLabs: generate voiceover per scene (3-5 audio files)
   └─ Pexels API: fetch stock footage per b_roll_keywords
       ↓
5. Music selection (random dari library berdasarkan vibe)
       ↓
6. Remotion Lambda render:
   - Compose scenes (footage + audio + text overlay + music)
   - Render mp4 1080x1920 @ 30fps
   - Output to S3/R2
       ↓
7. Save to DB: video metadata + URL
       ↓
8. Notify user (push + email): "Video ready"
       ↓
9. Total time: 30-60 seconds
```

**Cost per video (target):**

| Component | Cost |
|---|---|
| Claude Haiku (script ~2K tokens) | $0.002 |
| ElevenLabs (30s voice ~500 chars) | $0.15 |
| Pexels (free) | $0.00 |
| Remotion Lambda render (30s video) | $0.05 |
| R2 storage (5MB/video × 90 days) | $0.0007 |
| **Total** | **~$0.20** (Rp 3.000) |

Pro user pays Rp 99.000 ÷ 50 video = Rp 1.980/video. **Margin: 85%**.

## Folder Structure

```
kontenkilat/
├── app/
│   ├── (marketing)/
│   │   ├── page.tsx                # Landing
│   │   ├── pricing/page.tsx
│   │   └── templates/[slug]/page.tsx
│   ├── (app)/
│   │   ├── generator/page.tsx
│   │   ├── scheduler/page.tsx
│   │   ├── library/page.tsx
│   │   ├── analytics/page.tsx
│   │   ├── settings/page.tsx
│   │   └── layout.tsx              # Auth gate
│   ├── api/
│   │   ├── generate/route.ts       # Trigger generation
│   │   ├── schedule/route.ts
│   │   ├── analytics/sync/route.ts
│   │   ├── webhooks/
│   │   │   ├── meta/route.ts       # Meta callback
│   │   │   ├── tiktok/route.ts
│   │   │   └── stripe/route.ts
│   │   └── jobs/
│   │       ├── render/route.ts     # Triggered by queue
│   │       └── post/route.ts       # Triggered by scheduler
│   └── auth/
├── remotion/
│   ├── compositions/
│   │   ├── PromoFlash.tsx
│   │   ├── ProductShowcase.tsx
│   │   ├── Testimonial.tsx
│   │   ├── TipsTricks.tsx
│   │   ├── BehindScenes.tsx
│   │   └── Quote.tsx
│   ├── components/                 # Shared scene components
│   ├── Root.tsx                    # Remotion entry
│   └── render.ts                   # Lambda render trigger
├── lib/
│   ├── ai/
│   │   ├── script-generator.ts
│   │   ├── caption-generator.ts
│   │   └── prompts.ts
│   ├── voice/
│   │   └── elevenlabs.ts
│   ├── footage/
│   │   └── pexels.ts
│   ├── music/
│   │   └── library.ts
│   ├── render/
│   │   └── remotion-lambda.ts
│   ├── platforms/
│   │   ├── instagram.ts            # Meta Graph API
│   │   └── tiktok.ts               # TikTok Content Posting API
│   ├── storage/
│   │   └── r2.ts
│   └── supabase/
├── components/
│   ├── generator/
│   ├── scheduler/
│   └── shared/
├── content/
│   ├── industries.json
│   ├── voices.json
│   └── music-library.json
└── public/
```

## Database Schema

```sql
-- ============================================
-- PROFILES (extends auth.users)
-- ============================================
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT UNIQUE NOT NULL,
  business_name TEXT,
  industry TEXT,
  tier TEXT DEFAULT 'free' CHECK (tier IN ('free', 'pro', 'pro_aiyoo_bundle')),
  tier_expires_at TIMESTAMPTZ,
  monthly_quota_used INT DEFAULT 0,
  monthly_quota_reset_at TIMESTAMPTZ DEFAULT (NOW() + INTERVAL '30 days'),
  storage_used_mb NUMERIC DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- SOCIAL_ACCOUNTS (OAuth tokens)
-- ============================================
CREATE TABLE social_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  platform TEXT NOT NULL CHECK (platform IN ('instagram', 'tiktok')),
  platform_user_id TEXT NOT NULL,
  platform_username TEXT,
  access_token TEXT NOT NULL,            -- ENCRYPT IN PRODUCTION
  refresh_token TEXT,
  token_expires_at TIMESTAMPTZ,
  scopes TEXT[],
  is_active BOOLEAN DEFAULT TRUE,
  connected_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, platform)
);

-- ============================================
-- VIDEOS (generated content)
-- ============================================
CREATE TABLE videos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  template_id TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'queued' 
    CHECK (status IN ('queued', 'rendering', 'completed', 'failed')),
  
  -- Input
  prompt TEXT NOT NULL,
  industry TEXT,
  tone TEXT,
  voice_id TEXT,
  music_id TEXT,
  product_images TEXT[], -- URLs
  
  -- AI-generated
  script JSONB, -- {scenes: [...], voiceover_text}
  caption_instagram TEXT,
  caption_tiktok TEXT,
  hashtags TEXT[],
  
  -- Output
  video_url TEXT,
  thumbnail_url TEXT,
  duration_sec INT,
  size_mb NUMERIC,
  
  -- Metadata
  generation_time_ms INT,
  error_message TEXT,
  has_watermark BOOLEAN DEFAULT TRUE,
  expires_at TIMESTAMPTZ, -- auto-delete date
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);

CREATE INDEX idx_videos_user_id ON videos(user_id);
CREATE INDEX idx_videos_status ON videos(status);
CREATE INDEX idx_videos_expires_at ON videos(expires_at) WHERE status = 'completed';

-- ============================================
-- SCHEDULED_POSTS
-- ============================================
CREATE TABLE scheduled_posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  video_id UUID REFERENCES videos(id) ON DELETE CASCADE,
  platform TEXT NOT NULL CHECK (platform IN ('instagram', 'tiktok')),
  caption TEXT NOT NULL,
  scheduled_at TIMESTAMPTZ NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'posting', 'posted', 'failed', 'cancelled')),
  retry_count INT DEFAULT 0,
  
  -- After posting
  platform_post_id TEXT,
  platform_post_url TEXT,
  posted_at TIMESTAMPTZ,
  error_message TEXT,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_scheduled_status_time ON scheduled_posts(status, scheduled_at)
  WHERE status = 'pending';

-- ============================================
-- POST_ANALYTICS (synced from platforms)
-- ============================================
CREATE TABLE post_analytics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scheduled_post_id UUID REFERENCES scheduled_posts(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  platform TEXT NOT NULL,
  
  views INT DEFAULT 0,
  likes INT DEFAULT 0,
  comments INT DEFAULT 0,
  shares INT DEFAULT 0,
  saves INT DEFAULT 0,
  reach INT DEFAULT 0,
  impressions INT DEFAULT 0,
  avg_watch_time_sec NUMERIC,
  engagement_rate NUMERIC,
  
  synced_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_analytics_user_post ON post_analytics(user_id, scheduled_post_id);

-- ============================================
-- USAGE_LOG (for quota tracking + cost analysis)
-- ============================================
CREATE TABLE usage_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  action TEXT NOT NULL, -- 'video_generated', 'video_posted'
  resource_id UUID, -- references video_id or scheduled_post_id
  cost_estimate_usd NUMERIC,
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- SUBSCRIPTIONS (Stripe sync)
-- ============================================
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,
  stripe_customer_id TEXT,
  stripe_subscription_id TEXT,
  plan TEXT NOT NULL,
  status TEXT NOT NULL, -- active, trialing, past_due, cancelled
  current_period_end TIMESTAMPTZ,
  cancel_at_period_end BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- RLS POLICIES (similar pattern as BalasChat)
-- ============================================
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE videos ENABLE ROW LEVEL SECURITY;
ALTER TABLE scheduled_posts ENABLE ROW LEVEL SECURITY;
ALTER TABLE post_analytics ENABLE ROW LEVEL SECURITY;
ALTER TABLE social_accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "User reads own videos" ON videos FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "User reads own posts" ON scheduled_posts FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "User reads own analytics" ON post_analytics FOR SELECT USING (auth.uid() = user_id);
-- Insert/update via service role only di backend
```

## Environment Variables

```bash
# Core
NEXT_PUBLIC_APP_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# AI
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=claude-haiku-4-5-20251001
ELEVENLABS_API_KEY=
ELEVENLABS_VOICE_M1=21m00Tcm4TlvDq8ikWAM  # Sample voice IDs
ELEVENLABS_VOICE_M2=...
ELEVENLABS_VOICE_F1=...
ELEVENLABS_VOICE_F2=...

# Footage & Storage
PEXELS_API_KEY=
PIXABAY_API_KEY=
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=kontenkilat-videos
R2_PUBLIC_URL=https://cdn.kontenkilat.id

# Remotion Lambda
AWS_REGION=ap-southeast-1
AWS_LAMBDA_FUNCTION=remotion-render-prod
REMOTION_SERVE_URL=https://...remotionlambda.s3.../site/

# Platforms
META_APP_ID=
META_APP_SECRET=
META_REDIRECT_URI=https://kontenkilat.id/api/webhooks/meta
TIKTOK_CLIENT_KEY=
TIKTOK_CLIENT_SECRET=
TIKTOK_REDIRECT_URI=https://kontenkilat.id/api/webhooks/tiktok

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PRICE_PRO=price_xxx
STRIPE_PRICE_BUNDLE=price_xxx

# Misc
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=
RESEND_API_KEY=
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PUBLIC_SENTRY_DSN=

# Aiyoo Cross-sell
AIYOO_BUNDLE_API_KEY=
AIYOO_PROVISION_URL=
```

## Meta (Instagram) Integration

### Required Permissions

- `instagram_basic`
- `instagram_content_publish`
- `pages_show_list`
- `pages_read_engagement`
- `business_management`

### Approval Process

1. Create Meta App at https://developers.facebook.com/apps
2. Add Instagram Graph API product
3. Submit App Review dengan:
   - Use case description
   - Screencast demo
   - Privacy policy URL
   - Terms of service URL
4. Wait 2-4 weeks
5. Once approved, switch app to Live mode

### Posting Flow

```
1. POST /api/v18.0/{ig-user-id}/media
   { image_url / video_url, caption, media_type: 'REELS' }
   → returns creation_id
   
2. Poll GET /api/v18.0/{creation_id} until status_code = 'FINISHED'

3. POST /api/v18.0/{ig-user-id}/media_publish
   { creation_id }
   → returns post_id
```

## TikTok Integration

### Required Permissions

- `video.upload`
- `video.publish`
- `user.info.basic`

### Approval Process

1. Apply at https://developers.tiktok.com/
2. Create app with Content Posting API
3. Submit for review (2-8 weeks)
4. Approval requires TikTok for Business account

### Posting Flow (Content Posting API)

```
1. POST /v2/post/publish/inbox/video/init
   { source_info: { video_url } }
   → returns publish_id
   
2. Server-to-server: TikTok downloads video
   Poll GET /v2/post/publish/status/fetch
   
3. When status = 'PUBLISH_COMPLETE', post_id available
```

### Fallback if API Not Approved

**Direct Post API** (for tester accounts only — limited):
- User explicit consent each post
- Manual approval each post in TikTok app

**Or**: provide download + clipboard copy caption, user post manually.

## Cost Model

### Bulan 1-3 (low traffic)

| Service | Cost |
|---|---|
| Vercel Pro | $20 |
| Supabase Pro | $25 |
| Anthropic Claude | $5 (~2K videos × $0.002 script) |
| ElevenLabs (Starter) | $5 (30K char/bln) |
| Pexels | $0 |
| R2 Storage | $2 (~100GB) |
| Remotion Lambda | $20 (~2K renders) |
| Upstash | $0 |
| Resend | $0 |
| **Total** | **~$77/bulan** |

### Bulan 6+ (scale: 10K videos/bln)

| Service | Cost |
|---|---|
| Vercel Pro | $20 |
| Supabase Pro | $25 |
| Anthropic | $25 |
| ElevenLabs (Creator) | $22 (100K char) atau Pro $99 (500K) |
| R2 | $30 (~2TB) |
| Remotion Lambda | $200 (~10K renders) |
| Resend Pro | $20 |
| PostHog Pro (kalau lewat 1M events) | $0-450 |
| **Total** | **~$320-770/bulan** |

Revenue 250 Pro user × Rp 99K = Rp 24.75jt (~$1,600). **Margin: 80%+**.

## Security Considerations

- Encrypt social_accounts.access_token at rest (use Supabase Vault atau application-level encryption)
- Refresh tokens auto-rotated via cron
- Rate limit /api/generate: 5 concurrent per user, 10/hour anonymous (no auth = no generate)
- Video URLs signed dengan expiration (R2 signed URLs)
- Sanitize user prompt sebelum kirim ke AI (prompt injection)
- Webhook signature verification untuk Meta + TikTok + Stripe
- CORS strict
- CSP headers untuk prevent XSS
- Storage quota enforcement before generate
- Watermark validation (Pro user dapat downgrade, video lama tetap no-watermark — itu OK)

## Performance Targets

| Metrik | Target |
|---|---|
| Video generation end-to-end | <60s (P95) |
| Generator page load | <2s LCP |
| Scheduler responsiveness | <300ms tiap action |
| Analytics dashboard | <3s initial load |
| Scheduled post posting accuracy | ±2 minutes |
| API uptime | 99.5% |
