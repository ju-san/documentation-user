# 04 — Critical Code (Ready to Copy)

Kode di sini sudah production-ready untuk komponen yang paling tricky. Copy-paste, adjust dengan setup Anda.

## 1. Anthropic Client + Prompts

`lib/anthropic/client.ts`:

```typescript
import Anthropic from '@anthropic-ai/sdk';

if (!process.env.ANTHROPIC_API_KEY) {
  throw new Error('ANTHROPIC_API_KEY is required');
}

export const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

export const MODEL = process.env.ANTHROPIC_MODEL || 'claude-haiku-4-5-20251001';
```

`lib/anthropic/prompts.ts`:

```typescript
export const SYSTEM_PROMPT_GENERATOR = `Kamu adalah penulis customer service WhatsApp profesional untuk UMKM Indonesia.

TUGASMU: Generate 3 variasi balasan WhatsApp dalam Bahasa Indonesia.

ATURAN PENULISAN:
1. Pakai Bahasa Indonesia natural seperti pemilik toko online beneran (BUKAN bahasa baku textbook)
2. Sesuaikan dengan tone yang diminta:
   - "santai" → casual, akrab, boleh pakai "kak", "sis", "min", sedikit emoji oke
   - "semi-formal" → ramah tapi profesional, hindari emoji berlebihan
   - "formal" → bahasa baku, untuk B2B atau bisnis premium
3. KEEP IT SHORT — max 3 kalimat untuk santai, 5 kalimat untuk formal
4. Selalu akhiri dengan clear next action: pertanyaan, CTA, atau closing yang sopan
5. Convention Indonesia: format harga "Rp 150.000" (titik bukan koma), "kak" untuk sapaan umum
6. JANGAN pakai filler English: "anyway", "btw", "FYI" — pakai padanan Indonesia
7. Untuk pertanyaan harga: kasih info + ajak diskusi lanjut
8. Untuk komplain: empati dulu, baru solusi
9. Untuk konfirmasi order: restate item + harga + next step

TUJUAN BALASAN:
- "close_sale" → push ke transaksi, ciptakan urgency natural
- "answer_info" → kasih info jelas + ajak interaksi lanjut
- "follow_up" → reminder yang sopan, kasih reason to reply
- "handle_complaint" → empati + solusi konkret + recovery offer
- "confirm_order" → konfirmasi detail + next step jelas

OUTPUT FORMAT (HARUS JSON valid):
{
  "variations": [
    {
      "reply": "string — balasan utuh siap copy",
      "approach": "string — 1 kalimat jelasin pendekatan (direct/friendly/FOMO/empathetic)",
      "estimated_conversion": "Tinggi" | "Sedang" | "Rendah"
    },
    ... 3 variasi total
  ],
  "tips": "string — 1-2 kalimat tips kapan pakai variasi mana"
}

PENTING:
- HANYA output JSON, jangan ada text lain
- Pastikan JSON valid (escape quotes properly)
- 3 variasi WAJIB berbeda pendekatan, jangan duplikasi`;

interface GenerateInput {
  industry: string;
  message: string;
  tone: 'santai' | 'semi-formal' | 'formal';
  goal: 'close_sale' | 'answer_info' | 'follow_up' | 'handle_complaint' | 'confirm_order';
  additionalInfo?: string;
}

export function buildUserPrompt(input: GenerateInput): string {
  return `INDUSTRI: ${input.industry}
PESAN CUSTOMER: "${input.message}"
TONE: ${input.tone}
TUJUAN BALASAN: ${input.goal}
${input.additionalInfo ? `INFO TAMBAHAN: ${input.additionalInfo}` : ''}

Generate 3 variasi balasan sesuai aturan. Output JSON only.`;
}

// Industry-specific context untuk improve quality
export const INDUSTRY_CONTEXT: Record<string, string> = {
  'toko-baju': 'Fokus: size, warna, bahan, foto produk, ongkir',
  'skincare': 'Fokus: jenis kulit, ingredients, hasil, BPOM',
  'fnb': 'Fokus: menu, jam buka, pengiriman, GoFood/GrabFood',
  'klinik-kecantikan': 'Fokus: jenis treatment, dokter, jadwal, harga paket',
  'jasa-design': 'Fokus: scope, revisi, timeline, format file',
  // ... tambah sesuai content/industries.json
};
```

## 2. AI Generate API Route

`app/api/generate/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { anthropic, MODEL } from '@/lib/anthropic/client';
import { SYSTEM_PROMPT_GENERATOR, buildUserPrompt, INDUSTRY_CONTEXT } from '@/lib/anthropic/prompts';
import { checkAndIncrementQuota } from '@/lib/ratelimit';
import { getAnonymousId } from '@/lib/anonymous';
import { createServerSupabase } from '@/lib/supabase/server';

export const runtime = 'edge';

const InputSchema = z.object({
  industry: z.string().min(1).max(50),
  message: z.string().min(1).max(500),
  tone: z.enum(['santai', 'semi-formal', 'formal']),
  goal: z.enum(['close_sale', 'answer_info', 'follow_up', 'handle_complaint', 'confirm_order']),
  additional_info: z.string().max(300).optional(),
});

export async function POST(req: NextRequest) {
  const startTime = Date.now();
  
  try {
    // 1. Parse & validate input
    const body = await req.json();
    const parsed = InputSchema.safeParse(body);
    if (!parsed.success) {
      return NextResponse.json(
        { ok: false, error: 'INVALID_INPUT', issues: parsed.error.issues },
        { status: 400 }
      );
    }
    const input = parsed.data;

    // 2. Identify user (anonymous or authenticated)
    const supabase = createServerSupabase();
    const { data: { user } } = await supabase.auth.getUser();
    const anonymousId = user ? null : await getAnonymousId(req);
    const identifier = user?.id || anonymousId;
    
    if (!identifier) {
      return NextResponse.json({ ok: false, error: 'NO_IDENTITY' }, { status: 400 });
    }

    // 3. Check quota
    const tier = user ? (user.user_metadata?.is_pro ? 'pro' : 'user') : 'anonymous';
    const quotaResult = await checkAndIncrementQuota(identifier, tier);
    if (!quotaResult.allowed) {
      return NextResponse.json({
        ok: false,
        error: 'QUOTA_EXCEEDED',
        message: tier === 'anonymous' 
          ? 'Kuota harian habis. Masukkan email untuk lanjut gratis.'
          : 'Kuota harian habis. Upgrade ke Aiyoo untuk auto-reply unlimited.',
        next_action: tier === 'anonymous' ? 'email_gate' : 'aiyoo_upgrade',
        quota: quotaResult,
      }, { status: 429 });
    }

    // 4. Sanitize input (prompt injection prevention)
    const sanitizedMessage = sanitizeInput(input.message);
    const sanitizedInfo = input.additional_info ? sanitizeInput(input.additional_info) : undefined;
    
    // 5. Build prompt with industry context
    const industryContext = INDUSTRY_CONTEXT[input.industry] || '';
    const userPrompt = buildUserPrompt({
      industry: input.industry + (industryContext ? ` (${industryContext})` : ''),
      message: sanitizedMessage,
      tone: input.tone,
      goal: input.goal,
      additionalInfo: sanitizedInfo,
    });

    // 6. Call Claude API
    const response = await anthropic.messages.create({
      model: MODEL,
      max_tokens: 1024,
      system: SYSTEM_PROMPT_GENERATOR,
      messages: [{ role: 'user', content: userPrompt }],
    });

    const rawText = response.content[0].type === 'text' ? response.content[0].text : '';
    
    // 7. Parse JSON output (with fallback for malformed)
    let parsedOutput;
    try {
      const jsonMatch = rawText.match(/\{[\s\S]*\}/);
      parsedOutput = JSON.parse(jsonMatch ? jsonMatch[0] : rawText);
    } catch (err) {
      console.error('Failed to parse AI output:', rawText);
      return NextResponse.json({
        ok: false,
        error: 'AI_OUTPUT_INVALID',
        message: 'Terjadi error generate balasan. Silakan coba lagi.',
      }, { status: 500 });
    }

    // 8. Validate output structure
    if (!parsedOutput.variations || !Array.isArray(parsedOutput.variations) || parsedOutput.variations.length === 0) {
      return NextResponse.json({
        ok: false,
        error: 'AI_OUTPUT_INVALID',
      }, { status: 500 });
    }

    // 9. Save to DB (fire & forget — don't block response)
    const duration = Date.now() - startTime;
    void supabase.from('generations').insert({
      user_id: user?.id,
      anonymous_id: anonymousId,
      type: 'reply',
      input: input,
      output: parsedOutput,
      ip_address: req.headers.get('x-forwarded-for')?.split(',')[0],
      user_agent: req.headers.get('user-agent'),
      duration_ms: duration,
      success: true,
    });

    // 10. Return response
    return NextResponse.json({
      ok: true,
      data: parsedOutput,
      quota: quotaResult,
    });

  } catch (err: any) {
    console.error('Generate API error:', err);
    return NextResponse.json({
      ok: false,
      error: 'INTERNAL_ERROR',
      message: 'Terjadi kesalahan server. Silakan coba lagi.',
    }, { status: 500 });
  }
}

// Prevent prompt injection
function sanitizeInput(text: string): string {
  return text
    .replace(/\b(ignore|disregard|forget)\s+(previous|all|above)\s+(instructions?|prompts?|rules?)/gi, '[filtered]')
    .replace(/\b(system|assistant)\s*:\s*/gi, '[filtered]: ')
    .substring(0, 500);
}
```

## 3. Rate Limiting (Tier-Based)

`lib/ratelimit.ts`:

```typescript
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

const DAILY_LIMITS: Record<string, number> = {
  anonymous: 3,
  email: 10,
  user: 30,
  pro: 999,
};

export interface QuotaResult {
  allowed: boolean;
  used: number;
  limit: number;
  tier: string;
  resetAt: number; // unix timestamp
}

export async function checkAndIncrementQuota(
  identifier: string,
  tier: keyof typeof DAILY_LIMITS
): Promise<QuotaResult> {
  const today = new Date().toISOString().split('T')[0]; // YYYY-MM-DD
  const key = `quota:${tier}:${identifier}:${today}`;
  const limit = DAILY_LIMITS[tier];

  // Increment counter
  const used = await redis.incr(key);
  
  // Set expiry on first increment (25 hours to be safe across timezones)
  if (used === 1) {
    await redis.expire(key, 90000);
  }

  // Calculate reset time (midnight WIB = UTC+7)
  const now = new Date();
  const tomorrow = new Date(now);
  tomorrow.setUTCDate(now.getUTCDate() + 1);
  tomorrow.setUTCHours(17, 0, 0, 0); // 00:00 WIB = 17:00 UTC prev day
  const resetAt = Math.floor(tomorrow.getTime() / 1000);

  return {
    allowed: used <= limit,
    used,
    limit,
    tier,
    resetAt,
  };
}

export async function getCurrentQuota(
  identifier: string,
  tier: keyof typeof DAILY_LIMITS
): Promise<QuotaResult> {
  const today = new Date().toISOString().split('T')[0];
  const key = `quota:${tier}:${identifier}:${today}`;
  const used = (await redis.get<number>(key)) || 0;
  const limit = DAILY_LIMITS[tier];

  const now = new Date();
  const tomorrow = new Date(now);
  tomorrow.setUTCDate(now.getUTCDate() + 1);
  tomorrow.setUTCHours(17, 0, 0, 0);
  const resetAt = Math.floor(tomorrow.getTime() / 1000);

  return { allowed: used < limit, used, limit, tier, resetAt };
}
```

## 4. Anonymous Tracking

`lib/anonymous.ts`:

```typescript
import { NextRequest } from 'next/server';
import { cookies } from 'next/headers';

const ANON_COOKIE = 'bc_anon_id';

export async function getAnonymousId(req: NextRequest): Promise<string> {
  const cookieStore = await cookies();
  let id = cookieStore.get(ANON_COOKIE)?.value;
  
  if (!id) {
    id = generateId();
    cookieStore.set(ANON_COOKIE, id, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 60 * 60 * 24 * 365, // 1 year
    });
  }
  
  return id;
}

function generateId(): string {
  // Random + timestamp for uniqueness
  return `${Date.now().toString(36)}_${Math.random().toString(36).slice(2, 11)}`;
}
```

## 5. ROI Calculator Logic

`lib/calculator.ts`:

```typescript
// Conversion rates berdasarkan response time
// Source: industry benchmark + estimasi konservatif
const RESPONSE_TIME_CONVERSION: Record<string, number> = {
  '<5_min': 0.85,
  '5-30_min': 0.62,
  '30-60_min': 0.42,
  '1-3_hr': 0.21,
  '>3_hr': 0.08,
};

const AIYOO_RESPONSE_RATE = 0.85; // Aiyoo balas <5 menit, but factor in AI handling quality
const ORDER_FROM_ENGAGED = 0.40; // 40% engaged customer → order (UMKM industri avg)

interface CalculatorInput {
  chatsPerDay: number;
  avgOrderValue: number;
  currentResponseTime: keyof typeof RESPONSE_TIME_CONVERSION;
  hoursOpen: number;
}

export interface CalculatorResult {
  monthlyChats: number;
  afterHoursChats: number;
  
  currentEngaged: number;
  currentOrders: number;
  currentRevenue: number;
  
  aiyooEngaged: number;
  aiyooOrders: number;
  aiyooRevenue: number;
  
  lostRevenue: number;
  lostOrders: number;
  
  aiyooMonthlyCost: number;
  roiMultiplier: number;
  paybackDays: number;
}

export function calculateROI(input: CalculatorInput): CalculatorResult {
  const { chatsPerDay, avgOrderValue, currentResponseTime, hoursOpen } = input;
  
  const monthlyChats = chatsPerDay * 30;
  const afterHoursRatio = (24 - hoursOpen) / 24;
  const afterHoursChats = Math.round(monthlyChats * afterHoursRatio);
  
  const currentRate = RESPONSE_TIME_CONVERSION[currentResponseTime];
  
  // CURRENT: hanya bisa handle saat jam kerja
  const currentEngagedDuringHours = monthlyChats * (1 - afterHoursRatio) * currentRate;
  const currentEngagedAfterHours = monthlyChats * afterHoursRatio * RESPONSE_TIME_CONVERSION['>3_hr'];
  const currentEngaged = currentEngagedDuringHours + currentEngagedAfterHours;
  const currentOrders = currentEngaged * ORDER_FROM_ENGAGED;
  const currentRevenue = currentOrders * avgOrderValue;
  
  // WITH AIYOO: 24/7 fast response
  const aiyooEngaged = monthlyChats * AIYOO_RESPONSE_RATE;
  const aiyooOrders = aiyooEngaged * ORDER_FROM_ENGAGED;
  const aiyooRevenue = aiyooOrders * avgOrderValue;
  
  const lostRevenue = aiyooRevenue - currentRevenue;
  const lostOrders = aiyooOrders - currentOrders;
  
  const aiyooMonthlyCost = 199000; // Starter plan
  const roiMultiplier = Math.round((lostRevenue / aiyooMonthlyCost) * 10) / 10;
  const paybackDays = lostRevenue > 0 
    ? Math.ceil((aiyooMonthlyCost / lostRevenue) * 30)
    : 999;
  
  return {
    monthlyChats,
    afterHoursChats,
    currentEngaged: Math.round(currentEngaged),
    currentOrders: Math.round(currentOrders),
    currentRevenue: Math.round(currentRevenue),
    aiyooEngaged: Math.round(aiyooEngaged),
    aiyooOrders: Math.round(aiyooOrders),
    aiyooRevenue: Math.round(aiyooRevenue),
    lostRevenue: Math.round(lostRevenue),
    lostOrders: Math.round(lostOrders),
    aiyooMonthlyCost,
    roiMultiplier,
    paybackDays,
  };
}

export function formatRupiah(n: number): string {
  return 'Rp ' + n.toLocaleString('id-ID');
}
```

## 6. Generator Form Component (Skeleton)

`components/generator/GeneratorForm.tsx`:

```typescript
'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { RadioGroup, RadioGroupItem } from '@/components/ui/radio-group';
import { Label } from '@/components/ui/label';
import { ReplyCard } from './ReplyCard';
import { EmailGateModal } from './EmailGateModal';
import { trackEvent } from '@/lib/analytics';
import industries from '@/content/industries.json';

type Tone = 'santai' | 'semi-formal' | 'formal';
type Goal = 'close_sale' | 'answer_info' | 'follow_up' | 'handle_complaint' | 'confirm_order';

interface Variation {
  reply: string;
  approach: string;
  estimated_conversion: string;
}

export function GeneratorForm() {
  const [industry, setIndustry] = useState('');
  const [message, setMessage] = useState('');
  const [tone, setTone] = useState<Tone>('santai');
  const [goal, setGoal] = useState<Goal>('answer_info');
  const [additionalInfo, setAdditionalInfo] = useState('');
  
  const [loading, setLoading] = useState(false);
  const [variations, setVariations] = useState<Variation[] | null>(null);
  const [tips, setTips] = useState<string>('');
  const [error, setError] = useState<string | null>(null);
  const [showEmailGate, setShowEmailGate] = useState(false);

  const handleGenerate = async () => {
    if (!industry || !message) {
      setError('Mohon isi industri dan pesan customer');
      return;
    }
    
    setLoading(true);
    setError(null);
    trackEvent('generate_started', { industry, tone, goal });
    
    try {
      const res = await fetch('/api/generate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          industry,
          message,
          tone,
          goal,
          additional_info: additionalInfo || undefined,
        }),
      });
      
      const data = await res.json();
      
      if (!data.ok) {
        if (data.error === 'QUOTA_EXCEEDED' && data.next_action === 'email_gate') {
          setShowEmailGate(true);
          trackEvent('email_gate_shown', { trigger_count: data.quota?.used });
        } else {
          setError(data.message || 'Terjadi kesalahan. Coba lagi.');
        }
        return;
      }
      
      setVariations(data.data.variations);
      setTips(data.data.tips || '');
      trackEvent('generate_completed', { success: true });
      
    } catch (err) {
      setError('Koneksi error. Cek internet Anda.');
      trackEvent('generate_completed', { success: false });
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="space-y-6">
      <div className="grid gap-4">
        <div>
          <Label htmlFor="industry">Industri Toko Anda</Label>
          <Select value={industry} onValueChange={setIndustry}>
            <SelectTrigger id="industry">
              <SelectValue placeholder="Pilih industri..." />
            </SelectTrigger>
            <SelectContent>
              {industries.map((ind) => (
                <SelectItem key={ind.slug} value={ind.slug}>{ind.label}</SelectItem>
              ))}
            </SelectContent>
          </Select>
        </div>

        <div>
          <Label htmlFor="message">Pesan Customer</Label>
          <Textarea
            id="message"
            placeholder="Contoh: Halo kak, harga produk A berapa ya?"
            value={message}
            onChange={(e) => setMessage(e.target.value)}
            maxLength={500}
            rows={3}
          />
          <p className="text-xs text-muted-foreground mt-1">{message.length}/500</p>
        </div>

        <div>
          <Label>Tone Balasan</Label>
          <RadioGroup value={tone} onValueChange={(v) => setTone(v as Tone)} className="flex gap-4 mt-2">
            <div className="flex items-center space-x-2">
              <RadioGroupItem value="santai" id="t-santai" />
              <Label htmlFor="t-santai">Santai</Label>
            </div>
            <div className="flex items-center space-x-2">
              <RadioGroupItem value="semi-formal" id="t-semi" />
              <Label htmlFor="t-semi">Semi-formal</Label>
            </div>
            <div className="flex items-center space-x-2">
              <RadioGroupItem value="formal" id="t-formal" />
              <Label htmlFor="t-formal">Formal</Label>
            </div>
          </RadioGroup>
        </div>

        <div>
          <Label htmlFor="goal">Tujuan Balasan</Label>
          <Select value={goal} onValueChange={(v) => setGoal(v as Goal)}>
            <SelectTrigger id="goal">
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="answer_info">Jawab info</SelectItem>
              <SelectItem value="close_sale">Tutup penjualan</SelectItem>
              <SelectItem value="follow_up">Follow-up customer</SelectItem>
              <SelectItem value="handle_complaint">Tangani komplain</SelectItem>
              <SelectItem value="confirm_order">Konfirmasi order</SelectItem>
            </SelectContent>
          </Select>
        </div>

        <div>
          <Label htmlFor="info">Info Tambahan (opsional)</Label>
          <Textarea
            id="info"
            placeholder="Contoh: Produk A Rp 150.000, stok 5 pcs"
            value={additionalInfo}
            onChange={(e) => setAdditionalInfo(e.target.value)}
            maxLength={300}
            rows={2}
          />
        </div>

        <Button onClick={handleGenerate} disabled={loading} size="lg" className="w-full">
          {loading ? 'Generate...' : 'Generate Balasan'}
        </Button>

        {error && <p className="text-red-500 text-sm">{error}</p>}
      </div>

      {variations && (
        <div className="space-y-4">
          <h3 className="text-lg font-semibold">3 Variasi Balasan:</h3>
          {variations.map((v, i) => (
            <ReplyCard key={i} variation={v} index={i} />
          ))}
          {tips && (
            <div className="bg-blue-50 dark:bg-blue-950 p-4 rounded-lg text-sm">
              💡 <strong>Tips:</strong> {tips}
            </div>
          )}
        </div>
      )}

      <EmailGateModal open={showEmailGate} onOpenChange={setShowEmailGate} />
    </div>
  );
}
```

## 7. Reply Card with Copy + WA Share

`components/generator/ReplyCard.tsx`:

```typescript
'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Card } from '@/components/ui/card';
import { Copy, MessageCircle, ThumbsUp, ThumbsDown, RefreshCw } from 'lucide-react';
import { toast } from 'sonner';
import { trackEvent } from '@/lib/analytics';

interface Props {
  variation: {
    reply: string;
    approach: string;
    estimated_conversion: string;
  };
  index: number;
}

export function ReplyCard({ variation, index }: Props) {
  const [feedback, setFeedback] = useState<'up' | 'down' | null>(null);

  const handleCopy = async () => {
    await navigator.clipboard.writeText(variation.reply);
    toast.success('Balasan dicopy ke clipboard');
    trackEvent('reply_copied', { variation_index: index });
  };

  const handleWhatsApp = () => {
    const url = `https://wa.me/?text=${encodeURIComponent(variation.reply)}`;
    window.open(url, '_blank');
    trackEvent('reply_shared_wa', { variation_index: index });
  };

  return (
    <Card className="p-4">
      <div className="flex items-start justify-between gap-2 mb-3">
        <div>
          <span className="inline-block text-xs bg-teal-100 text-teal-800 px-2 py-1 rounded-full">
            Variasi {index + 1} • {variation.approach}
          </span>
          <span className="inline-block ml-2 text-xs text-muted-foreground">
            Konversi: {variation.estimated_conversion}
          </span>
        </div>
      </div>

      <p className="whitespace-pre-wrap text-base mb-4">{variation.reply}</p>

      <div className="flex flex-wrap gap-2">
        <Button size="sm" onClick={handleCopy}>
          <Copy className="w-4 h-4 mr-1" /> Copy
        </Button>
        <Button size="sm" variant="outline" onClick={handleWhatsApp}>
          <MessageCircle className="w-4 h-4 mr-1" /> Share via WA
        </Button>
        <Button
          size="sm"
          variant="ghost"
          onClick={() => setFeedback('up')}
          className={feedback === 'up' ? 'text-green-600' : ''}
        >
          <ThumbsUp className="w-4 h-4" />
        </Button>
        <Button
          size="sm"
          variant="ghost"
          onClick={() => setFeedback('down')}
          className={feedback === 'down' ? 'text-red-600' : ''}
        >
          <ThumbsDown className="w-4 h-4" />
        </Button>
      </div>
    </Card>
  );
}
```

## 8. Email Gate Modal

`components/generator/EmailGateModal.tsx`:

```typescript
'use client';

import { useState } from 'react';
import { Dialog, DialogContent, DialogTitle, DialogDescription } from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { trackEvent } from '@/lib/analytics';
import { toast } from 'sonner';

interface Props {
  open: boolean;
  onOpenChange: (open: boolean) => void;
}

export function EmailGateModal({ open, onOpenChange }: Props) {
  const [email, setEmail] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!email.match(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)) {
      toast.error('Email tidak valid');
      return;
    }
    
    setLoading(true);
    try {
      const res = await fetch('/api/leads', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, source: 'generator_gate' }),
      });
      const data = await res.json();
      
      if (data.ok) {
        trackEvent('email_submitted', { source: 'generator_gate' });
        toast.success('Sip! Kuota Anda terbuka 10 generate hari ini.');
        onOpenChange(false);
        // Trigger re-fetch or reload generation count
        window.location.reload();
      } else {
        toast.error(data.message || 'Gagal submit email');
      }
    } catch (err) {
      toast.error('Koneksi error');
    } finally {
      setLoading(false);
    }
  };

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent>
        <DialogTitle>Kuota harian habis 😅</DialogTitle>
        <DialogDescription>
          Masukkan email untuk lanjut gratis. Anda dapat 10 generate/hari + akses template library.
        </DialogDescription>
        
        <form onSubmit={handleSubmit} className="space-y-3 mt-4">
          <Input
            type="email"
            placeholder="email@bisnis.com"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
          <Button type="submit" disabled={loading} className="w-full">
            {loading ? 'Loading...' : 'Buka Kuota 10/hari'}
          </Button>
          <p className="text-xs text-muted-foreground text-center">
            Kami tidak spam. Cuma kirim tips marketing 1x/minggu. Unsubscribe kapan saja.
          </p>
        </form>
      </DialogContent>
    </Dialog>
  );
}
```

## 9. Analytics Helper

`lib/analytics.ts`:

```typescript
'use client';

import posthog from 'posthog-js';

let initialized = false;

export function initAnalytics() {
  if (typeof window === 'undefined' || initialized) return;
  
  posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
    api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST,
    person_profiles: 'identified_only',
    capture_pageview: true,
    capture_pageleave: true,
  });
  
  initialized = true;
}

export function trackEvent(name: string, properties?: Record<string, any>) {
  if (typeof window === 'undefined') return;
  posthog.capture(name, properties);
}

export function identifyUser(userId: string, traits?: Record<string, any>) {
  if (typeof window === 'undefined') return;
  posthog.identify(userId, traits);
}
```

## 10. Industry Data (Sample)

`content/industries.json`:

```json
[
  { "slug": "toko-baju", "label": "Toko Baju & Fashion", "icon": "👕" },
  { "slug": "skincare", "label": "Skincare & Kosmetik", "icon": "💄" },
  { "slug": "fnb", "label": "F&B / Restoran / Kafe", "icon": "🍽️" },
  { "slug": "klinik-kecantikan", "label": "Klinik Kecantikan", "icon": "✨" },
  { "slug": "barbershop-salon", "label": "Barbershop / Salon", "icon": "💇" },
  { "slug": "jasa-design", "label": "Jasa Desain Grafis", "icon": "🎨" },
  { "slug": "jasa-foto", "label": "Jasa Fotografi", "icon": "📷" },
  { "slug": "jasa-bersih", "label": "Jasa Bersih-bersih", "icon": "🧹" },
  { "slug": "elektronik", "label": "Toko Elektronik", "icon": "📱" },
  { "slug": "furniture", "label": "Furniture & Interior", "icon": "🪑" },
  { "slug": "olahraga", "label": "Toko Olahraga", "icon": "⚽" },
  { "slug": "buku", "label": "Toko Buku", "icon": "📚" },
  { "slug": "mainan", "label": "Toko Mainan", "icon": "🧸" },
  { "slug": "tas-aksesoris", "label": "Tas & Aksesoris", "icon": "👜" },
  { "slug": "sepatu", "label": "Toko Sepatu", "icon": "👟" },
  { "slug": "perhiasan", "label": "Perhiasan & Jam", "icon": "💍" },
  { "slug": "obat-herbal", "label": "Obat & Herbal", "icon": "💊" },
  { "slug": "kursus-online", "label": "Kursus Online / E-Learning", "icon": "🎓" },
  { "slug": "konsultan-bisnis", "label": "Konsultan Bisnis", "icon": "💼" },
  { "slug": "real-estate", "label": "Properti / Real Estate", "icon": "🏠" },
  { "slug": "ev-organizer", "label": "Event Organizer", "icon": "🎉" },
  { "slug": "vendor-pernikahan", "label": "Vendor Pernikahan", "icon": "💒" },
  { "slug": "travel", "label": "Travel Agent", "icon": "✈️" },
  { "slug": "rental-mobil", "label": "Rental Kendaraan", "icon": "🚗" },
  { "slug": "fitness-gym", "label": "Fitness / Gym", "icon": "💪" },
  { "slug": "tutor-les", "label": "Tutor / Les Privat", "icon": "👨‍🏫" },
  { "slug": "catering", "label": "Catering", "icon": "🍱" },
  { "slug": "florist", "label": "Florist / Toko Bunga", "icon": "💐" },
  { "slug": "kue-bakery", "label": "Kue & Bakery", "icon": "🎂" },
  { "slug": "fashion-muslim", "label": "Fashion Muslim", "icon": "🧕" },
  { "slug": "kerajinan", "label": "Kerajinan Tangan", "icon": "🎨" },
  { "slug": "obat-hewan", "label": "Pet Shop / Hewan", "icon": "🐶" },
  { "slug": "tanaman-hias", "label": "Tanaman Hias", "icon": "🌿" },
  { "slug": "alat-tulis", "label": "Alat Tulis Kantor", "icon": "✏️" },
  { "slug": "perlengkapan-bayi", "label": "Perlengkapan Bayi", "icon": "👶" },
  { "slug": "vape-rokok", "label": "Vape / Rokok Elektrik", "icon": "💨" },
  { "slug": "supplier-grosir", "label": "Supplier Grosir", "icon": "📦" },
  { "slug": "import-china", "label": "Import dari China", "icon": "🚢" },
  { "slug": "agen-pulsa", "label": "Agen Pulsa & PPOB", "icon": "📱" },
  { "slug": "laundry", "label": "Laundry", "icon": "🧺" },
  { "slug": "bengkel-motor", "label": "Bengkel Motor", "icon": "🔧" },
  { "slug": "bengkel-mobil", "label": "Bengkel Mobil", "icon": "🚗" },
  { "slug": "service-elektronik", "label": "Service Elektronik", "icon": "🔌" },
  { "slug": "dokter-praktek", "label": "Dokter / Praktek", "icon": "⚕️" },
  { "slug": "psikolog", "label": "Psikolog / Konselor", "icon": "🧠" },
  { "slug": "advokat", "label": "Advokat / Pengacara", "icon": "⚖️" },
  { "slug": "akuntan", "label": "Akuntan / Pajak", "icon": "🧾" },
  { "slug": "kontraktor", "label": "Kontraktor", "icon": "🏗️" },
  { "slug": "supplier-fnb", "label": "Supplier F&B", "icon": "🥘" },
  { "slug": "umkm-makanan", "label": "UMKM Makanan Rumahan", "icon": "🥟" }
]
```

## 11. Welcome Email Template

`lib/emails/welcome.tsx`:

```typescript
export function welcomeEmailHtml(name?: string): string {
  return `
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Welcome to BalasChat</title>
</head>
<body style="font-family: -apple-system, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; color: #333;">
  <h1 style="color: #00BFA6;">Hai ${name || 'kak'}! 👋</h1>
  
  <p>Makasih udah daftar BalasChat. Mulai sekarang Anda bisa:</p>
  
  <ul>
    <li>✅ Generate 10 balasan WA/hari (vs 3 sebelumnya)</li>
    <li>✅ Akses 50+ template industri</li>
    <li>✅ Simpan history balasan favorit</li>
  </ul>
  
  <p style="background: #f0fdfa; border-left: 4px solid #00BFA6; padding: 12px; margin: 24px 0;">
    💡 <strong>Tips minggu ini:</strong> Customer 5x lebih sering jadi order kalau dibalas dalam 5 menit. 
    Setup template balasan favorit Anda sekarang.
  </p>
  
  <a href="https://balaschat.aiyoo.id" style="display: inline-block; background: #00BFA6; color: white; padding: 12px 24px; text-decoration: none; border-radius: 8px; margin: 16px 0;">
    Mulai Generate Balasan
  </a>
  
  <hr style="margin: 32px 0; border: none; border-top: 1px solid #eee;">
  
  <p style="font-size: 14px; color: #666;">
    <strong>Capek copy-paste balasan?</strong><br>
    Aiyoo bisa otomatis balas customer Anda 24/7 di WhatsApp, tanpa Anda perlu pegang HP. 
    <a href="https://aiyoo.id?ref=balaschat_welcome">Coba gratis 14 hari →</a>
  </p>
  
  <p style="font-size: 12px; color: #999; margin-top: 24px;">
    Anda terima email ini karena daftar di balaschat.aiyoo.id<br>
    <a href="{{unsubscribe_url}}" style="color: #999;">Unsubscribe</a>
  </p>
</body>
</html>
  `;
}
```
