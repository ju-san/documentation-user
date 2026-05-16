# 01 — Product Spec

## Site Map

```
balaschat.aiyoo.id/
├── /                          # Landing page + AI Reply Generator (hero)
├── /kalkulator                # ROI Calculator
├── /template                  # Template library (SEO pages)
│   ├── /balasan-toko-baju
│   ├── /balasan-skincare
│   └── ... (50+ industries)
├── /dashboard                 # User dashboard (history, settings)
├── /pro                       # Upgrade page
├── /api/generate              # AI generation endpoint
├── /api/calculate             # ROI calculation endpoint
└── /api/leads                 # Lead capture endpoint
```

## Fitur 1: AI Reply Generator

### User Flow

```
Visit homepage
   ↓
See input form (no signup required)
   ↓
Fill: Industry + Customer Message + Tone + Goal
   ↓
Click "Generate"
   ↓
3 reply variations appear with copy buttons
   ↓
Free quota: 3 generates/day (by browser fingerprint + IP)
   ↓
4th generate → email gate modal: "Lanjut gratis dengan email"
   ↓
Email submitted → unlock 7 more generates/day
   ↓
11th generate → "Daftar akun gratis untuk unlimited + history"
   ↓
Account created → unlimited generates, history saved
   ↓
After 20 generates total → "Mau auto-balas WA tanpa copy-paste? Coba Aiyoo →"
```

### Input Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| Industri | Dropdown (50+ options) | Yes | Toko Baju, Skincare, F&B, Jasa, Klinik, etc |
| Pesan Customer | Textarea (max 500 char) | Yes | "Halo kak, harga produk A berapa?" |
| Tone | Radio (Santai / Semi-formal / Formal) | Yes | Default: Santai |
| Tujuan Balasan | Dropdown | Yes | Close sale / Jawab info / Follow-up / Tangani komplain / Konfirmasi order |
| Info Tambahan | Textarea (opsional) | No | "Produk A harganya Rp 150.000, stok 5 pcs" |

### Output

3 reply variations dalam JSON:

```json
{
  "variations": [
    {
      "reply": "Halo kak! Produk A kami harganya Rp 150.000. Saat ini ready stock 5 pcs. Mau langsung order berapa pcs?",
      "approach": "Direct: kasih info + push action",
      "estimated_conversion": "Tinggi"
    },
    {
      "reply": "Hai kak, makasih udah tanya 🙏 Produk A Rp 150.000/pcs ya. Ada warna/ukuran preferred?",
      "approach": "Friendly: bangun rapport dulu",
      "estimated_conversion": "Sedang"
    },
    {
      "reply": "Halo kak, untuk produk A harga Rp 150.000. Stok terbatas 5 pcs nih kak, mau diamankan dulu?",
      "approach": "FOMO: ciptakan urgensi",
      "estimated_conversion": "Tinggi"
    }
  ],
  "tips": "Untuk customer pertama kali, variasi #2 paling aman. Variasi #3 cocok kalau customer udah keliatan minat."
}
```

### Action Buttons per Reply

- 📋 **Copy** — copy ke clipboard
- 💬 **Share via WA** — buka WhatsApp dengan reply pre-filled (wa.me/?text=...)
- 👍 / 👎 — feedback untuk improve AI (training data)
- 🔄 **Regenerate** — generate ulang variasi ini saja

## Fitur 2: ROI Calculator

### User Flow

```
Visit /kalkulator
   ↓
4 input fields, ada tooltips bantu isi
   ↓
Real-time calculation di sidebar (debounce 500ms)
   ↓
Show result: Big number "Anda kehilangan Rp X juta/bulan"
   ↓
Visual: donut chart "Order tertangkap vs hilang"
   ↓
Email gate: "Mau laporan detail + benchmark industri? Masukkan email"
   ↓
Email submitted → unlock PDF report + 3 tips email follow-up
   ↓
CTA besar: "Hentikan kebocoran ini → Coba Aiyoo gratis 14 hari"
```

### Input Fields

| Field | Type | Default | Validasi |
|---|---|---|---|
| Chat masuk/hari (estimasi) | Number slider | 50 | 1–1000 |
| Rata-rata nilai order (Rp) | Number | 150.000 | 10rb–10jt |
| Berapa lama Anda balas chat? | Dropdown | 30 menit | <5 min, 5-30 min, 30 min - 1 jam, 1-3 jam, >3 jam |
| Jam operasional balas chat | Range | 09:00 - 21:00 | Standar UMKM |

### Calculation Logic

```typescript
// Konstanta riset (replace dengan data riil kalau ada)
const RESPONSE_TIME_CONVERSION = {
  '<5_min': 0.85,      // 85% inquiry jadi engagement
  '5-30_min': 0.62,
  '30-60_min': 0.42,
  '1-3_hr': 0.21,
  '>3_hr': 0.08
};

const AIYOO_CONVERSION = 0.85; // Aiyoo response <5 min → 85%

function calculateLoss({chatsPerDay, avgOrderValue, currentResponseTime, hoursOpen}) {
  const monthlyChats = chatsPerDay * 30;
  const afterHoursChats = monthlyChats * ((24 - hoursOpen) / 24); // Chat di luar jam kerja
  
  const currentRate = RESPONSE_TIME_CONVERSION[currentResponseTime];
  const aiyooRate = AIYOO_CONVERSION;
  
  // Hari ini: orders tertangkap
  const currentOrders = monthlyChats * currentRate * 0.4; // 40% engaged → ordered (industry avg)
  const currentRevenue = currentOrders * avgOrderValue;
  
  // Dengan Aiyoo: orders tertangkap
  const aiyooOrders = monthlyChats * aiyooRate * 0.4;
  const aiyooRevenue = aiyooOrders * avgOrderValue;
  
  const lostRevenue = aiyooRevenue - currentRevenue;
  
  return {
    monthlyChats,
    afterHoursChats: Math.round(afterHoursChats),
    currentRevenue: Math.round(currentRevenue),
    aiyooRevenue: Math.round(aiyooRevenue),
    lostRevenue: Math.round(lostRevenue),
    lostOrders: Math.round(aiyooOrders - currentOrders),
    aiyooMonthlyCost: 199000, // Starter plan
    roi: Math.round((lostRevenue / 199000) * 100) / 100, // berapa kali balik modal
  };
}
```

### Output Example

```
🔴 Anda kehilangan:
   Rp 4.250.000 per bulan
   ≈ 28 order yang seharusnya jadi

💡 Dengan Aiyoo:
   Rp 199.000/bulan
   ROI: 21x lipat dalam 1 bulan

[Bayangan visual: donut chart 65% hilang / 35% tertangkap]
[Tombol besar hijau: "Coba Aiyoo Gratis 14 Hari →"]
```

## Fitur 3: Template Library (SEO Pages)

Static pages untuk long-tail SEO. Tiap halaman = 1 industri × 5-10 use case.

### Struktur URL

```
/template/[industri]/[use-case]

Contoh:
/template/toko-baju/balas-tanya-harga
/template/skincare/handle-komplain
/template/fnb/konfirmasi-order
/template/jasa-design/follow-up-prospek
```

### Konten per Page

```markdown
# [Use Case]: Template Balasan WhatsApp untuk [Industri]

[Intro paragraph 150-300 kata, keyword-rich tapi natural]

## 5 Template Siap Pakai

### Template 1: [Skenario]
"[Template text]"

**Kapan dipakai:** [explanation]
**Tips:** [explanation]

[... 5 templates ...]

## Mau Tanpa Copy-Paste?
[Embedded CTA box → AI Reply Generator + Aiyoo upsell]

## Tips Tambahan untuk [Industri]
[200-300 kata tips]
```

**Target:** 50 industri × 10 use case = 500 SEO pages.
**Bangun cara:** Pakai AI generate konten awal, edit manual untuk top 20 prioritas.

## Pricing & Freemium Logic

### Tier Free (BalasChat)

| Status User | Quota Generate/Hari | Fitur |
|---|---|---|
| Anonymous | 3 | Generate only, no save |
| Email-verified | 10 | + Last 7 history |
| Full account | 30 | + Unlimited history, custom industries |

### Tier Pro (BalasChat)

Sebenarnya, **kita TIDAK menjual Pro tier di BalasChat**. BalasChat = full free.

Upsell-nya langsung ke **Aiyoo** (produk utama). Strategi:
- Gratis pakai BalasChat = sample value Aiyoo
- Setiap 5 generate, banner "Lelah copy-paste? Aiyoo otomatis balas 24/7 →"
- Email sequence: hari 3 & 7 promote Aiyoo trial

**Alternatif** kalau mau revenue langsung dari BalasChat:
- BalasChat Pro Rp 49rb/bulan: unlimited + brand voice training + bulk generate
- Tapi rekomendasi: keep free, fokus konversi ke Aiyoo (LTV jauh lebih tinggi)

## Lead Capture Strategy

3 lapisan email gate:

### Lapisan 1: Soft Gate (setelah 3 generates)
Modal: "Masukkan email untuk lanjut gratis"
- Optional skip → tapi reset besok
- Submit → unlock 10/hari + simpan history 7 hari

### Lapisan 2: Account Gate (setelah 10 generates)
Modal: "Daftar gratis untuk unlimited + history permanen"
- Google one-click login
- Email + password

### Lapisan 3: Aiyoo Upsell (setelah 20 generates total)
Banner persistent + email sequence
- "Anda udah generate 20 balasan minggu ini. Capek copy-paste? Aiyoo otomatis."
- CTA: "Coba Aiyoo gratis 14 hari"

## Analytics Events to Track

Critical events untuk optimasi funnel:

| Event | Properties | Trigger |
|---|---|---|
| `page_view` | path, referrer | Page load |
| `generate_started` | industry, tone, goal | Click Generate |
| `generate_completed` | success, duration_ms | API response |
| `reply_copied` | variation_index | Click copy button |
| `reply_shared_wa` | variation_index | Click WA share |
| `email_gate_shown` | trigger_count | Modal shown |
| `email_submitted` | source | Form submit |
| `account_created` | source | Signup complete |
| `aiyoo_cta_clicked` | placement | Click upsell |
| `calculator_completed` | inputs, output_lost_revenue | ROI calc done |
