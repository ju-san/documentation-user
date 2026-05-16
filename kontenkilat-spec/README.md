# KontenKilat — AI Video Generator + Auto-Post untuk UMKM

**Status:** Build Spec v1.0
**Target Launch:** 45 hari (lebih lama dari BalasChat karena video pipeline + API approval)
**Stack:** Next.js 14 + Supabase + Anthropic Claude + ElevenLabs + Pexels + Remotion + Meta/TikTok API

---

## Apa Itu KontenKilat?

Free AI tool yang **otomatis bikin video marketing 15-30 detik untuk Instagram Reels & TikTok**, lengkap dengan scheduler dan analytics.

User cukup ketik: "Promo flash sale skincare diskon 50%" → KontenKilat generate:
- Script Bahasa Indonesia (AI)
- Voiceover natural (AI)
- Stock footage relevan (auto-pick)
- Caption + hashtag (AI)
- Auto-post sesuai jadwal ke IG & TikTok
- Analytics tracking otomatis

**Positioning:** Free dengan tier Pro berbayar, sekaligus jadi acquisition channel untuk Aiyoo (cross-sell ke user yang aktif marketing).

## Target KPI 90 Hari

| Metrik | Bulan 1 | Bulan 3 |
|---|---|---|
| Unique visitor/bln | 3.000 | 30.000 |
| Active free user | 300 | 3.500 |
| Paid Pro subscribers | 15 (Rp 1,5jt MRR) | 250 (Rp 25jt MRR) |
| Video generated/bulan | 1.500 | 25.000 |
| Aiyoo cross-sell | 5 | 80 |
| **Total MRR potensial** | **Rp 2,5jt** | **Rp 41jt** |

## Pricing Model

| Tier | Harga | Quota |
|---|---|---|
| **Free** | Rp 0 | 5 video/bulan, watermark KontenKilat, schedule 7 hari ahead |
| **Pro** | Rp 99.000/bln | 50 video/bulan, no watermark, schedule unlimited, analytics lengkap |
| **Pro+ Aiyoo Bundle** | Rp 249.000/bln | Pro features + Aiyoo Starter (cross-sell) |

## Diferensiasi vs Kompetitor

| Kompetitor | Kelemahan | Diferensiasi KontenKilat |
|---|---|---|
| Canva | Bukan AI generative, perlu skill design | One-prompt → siap post |
| CapCut | Manual editing, time-consuming | Otomatis full pipeline |
| Pictory/InVideo | English-first, mahal ($25+/bulan), nggak ada IG scheduler | Bahasa Indonesia native + scheduler built-in |
| Buffer/Later | Cuma scheduler, no content generation | Generate + schedule + analytics 1 tool |
| HeyGen/Synthesia | Hanya AI avatar, expensive | Faceless video lebih relevan untuk UMKM |

## Struktur Dokumen

| File | Isi |
|---|---|
| `01-product-spec.md` | Fitur, video templates, user flow, pricing detail |
| `02-architecture.md` | Tech stack, video pipeline, API integration, cost model |
| `03-implementation.md` | Roadmap 6 minggu day-by-day |
| `04-code-critical.md` | AI prompts script, Remotion composition, IG/TikTok API |
| `05-launch-marketing.md` | Go-to-market 30 hari pasca-launch |

## Critical Path Risk (Baca Dulu)

Tiga risk utama yang harus di-de-risk minggu pertama:

1. **Meta Graph API approval** — `pages_manage_posts` + `instagram_content_publish` butuh app review Meta (2-4 minggu). Apply hari ke-1, paralel develop.
2. **TikTok Content Posting API** — butuh approval (2-8 minggu) & TikTok Business account. Apply hari ke-1.
3. **Video generation cost** — full AI video bisa $0.50-5/video. Pakai **hybrid pipeline** (script AI + voice AI + stock footage + Remotion compose) untuk turunin ke ~$0.05-0.15/video.

Kalau approval Meta/TikTok lama, fallback: **"Download & post manual"** mode, tetap valuable.

## Quick Start

```bash
# Setup project
npx create-next-app@latest kontenkilat --typescript --tailwind --app

# Critical dependencies
npm install @anthropic-ai/sdk @supabase/supabase-js
npm install remotion @remotion/cli @remotion/lambda  # video composition
npm install elevenlabs                                # AI voice
# Pexels via direct fetch (no SDK needed)

# Apply Meta + TikTok API SEKARANG (paralel develop)
# - Meta: https://developers.facebook.com/apps/create/
# - TikTok: https://developers.tiktok.com/

# Setup env (lihat 02-architecture.md untuk lengkapnya)
cp .env.example .env.local

npm run dev
```

## Prinsip Desain

1. **Speed of generation matter** — target <60 detik dari prompt ke video siap
2. **Output langsung post-ready** — no "needs manual editing"
3. **Bahasa Indonesia native** — voiceover, caption, hashtag semua di-optimize untuk pasar Indonesia
4. **Free tier real value** — 5 video/bulan cukup buat UMKM kecil, beneran berguna
5. **Upgrade path natural** — yang aktif post, organic pengin Pro
