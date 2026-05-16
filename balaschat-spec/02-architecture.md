# 02 — Architecture

## Tech Stack Decisions

| Layer | Pilihan | Alasan |
|---|---|---|
| **Framework** | Next.js 14 (App Router) + TypeScript | SSR untuk SEO, edge functions untuk speed, ekosistem matang |
| **Styling** | Tailwind CSS + shadcn/ui | Komponen production-ready, customizable, no runtime cost |
| **Database** | Supabase Postgres | Free tier 500MB, auth + db + storage in one |
| **Auth** | Supabase Auth | Email + Google + magic link out of the box |
| **AI** | Anthropic Claude Haiku 4.5 | Bahasa Indonesia bagus, cepat (<2s), murah (~$0.0008/generate) |
| **Rate Limit** | Upstash Redis | Serverless Redis, free 10K commands/day |
| **Email** | Resend | 3.000 email/bulan gratis, dev-friendly API |
| **Analytics** | PostHog | Free 1M events/bulan, funnel + recording |
| **Hosting** | Vercel | Free hobby cocok untuk traffic awal, instant deploy |
| **Monitoring** | Sentry | Free 5K errors/bulan, performance monitoring |
| **CDN** | Cloudflare (di depan Vercel) | DDoS protection + custom caching rules |

## Cost Estimate

### Bulan 1-3 (traffic kecil)
| Service | Tier | Cost |
|---|---|---|
| Vercel | Hobby | $0 |
| Supabase | Free | $0 |
| Anthropic API | ~5K generates × $0.0008 | $4 |
| Upstash | Free | $0 |
| Resend | Free | $0 |
| PostHog | Free | $0 |
| Sentry | Free | $0 |
| Domain (subdomain dari aiyoo.id) | $0 | $0 |
| **Total** | | **~$4/bulan** |

### Bulan 4-6 (scale to 50K users)
| Service | Tier | Cost |
|---|---|---|
| Vercel | Pro | $20 |
| Supabase | Pro | $25 |
| Anthropic API | ~50K generates | $40 |
| Upstash | Pay as you go | ~$10 |
| Resend | Pro (50K emails) | $20 |
| PostHog | Free (still under limit) | $0 |
| **Total** | | **~$115/bulan** |

ROI: kalau 1% leads convert ke Aiyoo Rp 199rb/bulan = 500 customer × Rp 199rb = Rp 99,5jt MRR. Cost $115 = Rp 1,8jt. **ROI 55x**.

## Folder Structure

```
balaschat/
├── app/
│   ├── (marketing)/
│   │   ├── page.tsx              # Homepage + AI Reply Generator
│   │   ├── kalkulator/
│   │   │   └── page.tsx          # ROI Calculator
│   │   ├── template/
│   │   │   ├── page.tsx          # Template index
│   │   │   └── [industri]/
│   │   │       └── [usecase]/
│   │   │           └── page.tsx  # SEO pages
│   │   └── pro/
│   │       └── page.tsx          # Aiyoo upsell page
│   ├── (app)/
│   │   ├── dashboard/
│   │   │   └── page.tsx          # User dashboard
│   │   └── layout.tsx            # Auth-gated layout
│   ├── api/
│   │   ├── generate/route.ts     # AI Reply Generator endpoint
│   │   ├── calculate/route.ts    # ROI Calculator endpoint
│   │   ├── leads/route.ts        # Lead capture
│   │   └── webhooks/
│   │       └── aiyoo/route.ts    # Webhook untuk track conversion
│   ├── auth/
│   │   ├── callback/route.ts     # Supabase OAuth callback
│   │   └── login/page.tsx
│   ├── layout.tsx
│   └── globals.css
├── components/
│   ├── ui/                       # shadcn components
│   ├── generator/
│   │   ├── GeneratorForm.tsx
│   │   ├── ReplyCard.tsx
│   │   └── EmailGateModal.tsx
│   ├── calculator/
│   │   ├── CalculatorForm.tsx
│   │   ├── ResultDisplay.tsx
│   │   └── ROIChart.tsx
│   └── shared/
│       ├── AiyooCTA.tsx
│       ├── Header.tsx
│       └── Footer.tsx
├── lib/
│   ├── supabase/
│   │   ├── client.ts
│   │   ├── server.ts
│   │   └── middleware.ts
│   ├── anthropic/
│   │   ├── client.ts
│   │   └── prompts.ts
│   ├── ratelimit.ts
│   ├── analytics.ts
│   └── utils.ts
├── content/
│   ├── industries.json           # 50+ industri options
│   └── templates/                # Pre-written template content
├── public/
├── middleware.ts                 # Auth + rate limit middleware
├── .env.example
└── next.config.js
```

## Database Schema (Supabase Postgres)

```sql
-- ============================================
-- 1. PROFILES (extends auth.users)
-- ============================================
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT UNIQUE NOT NULL,
  business_name TEXT,
  industry TEXT,
  whatsapp_number TEXT,
  is_pro BOOLEAN DEFAULT FALSE,
  aiyoo_trial_clicked_at TIMESTAMPTZ,
  aiyoo_signup_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_profiles_email ON profiles(email);

-- Auto-create profile on signup
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, email)
  VALUES (NEW.id, NEW.email);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- ============================================
-- 2. GENERATIONS (history & analytics)
-- ============================================
CREATE TABLE generations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  anonymous_id TEXT, -- browser fingerprint for non-logged-in
  type TEXT NOT NULL CHECK (type IN ('reply', 'calculator')),
  input JSONB NOT NULL,
  output JSONB,
  ip_address INET,
  user_agent TEXT,
  duration_ms INT,
  success BOOLEAN DEFAULT TRUE,
  error TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_generations_user_id ON generations(user_id);
CREATE INDEX idx_generations_anonymous_id ON generations(anonymous_id);
CREATE INDEX idx_generations_created_at ON generations(created_at DESC);

-- ============================================
-- 3. USAGE_QUOTAS (rate limit tracking)
-- ============================================
CREATE TABLE usage_quotas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  anonymous_id TEXT,
  date DATE NOT NULL DEFAULT CURRENT_DATE,
  count INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_quota_user_date ON usage_quotas(user_id, date)
  WHERE user_id IS NOT NULL;
CREATE UNIQUE INDEX idx_quota_anon_date ON usage_quotas(anonymous_id, date)
  WHERE anonymous_id IS NOT NULL;

-- ============================================
-- 4. LEADS (email captures)
-- ============================================
CREATE TABLE leads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  source TEXT NOT NULL, -- 'generator_gate' | 'calculator' | 'pro_page'
  business_name TEXT,
  industry TEXT,
  utm_source TEXT,
  utm_medium TEXT,
  utm_campaign TEXT,
  referrer TEXT,
  converted_to_aiyoo BOOLEAN DEFAULT FALSE,
  converted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_leads_email ON leads(email);
CREATE INDEX idx_leads_source ON leads(source);

-- ============================================
-- 5. CALCULATOR_RESULTS (for follow-up emails)
-- ============================================
CREATE TABLE calculator_results (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id UUID REFERENCES leads(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  chats_per_day INT,
  avg_order_value NUMERIC,
  current_response_time TEXT,
  hours_open INT,
  lost_revenue_per_month NUMERIC,
  lost_orders_per_month INT,
  potential_revenue_per_month NUMERIC,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- 6. ROW LEVEL SECURITY
-- ============================================
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE generations ENABLE ROW LEVEL SECURITY;
ALTER TABLE usage_quotas ENABLE ROW LEVEL SECURITY;
ALTER TABLE leads ENABLE ROW LEVEL SECURITY;
ALTER TABLE calculator_results ENABLE ROW LEVEL SECURITY;

-- Profiles: users can read/update own profile
CREATE POLICY "Users can view own profile"
  ON profiles FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users can update own profile"
  ON profiles FOR UPDATE USING (auth.uid() = id);

-- Generations: users can view own history
CREATE POLICY "Users can view own generations"
  ON generations FOR SELECT USING (auth.uid() = user_id);

-- Insert allowed for all (validated via service role in API)
CREATE POLICY "Service role can insert generations"
  ON generations FOR INSERT WITH CHECK (true);
```

## Environment Variables

`.env.example`:

```bash
# Next.js
NEXT_PUBLIC_APP_URL=http://localhost:3000

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx
SUPABASE_SERVICE_ROLE_KEY=eyJxxx

# Anthropic
ANTHROPIC_API_KEY=sk-ant-xxx
ANTHROPIC_MODEL=claude-haiku-4-5-20251001

# Upstash Redis (rate limiting)
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=xxx

# Resend (email)
RESEND_API_KEY=re_xxx
RESEND_FROM_EMAIL=hello@balaschat.aiyoo.id

# PostHog
NEXT_PUBLIC_POSTHOG_KEY=phc_xxx
NEXT_PUBLIC_POSTHOG_HOST=https://app.posthog.com

# Aiyoo Integration
AIYOO_TRIAL_URL=https://aiyoo.id/signup?ref=balaschat
AIYOO_API_KEY=xxx # for tracking conversion

# Sentry
NEXT_PUBLIC_SENTRY_DSN=https://xxx@sentry.io/xxx
```

## API Endpoints Design

### POST `/api/generate`

**Request:**
```json
{
  "industry": "toko-baju",
  "message": "Halo kak, harga produk A berapa?",
  "tone": "santai",
  "goal": "close_sale",
  "additional_info": "Produk A Rp 150.000, stok 5"
}
```

**Response (success):**
```json
{
  "ok": true,
  "data": {
    "variations": [...],
    "tips": "..."
  },
  "quota": {
    "used": 1,
    "limit": 3,
    "tier": "anonymous"
  }
}
```

**Response (quota exceeded):**
```json
{
  "ok": false,
  "error": "QUOTA_EXCEEDED",
  "message": "Kuota harian habis. Masukkan email untuk lanjut gratis.",
  "next_action": "email_gate"
}
```

### POST `/api/calculate`

**Request:**
```json
{
  "chats_per_day": 50,
  "avg_order_value": 150000,
  "current_response_time": "30-60_min",
  "hours_open": 12
}
```

**Response:**
```json
{
  "ok": true,
  "data": {
    "monthly_chats": 1500,
    "current_revenue": 1890000,
    "potential_revenue": 6120000,
    "lost_revenue": 4230000,
    "lost_orders": 28,
    "aiyoo_cost": 199000,
    "roi_multiplier": 21.3
  }
}
```

### POST `/api/leads`

**Request:**
```json
{
  "email": "user@example.com",
  "source": "calculator",
  "metadata": {
    "calculator_result_id": "uuid",
    "utm_source": "tiktok"
  }
}
```

## Middleware Strategy

`middleware.ts` di root project:

1. **Auth refresh** — Supabase session refresh untuk SSR
2. **Anonymous ID** — generate & set cookie kalau belum ada
3. **Rate limit** — quick check sebelum hit API route (heavy ratelimit di dalam route)
4. **PostHog** — track page views server-side
5. **Geo detection** — block non-Indonesia? (TBD — bagusnya allow worldwide untuk SEO)

## Performance Targets

| Metrik | Target |
|---|---|
| LCP (Largest Contentful Paint) | <2.0s |
| FID (First Input Delay) | <100ms |
| CLS (Cumulative Layout Shift) | <0.1 |
| TTFB (Time to First Byte) | <500ms |
| Lighthouse score | >90 (mobile) |
| AI generate response time | <3s end-to-end |

## Security Checklist

- [ ] API routes pakai `runtime = 'edge'` untuk speed + DDoS resistance
- [ ] Rate limiting per IP (10/min) + per anonymous_id (3/day) + per user (10-30/day)
- [ ] Input validation pakai Zod schema
- [ ] Anthropic API key hanya di server, never expose
- [ ] Supabase service_role key hanya di server
- [ ] RLS policies aktif untuk semua tables
- [ ] Email validation strict (no disposable email — bisa pakai library `disposable-email-domains`)
- [ ] CSRF protection (Next.js default cukup untuk same-origin)
- [ ] CORS strict (hanya allow aiyoo.id + balaschat.aiyoo.id)
- [ ] Sanitize user input sebelum kirim ke AI (prompt injection prevention)
- [ ] Sentry untuk monitor anomali
