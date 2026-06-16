# SCORE System — Architecture Step 1
## Foundation, Auth, Database & Participant Dashboard (Core Flow)

> **Prioritas utama:** Flow peserta end-to-end dari login → pilih kelas → isi form tahapan → laporan otomatis
> **Stack:** Next.js 14 (App Router) · Supabase · Tailwind CSS · Claude Haiku API
> **Target Step 1:** Participant Dashboard fully functional + 1 demo class (ACE Batch 3 BRI Life)

---

## 1. Struktur Folder Project

```
score-system/
├── app/
│   ├── (auth)/
│   │   └── login/
│   │       └── page.tsx              # Login page dengan demo buttons
│   ├── (participant)/
│   │   ├── layout.tsx                # Participant shell layout
│   │   ├── dashboard/
│   │   │   └── page.tsx              # Home: daftar kelas tersedia
│   │   ├── class/
│   │   │   ├── page.tsx              # Browse semua kelas
│   │   │   └── [classId]/
│   │   │       ├── page.tsx          # Detail kelas + password gate
│   │   │       └── session/
│   │   │           ├── layout.tsx    # Session stepper layout
│   │   │           ├── step-1-pre-assessment/page.tsx
│   │   │           ├── step-2-learning/page.tsx
│   │   │           ├── step-3-competency-assessment/page.tsx
│   │   │           ├── step-4-evaluation/page.tsx
│   │   │           ├── step-5-action-plan/page.tsx
│   │   │           └── complete/page.tsx
│   ├── (internal)/
│   │   └── layout.tsx                # Placeholder Step 2
│   ├── (client)/
│   │   └── layout.tsx                # Placeholder Step 3
│   └── api/
│       ├── auth/
│       │   └── route.ts
│       └── ai/
│           ├── generate-insight/route.ts
│           └── generate-report/route.ts
├── components/
│   ├── auth/
│   │   ├── LoginForm.tsx
│   │   └── DemoLoginButtons.tsx      # Tombol demo langsung login
│   ├── participant/
│   │   ├── layout/
│   │   │   ├── ParticipantSidebar.tsx
│   │   │   ├── ParticipantHeader.tsx
│   │   │   └── SessionStepper.tsx    # Progress bar antar steps
│   │   ├── class/
│   │   │   ├── ClassCard.tsx         # Card kelas seperti referensi gambar
│   │   │   ├── ClassGrid.tsx
│   │   │   └── PasswordGate.tsx      # Modal input password kelas
│   │   ├── session/
│   │   │   ├── PreAssessmentForm.tsx
│   │   │   ├── LearningPage.tsx
│   │   │   ├── CompetencyForm.tsx    # Form akreditasi dengan rubrik
│   │   │   ├── EvaluationForm.tsx
│   │   │   ├── ActionPlanForm.tsx
│   │   │   └── CompletionPage.tsx
│   │   └── profile/
│   │       └── ParticipantProfile.tsx
│   └── ui/
│       ├── Button.tsx
│       ├── Input.tsx
│       ├── Card.tsx
│       ├── Badge.tsx
│       ├── ProgressBar.tsx
│       ├── Modal.tsx
│       ├── Stepper.tsx
│       └── StarRating.tsx
├── lib/
│   ├── supabase/
│   │   ├── client.ts                 # Browser client
│   │   ├── server.ts                 # Server client
│   │   └── middleware.ts
│   ├── ai/
│   │   └── claude.ts                 # Claude Haiku client
│   └── utils/
│       ├── accreditation.ts          # Hitung score akreditasi
│       └── formatters.ts
├── types/
│   └── index.ts                      # Semua TypeScript types
├── supabase/
│   ├── schema.sql                    # Full database schema
│   ├── seed.sql                      # Demo data ACE Batch 3
│   └── rls.sql                       # Row Level Security policies
└── middleware.ts                     # Auth route protection
```

---

## 2. Database Schema (Supabase PostgreSQL)

### Jalankan di Supabase SQL Editor — urut dari atas ke bawah

```sql
-- ============================================================
-- EXTENSIONS
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ============================================================
-- TABLE: users
-- ============================================================
CREATE TABLE public.users (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email         TEXT UNIQUE NOT NULL,
  full_name     TEXT NOT NULL,
  role          TEXT NOT NULL CHECK (role IN (
                  'super_admin','project_manager','administrator',
                  'facilitator','coach','mentor',
                  'participant','manager','client','business_sponsor'
                )),
  avatar_url    TEXT,
  phone         TEXT,
  is_demo       BOOLEAN DEFAULT FALSE,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: clients
-- ============================================================
CREATE TABLE public.clients (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name          TEXT NOT NULL,
  industry      TEXT,
  logo_url      TEXT,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: competency_frameworks
-- Configurable — tidak hardcoded
-- ============================================================
CREATE TABLE public.competency_frameworks (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name          TEXT NOT NULL,
  description   TEXT,
  competencies  JSONB NOT NULL DEFAULT '[]',
  -- Struktur: [{"id":"c1","name":"Professional Grooming","weight":15,"rating_scale":5,
  --             "indicators":["Penampilan rapi","Hygiene baik",...]}]
  rating_scale  INTEGER DEFAULT 5,
  passing_score NUMERIC(5,2) DEFAULT 65,
  is_active     BOOLEAN DEFAULT TRUE,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: classes (setiap "kelas" = 1 program training)
-- ============================================================
CREATE TABLE public.classes (
  id                      UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  client_id               UUID REFERENCES public.clients(id),
  competency_framework_id UUID REFERENCES public.competency_frameworks(id),
  title                   TEXT NOT NULL,
  subtitle                TEXT,                       -- e.g. "A.C.E Program & Certification"
  batch                   TEXT,                       -- "Batch 3"
  description             TEXT,
  cover_image_url         TEXT,
  facilitator_names       TEXT[],                     -- ["Oki T. Wikan", "Arike Agung Widjaja"]
  location                TEXT,
  start_date              DATE,
  end_date                DATE,
  duration_days           INTEGER,
  objectives              TEXT[],
  password_hash           TEXT NOT NULL,              -- Bcrypt hash dari password kelas
  max_participants        INTEGER DEFAULT 30,
  status                  TEXT DEFAULT 'active' CHECK (status IN ('draft','active','completed','archived')),
  -- Session steps yang aktif untuk kelas ini (configurable)
  active_steps            TEXT[] DEFAULT ARRAY[
                            'pre_assessment','learning',
                            'competency_assessment','evaluation',
                            'action_plan'
                          ],
  created_by              UUID REFERENCES public.users(id),
  created_at              TIMESTAMPTZ DEFAULT NOW(),
  updated_at              TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: class_enrollments
-- Peserta join kelas (setelah input password)
-- ============================================================
CREATE TABLE public.class_enrollments (
  id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  class_id       UUID NOT NULL REFERENCES public.classes(id) ON DELETE CASCADE,
  participant_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  enrolled_at    TIMESTAMPTZ DEFAULT NOW(),
  current_step   TEXT DEFAULT 'pre_assessment',
  is_completed   BOOLEAN DEFAULT FALSE,
  completed_at   TIMESTAMPTZ,
  UNIQUE(class_id, participant_id)
);

-- ============================================================
-- TABLE: step_progress
-- Track progress tiap step per peserta per kelas
-- ============================================================
CREATE TABLE public.step_progress (
  id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enrollment_id  UUID NOT NULL REFERENCES public.class_enrollments(id) ON DELETE CASCADE,
  step_name      TEXT NOT NULL CHECK (step_name IN (
                   'pre_assessment','learning','competency_assessment',
                   'evaluation','action_plan'
                 )),
  status         TEXT DEFAULT 'not_started' CHECK (status IN ('not_started','in_progress','completed')),
  started_at     TIMESTAMPTZ,
  completed_at   TIMESTAMPTZ,
  UNIQUE(enrollment_id, step_name)
);

-- ============================================================
-- TABLE: pre_assessments (Step 1)
-- ============================================================
CREATE TABLE public.pre_assessments (
  id                      UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enrollment_id           UUID NOT NULL REFERENCES public.class_enrollments(id) ON DELETE CASCADE,
  -- Form fields sesuai SCORE Module 3
  business_challenge      TEXT NOT NULL,
  learning_needs          TEXT NOT NULL,
  existing_strengths      TEXT NOT NULL,
  development_areas       TEXT NOT NULL,
  stakeholder_expectations TEXT,
  years_of_experience     INTEGER,
  current_role            TEXT,
  -- Profile tambahan
  gender                  TEXT CHECK (gender IN ('male','female','other')),
  region                  TEXT,
  function_area           TEXT,
  direct_manager_name     TEXT,
  submitted_at            TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: learning_confirmations (Step 2)
-- Peserta konfirmasi materi yang dipelajari
-- ============================================================
CREATE TABLE public.learning_confirmations (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enrollment_id   UUID NOT NULL REFERENCES public.class_enrollments(id) ON DELETE CASCADE,
  -- Materi yang dikonfirmasi dipahami (dari agenda training)
  confirmed_topics JSONB DEFAULT '[]',
  -- [{"topic":"ACE Framework","understood":true},{"topic":"MEET Process","understood":true}]
  notes           TEXT,
  confirmed_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: competency_assessments (Step 3)
-- Form akreditasi dengan rubrik — sesuai laporan ACE
-- ============================================================
CREATE TABLE public.competency_assessments (
  id                 UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enrollment_id      UUID NOT NULL REFERENCES public.class_enrollments(id) ON DELETE CASCADE,
  framework_id       UUID REFERENCES public.competency_frameworks(id),
  -- Scores per komponen — sesuai struktur framework
  scores             JSONB NOT NULL DEFAULT '[]',
  -- [{"competency_id":"c1","competency_name":"Professional Grooming",
  --   "weight":15,"score":4,"weighted_score":0.6,
  --   "sub_scores":[{"id":"c1a","name":"Penampilan","score":4},...]
  --   "notes":"Penampilan rapi dan profesional"}]
  final_score        NUMERIC(5,2),           -- Total weighted score (0-100)
  category           TEXT CHECK (category IN (
                       'kompeten_unggul','kompeten','perlu_pengembangan'
                     )),
  -- Rekomendasi AI
  ai_strength_notes  TEXT,
  ai_dev_notes       TEXT,
  -- Self-assessment (diisi peserta) vs facilitator assessment
  assessment_by      TEXT DEFAULT 'self' CHECK (assessment_by IN ('self','facilitator','both')),
  assessed_at        TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: evaluations (Step 4)
-- Feedback peserta terhadap program — sesuai laporan ACE
-- ============================================================
CREATE TABLE public.evaluations (
  id                   UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enrollment_id        UUID NOT NULL REFERENCES public.class_enrollments(id) ON DELETE CASCADE,
  -- Bagian A: Persepsi terhadap Program (skala 1-5)
  goal_clarity         NUMERIC(3,2),   -- Tujuan disampaikan jelas
  material_relevance   NUMERIC(3,2),   -- Relevansi materi
  trusted_advisor_help NUMERIC(3,2),   -- Membantu jadi Trusted Advisor
  flow_clarity         NUMERIC(3,2),   -- Urutan mudah dipahami
  case_study_help      NUMERIC(3,2),   -- Contoh kasus membantu
  duration_adequate    NUMERIC(3,2),   -- Durasi sesuai kebutuhan
  media_effective      NUMERIC(3,2),   -- Media efektif
  facility_support     NUMERIC(3,2),   -- Lokasi & fasilitas
  confidence_increase  NUMERIC(3,2),   -- Lebih percaya diri
  overall_satisfaction NUMERIC(3,2),   -- Kepuasan keseluruhan
  program_avg          NUMERIC(3,2),   -- Auto-calculated
  -- Bagian B: Penilaian Trainer (skala 1-5)
  trainer_mastery      NUMERIC(3,2),
  trainer_delivery     NUMERIC(3,2),
  trainer_engagement   NUMERIC(3,2),
  trainer_examples     NUMERIC(3,2),
  trainer_feedback     NUMERIC(3,2),
  trainer_atmosphere   NUMERIC(3,2),
  trainer_avg          NUMERIC(3,2),   -- Auto-calculated
  -- Bagian C: Umpan Balik Kualitatif
  most_beneficial      TEXT,
  improvement_suggestions TEXT,
  additional_topics    TEXT,
  nps_score            INTEGER CHECK (nps_score BETWEEN 0 AND 10),
  submitted_at         TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: action_plans (Step 5)
-- ============================================================
CREATE TABLE public.action_plans (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enrollment_id       UUID NOT NULL REFERENCES public.class_enrollments(id) ON DELETE CASCADE,
  -- Peserta bisa buat multiple action plan items
  items               JSONB NOT NULL DEFAULT '[]',
  -- [{"id":"ap1","action":"Implementasi ACE framework dengan 3 nasabah per minggu",
  --   "due_date":"2025-09-06","success_indicator":"3 nasabah ter-onboard",
  --   "status":"not_started","commitment_level":"high"}]
  commitment_statement TEXT,            -- Pernyataan komitmen peserta
  coach_name          TEXT,             -- Nama coach untuk follow-up
  next_check_date     DATE,
  submitted_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: generated_reports
-- Laporan yang di-generate AI setelah peserta selesai semua step
-- ============================================================
CREATE TABLE public.generated_reports (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enrollment_id     UUID NOT NULL REFERENCES public.class_enrollments(id),
  class_id          UUID NOT NULL REFERENCES public.classes(id),
  report_type       TEXT NOT NULL CHECK (report_type IN (
                      'individual_profile','training_impact','executive_summary'
                    )),
  -- Full content dari AI
  content           JSONB NOT NULL DEFAULT '{}',
  -- {"executive_summary":"...","strength_profile":"...","dev_priorities":"...",
  --  "competency_summary":"...","learning_path":"...","recommendations":"..."}
  raw_text          TEXT,               -- Markdown format
  ai_model          TEXT DEFAULT 'claude-haiku-4-5-20251001',
  generated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- INDEXES
-- ============================================================
CREATE INDEX idx_enrollments_class ON public.class_enrollments(class_id);
CREATE INDEX idx_enrollments_participant ON public.class_enrollments(participant_id);
CREATE INDEX idx_step_progress_enrollment ON public.step_progress(enrollment_id);
CREATE INDEX idx_pre_assessments_enrollment ON public.pre_assessments(enrollment_id);
CREATE INDEX idx_evaluations_enrollment ON public.evaluations(enrollment_id);
CREATE INDEX idx_action_plans_enrollment ON public.action_plans(enrollment_id);
CREATE INDEX idx_reports_enrollment ON public.generated_reports(enrollment_id);
CREATE INDEX idx_reports_class ON public.generated_reports(class_id);

-- ============================================================
-- ROW LEVEL SECURITY
-- ============================================================
ALTER TABLE public.classes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.class_enrollments ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.pre_assessments ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.competency_assessments ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.evaluations ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.action_plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.generated_reports ENABLE ROW LEVEL SECURITY;

-- Peserta hanya bisa lihat data mereka sendiri
CREATE POLICY "participant_own_enrollment" ON public.class_enrollments
  FOR ALL USING (participant_id = auth.uid());

CREATE POLICY "participant_own_pre_assessment" ON public.pre_assessments
  FOR ALL USING (
    enrollment_id IN (
      SELECT id FROM public.class_enrollments WHERE participant_id = auth.uid()
    )
  );

CREATE POLICY "participant_own_assessment" ON public.competency_assessments
  FOR ALL USING (
    enrollment_id IN (
      SELECT id FROM public.class_enrollments WHERE participant_id = auth.uid()
    )
  );

CREATE POLICY "participant_own_evaluation" ON public.evaluations
  FOR ALL USING (
    enrollment_id IN (
      SELECT id FROM public.class_enrollments WHERE participant_id = auth.uid()
    )
  );

CREATE POLICY "participant_own_action_plan" ON public.action_plans
  FOR ALL USING (
    enrollment_id IN (
      SELECT id FROM public.class_enrollments WHERE participant_id = auth.uid()
    )
  );

CREATE POLICY "participant_own_report" ON public.generated_reports
  FOR ALL USING (
    enrollment_id IN (
      SELECT id FROM public.class_enrollments WHERE participant_id = auth.uid()
    )
  );

-- Kelas bisa dilihat semua orang yang authenticated
CREATE POLICY "classes_viewable_by_authenticated" ON public.classes
  FOR SELECT USING (auth.role() = 'authenticated');
```

---

## 3. Seed Data — Demo Class ACE Batch 3

```sql
-- File: supabase/seed.sql
-- Jalankan SETELAH schema.sql

-- Demo users (password semua: "demo123")
INSERT INTO public.users (id, email, full_name, role, is_demo) VALUES
  -- Participant demo accounts
  ('u0000001-0000-0000-0000-000000000001', 'nadia@demo.score', 'Nadia Anggraini Putri', 'participant', TRUE),
  ('u0000001-0000-0000-0000-000000000002', 'sahadatu@demo.score', 'Sahadatu Chandra', 'participant', TRUE),
  ('u0000001-0000-0000-0000-000000000003', 'brian@demo.score', 'Brian Bagus Sadewo', 'participant', TRUE),
  ('u0000001-0000-0000-0000-000000000004', 'siska@demo.score', 'Siska Febriani Sihombing', 'participant', TRUE),
  -- Internal staff demo
  ('u0000002-0000-0000-0000-000000000001', 'admin@demo.score', 'Demo Admin', 'super_admin', TRUE),
  ('u0000002-0000-0000-0000-000000000002', 'fasilitator@demo.score', 'Oki T. Wikan', 'facilitator', TRUE),
  -- Client demo
  ('u0000003-0000-0000-0000-000000000001', 'client@demo.score', 'BRI Life HR', 'client', TRUE);

-- Demo client
INSERT INTO public.clients (id, name, industry) VALUES
  ('c0000001-0000-0000-0000-000000000001', 'BRI Life', 'Insurance & Bancassurance');

-- ACE Competency Framework
INSERT INTO public.competency_frameworks (id, name, description, competencies, rating_scale, passing_score) VALUES (
  'f0000001-0000-0000-0000-000000000001',
  'ACE Accreditation Framework — PBS',
  'Affluent Customer Experience framework untuk Priority Bancassurance Specialist',
  '[
    {
      "id": "c1", "name": "1. Professional Grooming", "weight": 15,
      "rating_scale": 5,
      "sub_competencies": [
        {
          "id": "c1a", "name": "Penampilan rapi sesuai standar korporat",
          "weight": 10, "description": "Pakaian, sepatu, rambut, aksesori sesuai standar BRI Life",
          "rubric": {
            "1": "Pakaian tidak rapi, tidak sesuai standar",
            "2": "Masih ada elemen tidak rapi",
            "3": "Penampilan cukup baik, sebagian besar sesuai standar",
            "4": "Penampilan rapi, sesuai standar korporat",
            "5": "Sangat rapi, profesional, konsisten dengan citra brand"
          }
        },
        {
          "id": "c1b", "name": "Hygiene & kesegaran diri",
          "weight": 5, "description": "Kebersihan dan kerapian diri secara keseluruhan",
          "rubric": {
            "1": "Terlihat tidak segar, kebersihan tidak terjaga",
            "2": "Kebersihan cukup, ada beberapa hal kurang terawat",
            "3": "Kebersihan diri cukup baik dan layak",
            "4": "Bersih, segar, terlihat terawat",
            "5": "Sangat bersih, segar, menampilkan kesan profesional"
          }
        }
      ]
    },
    {
      "id": "c2", "name": "2. Workplace Professionalism", "weight": 15,
      "rating_scale": 5,
      "sub_competencies": [
        {
          "id": "c2a", "name": "Sikap positif & manner saat interaksi",
          "weight": 5, "description": "Ramah, sopan, percaya diri dalam berinteraksi",
          "rubric": {
            "1": "Sikap kurang sopan, kurang percaya diri",
            "2": "Sikap belum konsisten, ada momen tidak sesuai",
            "3": "Sikap cukup baik, ramah dan sopan",
            "4": "Sikap profesional, percaya diri",
            "5": "Sangat profesional, manner prima, penuh kehangatan"
          }
        },
        {
          "id": "c2b", "name": "Disiplin waktu & kesiapan",
          "weight": 5, "description": "Ketepatan waktu dan kesiapan dalam sesi",
          "rubric": {
            "1": "Datang terlambat, tidak siap",
            "2": "Datang mendekati waktu, persiapan minim",
            "3": "Siap sebagian, hadir tepat waktu",
            "4": "Persiapan baik, hadir sebelum waktu",
            "5": "Sangat siap, hadir lebih awal, proaktif"
          }
        },
        {
          "id": "c2c", "name": "Professional presence keseluruhan",
          "weight": 5, "description": "Kehadiran profesional secara keseluruhan selama sesi",
          "rubric": {
            "1": "Tidak menunjukkan presence profesional",
            "2": "Presence profesional belum konsisten",
            "3": "Cukup profesional dalam sebagian besar waktu",
            "4": "Profesional dan konsisten sepanjang sesi",
            "5": "Luar biasa profesional, menjadi role model"
          }
        }
      ]
    },
    {
      "id": "c3", "name": "3. ACE Process — MEET", "weight": 30,
      "rating_scale": 5,
      "sub_competencies": [
        {
          "id": "c3a", "name": "Open Meeting (Model W.I.N: Why–Insight–Next Step)",
          "weight": 10,
          "rubric": {
            "1": "Tidak terstruktur, tidak mengikuti alur W.I.N",
            "2": "Struktur lemah, sebagian langkah terlewat",
            "3": "Cukup jelas, memenuhi sebagian alur",
            "4": "Terstruktur baik dan jelas",
            "5": "Sangat terstruktur, membangun kepercayaan nasabah"
          }
        },
        {
          "id": "c3b", "name": "Identify Goals (4 Komponen Wealth Management)",
          "weight": 10,
          "rubric": {
            "1": "Tidak menggunakan prosedur, pertanyaan tidak relevan",
            "2": "Beberapa komponen WM tidak digali",
            "3": "Sesuai sebagian, masih ada area kurang",
            "4": "Hampir lengkap, mencakup komponen WM utama",
            "5": "Lengkap: Wealth Creation, Preservation, Distribution, Business Insurance"
          }
        },
        {
          "id": "c3c", "name": "Understand Goals (Probing C.E.O: Concern–Emotion–Outcome)",
          "weight": 10,
          "rubric": {
            "1": "Probing dangkal, tidak menggali C.E.O",
            "2": "Probing kurang menyentuh motivasi nasabah",
            "3": "Probing cukup, sebagian relevan",
            "4": "Probing mendalam & relevan",
            "5": "Probing sangat kuat, menggali motivasi emosional & kebutuhan jangka panjang"
          }
        }
      ]
    },
    {
      "id": "c4", "name": "4. ACE Process — PRESENT", "weight": 30,
      "rating_scale": 5,
      "sub_competencies": [
        {
          "id": "c4a", "name": "Menyampaikan solusi sesuai kebutuhan (4 pilar WM)",
          "weight": 10,
          "rubric": {
            "1": "Solusi tidak relevan dengan kebutuhan nasabah",
            "2": "Relevansi lemah, hanya sebagian sesuai",
            "3": "Cukup relevan & layak",
            "4": "Jelas, relevan, sesuai kebutuhan",
            "5": "Sangat relevan, langsung menjawab kebutuhan inti nasabah"
          }
        },
        {
          "id": "c4b", "name": "Storytelling/data nasabah saat menjelaskan FAB Produk",
          "weight": 10,
          "rubric": {
            "1": "Tidak tepat, tidak terhubung dengan kebutuhan nasabah",
            "2": "Kurang sesuai konteks, contoh kurang kuat",
            "3": "Cukup tepat",
            "4": "Tepat, menarik, membantu nasabah memahami solusi",
            "5": "Sangat kuat, emosional/logis, membuat nasabah engaged"
          }
        },
        {
          "id": "c4c", "name": "Menangani keberatan dengan model C.A.R (Connect–Alignment–Reassure)",
          "weight": 10,
          "rubric": {
            "1": "Jawaban tidak tepat, tidak mengatasi keberatan",
            "2": "Jawaban kurang selaras & tidak menenangkan",
            "3": "Jawaban cukup tepat",
            "4": "Tepat, relevan, membantu nasabah",
            "5": "Sangat efektif, menenangkan kekhawatiran, meningkatkan kepercayaan"
          }
        }
      ]
    },
    {
      "id": "c5", "name": "5. ACE Process — ASK", "weight": 10,
      "rating_scale": 5,
      "sub_competencies": [
        {
          "id": "c5a", "name": "Trial to Close 4x dengan model L.A.S.T",
          "weight": 5,
          "rubric": {
            "1": "Tidak melakukan trial to close",
            "2": "Melakukan tetapi tidak terstruktur",
            "3": "Cukup jelas",
            "4": "Terstruktur & jelas",
            "5": "Sangat terstruktur, membangun keputusan dengan elegan"
          }
        },
        {
          "id": "c5b", "name": "Closing & ajakan langkah lanjut",
          "weight": 2.5,
          "rubric": {
            "1": "Tidak ada upaya closing",
            "2": "Upaya sangat lemah",
            "3": "Closing lemah atau ragu",
            "4": "Closing baik & jelas",
            "5": "Closing kuat, meyakinkan, nasabah jelas next step"
          }
        },
        {
          "id": "c5c", "name": "Meminta referral & Reciprocal business produk BRI",
          "weight": 2.5,
          "rubric": {
            "1": "Tidak meminta referral",
            "2": "Meminta referral dengan pasif/tidak yakin",
            "3": "Cukup, sesuai prosedur minimal",
            "4": "Meminta referral dengan percaya diri & natural",
            "5": "Sangat meyakinkan, menghasilkan referral berkualitas"
          }
        }
      ]
    }
  ]',
  5,
  65.0
);

-- Demo Class: ACE Batch 3
-- Password kelas: "ACE2025" (hash ini adalah bcrypt dari "ACE2025")
INSERT INTO public.classes (
  id, client_id, competency_framework_id,
  title, subtitle, batch, description, cover_image_url,
  facilitator_names, location, start_date, end_date, duration_days,
  objectives, password_hash, max_participants, status
) VALUES (
  'cl000001-0000-0000-0000-000000000001',
  'c0000001-0000-0000-0000-000000000001',
  'f0000001-0000-0000-0000-000000000001',
  'Affluent Customer Experience (A.C.E)',
  'Program & Certification — Priority Bancassurance Specialist',
  'Batch 3',
  'Program strategis untuk mempersiapkan Priority Bancassurance Specialist (PBS) sebagai Trusted Insurance Advisor di segmen affluent. Pelatihan intensif selama 3 hari mencakup workshop, simulasi, roleplay, coaching clinic, dan akreditasi.',
  '/images/ace-cover.jpg',
  ARRAY['Oki T. Wikan', 'Arike Agung Widjaja'],
  'Hotel Harris Sentul',
  '2025-08-06',
  '2025-08-08',
  3,
  ARRAY[
    'Meningkatkan kapabilitas PBS dalam financial needs discovery & consultative selling',
    'Menguasai teknik presentasi, objection handling, dan closing yang terstruktur',
    'Menjalankan end-to-end advisory menggunakan framework A.C.E (PREPARE–CONNECT–PRESENT–CLOSE–RETAIN)',
    'Lulus Akreditasi ACE sebagai validasi kompetensi profesional'
  ],
  '$2a$10$demo_hash_ACE2025_replace_with_real_bcrypt',
  30,
  'active'
);

-- CATATAN: Generate bcrypt hash "ACE2025" di aplikasi saat setup
-- Atau gunakan: SELECT crypt('ACE2025', gen_salt('bf')) dari pgcrypto extension
```

---

## 4. Auth System

### `lib/supabase/client.ts`
```typescript
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

### `lib/supabase/server.ts`
```typescript
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options))
        }
      }
    }
  )
}
```

### `middleware.ts` — Route Protection
```typescript
import { NextResponse, type NextRequest } from 'next/server'
import { createServerClient } from '@supabase/ssr'

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request })
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: { /* ... */ } }
  )

  const { data: { user } } = await supabase.auth.getUser()
  const path = request.nextUrl.pathname

  // Redirect ke login jika belum auth
  if (!user && path.startsWith('/(participant)')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Redirect based on role
  if (user) {
    const { data: profile } = await supabase
      .from('users').select('role').eq('id', user.id).single()

    if (profile?.role === 'participant' && path.startsWith('/internal')) {
      return NextResponse.redirect(new URL('/dashboard', request.url))
    }
  }

  return supabaseResponse
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|api).*)']
}
```

---

## 5. Login Page dengan Demo Buttons

### `app/(auth)/login/page.tsx`

**UI Behavior:**
```
┌─────────────────────────────────────────────┐
│  [Logo SCORE]   Human Development Ecosystem  │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │           Masuk ke SCORE                │ │
│  │                                         │ │
│  │  Email: [________________________]      │ │
│  │  Password: [____________________]       │ │
│  │                                         │ │
│  │  [        Masuk        ] (primary btn)  │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  ─── Demo Login (untuk testing) ───          │
│                                              │
│  [Peserta: Nadia]  [Peserta: Sahadatu]       │
│  [Peserta: Brian]  [Peserta: Siska]          │
│  [Internal: Admin] [Client: BRI Life HR]     │
└─────────────────────────────────────────────┘
```

**Logic demo button:**
```typescript
// Setiap demo button langsung signIn dengan credential dummy
const demoLogin = async (role: string) => {
  const credentials = {
    'nadia':    { email: 'nadia@demo.score',     password: 'demo123' },
    'sahadatu': { email: 'sahadatu@demo.score',  password: 'demo123' },
    'brian':    { email: 'brian@demo.score',     password: 'demo123' },
    'admin':    { email: 'admin@demo.score',     password: 'demo123' },
    'client':   { email: 'client@demo.score',    password: 'demo123' },
  }
  await supabase.auth.signInWithPassword(credentials[role])
  // Redirect berdasarkan role dari tabel users
}
```

---

## 6. Participant Dashboard Layout

### Visual Reference (dari gambar yang diberikan)

Adaptasi dari desain Academy ke SCORE:
```
┌──────────────────────────────────────────────────────────────┐
│ SCORE │  [Search kelas...]      │  [Notif] [Profil] [Keluar] │
├───────┤                                                       │
│       │  Selamat datang, Nadia!                               │
│ MAIN  │                                                       │
│       │  ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│ Home  │  │ Kelas    │ │Progress  │ │Sertifikat│ [Daftar Kelas]│
│       │  │ Diikuti  │ │ Overall  │ │ Diraih   │              │
│ Kelas │  │    1     │ │   67%    │ │    0     │              │
│       │  └──────────┘ └──────────┘ └──────────┘              │
│ Progr.│                                                       │
│       │  ── Kelas Aktif ──────────────────────── [Lihat Semua]│
│ Sert. │                                                       │
│       │  ┌─────────────────────────────────────────────────┐  │
│       │  │ [IMG] ACE Program — Batch 3         AKTIF       │  │
│       │  │       BRI Life · Hotel Harris Sentul            │  │
│       │  │       Oki T. Wikan & Arike Agung Widjaja        │  │
│       │  │       6–8 Agustus 2025                          │  │
│       │  │       ████████░░ Step 3/5 — Assessment          │  │
│       │  │                              [Lanjutkan →]      │  │
│       │  └─────────────────────────────────────────────────┘  │
│       │                                                       │
│       │  ── Kelas Tersedia ──────────────────── [Lihat Semua] │
│       │                                                       │
│       │  ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│       │  │ ACE      │ │ Advanced │ │ Wealth   │              │
│       │  │ Batch 4  │ │ Sales    │ │ Advisory │              │
│       │  │ (locked) │ │ (locked) │ │ (locked) │              │
│       │  │[+Daftar] │ │[+Daftar] │ │[+Daftar] │              │
│       │  └──────────┘ └──────────┘ └──────────┘              │
└───────┴───────────────────────────────────────────────────────┘
```

---

## 7. Password Gate — Join Kelas

```
Saat peserta klik "Daftar" atau "Buka Kelas":

┌──────────────────────────────────────────┐
│  Masuk ke Kelas                    [✕]   │
│  ─────────────────────────────────────   │
│  ACE Program Batch 4                     │
│  BRI Life · 3 Hari · Sentul              │
│                                          │
│  Masukkan password kelas yang diberikan  │
│  oleh trainer saat sesi offline:         │
│                                          │
│  [ _ _ _ _ _ _ _ _ ] (password input)   │
│                                          │
│  [     Masuk ke Kelas     ]              │
│                                          │
│  Belum punya password? Hubungi trainer.  │
└──────────────────────────────────────────┘
```

**Validasi password:**
```typescript
// app/api/class/join/route.ts
import bcrypt from 'bcrypt'

export async function POST(req: Request) {
  const { classId, password, participantId } = await req.json()

  const { data: classData } = await supabase
    .from('classes').select('password_hash').eq('id', classId).single()

  const isValid = await bcrypt.compare(password, classData.password_hash)
  if (!isValid) return Response.json({ error: 'Password salah' }, { status: 401 })

  // Create enrollment
  await supabase.from('class_enrollments').upsert({
    class_id: classId, participant_id: participantId,
    current_step: 'pre_assessment'
  })

  // Init step_progress untuk semua steps
  const steps = ['pre_assessment','learning','competency_assessment','evaluation','action_plan']
  await supabase.from('step_progress').insert(
    steps.map(step => ({ enrollment_id: newEnrollment.id, step_name: step, status: 'not_started' }))
  )

  return Response.json({ success: true })
}
```

---

## 8. Session Flow — Stepper Component

```
Step Progress Bar (selalu tampil di atas saat dalam sesi):

[✓] Pre-Assessment  →  [✓] Materi  →  [●] Assessment  →  [○] Evaluasi  →  [○] Action Plan
     Selesai               Selesai        Sedang             Belum              Belum

✓ = completed (hijau)
● = current (biru, pulsing)
○ = locked (abu-abu)
```

**Aturan navigasi:**
- Step hanya bisa diakses secara berurutan
- Step sebelumnya bisa dilihat tapi tidak bisa diedit
- Setelah semua step selesai → redirect ke `/class/[id]/complete`

---

## 9. TypeScript Types

### `types/index.ts`
```typescript
export type UserRole = 
  'super_admin' | 'project_manager' | 'administrator' |
  'facilitator' | 'coach' | 'mentor' |
  'participant' | 'manager' | 'client' | 'business_sponsor'

export type StepName = 
  'pre_assessment' | 'learning' | 'competency_assessment' | 
  'evaluation' | 'action_plan'

export type StepStatus = 'not_started' | 'in_progress' | 'completed'

export type AccreditationCategory = 
  'kompeten_unggul' | 'kompeten' | 'perlu_pengembangan'

export interface User {
  id: string
  email: string
  full_name: string
  role: UserRole
  avatar_url?: string
  is_demo: boolean
}

export interface Class {
  id: string
  title: string
  subtitle?: string
  batch?: string
  description?: string
  cover_image_url?: string
  facilitator_names: string[]
  location?: string
  start_date?: string
  end_date?: string
  duration_days?: number
  objectives: string[]
  max_participants: number
  status: 'draft' | 'active' | 'completed' | 'archived'
  client?: { name: string; logo_url?: string }
  competency_framework?: CompetencyFramework
}

export interface Enrollment {
  id: string
  class_id: string
  participant_id: string
  current_step: StepName
  is_completed: boolean
  completed_at?: string
  enrolled_at: string
  class?: Class
  step_progress?: StepProgress[]
}

export interface StepProgress {
  id: string
  enrollment_id: string
  step_name: StepName
  status: StepStatus
  started_at?: string
  completed_at?: string
}

export interface SubCompetency {
  id: string
  name: string
  weight: number
  description?: string
  rubric: Record<string, string> // {"1":"...", "2":"...", "5":"..."}
}

export interface Competency {
  id: string
  name: string
  weight: number
  rating_scale: number
  sub_competencies: SubCompetency[]
}

export interface CompetencyFramework {
  id: string
  name: string
  competencies: Competency[]
  rating_scale: number
  passing_score: number
}

export interface ActionPlanItem {
  id: string
  action: string
  due_date: string
  success_indicator: string
  status: 'not_started' | 'in_progress' | 'completed'
  commitment_level: 'low' | 'medium' | 'high'
}

export interface GeneratedReport {
  id: string
  enrollment_id: string
  report_type: 'individual_profile' | 'training_impact' | 'executive_summary'
  content: {
    executive_summary: string
    strength_profile: string
    development_priorities: string
    competency_summary: string
    learning_path: string
    recommendations: string
    action_plan_summary: string
  }
  raw_text: string
  generated_at: string
}
```

---

## 10. Environment Variables

```env
# .env.local

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...   # Hanya di server-side

# Claude AI (Anthropic)
ANTHROPIC_API_KEY=sk-ant-...

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 11. Package Dependencies

```json
{
  "dependencies": {
    "next": "14.2.x",
    "react": "^18.3.x",
    "react-dom": "^18.3.x",
    "@supabase/supabase-js": "^2.45.x",
    "@supabase/ssr": "^0.5.x",
    "@anthropic-ai/sdk": "^0.30.x",
    "bcrypt": "^5.1.x",
    "tailwindcss": "^3.4.x",
    "class-variance-authority": "^0.7.x",
    "clsx": "^2.1.x",
    "tailwind-merge": "^2.5.x",
    "lucide-react": "^0.447.x",
    "react-hook-form": "^7.53.x",
    "zod": "^3.23.x",
    "@hookform/resolvers": "^3.9.x",
    "framer-motion": "^11.x",
    "sonner": "^1.5.x"
  },
  "devDependencies": {
    "@types/bcrypt": "^5.0.x",
    "@types/node": "^20.x",
    "@types/react": "^18.x",
    "typescript": "^5.x"
  }
}
```

---

## 12. Checklist Build Step 1

```
FOUNDATION
[ ] Setup Next.js 14 + Tailwind + TypeScript
[ ] Setup Supabase project
[ ] Jalankan schema.sql di Supabase
[ ] Jalankan seed.sql (demo data)
[ ] Setup environment variables

AUTH
[ ] Login page dengan form email/password
[ ] Demo login buttons (6 akun demo)
[ ] Middleware route protection berdasarkan role
[ ] Redirect berdasarkan role setelah login

PARTICIPANT DASHBOARD HOME
[ ] Layout sidebar + header (sesuai desain referensi)
[ ] Stat cards: kelas diikuti, progress, sertifikat
[ ] Kelas aktif dengan progress bar
[ ] Grid kelas tersedia
[ ] Password gate modal

SESSION FLOW (Core — Step 1 Deliverable Utama)
[ ] Session stepper component
[ ] Step 1: Pre-Assessment form
[ ] Step 2: Learning confirmation page
[ ] Step 3: Competency Assessment form (rubrik ACE)
[ ] Step 4: Evaluation form (sesuai laporan)
[ ] Step 5: Action Plan form
[ ] Completion page

DATABASE
[ ] Semua form tersimpan ke Supabase
[ ] Progress tracking per step
[ ] Auto-calculate competency scores
```

---

> **Lanjut ke ARCHITECTURE_STEP2.md** untuk:
> Internal Dashboard (Super Admin, Facilitator, Coach) + AI Report Generation
