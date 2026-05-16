# 01 — Product Spec

## Site Map

```
kontenkilat.id/
├── /                          # Landing + demo video
├── /generator                 # AI video generator (main tool)
├── /scheduler                 # Calendar + queue
├── /analytics                 # Performance dashboard
├── /library                   # Generated video library
├── /templates                 # Pre-made video templates per industri
├── /pricing                   # Free vs Pro vs Bundle
├── /dashboard                 # User home
└── /api/
    ├── /generate              # Trigger video generation
    ├── /schedule              # CRUD jadwal posting
    ├── /analytics             # Fetch IG/TikTok metrics
    └── /webhooks/             # Meta/TikTok callbacks
```

## Fitur 1: AI Video Generator (Core)

### User Flow

```
Pick template type
   ↓
Fill prompt (e.g. "Promo skincare diskon 50% sampai akhir bulan")
   ↓
[Optional] Upload product photo
   ↓
Select voice (M1, M2, F1, F2) + music vibe (energetic/calm/trendy)
   ↓
Click Generate (loading 30-60 sec)
   ↓
Preview video + caption + hashtag
   ↓
Actions:
   - Download (mp4)
   - Schedule to IG
   - Schedule to TikTok
   - Edit caption manually
   - Regenerate variation
```

### Video Templates (V1: 6 Templates)

Setiap template = struktur scene + animasi yang berbeda. AI fill content.

| Template | Use Case | Durasi | Format |
|---|---|---|---|
| **Promo Flash** | Diskon, flash sale, limited offer | 15s | Vertical 9:16 |
| **Product Showcase** | Showcase 1 produk lengkap dengan benefit | 30s | Vertical 9:16 |
| **Testimonial Card** | Customer review animated | 20s | Vertical 9:16 |
| **Tips & Tricks** | Edukatif, tips industri | 30s | Vertical 9:16 |
| **Behind The Scenes** | BTS, proses produksi/jasa | 25s | Vertical 9:16 |
| **Quote/Inspiration** | Inspirational quote bisnis | 15s | Vertical 9:16 |

### Generator Form

| Field | Type | Required | Notes |
|---|---|---|---|
| Template | Card selector | Yes | 6 templates dengan thumbnail |
| Prompt utama | Textarea (max 500) | Yes | Deskripsi konten yang mau dibuat |
| Industri | Dropdown | Yes | Mempengaruhi pilihan stock footage |
| Tone | Radio | Yes | Energetic / Profesional / Santai / Edukatif |
| Voice | Selector dengan preview | Yes | 4 pilihan suara Indonesia (2M + 2F) |
| Music | Selector dengan preview | Yes | 6 vibe: trendy, energetic, calm, dramatic, fun, romantic |
| Upload produk (opsional) | File upload | No | Max 3 foto, akan di-blend ke video |
| CTA | Input | No | Default: "Order sekarang" |

### Output

```json
{
  "video_id": "uuid",
  "video_url": "https://cdn.kontenkilat.id/videos/xxx.mp4",
  "thumbnail_url": "https://cdn.kontenkilat.id/thumbs/xxx.jpg",
  "duration_sec": 15,
  "format": "9:16",
  "size_mb": 4.2,
  "caption": {
    "instagram": "✨ FLASH SALE skincare ALL ITEM diskon 50%! Khusus 24 jam saja, jangan sampai kelewat ya kak!\n\nKlik link di bio buat order langsung 🛒\n\n#flashsale #skincareindonesia #...",
    "tiktok": "Diskon 50% SEMUA skincare? Yes please! 🔥 Tag temenmu yang butuh promo ini! #flashsale #skincare #fyp"
  },
  "hashtags": ["#flashsale", "#skincareindonesia", "#promobeauty", "..."],
  "music_credit": "Track Name by Artist (royalty-free)",
  "regenerate_token": "xxx" // pakai untuk regenerate variasi
}
```

## Fitur 2: Multi-Channel Scheduler

### User Flow

```
From video preview → "Schedule Post"
   ↓
Pick platforms (IG / TikTok / both)
   ↓
Pick date + time (recommended slots highlighted)
   ↓
Edit caption per platform if needed
   ↓
[Free] Auto-add #kontenkilat watermark+caption credit
[Pro] No watermark
   ↓
Confirm → Saved to queue
   ↓
Background worker triggers post at scheduled time
   ↓
On success: status "Posted" + link to post
On fail: notify user, retry once
```

### Calendar View

- Bulanan view dengan dots per scheduled post
- Click date → daily view dengan time slots
- Drag-and-drop reschedule
- Color coding: 🟢 Posted | 🟡 Scheduled | 🔴 Failed | 🔵 Draft
- "Smart Time" suggestions berdasarkan industri benchmark (peak engagement)

### Recommended Time Slots (Indonesia)

| Platform | Hari kerja | Weekend |
|---|---|---|
| Instagram | 11:00-13:00, 19:00-21:00 | 09:00-11:00, 20:00-22:00 |
| TikTok | 12:00-14:00, 20:00-23:00 | 10:00-12:00, 21:00-23:00 |

## Fitur 3: Analytics Dashboard

### Metrics yang Ditampilkan

**Per Video:**
- Views, likes, comments, shares
- Engagement rate
- Saves (IG) / Saves & follows (TikTok)
- Reach + impressions (kalau IG Business)
- Average watch time (TikTok)

**Aggregated (Weekly/Monthly):**
- Total reach across platforms
- Best performing video (top 3)
- Best posting time analysis
- Follower growth chart
- Content type performance (Reels vs Story vs Carousel)

### Auto-Report (Pro Feature)

Email otomatis tiap Senin pagi:
- Performance summary minggu lalu
- Top 3 video
- 3 actionable insight (AI-generated)
- Suggested content untuk minggu depan
- Comparison vs minggu sebelumnya

## Fitur 4: Video Library

Storage semua video yang pernah di-generate:
- Filter: by template, by date, by status (posted/draft)
- Search by caption keyword
- Bulk download
- Re-schedule (post lagi ke platform berbeda)
- Delete (free up storage quota)

## Fitur 5: Template Library (SEO Pages)

Strategi SEO yang sama seperti BalasChat: long-tail keywords.

```
/templates/
├── /promo-flash-sale-skincare
├── /testimonial-fnb
├── /product-showcase-fashion
└── ... (60 SEO pages: 6 template × 10 industri)
```

Setiap page:
- Preview video example (2-3 sample yang sudah generate)
- "Generate variasi Anda sendiri" CTA
- Industry-specific tips
- SEO-rich content 500-800 kata

## Freemium Logic Detail

### Free Tier

| Resource | Limit |
|---|---|
| Video/bulan | 5 |
| Storage | 100MB (video tersimpan 30 hari, auto-delete) |
| Schedule ahead | Max 7 hari |
| Watermark | Wajib (logo kecil pojok bawah + "Made with KontenKilat" di akhir caption) |
| Platforms | 1 (pilih IG atau TikTok, bukan both) |
| Analytics | Basic (views & likes only) |
| Templates | 3 dari 6 |
| Voice options | 2 dari 4 |

### Pro Tier (Rp 99.000/bulan)

| Resource | Limit |
|---|---|
| Video/bulan | 50 |
| Storage | 5GB (90 hari retention) |
| Schedule ahead | Unlimited |
| Watermark | Tidak ada |
| Platforms | IG + TikTok |
| Analytics | Full + AI insights + weekly email report |
| Templates | All 6 |
| Voice options | All 4 |
| Custom brand voice | Coming soon (Pro+ feature) |

### Pro+ Aiyoo Bundle (Rp 249.000/bulan)

- Semua fitur Pro
- + Aiyoo Starter plan (AI chatbot WhatsApp)
- Saving Rp 49.000/bulan vs beli terpisah

## User Account Tiers

```
Anonymous (browse only, demo video preview)
   ↓ (signup)
Free user (5 video/bulan)
   ↓ (upgrade)
Pro user (50 video/bulan)
   ↓ (cross-sell)
Pro+ Aiyoo user (full stack)
```

## Onboarding Flow (Critical untuk retention)

**Step 1 (signup):** Email + Google OAuth
**Step 2 (welcome):** "Apa industri Anda?" → personalize template recommendation
**Step 3 (first video):** Guided tour bikin video pertama (sample prompt diberikan)
**Step 4 (connect):** Connect IG/TikTok account (optional skip)
**Step 5 (schedule):** "Mau auto-post atau download dulu?"
**Step 6 (success):** Confetti animation + "Video pertama kamu siap! Lihat di Library"

## Critical Edge Cases

1. **Meta API gagal posting** → notify user, save sebagai draft, suggest manual post + download
2. **Video generation timeout >120s** → notify user, refund quota, retry background
3. **TikTok rejects content** (e.g., music copyright) → notify dengan reason, suggest re-generate
4. **User exceeds storage** → notify, force delete oldest before generate new
5. **Pro user downgrade** → keep generated videos, but quota reset to free, scheduled posts beyond 7 days canceled
6. **Token IG/TikTok expired** → notify user, redirect ke reconnect flow

## Analytics Events to Track

| Event | Critical? | Properties |
|---|---|---|
| `signup_complete` | Y | source, industry |
| `template_selected` | Y | template_id |
| `video_generate_started` | Y | template, industry |
| `video_generate_completed` | Y | duration_sec, success, generation_time_ms |
| `video_generate_failed` | Y | error_type |
| `video_downloaded` | Y | platform |
| `video_scheduled` | Y | platform, scheduled_at |
| `video_posted_success` | Y | platform, post_url |
| `video_posted_failed` | Y | platform, error |
| `pro_upgrade_clicked` | Y | placement |
| `pro_upgrade_completed` | Y | plan |
| `aiyoo_bundle_clicked` | Y | placement |
| `analytics_viewed` | N | section |
| `instagram_connected` | Y | - |
| `tiktok_connected` | Y | - |
