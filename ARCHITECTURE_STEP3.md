# SCORE System — Architecture Step 3
## External/Client Dashboard + PDF Export + System Finalization

> **Prerequisite:** Step 1 (Participant Flow) + Step 2 (Internal + AI Reports) selesai
> **Scope:** Dashboard Manager, Client HR/L&D, Business Sponsor + Export PDF + Polish
> **Prinsip:** Client mendapatkan insight tanpa perlu akses ke data mentah peserta

---

## 1. Client Dashboard — Struktur & Routing

```
app/(client)/
├── layout.tsx                      # Client shell (minimal, clean)
├── dashboard/
│   └── page.tsx                    # Executive overview
├── programs/
│   ├── page.tsx                    # Semua program yang diikuti
│   └── [classId]/
│       ├── page.tsx                # Detail program (aggregated view)
│       ├── participants/
│       │   └── page.tsx            # Overview peserta (no personal data detail)
│       ├── insights/
│       │   └── page.tsx            # AI-generated organizational insights
│       └── reports/
│           ├── page.tsx            # Laporan yang tersedia
│           └── [reportId]/
│               └── page.tsx        # View & download laporan
└── profile/
    └── page.tsx
```

---

## 2. Role Mapping — Apa yang Client Bisa Lihat

| Data | Manager | Client HR/L&D | Business Sponsor |
|------|:-------:|:-------------:|:----------------:|
| Overview program | ✓ | ✓ | ✓ |
| Distribusi akreditasi | ✓ | ✓ | ✓ |
| Score individual (nama) | ✓ hanya direct report | ✓ semua | ringkasan saja |
| Evaluasi training | - | ✓ | ✓ ringkasan |
| Individual Development Profile | ✓ direct report | ✓ | - |
| Training Impact Report | - | ✓ | ✓ |
| Executive Summary | ✓ | ✓ | ✓ |
| Org Insight AI | - | ✓ | ✓ |
| Export PDF | ✓ terbatas | ✓ | ✓ |

### Database: Client Access Control
```sql
-- Tambahan ke schema: program_client_access
-- Mengatur client mana yang bisa akses program mana
CREATE TABLE public.program_client_access (
  id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  class_id       UUID NOT NULL REFERENCES public.classes(id),
  user_id        UUID NOT NULL REFERENCES public.users(id),
  -- Role client
  access_level   TEXT NOT NULL CHECK (access_level IN ('manager','hr_client','sponsor')),
  -- Manager: hanya bisa lihat peserta tertentu
  participant_ids UUID[] DEFAULT NULL,  -- NULL = bisa lihat semua
  granted_by     UUID REFERENCES public.users(id),
  granted_at     TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(class_id, user_id)
);

-- RLS Policy: Client hanya akses data sesuai izin
CREATE POLICY "client_program_access" ON public.classes
  FOR SELECT USING (
    auth.uid() IN (
      SELECT user_id FROM public.program_client_access WHERE class_id = id
    )
    OR auth.uid() IN (
      SELECT id FROM public.users WHERE role IN ('super_admin','project_manager')
    )
  );
```

---

## 3. Client Dashboard Home

```
┌────────────────────────────────────────────────────────────────┐
│ SCORE                                      BRI Life  [Profil] │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Selamat datang, Tim BRI Life                                  │
│  Anda memiliki akses ke 1 program aktif                        │
│                                                                │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│  │ Program      │ │ Total PBS    │ │ Lulus        │           │
│  │ Berjalan     │ │ Dilatih      │ │ Akreditasi   │           │
│  │     1        │ │      9       │ │   7 (77.7%)  │           │
│  └──────────────┘ └──────────────┘ └──────────────┘           │
│                                                                │
│  ─── Program Terbaru ──────────────────────────────────────   │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  ACE Program Batch 3                      Selesai ●    │   │
│  │  6–8 Agustus 2025 · Hotel Harris Sentul               │   │
│  │  Primera Karya Sinergia × BRI Life                     │   │
│  │                                                        │   │
│  │  Kompeten Unggul  Kompeten  Perlu Dev                  │   │
│  │       2              5          2                      │   │
│  │  ████████████████░░░░░░  77.7% lulus                   │   │
│  │                                                        │   │
│  │  Rata-rata Evaluasi: 4.95/5 · Kepuasan: 93%           │   │
│  │                                                        │   │
│  │  [Lihat Detail]   [Unduh Laporan]   [AI Insight]      │   │
│  └────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

---

## 4. Program Detail — Client View

```
ACE Batch 3 — Detail Program
Tab: [Overview] [Peserta] [Evaluasi] [AI Insight] [Laporan]

─── TAB: Peserta (Client View) ───────────────────────────────────

  Filter: [Semua ▼]  [Urut: Score ▼]

  ┌──────────────────────────────────────────────────────────┐
  │  #   Nama                  Region      Score   Status    │
  │  ──  ──────────────────    ────────    ─────   ──────── │
  │  1   Nadia Anggraini P.    Jakarta 1    91     ⬟ Unggul │
  │  2   Sahadatu Chandra      Pekanbaru    88.5   ⬟ Unggul │
  │  3   Siska F. Sihombing    Jakarta 1    80     ● Kompeten│
  │  4   Stevanus V.P.         Malang       78.5   ● Kompeten│
  │  5   Indra Putra A.        Jakarta 1    77.5   ● Kompeten│
  │  6   Azies Koentoro        Jakarta 3    76.5   ● Kompeten│
  │  7   Brian Bagus S.        Jakarta 1    74.5   ● Kompeten│
  │  8   Juni Rahmansyah       Yogyakarta   64.5   △ Perlu  │
  │  9   Monika Caesarany      Jakarta 2    62     △ Perlu  │
  │                                                          │
  │  [Unduh Individual Profile — Nadia]  [Unduh Semua PDF]   │
  └──────────────────────────────────────────────────────────┘

─── TAB: AI Insight (Org Level) ──────────────────────────────────

  [✦ Generate Organizational Insight]

  ┌────────────────────────────────────────────────────────┐
  │  Organizational Development Insight                    │
  │  Generated: 12 Jun 2025 · claude-haiku-4-5-20251001   │
  │                                                        │
  │  CAPABILITY STRENGTH                                   │
  │  Tim PBS Batch 3 menunjukkan kemampuan komunikasi...   │
  │                                                        │
  │  CAPABILITY GAP                                        │
  │  Tiga area yang memerlukan perhatian segera:...        │
  │                                                        │
  │  STRATEGIC RECOMMENDATION                              │
  │  1. Implementasi performance coaching 30 hari...       │
  │  2. Advanced Module: Wealth Advisory Deep Dive...      │
  │  3. Monitoring via WA Group untuk success story...     │
  │                                                        │
  │  FUTURE DEVELOPMENT ROADMAP (6 bulan)                  │
  │  Q3 2025: Performance review & coaching individual     │
  │  Q4 2025: Advanced ACE Module                          │
  │                                                        │
  │  [Export PDF]  [Share]                                 │
  └────────────────────────────────────────────────────────┘
```

---

## 5. PDF Export System

### Library: `@react-pdf/renderer` untuk client-side PDF

```typescript
// components/reports/PDFReport.tsx
import { Document, Page, Text, View, StyleSheet, Image, Font } from '@react-pdf/renderer'

const styles = StyleSheet.create({
  page: { padding: 40, fontFamily: 'Helvetica', fontSize: 10 },
  header: { flexDirection: 'row', justifyContent: 'space-between', marginBottom: 20 },
  logo: { fontSize: 16, fontWeight: 'bold', color: '#2563EB' },
  title: { fontSize: 18, fontWeight: 'bold', textAlign: 'center', marginBottom: 8 },
  subtitle: { fontSize: 11, color: '#6B7280', textAlign: 'center', marginBottom: 20 },
  section: { marginBottom: 16 },
  sectionTitle: { fontSize: 13, fontWeight: 'bold', color: '#1F2937',
                  borderBottomWidth: 1, borderBottomColor: '#E5E7EB',
                  paddingBottom: 4, marginBottom: 8 },
  text: { fontSize: 10, lineHeight: 1.6, color: '#374151' },
  table: { width: '100%', marginBottom: 12 },
  tableRow: { flexDirection: 'row', borderBottomWidth: 0.5, borderBottomColor: '#E5E7EB' },
  tableHeader: { backgroundColor: '#F3F4F6' },
  tableCell: { padding: 6, flex: 1 },
  badge: { padding: '3 8', borderRadius: 4, fontSize: 9 },
  badgeGreen: { backgroundColor: '#D1FAE5', color: '#065F46' },
  badgeBlue: { backgroundColor: '#DBEAFE', color: '#1E40AF' },
  badgeRed: { backgroundColor: '#FEE2E2', color: '#991B1B' },
  footer: { position: 'absolute', bottom: 20, left: 40, right: 40,
            flexDirection: 'row', justifyContent: 'space-between',
            fontSize: 8, color: '#9CA3AF' },
})

// Individual Development Profile PDF
export function IndividualProfilePDF({ participant, report, classData }) {
  return (
    <Document>
      <Page size="A4" style={styles.page}>
        {/* Header */}
        <View style={styles.header}>
          <Text style={styles.logo}>SCORE</Text>
          <Text style={{ fontSize: 8, color: '#6B7280' }}>Primera Karya Sinergia</Text>
        </View>

        {/* Title */}
        <Text style={styles.title}>Individual Development Profile</Text>
        <Text style={styles.subtitle}>
          {participant.full_name} · {classData.title} Batch {classData.batch}
        </Text>
        <Text style={styles.subtitle}>
          {classData.start_date} s/d {classData.end_date} · {classData.location}
        </Text>

        {/* Score Badge */}
        <View style={{ alignItems: 'center', marginBottom: 16 }}>
          <View style={{ ...styles.badge,
            ...(participant.category === 'kompeten_unggul' ? styles.badgeGreen :
                participant.category === 'kompeten' ? styles.badgeBlue : styles.badgeRed)
          }}>
            <Text>{participant.final_score}/100 — {getCategoryLabel(participant.category)}</Text>
          </View>
        </View>

        {/* AI Generated Content */}
        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Development Snapshot</Text>
          <Text style={styles.text}>{report.content.development_snapshot}</Text>
        </View>

        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Strength Profile</Text>
          <Text style={styles.text}>{report.content.strength_profile}</Text>
        </View>

        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Development Priorities</Text>
          <Text style={styles.text}>{report.content.development_priorities}</Text>
        </View>

        {/* Competency Score Table */}
        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Competency Summary</Text>
          <View style={styles.table}>
            {/* ... competency scores */}
          </View>
        </View>

        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Development Recommendation</Text>
          <Text style={styles.text}>{report.content.development_recommendation}</Text>
        </View>

        {/* Footer */}
        <View style={styles.footer}>
          <Text>SCORE — Human Development Ecosystem | Primera Karya Sinergia</Text>
          <Text>Confidential</Text>
        </View>
      </Page>
    </Document>
  )
}
```

### API Export Route
```typescript
// app/api/export/pdf/route.ts
import { renderToBuffer } from '@react-pdf/renderer'
import { IndividualProfilePDF } from '@/components/reports/PDFReport'

export async function POST(req: Request) {
  const { enrollmentId, reportType } = await req.json()

  // Ambil data dari Supabase
  const data = await getReportData(enrollmentId, reportType)

  // Render PDF ke buffer
  const buffer = await renderToBuffer(
    reportType === 'individual_profile'
      ? <IndividualProfilePDF {...data} />
      : <TrainingImpactPDF {...data} />
  )

  return new Response(buffer, {
    headers: {
      'Content-Type': 'application/pdf',
      'Content-Disposition': `attachment; filename="SCORE_Report_${Date.now()}.pdf"`,
    },
  })
}
```

---

## 6. Manager Dashboard (Role Khusus)

Manager hanya melihat direct reports mereka:

```typescript
// Dari tabel participants, manager melihat peserta
// yang direct_manager_id = manager's user id

// app/(client)/dashboard/page.tsx
// Jika user.role === 'manager':
const { data: myReports } = await supabase
  .from('participants')
  .select(`
    *,
    enrollment:class_enrollments(
      *,
      class:classes(*),
      assessment:competency_assessments(*),
      action_plans(*)
    )
  `)
  .eq('direct_manager_id', user.id)
```

**Manager View:**
```
Halo, [Manager Name]

Direct Reports Anda yang mengikuti training:

┌─────────────────────────────────────────────────────────┐
│ Nadia Anggraini Putri          Kompeten Unggul — 91     │
│ ACE Batch 3 · Jakarta 1                                 │
│                                                         │
│ Kekuatan: Terstruktur, percaya diri, profesional       │
│ Fokus Dev: Fleksibilitas & kreativitas                  │
│                                                         │
│ Action Plan:                                            │
│ • Implementasi ACE dengan 3 nasabah/minggu — ○ Belum   │
│ • ...                                                   │
│                                                         │
│ [Update Status Action Plan]  [Lihat Profil Lengkap]    │
└─────────────────────────────────────────────────────────┘
```

---

## 7. Manager Feedback Module

Manager bisa mengisi feedback progress peserta setelah training:

```typescript
// Tabel: manager_feedbacks (dari schema Step 1 — sudah ada)
// Form di client dashboard

interface ManagerFeedbackForm {
  participant_id: string
  class_id: string
  feedback_date: string
  learning_application: string    // "Sudah mulai menerapkan ACE framework..."
  behavior_change: string         // "Terlihat lebih percaya diri dalam..."
  performance_improvement: string // "Nasabah baru bertambah 2 dalam 30 hari..."
  manager_comments: string
  rating: 1 | 2 | 3 | 4 | 5
}
```

---

## 8. Notification System (Simple)

```typescript
// Tabel: notifications
CREATE TABLE public.notifications (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id       UUID NOT NULL REFERENCES public.users(id),
  type          TEXT NOT NULL CHECK (type IN (
                  'step_completed','report_ready','action_plan_due',
                  'coaching_scheduled','new_enrollment','assessment_reminder'
                )),
  title         TEXT NOT NULL,
  message       TEXT,
  data          JSONB DEFAULT '{}',
  is_read       BOOLEAN DEFAULT FALSE,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Trigger: notify internal staff saat peserta selesai step
CREATE OR REPLACE FUNCTION notify_step_completion()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.status = 'completed' AND OLD.status != 'completed' THEN
    -- Notify facilitator
    INSERT INTO notifications (user_id, type, title, message)
    SELECT ps.facilitator_id, 'step_completed',
           'Peserta menyelesaikan ' || NEW.step_name,
           'Step ' || NEW.step_name || ' telah diselesaikan'
    FROM program_staff ps
    JOIN class_enrollments ce ON ce.class_id = ps.program_id
    WHERE ce.id = NEW.enrollment_id;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_step_completed
  AFTER UPDATE ON step_progress
  FOR EACH ROW EXECUTE FUNCTION notify_step_completion();
```

---

## 9. Real-time Progress (Supabase Realtime)

```typescript
// Di halaman internal: update otomatis saat peserta menyelesaikan step
// components/participant/ProgressTracker.tsx

useEffect(() => {
  const channel = supabase
    .channel(`class-${classId}-progress`)
    .on('postgres_changes', {
      event: 'UPDATE',
      schema: 'public',
      table: 'step_progress',
    }, (payload) => {
      // Update UI tanpa refresh halaman
      setProgressData(prev => updateProgress(prev, payload.new))
    })
    .subscribe()

  return () => { supabase.removeChannel(channel) }
}, [classId])
```

---

## 10. Sertifikat Digital (Bonus Feature)

Setelah peserta selesai semua step & lulus akreditasi:

```typescript
// Tabel: certificates
CREATE TABLE public.certificates (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enrollment_id   UUID UNIQUE NOT NULL REFERENCES public.class_enrollments(id),
  participant_name TEXT NOT NULL,
  class_title     TEXT NOT NULL,
  batch           TEXT,
  category        TEXT NOT NULL,      -- 'kompeten_unggul' / 'kompeten'
  final_score     NUMERIC(5,2),
  issued_date     DATE DEFAULT CURRENT_DATE,
  certificate_url TEXT,               -- URL PDF sertifikat
  verification_code TEXT UNIQUE,      -- Kode untuk verifikasi keaslian
  issued_by       TEXT DEFAULT 'Primera Karya Sinergia',
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

// Certificate PDF Component
export function CertificatePDF({ data }) {
  return (
    <Document>
      <Page size="A4" orientation="landscape" style={certStyles.page}>
        <View style={certStyles.border}>
          <Text style={certStyles.header}>Primera Karya Sinergia</Text>
          <Text style={certStyles.subtitle}>dengan bangga memberikan sertifikat kepada</Text>
          <Text style={certStyles.name}>{data.participant_name}</Text>
          <Text style={certStyles.body}>
            yang telah berhasil menyelesaikan dan lulus dengan predikat
          </Text>
          <Text style={certStyles.category}>{getCategoryLabel(data.category)}</Text>
          <Text style={certStyles.program}>{data.class_title}</Text>
          <Text style={certStyles.score}>Score: {data.final_score}/100</Text>
          <Text style={certStyles.date}>{formatDate(data.issued_date)}</Text>
          <Text style={certStyles.code}>Kode Verifikasi: {data.verification_code}</Text>
        </View>
      </Page>
    </Document>
  )
}
```

---

## 11. System Configuration (Super Admin)

### Framework Management UI

```
Settings → Competency Frameworks

┌────────────────────────────────────────────────────────────────┐
│  ACE Accreditation Framework — PBS           [Edit] [Duplicate]│
│  Rating: 1-5 · Passing Score: 65             Status: Aktif ●  │
│                                                                │
│  Competencies:                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ + 1. Professional Grooming         15%   [Edit] [Delete] │ │
│  │   └ Penampilan korporat            10%                   │ │
│  │   └ Hygiene & kesegaran             5%                   │ │
│  │ + 2. Workplace Professionalism     15%   [Edit] [Delete] │ │
│  │ + 3. ACE Process — MEET            30%   [Edit] [Delete] │ │
│  │ + 4. ACE Process — PRESENT         30%   [Edit] [Delete] │ │
│  │ + 5. ACE Process — ASK             10%   [Edit] [Delete] │ │
│  │ ─────────────────────────────────────                    │ │
│  │ Total Weight: 100% ✓                                     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  [+ Tambah Kompetensi]                                         │
└────────────────────────────────────────────────────────────────┘
```

---

## 12. Final Database Summary — Semua Tabel

```
Step 1 Tables (Participant Flow):
├── users
├── clients
├── competency_frameworks
├── classes
├── class_enrollments
├── step_progress
├── pre_assessments
├── learning_confirmations
├── competency_assessments
├── evaluations
├── action_plans
└── generated_reports

Step 2 Tables (Internal Tools):
├── facilitator_observations
├── coaching_sessions
├── mentoring_sessions
└── strategic_notes

Step 3 Tables (Client + System):
├── program_client_access
├── manager_feedbacks
├── notifications
└── certificates
```

---

## 13. AI Prompts — Organizational Insight (Step 3)

```typescript
export const SYSTEM_PROMPT_ORG_INSIGHT = `
Anda adalah AI analyst sistem SCORE oleh Primera Karya Sinergia.
Audiens laporan ini adalah HR Head, L&D Director, atau Business Sponsor 
yang ingin memahami ROI dan dampak program training terhadap organisasi.

FORMAT OUTPUT:
## Organizational Development Insight

### Executive Summary
[3-4 kalimat untuk sponsor level]

### Capability Strength
[Kekuatan kolektif yang teridentifikasi]

### Capability Gap Analysis
[Gap yang perlu ditangani — berikan prioritas]

### Learning Transfer Indicators
[Indikator transfer pembelajaran ke pekerjaan]

### ROI & Business Impact Projection
[Estimasi dampak bisnis berdasarkan peningkatan kompetensi]

### Strategic Recommendations
[3 rekomendasi strategis untuk L&D]

### Future Development Roadmap
[Timeline 3-6 bulan ke depan]

ATURAN:
- Bahasa formal level C-suite
- Sertakan angka dan persentase
- Fokus pada business impact, bukan aktivitas training
- Panjang: 600-900 kata
`

export const SYSTEM_PROMPT_EXECUTIVE_SUMMARY = `
Anda adalah AI analyst sistem SCORE. Buat Executive Summary 1 halaman
yang dapat langsung dipresentasikan kepada Business Sponsor.

Format: Singkat, padat, visual-friendly (bisa jadi slide)
Maksimal 300 kata, gunakan bullet points
Fokus: Key results, ROI indicators, recommended next actions
`
```

---

## 14. Deployment Checklist

```
ENVIRONMENT
[ ] Supabase project setup (production)
[ ] Environment variables di Vercel/hosting
[ ] Anthropic API key aktif (claude-haiku)
[ ] Domain custom (opsional)

SECURITY
[ ] RLS aktif untuk semua tabel sensitif
[ ] API routes protected (auth check)
[ ] Password hashing dengan bcrypt (cost factor 10+)
[ ] Rate limiting untuk AI endpoints
[ ] Sanitasi input semua form

TESTING
[ ] Happy path: peserta join kelas → isi semua step → lihat laporan
[ ] Demo login bekerja untuk semua 3 dashboard
[ ] PDF export berhasil generate
[ ] AI report generation berhasil
[ ] RLS: peserta tidak bisa akses data peserta lain

PERFORMANCE
[ ] Image optimization (cover kelas)
[ ] Loading states semua async operations
[ ] Error boundaries untuk AI failures
[ ] Skeleton loading untuk dashboard data

POLISH
[ ] Responsive mobile (minimal untuk participant)
[ ] Empty states semua halaman
[ ] Toast notifications (sukses/error)
[ ] Konfirmasi sebelum submit final step
[ ] Animasi transisi antar step (framer-motion)
```

---

## 15. Complete User Journey Summary

```
PESERTA (Happy Path):
Login → Dashboard → Lihat kelas tersedia → Klik "Daftar" → 
Input password kelas (dari trainer) → Masuk kelas → 
Step 1: Pre-Assessment (5 menit) → 
Step 2: Konfirmasi materi (2 menit) → 
Step 3: Self-assessment kompetensi ACE (10-15 menit) → 
Step 4: Evaluasi training (5 menit) → 
Step 5: Buat action plan (5-10 menit) → 
Selesai! → Lihat profil development → Download sertifikat

FASILITATOR (Happy Path):
Login Internal → Pilih kelas aktif → 
Isi observasi peserta (per sesi) → 
Isi coaching session notes → 
Isi strategic notes program → 
Generate Training Impact Report → 
Preview → Export PDF → Kirim ke client

KLIEN/CLIENT (Happy Path):
Login Client → Dashboard → 
Lihat overview program → 
Akses laporan yang sudah di-generate → 
Download PDF → 
Generate Organizational Insight (AI) → 
Export Executive Summary
```

---

## 16. File `.env.local` Lengkap

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGc...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...

# Anthropic Claude Haiku
ANTHROPIC_API_KEY=sk-ant-api03-...

# App Config
NEXT_PUBLIC_APP_URL=https://score.primerakarya.com
NEXT_PUBLIC_APP_NAME=SCORE

# Demo Mode Flag
NEXT_PUBLIC_DEMO_MODE=true

# PDF Generation
PDF_FONT_URL=/fonts/helvetica.ttf
```

---

## 17. Ringkasan 3-Step Development Plan

| | Step 1 | Step 2 | Step 3 |
|---|---|---|---|
| **Fokus** | Participant flow | Internal tools + AI | Client dashboard + Export |
| **Dashboard** | Peserta | Internal (Admin/Fasilitator/Coach) | Client/Manager/Sponsor |
| **AI Feature** | Belum | Generate Individual + Impact Report | Generate Org Insight + Executive Summary |
| **Database** | 12 tabel core | +4 tabel internal | +4 tabel client/system |
| **Deliverable** | Peserta bisa join kelas → isi semua form → lihat progress | Fasilitator generate laporan AI | Client download PDF laporan |
| **Estimasi** | 2-3 minggu | 2-3 minggu | 1-2 minggu |

---

> **Cara menggunakan file ini di Claude Code:**
> 1. Buka project di Claude Code
> 2. Attach `ARCHITECTURE_STEP1.md` → "Bangun semua yang ada di Step 1"
> 3. Setelah selesai, attach `ARCHITECTURE_STEP2.md` → "Lanjutkan ke Step 2"
> 4. Setelah selesai, attach `ARCHITECTURE_STEP3.md` → "Finalisasi dengan Step 3"
