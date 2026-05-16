# 04 — Critical Code (Ready to Copy)

Bagian paling tricky di KontenKilat: AI script prompt + Remotion composition + Platform API integration. Kode di sini production-ready.

## 1. AI Script Generator

`lib/ai/prompts.ts`:

```typescript
export const SCRIPT_SYSTEM_PROMPT = `Kamu adalah scriptwriter video TikTok/Reels untuk UMKM Indonesia.

TUGASMU: Generate script video pendek 15-30 detik dengan struktur scene-by-scene.

ATURAN UMUM:
1. Bahasa Indonesia natural, conversational, sesuai gaya creator TikTok/IG
2. Hook 3 detik pertama HARUS kuat (pertanyaan provokatif, statement bold, atau visual surprise)
3. Tiap scene 3-6 detik, max 12 kata per scene (untuk fit voiceover speed)
4. Akhiri dengan clear CTA
5. Vocabulary sesuai target: UMKM/customer e-commerce Indonesia
6. JANGAN bahasa baku textbook, pakai bahasa Instagram-native

STRUKTUR PER TEMPLATE:

PROMO_FLASH (15 detik, 3-4 scenes):
- Scene 1 (3s): HOOK - "STOP! [statement urgent]"
- Scene 2 (5s): OFFER - detail diskon/promo
- Scene 3 (4s): URGENCY - alasan harus cepat
- Scene 4 (3s): CTA - cara order

PRODUCT_SHOWCASE (30 detik, 5-6 scenes):
- Scene 1 (4s): HOOK - problem yang dialami target
- Scene 2 (5s): INTRODUCE - perkenalkan produk
- Scene 3 (5s): BENEFIT 1
- Scene 4 (5s): BENEFIT 2
- Scene 5 (5s): SOCIAL PROOF
- Scene 6 (6s): CTA

TESTIMONIAL (20s, 4 scenes):
- Scene 1 (3s): HOOK - "Customer ini awalnya..."
- Scene 2 (6s): PROBLEM customer
- Scene 3 (8s): SOLUTION + RESULT
- Scene 4 (3s): CTA - "Mau hasil yang sama?"

TIPS_TRICKS (30s, 5 scenes):
- Scene 1 (4s): HOOK - "3 cara [achieve outcome]"
- Scene 2-4 (each 7s): Tip 1, 2, 3
- Scene 5 (5s): CTA + brand mention

BEHIND_SCENES (25s, 4 scenes):
- Scene 1 (4s): HOOK - "Lihat gimana kita..."
- Scene 2 (8s): PROCESS step 1
- Scene 3 (8s): PROCESS step 2 + reveal
- Scene 4 (5s): CTA

QUOTE (15s, 2-3 scenes):
- Scene 1 (3s): SETUP context
- Scene 2 (8s): MAIN QUOTE
- Scene 3 (4s): REFLECTION + brand

KETENTUAN OUTPUT (HARUS JSON valid):
{
  "scenes": [
    {
      "text": "string - voiceover text (Bahasa Indonesia)",
      "duration_sec": number,
      "b_roll_keywords": ["keyword1", "keyword2"], 
      "text_overlay": "string - text muncul di layar (boleh berbeda dari voiceover, lebih singkat)",
      "emphasis": "normal" | "bold" | "highlight"
    }
  ],
  "voiceover_full_text": "string - gabungan semua scene text",
  "instagram_caption": "string - caption IG (2-3 paragraf, dengan line break, emoji secukupnya, mention CTA)",
  "tiktok_caption": "string - caption TikTok (1-2 kalimat singkat punchy)",
  "hashtags": ["#tag1", "#tag2", ...] // 10-15 hashtag mix mass + niche
}

PENTING:
- Output ONLY valid JSON, no markdown wrapper
- b_roll_keywords HARUS dalam bahasa English (untuk search Pexels)
- Total duration scenes harus sesuai template (PROMO_FLASH=15s, dst)
- voiceover text harus natural saat dibaca, bukan formal`;

interface GenerateScriptInput {
  template: 'promo_flash' | 'product_showcase' | 'testimonial' | 'tips_tricks' | 'behind_scenes' | 'quote';
  prompt: string;
  industry: string;
  tone: 'energetic' | 'profesional' | 'santai' | 'edukatif';
  cta?: string;
}

export function buildScriptPrompt(input: GenerateScriptInput): string {
  return `TEMPLATE: ${input.template.toUpperCase()}
INDUSTRI: ${input.industry}
TONE: ${input.tone}
CTA: ${input.cta || 'Order sekarang via link di bio'}
KONTEN YANG DIINGINKAN:
"${input.prompt}"

Generate script video sesuai aturan template ${input.template.toUpperCase()}. Output JSON only.`;
}
```

`lib/ai/script-generator.ts`:

```typescript
import { anthropic, MODEL } from './client';
import { SCRIPT_SYSTEM_PROMPT, buildScriptPrompt } from './prompts';
import { z } from 'zod';

const SceneSchema = z.object({
  text: z.string(),
  duration_sec: z.number().min(1).max(15),
  b_roll_keywords: z.array(z.string()),
  text_overlay: z.string(),
  emphasis: z.enum(['normal', 'bold', 'highlight']).default('normal'),
});

const ScriptSchema = z.object({
  scenes: z.array(SceneSchema).min(2).max(6),
  voiceover_full_text: z.string(),
  instagram_caption: z.string(),
  tiktok_caption: z.string(),
  hashtags: z.array(z.string()).min(5).max(20),
});

export type Script = z.infer<typeof ScriptSchema>;

export async function generateScript(input: {
  template: string;
  prompt: string;
  industry: string;
  tone: string;
  cta?: string;
}): Promise<Script> {
  const response = await anthropic.messages.create({
    model: MODEL,
    max_tokens: 2048,
    system: SCRIPT_SYSTEM_PROMPT,
    messages: [{ role: 'user', content: buildScriptPrompt(input as any) }],
  });

  const rawText = response.content[0].type === 'text' ? response.content[0].text : '';
  const jsonMatch = rawText.match(/\{[\s\S]*\}/);
  if (!jsonMatch) throw new Error('No JSON in AI output');
  
  const parsed = JSON.parse(jsonMatch[0]);
  return ScriptSchema.parse(parsed);
}
```

## 2. ElevenLabs Voice Generation

`lib/voice/elevenlabs.ts`:

```typescript
import { ElevenLabsClient } from 'elevenlabs';

const client = new ElevenLabsClient({
  apiKey: process.env.ELEVENLABS_API_KEY!,
});

const VOICE_IDS = {
  M1: process.env.ELEVENLABS_VOICE_M1!,
  M2: process.env.ELEVENLABS_VOICE_M2!,
  F1: process.env.ELEVENLABS_VOICE_F1!,
  F2: process.env.ELEVENLABS_VOICE_F2!,
};

export async function generateVoiceover(
  text: string,
  voiceId: keyof typeof VOICE_IDS = 'F1'
): Promise<Buffer> {
  const audioStream = await client.textToSpeech.convert(VOICE_IDS[voiceId], {
    text,
    model_id: 'eleven_multilingual_v2',
    voice_settings: {
      stability: 0.5,
      similarity_boost: 0.75,
      style: 0.3,
      use_speaker_boost: true,
    },
    output_format: 'mp3_44100_128',
  });

  // Convert stream to buffer
  const chunks: Buffer[] = [];
  for await (const chunk of audioStream) {
    chunks.push(Buffer.from(chunk));
  }
  return Buffer.concat(chunks);
}

export async function generateVoiceoverPerScene(
  scenes: { text: string; duration_sec: number }[],
  voiceId: keyof typeof VOICE_IDS
): Promise<Array<{ scene_index: number; audio_buffer: Buffer; duration_sec: number }>> {
  // Generate parallel
  const results = await Promise.all(
    scenes.map(async (scene, index) => ({
      scene_index: index,
      audio_buffer: await generateVoiceover(scene.text, voiceId),
      duration_sec: scene.duration_sec,
    }))
  );
  return results;
}
```

## 3. Pexels Footage Fetcher

`lib/footage/pexels.ts`:

```typescript
const PEXELS_BASE = 'https://api.pexels.com/videos';

interface PexelsVideo {
  id: number;
  duration: number;
  video_files: Array<{
    id: number;
    quality: string;
    file_type: string;
    width: number;
    height: number;
    link: string;
  }>;
}

export async function searchPortraitVideos(
  query: string,
  limit: number = 5
): Promise<string[]> {
  const url = `${PEXELS_BASE}/search?query=${encodeURIComponent(query)}&orientation=portrait&size=medium&per_page=${limit}`;
  
  const res = await fetch(url, {
    headers: { Authorization: process.env.PEXELS_API_KEY! },
  });
  
  if (!res.ok) {
    console.error('Pexels error:', await res.text());
    return [];
  }
  
  const data = await res.json();
  
  return data.videos
    .map((v: PexelsVideo) => {
      // Prefer HD portrait file
      const hdFile = v.video_files
        .filter(f => f.height >= 1280 && f.width <= f.height)
        .sort((a, b) => b.height - a.height)[0];
      return hdFile?.link;
    })
    .filter(Boolean)
    .slice(0, limit);
}

export async function fetchFootageForScenes(
  scenes: { b_roll_keywords: string[] }[]
): Promise<string[][]> {
  return Promise.all(
    scenes.map(async (scene) => {
      const keyword = scene.b_roll_keywords[0]; // primary
      const videos = await searchPortraitVideos(keyword, 3);
      
      // Fallback ke keyword #2 kalau kosong
      if (videos.length === 0 && scene.b_roll_keywords[1]) {
        return await searchPortraitVideos(scene.b_roll_keywords[1], 3);
      }
      return videos;
    })
  );
}
```

## 4. Remotion Composition (PromoFlash)

`remotion/compositions/PromoFlash.tsx`:

```typescript
import {
  AbsoluteFill,
  Series,
  Video,
  Audio,
  useVideoConfig,
  interpolate,
  useCurrentFrame,
  spring,
} from 'remotion';

interface Scene {
  text: string;
  duration_sec: number;
  text_overlay: string;
  emphasis: 'normal' | 'bold' | 'highlight';
  video_url: string;
  audio_url: string;
}

export interface PromoFlashProps {
  scenes: Scene[];
  music_url: string;
  watermark: boolean;
}

export const PromoFlash: React.FC<PromoFlashProps> = ({ scenes, music_url, watermark }) => {
  const { fps } = useVideoConfig();
  const totalDuration = scenes.reduce((sum, s) => sum + s.duration_sec, 0);
  
  return (
    <AbsoluteFill style={{ backgroundColor: '#000' }}>
      {/* Background music throughout */}
      <Audio src={music_url} volume={0.15} />
      
      <Series>
        {scenes.map((scene, i) => (
          <Series.Sequence key={i} durationInFrames={Math.round(scene.duration_sec * fps)}>
            <SceneRenderer scene={scene} />
          </Series.Sequence>
        ))}
      </Series>
      
      {watermark && <Watermark />}
    </AbsoluteFill>
  );
};

const SceneRenderer: React.FC<{ scene: Scene }> = ({ scene }) => {
  const frame = useCurrentFrame();
  const { fps, width, height } = useVideoConfig();
  
  // Text animation: spring scale in
  const textScale = spring({
    frame: frame - 5,
    fps,
    config: { damping: 12, stiffness: 200 },
  });
  
  // Text fade out di akhir
  const textOpacity = interpolate(
    frame,
    [0, 8, scene.duration_sec * fps - 10, scene.duration_sec * fps],
    [0, 1, 1, 0],
    { extrapolateLeft: 'clamp', extrapolateRight: 'clamp' }
  );

  return (
    <AbsoluteFill>
      {/* Background video */}
      <Video src={scene.video_url} muted style={{ objectFit: 'cover', width: '100%', height: '100%' }} />
      
      {/* Dark overlay for readability */}
      <AbsoluteFill style={{ backgroundColor: 'rgba(0,0,0,0.4)' }} />
      
      {/* Voiceover audio */}
      <Audio src={scene.audio_url} />
      
      {/* Text overlay */}
      <AbsoluteFill style={{
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        padding: '0 60px',
        transform: `scale(${textScale})`,
        opacity: textOpacity,
      }}>
        <h1 style={{
          fontFamily: 'Inter, sans-serif',
          fontSize: scene.emphasis === 'bold' ? 96 : 72,
          fontWeight: 900,
          color: scene.emphasis === 'highlight' ? '#FFD700' : '#FFFFFF',
          textAlign: 'center',
          lineHeight: 1.2,
          textShadow: '0 4px 12px rgba(0,0,0,0.8)',
          maxWidth: '90%',
        }}>
          {scene.text_overlay}
        </h1>
      </AbsoluteFill>
    </AbsoluteFill>
  );
};

const Watermark: React.FC = () => (
  <div style={{
    position: 'absolute',
    bottom: 30,
    right: 30,
    background: 'rgba(0,0,0,0.5)',
    padding: '8px 16px',
    borderRadius: 8,
    color: 'white',
    fontSize: 24,
    fontWeight: 600,
    fontFamily: 'Inter',
  }}>
    KontenKilat.id
  </div>
);
```

`remotion/Root.tsx`:

```typescript
import { Composition } from 'remotion';
import { PromoFlash } from './compositions/PromoFlash';
// import other compositions...

export const RemotionRoot: React.FC = () => {
  return (
    <>
      <Composition
        id="PromoFlash"
        component={PromoFlash}
        durationInFrames={450}  // 15 sec @ 30fps, akan di-override per render
        fps={30}
        width={1080}
        height={1920}
        defaultProps={{
          scenes: [],
          music_url: '',
          watermark: true,
        }}
      />
      {/* Register other compositions */}
    </>
  );
};
```

## 5. Remotion Lambda Render Trigger

`lib/render/remotion-lambda.ts`:

```typescript
import { renderMediaOnLambda, getRenderProgress } from '@remotion/lambda/client';

export async function triggerRender(input: {
  composition: string;
  inputProps: any;
  outputKey: string;
}): Promise<{ renderId: string; bucketName: string }> {
  const result = await renderMediaOnLambda({
    region: process.env.AWS_REGION as 'ap-southeast-1',
    functionName: process.env.AWS_LAMBDA_FUNCTION!,
    serveUrl: process.env.REMOTION_SERVE_URL!,
    composition: input.composition,
    inputProps: input.inputProps,
    codec: 'h264',
    imageFormat: 'jpeg',
    privacy: 'public',
    outName: {
      bucketName: process.env.R2_BUCKET!,
      key: input.outputKey,
    },
    maxRetries: 1,
  });

  return {
    renderId: result.renderId,
    bucketName: result.bucketName,
  };
}

export async function pollRenderProgress(renderId: string, bucketName: string) {
  return await getRenderProgress({
    renderId,
    bucketName,
    region: process.env.AWS_REGION as 'ap-southeast-1',
    functionName: process.env.AWS_LAMBDA_FUNCTION!,
  });
}
```

## 6. Generation Orchestrator (Main API)

`app/api/generate/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { createServerSupabase } from '@/lib/supabase/server';
import { generateScript } from '@/lib/ai/script-generator';
import { generateVoiceoverPerScene } from '@/lib/voice/elevenlabs';
import { fetchFootageForScenes } from '@/lib/footage/pexels';
import { pickMusic } from '@/lib/music/library';
import { uploadToR2 } from '@/lib/storage/r2';
import { triggerRender } from '@/lib/render/remotion-lambda';
import { checkQuota } from '@/lib/quota';

const InputSchema = z.object({
  template: z.enum(['promo_flash', 'product_showcase', 'testimonial', 'tips_tricks', 'behind_scenes', 'quote']),
  prompt: z.string().min(10).max(500),
  industry: z.string(),
  tone: z.enum(['energetic', 'profesional', 'santai', 'edukatif']),
  voice_id: z.enum(['M1', 'M2', 'F1', 'F2']),
  music_vibe: z.enum(['trendy', 'energetic', 'calm', 'dramatic', 'fun', 'romantic']),
  cta: z.string().optional(),
});

export async function POST(req: NextRequest) {
  const supabase = createServerSupabase();
  const { data: { user } } = await supabase.auth.getUser();
  
  if (!user) return NextResponse.json({ error: 'UNAUTHORIZED' }, { status: 401 });

  const body = await req.json();
  const parsed = InputSchema.safeParse(body);
  if (!parsed.success) return NextResponse.json({ error: 'INVALID_INPUT' }, { status: 400 });
  const input = parsed.data;

  // 1. Check quota
  const quota = await checkQuota(user.id);
  if (!quota.allowed) {
    return NextResponse.json({
      error: 'QUOTA_EXCEEDED',
      message: 'Kuota bulanan habis. Upgrade ke Pro untuk 50 video/bulan.',
      quota,
    }, { status: 429 });
  }

  // 2. Create video record (status: queued)
  const { data: video, error } = await supabase
    .from('videos')
    .insert({
      user_id: user.id,
      template_id: input.template,
      status: 'queued',
      prompt: input.prompt,
      industry: input.industry,
      tone: input.tone,
      voice_id: input.voice_id,
      music_id: input.music_vibe, // resolve to actual track later
      has_watermark: quota.tier === 'free',
    })
    .select()
    .single();
    
  if (error) return NextResponse.json({ error: 'DB_ERROR' }, { status: 500 });

  // 3. Trigger background job (don't await)
  void processVideoGeneration(video.id, input, quota.tier === 'free');

  return NextResponse.json({
    ok: true,
    video_id: video.id,
    estimated_time_sec: 60,
  });
}

async function processVideoGeneration(
  videoId: string,
  input: z.infer<typeof InputSchema>,
  hasWatermark: boolean
) {
  const supabase = createServerSupabase();
  
  try {
    await supabase.from('videos').update({ status: 'rendering' }).eq('id', videoId);
    
    // Step 1: Generate script
    const script = await generateScript(input);
    
    // Step 2 + 3 parallel: voice + footage
    const [voiceFiles, footageUrls, musicUrl] = await Promise.all([
      generateVoiceoverPerScene(script.scenes, input.voice_id),
      fetchFootageForScenes(script.scenes),
      pickMusic(input.music_vibe),
    ]);
    
    // Step 4: Upload voice files to R2
    const audioUrls = await Promise.all(
      voiceFiles.map((v, i) => 
        uploadToR2(v.audio_buffer, `audio/${videoId}/scene-${i}.mp3`, 'audio/mp3')
      )
    );
    
    // Step 5: Compose Remotion props
    const scenes = script.scenes.map((s, i) => ({
      text: s.text,
      duration_sec: s.duration_sec,
      text_overlay: s.text_overlay,
      emphasis: s.emphasis,
      video_url: footageUrls[i][0] || footageUrls[0][0], // fallback
      audio_url: audioUrls[i],
    }));
    
    const compositionId = templateToComposition(input.template);
    const totalDurationFrames = Math.round(
      scenes.reduce((sum, s) => sum + s.duration_sec, 0) * 30
    );
    
    // Step 6: Trigger render
    const renderResult = await triggerRender({
      composition: compositionId,
      inputProps: {
        scenes,
        music_url: musicUrl,
        watermark: hasWatermark,
        durationInFrames: totalDurationFrames,
      },
      outputKey: `videos/${videoId}.mp4`,
    });
    
    // Step 7: Poll until complete (or use webhook for production)
    let progress;
    let attempts = 0;
    do {
      await new Promise(r => setTimeout(r, 3000));
      progress = await pollRenderProgress(renderResult.renderId, renderResult.bucketName);
      attempts++;
      if (attempts > 40) throw new Error('Render timeout'); // 2 min max
    } while (!progress.done);
    
    if (progress.errors.length > 0) throw new Error(progress.errors[0].message);
    
    // Step 8: Update DB with final video URL
    const videoUrl = `${process.env.R2_PUBLIC_URL}/videos/${videoId}.mp4`;
    
    await supabase.from('videos').update({
      status: 'completed',
      video_url: videoUrl,
      thumbnail_url: videoUrl.replace('.mp4', '-thumb.jpg'), // generate separately
      duration_sec: Math.round(scenes.reduce((s, sc) => s + sc.duration_sec, 0)),
      size_mb: progress.outputSizeInBytes ? progress.outputSizeInBytes / 1024 / 1024 : null,
      script: script,
      caption_instagram: script.instagram_caption,
      caption_tiktok: script.tiktok_caption,
      hashtags: script.hashtags,
      completed_at: new Date().toISOString(),
    }).eq('id', videoId);
    
    // Step 9: Increment quota
    await supabase.rpc('increment_quota', { user_id_param: (await supabase.auth.getUser()).data.user?.id });
    
  } catch (err: any) {
    console.error('Generation failed:', err);
    await supabase.from('videos').update({
      status: 'failed',
      error_message: err.message,
    }).eq('id', videoId);
  }
}

function templateToComposition(template: string): string {
  const map: Record<string, string> = {
    promo_flash: 'PromoFlash',
    product_showcase: 'ProductShowcase',
    testimonial: 'Testimonial',
    tips_tricks: 'TipsTricks',
    behind_scenes: 'BehindScenes',
    quote: 'Quote',
  };
  return map[template];
}
```

## 7. Instagram Posting

`lib/platforms/instagram.ts`:

```typescript
const FB_API = 'https://graph.facebook.com/v19.0';

interface PostToInstagramInput {
  igUserId: string;
  accessToken: string;
  videoUrl: string;
  caption: string;
}

export async function postReelToInstagram(input: PostToInstagramInput): Promise<{
  post_id: string;
  permalink: string;
}> {
  // Step 1: Create media container
  const createRes = await fetch(`${FB_API}/${input.igUserId}/media`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      media_type: 'REELS',
      video_url: input.videoUrl,
      caption: input.caption,
      access_token: input.accessToken,
    }),
  });
  
  const createData = await createRes.json();
  if (!createData.id) throw new Error(`IG create failed: ${JSON.stringify(createData)}`);
  
  const creationId = createData.id;
  
  // Step 2: Poll status
  let status = 'IN_PROGRESS';
  let attempts = 0;
  while (status === 'IN_PROGRESS' && attempts < 60) {
    await new Promise(r => setTimeout(r, 5000));
    const statusRes = await fetch(
      `${FB_API}/${creationId}?fields=status_code&access_token=${input.accessToken}`
    );
    const statusData = await statusRes.json();
    status = statusData.status_code;
    attempts++;
  }
  
  if (status !== 'FINISHED') throw new Error(`IG status not finished: ${status}`);
  
  // Step 3: Publish
  const publishRes = await fetch(`${FB_API}/${input.igUserId}/media_publish`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      creation_id: creationId,
      access_token: input.accessToken,
    }),
  });
  
  const publishData = await publishRes.json();
  if (!publishData.id) throw new Error(`IG publish failed: ${JSON.stringify(publishData)}`);
  
  // Step 4: Get permalink
  const permalinkRes = await fetch(
    `${FB_API}/${publishData.id}?fields=permalink&access_token=${input.accessToken}`
  );
  const permalinkData = await permalinkRes.json();
  
  return {
    post_id: publishData.id,
    permalink: permalinkData.permalink,
  };
}

// Refresh long-lived token (60 days)
export async function refreshInstagramToken(currentToken: string): Promise<string> {
  const res = await fetch(
    `${FB_API}/oauth/access_token?grant_type=fb_exchange_token&client_id=${process.env.META_APP_ID}&client_secret=${process.env.META_APP_SECRET}&fb_exchange_token=${currentToken}`
  );
  const data = await res.json();
  return data.access_token;
}
```

## 8. TikTok Posting

`lib/platforms/tiktok.ts`:

```typescript
const TIKTOK_API = 'https://open.tiktokapis.com/v2';

interface PostToTikTokInput {
  accessToken: string;
  videoUrl: string;
  caption: string;
}

export async function postVideoToTikTok(input: PostToTikTokInput): Promise<{
  publish_id: string;
  status: string;
}> {
  // Step 1: Init upload (PULL_FROM_URL method)
  const initRes = await fetch(`${TIKTOK_API}/post/publish/inbox/video/init/`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${input.accessToken}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      source_info: {
        source: 'PULL_FROM_URL',
        video_url: input.videoUrl,
      },
    }),
  });
  
  const initData = await initRes.json();
  if (initData.error?.code !== 'ok') {
    throw new Error(`TikTok init failed: ${initData.error?.message}`);
  }
  
  const publishId = initData.data.publish_id;
  
  // Step 2: Poll status
  let status = 'PROCESSING_UPLOAD';
  let attempts = 0;
  while (status === 'PROCESSING_UPLOAD' && attempts < 60) {
    await new Promise(r => setTimeout(r, 5000));
    const statusRes = await fetch(`${TIKTOK_API}/post/publish/status/fetch/`, {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${input.accessToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ publish_id: publishId }),
    });
    const statusData = await statusRes.json();
    status = statusData.data?.status;
    attempts++;
  }
  
  // Inbox method = video uploaded ke akun, tapi user perlu publish manual di app
  // Untuk full auto-publish, perlu Content Posting API approval khusus
  
  return {
    publish_id: publishId,
    status,
  };
}
```

## 9. Scheduled Posts Worker

`app/api/jobs/post/route.ts`:

```typescript
// Triggered by Vercel Cron tiap 5 menit
// vercel.json: { "crons": [{ "path": "/api/jobs/post", "schedule": "*/5 * * * *" }] }

import { NextResponse } from 'next/server';
import { createAdminSupabase } from '@/lib/supabase/admin';
import { postReelToInstagram } from '@/lib/platforms/instagram';
import { postVideoToTikTok } from '@/lib/platforms/tiktok';

export async function GET(req: Request) {
  // Verify cron secret
  if (req.headers.get('Authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  const supabase = createAdminSupabase();
  
  // Find scheduled posts due (next 5 min window)
  const { data: duePosts } = await supabase
    .from('scheduled_posts')
    .select(`
      *,
      video:videos(*),
      account:social_accounts!inner(*)
    `)
    .eq('status', 'pending')
    .lte('scheduled_at', new Date(Date.now() + 5 * 60 * 1000).toISOString())
    .limit(50);
    
  if (!duePosts || duePosts.length === 0) return NextResponse.json({ processed: 0 });
  
  const results = await Promise.allSettled(
    duePosts.map(async (post: any) => {
      try {
        await supabase.from('scheduled_posts').update({ status: 'posting' }).eq('id', post.id);
        
        let result;
        if (post.platform === 'instagram') {
          result = await postReelToInstagram({
            igUserId: post.account.platform_user_id,
            accessToken: post.account.access_token,
            videoUrl: post.video.video_url,
            caption: post.caption,
          });
        } else if (post.platform === 'tiktok') {
          result = await postVideoToTikTok({
            accessToken: post.account.access_token,
            videoUrl: post.video.video_url,
            caption: post.caption,
          });
        } else {
          throw new Error(`Unsupported platform: ${post.platform}`);
        }
        
        await supabase.from('scheduled_posts').update({
          status: 'posted',
          platform_post_id: 'post_id' in result ? result.post_id : result.publish_id,
          platform_post_url: 'permalink' in result ? result.permalink : null,
          posted_at: new Date().toISOString(),
        }).eq('id', post.id);
        
        return { id: post.id, success: true };
      } catch (err: any) {
        await supabase.from('scheduled_posts').update({
          status: post.retry_count >= 1 ? 'failed' : 'pending',
          retry_count: post.retry_count + 1,
          error_message: err.message,
        }).eq('id', post.id);
        
        return { id: post.id, success: false, error: err.message };
      }
    })
  );
  
  return NextResponse.json({
    processed: duePosts.length,
    results,
  });
}
```

## 10. Music Library Helper

`lib/music/library.ts`:

```typescript
import musicLibrary from '@/content/music-library.json';

interface MusicTrack {
  id: string;
  title: string;
  artist: string;
  vibe: 'trendy' | 'energetic' | 'calm' | 'dramatic' | 'fun' | 'romantic';
  duration_sec: number;
  url: string;
  license: string;
}

export async function pickMusic(vibe: string): Promise<string> {
  const matching = (musicLibrary as MusicTrack[]).filter(t => t.vibe === vibe);
  if (matching.length === 0) {
    // Fallback to first track
    return musicLibrary[0].url;
  }
  
  // Random pick to avoid repetition
  return matching[Math.floor(Math.random() * matching.length)].url;
}
```

`content/music-library.json` (sample, populate dengan 30-50 tracks):

```json
[
  {
    "id": "trendy-1",
    "title": "Upbeat Pop",
    "artist": "YouTube Audio Library",
    "vibe": "trendy",
    "duration_sec": 60,
    "url": "https://cdn.kontenkilat.id/music/trendy-1.mp3",
    "license": "Royalty-free"
  }
]
```
