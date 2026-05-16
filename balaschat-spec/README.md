# BalasChat — Lead Magnet Tool untuk Aiyoo

**Status:** Build Spec v1.0
**Target Launch:** 30 hari dari hari ini
**Stack:** Next.js 14 + Supabase + Anthropic Claude API + Vercel

---

## Apa Itu BalasChat?

Free tool dengan 2 fitur utama yang dibungkus dalam 1 domain (`balaschat.aiyoo.id`):

1. **AI Reply Generator** — Generate 3 variasi balasan WhatsApp profesional dalam Bahasa Indonesia dari pertanyaan customer.
2. **ROI Calculator** — Hitung kerugian uang/bulan dari lambat balas chat customer.

Tujuan strategis: **lead magnet** yang menarik UMKM Indonesia (target market Aiyoo) → kumpulkan email + demonstrasi value → konversi ke paying customer Aiyoo.

## Target KPI 90 Hari Pasca-Launch

| Metrik | Target Bulan 1 | Target Bulan 3 |
|---|---|---|
| Unique visitor/bulan | 2.000 | 20.000 |
| Email leads | 200 | 3.000 |
| Aiyoo trial signup | 30 | 400 |
| Aiyoo paying customer | 5 | 60 |
| Estimated MRR contribution | Rp 1jt | Rp 18jt |

## Struktur Dokumen

| File | Isi |
|---|---|
| `01-product-spec.md` | Definisi produk, fitur, user flow, monetisasi |
| `02-architecture.md` | Tech stack, database schema, infrastruktur |
| `03-implementation.md` | Roadmap 4 minggu day-by-day |
| `04-code-critical.md` | Kode kritis: AI prompts, rate limiting, freemium logic |
| `05-launch-marketing.md` | Go-to-market 30 hari pertama |

## Quick Start untuk Developer

```bash
# 1. Clone template
npx create-next-app@latest balaschat --typescript --tailwind --app

# 2. Install dependencies kunci
cd balaschat
npm install @anthropic-ai/sdk @supabase/supabase-js @upstash/ratelimit @upstash/redis posthog-js resend

# 3. Setup env vars (lihat 02-architecture.md untuk daftar lengkap)
cp .env.example .env.local

# 4. Setup Supabase schema (jalankan SQL di 02-architecture.md)

# 5. Run dev
npm run dev
```

## Prinsip Desain

1. **Freemium dengan friksi terukur** — gratis cukup untuk merasakan magic, terbatas cukup untuk push upgrade
2. **Email gate progresif** — anonim → email → akun penuh (3 lapisan, kurangi drop-off)
3. **Tiap output = upsell opportunity** — setiap reply yang di-generate harus subtle promote Aiyoo
4. **SEO-first** — template library = ratusan halaman SEO long-tail
5. **Speed > polish** — launch v1 di hari 30, iterate dari data

## Diferensiasi vs Kompetitor

| Kompetitor | Kelemahan | Diferensiasi BalasChat |
|---|---|---|
| ChatGPT/Claude.ai generic | Output English-default, perlu prompt engineering | Output langsung Bahasa Indonesia natural untuk UMKM |
| Tool template WA gratis | Static template, nggak personal | AI generate kontekstual per pertanyaan |
| Wati/Qontak template | Bayar dari awal | Gratis dengan freemium clear |
| Tools korporat | UI rumit untuk UMKM | UI super sederhana, 1 layar |
