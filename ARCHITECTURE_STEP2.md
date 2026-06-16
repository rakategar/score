# SCORE System — Architecture Step 2
## Internal Dashboard + AI Report Generation

> **Prerequisite:** Step 1 selesai (database, auth, participant flow)
> **Scope:** Dashboard untuk Super Admin, Project Manager, Facilitator, Coach, Mentor
> **AI Feature:** Generate laporan otomatis menggunakan Claude Haiku
> **Prinsip:** Fasilitator tidak lagi membuat laporan — sistem & AI yang bekerja

---

## 1. Internal Dashboard — Struktur & Routing

```
app/(internal)/
├── layout.tsx                          # Internal shell (sidebar berbeda)
├── dashboard/
│   └── page.tsx                        # Overview semua program
├── programs/
│   ├── page.tsx                        # Daftar semua program/kelas
│   ├── new/page.tsx                    # Buat program baru
│   └── [classId]/
│       ├── page.tsx                    # Detail program + semua peserta
│       ├── participants/
│       │   ├── page.tsx                # Tabel semua peserta
│       │   └── [participantId]/
│       │       └── page.tsx            # Detail individual peserta
│       ├── observation/
│       │   └── page.tsx                # Input observasi fasilitator
│       ├── coaching/
│       │   └── page.tsx                # Sesi coaching per peserta
│       ├── mentoring/
│       │   └── page.tsx                # Sesi mentoring per peserta
│       ├── strategic-notes/
│       │   └── page.tsx                # Catatan strategis fasilitator
│       └── reports/
│           ├── page.tsx                # Report builder
│           └── [reportId]/page.tsx     # View generated report
├── settings/
│   ├── frameworks/
│   │   ├── page.tsx                    # Kelola competency frameworks
│   │   └── [frameworkId]/page.tsx
│   ├── classes/
│   │   └── page.tsx                    # Kelola kelas & password
│   └── users/
│       └── page.tsx                    # Kelola users & roles
└── profile/
    └── page.tsx
```

---

## 2. Internal Sidebar Navigation

```
┌─────────────────────┐
│  SCORE Internal     │
│  [Avatar] Oki T.W.  │
│  Facilitator        │
├─────────────────────┤
│ PROGRAM             │
│  Overview           │
│  Semua Program      │
│                     │
│ KELAS AKTIF         │
│  ACE Batch 3 ●      │
│   ├ Peserta (9)     │
│   ├ Observasi       │
│   ├ Coaching        │
│   ├ Mentoring       │
│   ├ Catatan         │
│   └ Laporan         │
│                     │
│ TOOLS               │
│  Framework Config   │
│  Kelola Kelas       │
│  Kelola User        │
│                     │
│ OTHER               │
│  Pengaturan         │
│  Keluar             │
└─────────────────────┘
```

**Role-based menu visibility:**
| Menu | Super Admin | PM | Facilitator | Coach | Mentor |
|------|:-----------:|:--:|:-----------:|:-----:|:------:|
| Semua Program | ✓ | ✓ | - | - | - |
| Detail Program | ✓ | ✓ | ✓ | ✓ | ✓ |
| Peserta | ✓ | ✓ | ✓ | ✓ | ✓ |
| Observasi | ✓ | ✓ | ✓ | - | - |
| Coaching | ✓ | ✓ | - | ✓ | - |
| Mentoring | ✓ | ✓ | - | - | ✓ |
| Strategic Notes | ✓ | ✓ | ✓ | - | - |
| Report Builder | ✓ | ✓ | ✓ | - | - |
| Framework Config | ✓ | - | - | - | - |
| Kelola User | ✓ | - | - | - | - |

---

## 3. Internal Dashboard Home

```
┌──────────────────────────────────────────────────────────────┐
│  Program Aktif                                               │
│                                                              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────┐  │
│  │ Total      │ │ Peserta    │ │ Selesai    │ │ Laporan  │  │
│  │ Kelas      │ │ Aktif      │ │ Assessment │ │ Generated│  │
│  │    1       │ │     9      │ │   7 / 9    │ │    5     │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────────┘  │
│                                                              │
│  ── Program Berjalan ─────────────────────────────────────   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ ACE Batch 3 · BRI Life           Aktif               │   │
│  │ 6–8 Agustus 2025 · Hotel Harris Sentul               │   │
│  │                                                      │   │
│  │ Progress Peserta:                                    │   │
│  │ Pre-Assess  Learning  Assessment  Evaluasi  Action   │   │
│  │   9/9  ●     9/9  ●    7/9  ●      5/9        2/9   │   │
│  │                                                      │   │
│  │ [Lihat Detail]  [Input Observasi]  [Generate Laporan]│   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Halaman Detail Peserta (Internal View)

Fasilitator bisa melihat semua data yang diisi peserta dalam 1 halaman:

```
Nadia Anggraini Putri                          [Generate Profil AI]
SLP Jakarta Rawamangun · Jakarta 1 · ESTJ

Tab: [Pre-Assessment] [Learning] [Assessment] [Evaluasi] [Action Plan] [Laporan]

─── TAB: Assessment ─────────────────────────────────────────────────
Score Akhir: 91 / 100                          [Kompeten Unggul]

┌─────────────────────────────────────────────────────────────────┐
│ Komponen                    Bobot  Score  Weighted  Notes       │
│ ─────────────────────────────────────────────────────────────── │
│ 1. Professional Grooming    15%    5/5    15.0      Sangat rapi │
│    ├ Penampilan korporat    10%    5      -                     │
│    └ Hygiene & kesegaran     5%    4      -                     │
│ 2. Workplace Professional   15%    5/5    15.0                  │
│ 3. ACE Process — MEET       30%    4/5    24.0      Terstruktur │
│ 4. ACE Process — PRESENT    30%    5/5    30.0      Excellent   │
│ 5. ACE Process — ASK        10%    5/5     7.0                  │
│ ─────────────────────────────────────────────────────────────── │
│                   TOTAL    100%          91.0                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Observasi Fasilitator (Module 4)

Form tambahan yang diisi FASILITATOR (bukan peserta) selama pelatihan:

```typescript
// Tabel: facilitator_observations (tambahan ke schema Step 1)
CREATE TABLE public.facilitator_observations (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  class_id          UUID NOT NULL REFERENCES public.classes(id),
  participant_id    UUID NOT NULL REFERENCES public.users(id),
  facilitator_id    UUID NOT NULL REFERENCES public.users(id),
  observation_date  DATE NOT NULL,
  session_name      TEXT,              -- "Hari 1 — Workshop MEET"
  -- Observasi per kompetensi (JSON sesuai framework)
  competency_obs    JSONB DEFAULT '[]',
  -- [{"comp_id":"c3","name":"ACE MEET","rating":4,
  --   "strength":"Membuka meeting dengan percaya diri",
  --   "dev_area":"Probing CEO masih kurang mendalam"}]
  mbti_type         TEXT,              -- Hasil observasi kepribadian
  mbti_strength     TEXT,
  mbti_dev_area     TEXT,
  overall_notes     TEXT,
  created_at        TIMESTAMPTZ DEFAULT NOW()
);

// Tabel: coaching_sessions (tambahan)
CREATE TABLE public.coaching_sessions (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  class_id         UUID NOT NULL REFERENCES public.classes(id),
  participant_id   UUID NOT NULL REFERENCES public.users(id),
  coach_id         UUID NOT NULL REFERENCES public.users(id),
  session_date     DATE NOT NULL,
  session_number   INTEGER DEFAULT 1,
  topic            TEXT,
  challenge        TEXT,
  root_cause       TEXT,
  agreed_action    TEXT,
  progress_status  TEXT CHECK (progress_status IN ('on_track','delayed','completed','at_risk')),
  coach_notes      TEXT,
  next_session_date DATE,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

// Tabel: mentoring_sessions
CREATE TABLE public.mentoring_sessions (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  class_id         UUID NOT NULL REFERENCES public.classes(id),
  participant_id   UUID NOT NULL REFERENCES public.users(id),
  mentor_id        UUID NOT NULL REFERENCES public.users(id),
  session_date     DATE NOT NULL,
  topic            TEXT,
  development_focus TEXT,
  advice           TEXT,
  agreed_action    TEXT,
  progress_notes   TEXT,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

// Tabel: strategic_notes
CREATE TABLE public.strategic_notes (
  id                    UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  class_id              UUID NOT NULL REFERENCES public.classes(id),
  author_id             UUID REFERENCES public.users(id),
  biggest_strength      TEXT,
  biggest_dev_area      TEXT,
  common_challenge      TEXT,
  organizational_insight TEXT,
  recommendation        TEXT,
  suggested_next_program TEXT,
  created_at            TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 6. AI Report Generation — Claude Haiku

### Architecture AI Pipeline

```
Data Input (dari Supabase)
        ↓
  Data Aggregator
  (lib/ai/aggregator.ts)
        ↓
  Prompt Builder
  (lib/ai/prompts.ts)
        ↓
  Claude Haiku API
  (claude-haiku-4-5-20251001)
        ↓
  Response Parser
        ↓
  Save to generated_reports
        ↓
  Render di UI / Export PDF
```

### `lib/ai/claude.ts`
```typescript
import Anthropic from '@anthropic-ai/sdk'

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
})

export async function generateWithHaiku(
  systemPrompt: string,
  userPrompt: string,
  maxTokens: number = 2048
): Promise<string> {
  const message = await anthropic.messages.create({
    model: 'claude-haiku-4-5-20251001',
    max_tokens: maxTokens,
    messages: [{ role: 'user', content: userPrompt }],
    system: systemPrompt,
  })

  const textBlock = message.content.find(b => b.type === 'text')
  return textBlock?.text ?? ''
}
```

### `lib/ai/aggregator.ts` — Kumpulkan semua data 1 peserta
```typescript
export async function aggregateParticipantData(enrollmentId: string) {
  const supabase = createClient()

  const [enrollment, preAssessment, assessment, evaluation, actionPlan, observations, coaching] =
    await Promise.all([
      supabase.from('class_enrollments')
        .select('*, class:classes(*, competency_framework:competency_frameworks(*))')
        .eq('id', enrollmentId).single(),
      supabase.from('pre_assessments').select('*').eq('enrollment_id', enrollmentId).single(),
      supabase.from('competency_assessments').select('*').eq('enrollment_id', enrollmentId).single(),
      supabase.from('evaluations').select('*').eq('enrollment_id', enrollmentId).single(),
      supabase.from('action_plans').select('*').eq('enrollment_id', enrollmentId).single(),
      supabase.from('facilitator_observations').select('*')
        .eq('participant_id', enrollment.data?.participant_id)
        .eq('class_id', enrollment.data?.class_id),
      supabase.from('coaching_sessions').select('*')
        .eq('participant_id', enrollment.data?.participant_id)
        .eq('class_id', enrollment.data?.class_id),
    ])

  return {
    participant: enrollment.data,
    preAssessment: preAssessment.data,
    assessment: assessment.data,
    evaluation: evaluation.data,
    actionPlan: actionPlan.data,
    observations: observations.data ?? [],
    coaching: coaching.data ?? [],
  }
}
```

### `lib/ai/prompts.ts` — System prompts

```typescript
export const SYSTEM_PROMPT_INDIVIDUAL_PROFILE = `
Anda adalah AI analyst sistem SCORE (Human Development Ecosystem) 
oleh Primera Karya Sinergia. Tugas Anda menghasilkan Individual 
Development Profile yang profesional, spesifik, dan actionable 
berdasarkan data peserta yang diberikan.

FORMAT OUTPUT (gunakan markdown):
## Development Snapshot
[2-3 kalimat ringkasan performa dan potensi peserta]

## Strength Profile  
[3-4 bullet kekuatan utama berdasarkan data]

## Development Priorities
[3 prioritas pengembangan yang harus difokuskan]

## Competency Summary
[Tabel atau ringkasan score per kompetensi]

## Coaching Observation
[Insight dari sesi coaching/observasi jika ada]

## Development Recommendation
[3 rekomendasi konkret dan spesifik]

## Suggested Learning Path
[Next program atau aktivitas yang direkomendasikan]

ATURAN:
- Gunakan Bahasa Indonesia yang profesional
- Selalu referensikan data aktual (score, tanggal, nama)
- Hindari generalisasi — setiap rekomendasi harus spesifik
- Tone: supportif dan membangun, bukan menghakimi
- Panjang: 400-600 kata
`

export const SYSTEM_PROMPT_TRAINING_IMPACT = `
Anda adalah AI analyst sistem SCORE oleh Primera Karya Sinergia.
Tugas Anda menghasilkan Training Impact Report yang komprehensif
untuk disajikan kepada klien (HR Head/Director/Sponsor).

FORMAT OUTPUT (gunakan markdown):
# Training Impact Report
## I. Executive Summary
## II. Program Overview  
## III. Participant Profile
## IV. Learning Outcomes & Key Insights
## V. Training Evaluation Results
## VI. Accreditation Results
## VII. Individual Development Highlights
## VIII. Organizational Insights
## IX. Next Steps & Recommendations

ATURAN:
- Bahasa Indonesia profesional, setara laporan consulting
- Sertakan angka dan persentase spesifik
- Highlight peserta berprestasi (Role Model)
- Identifikasi pola dari data keseluruhan
- Panjang: 800-1200 kata
`

export function buildIndividualPrompt(data: ParticipantData): string {
  const { participant, preAssessment, assessment, evaluation, actionPlan, observations } = data

  return `
DATA PESERTA:
Nama: ${participant.class?.participant?.full_name}
Program: ${participant.class?.title} — Batch ${participant.class?.batch}
Klien: ${participant.class?.client?.name}
Tanggal: ${participant.class?.start_date} s/d ${participant.class?.end_date}

PRE-ASSESSMENT:
- Tantangan bisnis: ${preAssessment?.business_challenge}
- Kebutuhan belajar: ${preAssessment?.learning_needs}
- Kekuatan yang ada: ${preAssessment?.existing_strengths}
- Area pengembangan: ${preAssessment?.development_areas}

HASIL AKREDITASI:
- Score akhir: ${assessment?.final_score}/100
- Kategori: ${assessment?.category}
- Detail scores: ${JSON.stringify(assessment?.scores, null, 2)}

EVALUASI PROGRAM:
- Rata-rata kepuasan: ${evaluation?.program_avg}/5
- Manfaat paling dirasakan: ${evaluation?.most_beneficial}
- Komitmen setelah training: ${evaluation?.improvement_suggestions}

ACTION PLAN:
${JSON.stringify(actionPlan?.items, null, 2)}

OBSERVASI FASILITATOR:
${observations.map(o => `- ${o.session_name}: ${o.overall_notes}`).join('\n')}

Berdasarkan data di atas, buatkan Individual Development Profile yang lengkap.
`
}

export function buildTrainingImpactPrompt(classData: ClassData, allParticipants: ParticipantData[]): string {
  const scores = allParticipants.map(p => p.assessment?.final_score ?? 0)
  const avgScore = scores.reduce((a, b) => a + b, 0) / scores.length
  const kompetentUnggul = allParticipants.filter(p => p.assessment?.category === 'kompeten_unggul')
  const kompeten = allParticipants.filter(p => p.assessment?.category === 'kompeten')
  const perluPengembangan = allParticipants.filter(p => p.assessment?.category === 'perlu_pengembangan')

  const avgEval = allParticipants.reduce((sum, p) => sum + (p.evaluation?.program_avg ?? 0), 0) / allParticipants.length

  return `
PROGRAM:
Nama: ${classData.title}
Batch: ${classData.batch}
Klien: ${classData.client?.name}
Tanggal: ${classData.start_date} s/d ${classData.end_date}
Lokasi: ${classData.location}
Fasilitator: ${classData.facilitator_names?.join(', ')}
Jumlah Peserta: ${allParticipants.length}

HASIL AKREDITASI:
- Kompeten Unggul (≥85): ${kompetentUnggul.length} peserta → ${kompetentUnggul.map(p => p.participant?.full_name).join(', ')}
- Kompeten (65-84): ${kompeten.length} peserta
- Perlu Pengembangan (<65): ${perluPengembangan.length} peserta
- Rata-rata Score: ${avgScore.toFixed(1)}/100

EVALUASI TRAINING:
- Rata-rata kepuasan program: ${avgEval.toFixed(2)}/5

PESERTA DAN SCORE:
${allParticipants.map(p => `- ${p.participant?.full_name}: ${p.assessment?.final_score} (${p.assessment?.category})`).join('\n')}

COMMON STRENGTHS (dari observasi):
${classData.strategic_notes?.biggest_strength ?? 'Tidak ada data'}

COMMON DEV AREAS:
${classData.strategic_notes?.biggest_dev_area ?? 'Tidak ada data'}

Buat Training Impact Report lengkap berdasarkan data di atas.
`
}
```

---

## 7. API Routes — Report Generation

### `app/api/ai/generate-individual/route.ts`
```typescript
import { generateWithHaiku } from '@/lib/ai/claude'
import { aggregateParticipantData } from '@/lib/ai/aggregator'
import { SYSTEM_PROMPT_INDIVIDUAL_PROFILE, buildIndividualPrompt } from '@/lib/ai/prompts'
import { createClient } from '@/lib/supabase/server'

export async function POST(req: Request) {
  const { enrollmentId } = await req.json()
  const supabase = await createClient()

  // Verifikasi auth & permission
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 })

  // Kumpulkan semua data
  const participantData = await aggregateParticipantData(enrollmentId)

  // Build prompt
  const userPrompt = buildIndividualPrompt(participantData)

  // Generate dengan Claude Haiku
  const reportText = await generateWithHaiku(
    SYSTEM_PROMPT_INDIVIDUAL_PROFILE,
    userPrompt,
    1500  // Haiku max tokens untuk hemat biaya
  )

  // Parse sections
  const content = parseReportSections(reportText)

  // Simpan ke database
  const { data: report } = await supabase.from('generated_reports').insert({
    enrollment_id: enrollmentId,
    class_id: participantData.participant.class_id,
    report_type: 'individual_profile',
    content,
    raw_text: reportText,
    ai_model: 'claude-haiku-4-5-20251001',
  }).select().single()

  return Response.json({ report })
}

function parseReportSections(text: string) {
  // Parse markdown sections menjadi object terstruktur
  const sections: Record<string, string> = {}
  const sectionRegex = /## (.+?)\n([\s\S]*?)(?=## |$)/g
  let match
  while ((match = sectionRegex.exec(text)) !== null) {
    const key = match[1].toLowerCase().replace(/\s+/g, '_')
    sections[key] = match[2].trim()
  }
  return sections
}
```

### `app/api/ai/generate-training-impact/route.ts`
```typescript
export async function POST(req: Request) {
  const { classId } = await req.json()
  const supabase = await createClient()

  // Ambil semua enrollments untuk kelas ini
  const { data: enrollments } = await supabase
    .from('class_enrollments')
    .select('id, participant_id')
    .eq('class_id', classId)

  // Aggregate semua data peserta secara parallel
  const allParticipantsData = await Promise.all(
    enrollments?.map(e => aggregateParticipantData(e.id)) ?? []
  )

  // Ambil data kelas + strategic notes
  const { data: classData } = await supabase
    .from('classes')
    .select('*, client:clients(*), strategic_notes(*)')
    .eq('id', classId).single()

  // Generate report
  const userPrompt = buildTrainingImpactPrompt(classData, allParticipantsData)
  const reportText = await generateWithHaiku(
    SYSTEM_PROMPT_TRAINING_IMPACT,
    userPrompt,
    3000
  )

  // Simpan
  const { data: report } = await supabase.from('generated_reports').insert({
    enrollment_id: enrollments?.[0]?.id, // Link ke enrollment pertama
    class_id: classId,
    report_type: 'training_impact',
    content: parseReportSections(reportText),
    raw_text: reportText,
  }).select().single()

  return Response.json({ report })
}
```

---

## 8. Report Builder UI

```
┌─────────────────────────────────────────────────────────────────┐
│  Report Builder — ACE Batch 3 BRI Life                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pilih Jenis Laporan:                                           │
│  ● Training Impact Report  ○ Individual Profiles  ○ Executive  │
│                                                                 │
│  Section yang Disertakan:                                       │
│  [✓] Executive Summary    [✓] Program Overview                 │
│  [✓] Profil Peserta       [✓] Learning Outcomes                │
│  [✓] Hasil Evaluasi       [✓] Hasil Akreditasi                 │
│  [✓] Individual Highlights [✓] Org Insights                    │
│  [✓] Rekomendasi & Next Steps                                  │
│                                                                 │
│  Filter:                                                        │
│  Batch: [Batch 3 ▼]   Tanggal: [6 Agu - 8 Agu ▼]             │
│  Peserta: [Semua ▼]                                            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  [✦] Generate dengan AI                                 │   │
│  │  Estimasi waktu: ~15-30 detik | Model: Claude Haiku    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Laporan Tersimpan:                                             │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Training Impact Report  │ 12 Jun 2025 │ [Preview] [Export]│ │
│  │ Individual: Nadia       │ 12 Jun 2025 │ [Preview] [Export]│ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Report Preview Component

Saat laporan di-generate, tampilkan dengan layout mirip laporan PDF asli:

```
┌──────────────────────────────────────────────────────────┐
│                                              prmr•       │
│              INDIVIDUAL DEVELOPMENT PROFILE              │
│              Nadia Anggraini Putri                       │
│              ACE Batch 3 · BRI Life · Agustus 2025       │
│ ─────────────────────────────────────────────────────── │
│ DEVELOPMENT SNAPSHOT                                     │
│ Nadia menunjukkan performa luar biasa dengan score 91... │
│                                                          │
│ STRENGTH PROFILE                                         │
│ • Terstruktur dan sistematis dalam menjalankan ACE...    │
│ • Percaya diri tinggi dalam presentasi kepada nasabah... │
│ • Detail-oriented dalam aspek professional grooming...   │
│                                                          │
│ DEVELOPMENT PRIORITIES                                   │
│ 1. Fleksibilitas dalam menghadapi ketidakpastian...     │
│ 2. Kreativitas dalam memvariasikan pendekatan ACE...    │
│ 3. ...                                                   │
│                                                          │
│ [Export PDF]  [Download Word]  [Share Link]              │
└──────────────────────────────────────────────────────────┘
```

---

## 10. Program Setup — Buat Kelas Baru

Form yang diisi Super Admin/PM untuk membuat kelas baru:

```typescript
// Form fields: app/internal/programs/new/page.tsx
interface NewClassForm {
  // Basic Info
  title: string
  subtitle: string
  batch: string
  client_id: string           // Select dari daftar clients
  
  // Schedule
  location: string
  start_date: string
  end_date: string
  duration_days: number
  
  // Staff
  facilitator_names: string[] // Input text, bisa multiple
  
  // Framework
  competency_framework_id: string  // Select dari frameworks yang ada
  
  // Password
  class_password: string      // Plain text → di-hash sebelum disimpan
  confirm_password: string
  
  // Objectives (dynamic list)
  objectives: string[]
  
  // Cover image
  cover_image?: File
}
```

**Password hashing saat create class:**
```typescript
import bcrypt from 'bcrypt'

const passwordHash = await bcrypt.hash(formData.class_password, 10)
await supabase.from('classes').insert({
  ...formData,
  password_hash: passwordHash,
  class_password: undefined,   // Jangan simpan plain text
  confirm_password: undefined,
})
```

---

## 11. Facilitator Observation Form

Form yang diisi fasilitator untuk setiap peserta selama training berlangsung:

```
┌────────────────────────────────────────────────────────┐
│  Form Observasi Peserta                                │
│  Nadia Anggraini Putri · ACE Batch 3                   │
│  Sesi: [Hari 2 — Workshop MEET & PRESENT ▼]            │
│  Tanggal: [07/08/2025]                                 │
│                                                        │
│  OBSERVASI KEPRIBADIAN (MBTI)                          │
│  Tipe MBTI: [ESTJ ▼]                                  │
│  Kekuatan terlihat: [________________________]         │
│  Area pengembangan: [________________________]         │
│                                                        │
│  OBSERVASI PER KOMPETENSI                              │
│                                                        │
│  1. Professional Grooming                              │
│     Rating: ○1 ○2 ○3 ○4 ●5                           │
│     Kekuatan: [Penampilan sangat profesional...]       │
│     Dev Area: [________________________]               │
│                                                        │
│  3. ACE Process — MEET                                 │
│     Rating: ○1 ○2 ○3 ●4 ○5                           │
│     Kekuatan: [Membuka meeting dengan percaya diri...] │
│     Dev Area: [Probing CEO perlu lebih mendalam...]    │
│                                                        │
│  CATATAN UMUM                                          │
│  [_________________________________________________]  │
│  [_________________________________________________]  │
│                                                        │
│  [Simpan Observasi]                                    │
└────────────────────────────────────────────────────────┘
```

---

## 12. Checklist Build Step 2

```
INTERNAL DASHBOARD LAYOUT
[ ] Internal sidebar dengan role-based menu
[ ] Route protection khusus internal roles
[ ] Dashboard home dengan overview stats

PROGRAM MANAGEMENT
[ ] List semua program/kelas
[ ] Buat kelas baru (form + password hashing)
[ ] Detail kelas dengan tab peserta

PESERTA MANAGEMENT (Internal View)
[ ] Tabel semua peserta dengan status per step
[ ] Detail individual: semua tab data peserta
[ ] Progress tracking real-time

FACILITATOR TOOLS
[ ] Form observasi fasilitator per peserta per sesi
[ ] Form coaching session
[ ] Form mentoring session
[ ] Form strategic notes program

AI REPORT GENERATION
[ ] API route: generate individual profile
[ ] API route: generate training impact report
[ ] Report builder UI (pilih type + sections)
[ ] Report preview (render markdown)
[ ] Tombol export (untuk Step 3)

SETTINGS
[ ] Kelola competency frameworks (CRUD)
[ ] Kelola kelas & password
[ ] Kelola users & roles (Super Admin only)

DATABASE ADDITIONS
[ ] facilitator_observations table
[ ] coaching_sessions table
[ ] mentoring_sessions table
[ ] strategic_notes table
[ ] RLS policies untuk tabel baru
```

---

> **Lanjut ke ARCHITECTURE_STEP3.md** untuk:
> External/Client Dashboard + Export & Finalisasi Sistem
