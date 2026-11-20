# Unmeasured — Engineering Specification (`docs/SPEC.md`)

**Stack:** Vite + React + TypeScript + Tailwind CSS + shadcn/ui + Vite PWA (Workbox) + `i18next` + Dexie.js (offline read cache & upload outbox) + Supabase (Auth + Postgres + RLS) + Cloudflare (Workers + Workflows + R2 + Turnstile) + `@google/genai` (Gemini Paid Tier).

**Core Architectural Principles:**
1. **Cloud-backed durability, offline-capable kitchen:** Supabase Postgres + Cloudflare R2 are the canonical source of truth. Dexie.js is strictly a local offline read cache and upload outbox—never a fragile single-device primary database.
2. **Durable async AI execution:** Long-running audio/OCR extractions run in **Cloudflare Workflows** (never request-scoped `ctx.waitUntil`). Temporary Gemini Files API uploads are explicitly deleted once extraction completes.
3. **2-Field Provenance:** Every ingredient field, step instruction, and duration carries `{ source, verified }`. Never invent precision or show countdown timers for unverified AI-guessed durations.
4. **Explicit Allergen Safety:** Zero AI auto-inference of allergens or "allergen-free" status. Unreviewed recipes explicitly display `not_reviewed`.
5. **Deterministic Data, Translated Prose:** LLMs translate only free-text fields (`title`, `story`, `instruction`, `sensory_cues`). Structured quantities and calibrated utensils are rendered deterministically from data.

---

## 1. Repository Structure

```text
unmeasured/
├── docs/
│   └── SPEC.md                     # This specification
├── supabase/
│   └── migrations/                 # Postgres schema, RLS policies, invite functions
├── workers/                        # Cloudflare Workers + Workflows (Wrangler)
│   ├── src/
│   │   ├── index.ts                # API router (Turnstile, Supabase JWT auth, quota check)
│   │   ├── workflows/
│   │   │   └── extractRecipe.ts    # Durable Cloudflare Workflow: R2 -> Gemini -> Postgres -> cleanup
│   │   ├── routes/
│   │   │   ├── upload.ts           # Multipart/R2 media upload
│   │   │   ├── clarify.ts          # Merge <= 3 clarification answers
│   │   │   └── translate.ts        # Lazy free-text 5-language translation + casual Romanization
│   │   └── lib/
│   │       ├── gemini.ts           # @google/genai client, usageMetadata spend tracker
│   │       └── schemas.ts          # Shared Zod contracts
├── src/
│   ├── db/
│   │   ├── cache.ts                # Dexie offline read cache + pending upload outbox
│   │   └── sync.ts                 # Background outbox flusher & Supabase pull
│   ├── i18n/                       # Offline UI bundles (en, mr, hi, te, gu) + culinary glossary
│   ├── schemas/                    # Shared Zod schemas (mirrored with workers)
│   ├── services/                   # AudioRecorder, WakeLock, ShareService (WhatsApp text + Web Link)
│   ├── components/                 # WCAG 2.2 AA primitives, ProvenancePill, AllergenBanner
│   ├── pages/                      # Vault, Capture, Review, Detail, Cook, SharedView, Settings
│   └── styles/                     # Parchment (#FAF6F0) / Terracotta (#B84A27 / #9A3412) tokens
└── tests/                          # Vitest unit/axe tests + Playwright phone/iPad E2E
```

---

## 2. Database Schema (Supabase Postgres + RLS)

All tables use `uuid` primary keys, `timestamptz`, and Row-Level Security (RLS) scoped to `vault_members` where `user_id = auth.uid()`.

### Core Tables
* **`vaults`**: `id`, `name`, `owner_id`, `max_seats` (default `6`), `ui_language` (`en|mr|hi|te|gu`), `created_at`
* **`vault_members`**: `vault_id`, `user_id`, `role` (`owner|contributor|cook`), `joined_at`
* **`vault_invites`**: `id`, `vault_id`, `token_hash`, `role`, `expires_at`, `used_by`, `used_at`
* **`vault_allergies`**: `id`, `vault_id`, `allergens` (`text[]` — FDA Big 9 + `mustard`, `coconut`), `updated_at` *(vault-scoped household profile)*
* **`measurement_profiles`**: `id`, `vault_id`, `display_name`, `capacity_ml`, `staple_grams_note`, `photo_r2_key`, `verified_by`, `updated_at`
* **`recipes`**:
  * `id`, `vault_id`, `title`, `cook_name`, `recorded_by`, `original_languages` (`text[]`), `capture_mode` (`voice|ocr|voice_plus_ocr|import`)
  * `lifecycle_status` (`draft|published|archived`) *(strictly separated from AI job state)*
  * `allergen_review_status` (`not_reviewed|reviewed|needs_recheck`)
  * `confirmed_allergens` (`text[]`), `dietary_tags` (`text[]`)
  * `prep_ahead` (`text[]`), `equipment` (`text[]`), `servings_display` (`text`)
  * `current_revision` (`int`), `share_token` (`uuid`, nullable — enables read-only public web link)
  * `working_draft_json` (`jsonb`, nullable — holds unpublished edits without mutating the published revision)
  * `updated_at`, `created_at`
* **`recipe_revisions`**: `id`, `recipe_id`, `revision_number`, `snapshot_json` (`jsonb`), `model_id`, `prompt_version`, `schema_version`, `created_by`, `created_at` *(immutable published snapshots)*
* **`recipe_media`**: `id`, `vault_id`, `recipe_id`, `media_type` (`audio_segment|audio_clarification|photo_cooking|scan_ocr_page`), `r2_key`, `mime_type`, `segment_index`, `offset_sec`, `duration_sec`, `created_at`
* **`ai_jobs`**: `id`, `vault_id`, `recipe_id`, `job_type` (`extract|clarify|translate`), `status` (`queued|uploading|processing|retrying|succeeded|failed`), `attempt_count`, `workflow_instance_id`, `error_message`, `token_usage_json`, `created_at`, `updated_at`
* **`recipe_translations`**: `id`, `recipe_id`, `target_lang` (`en|mr|hi|te|gu`), `source_revision_number`, `free_text_payload` (`jsonb`), `romanized_payload` (`jsonb`, nullable), `model_id`, `created_at` — Unique index on `(recipe_id, target_lang, source_revision_number)`.

### Dexie Offline Cache (`src/db/cache.ts`)
* `cached_recipes`: mirrors published `recipes` + `recipe_translations` for 100% offline Cook Mode.
* `media_cache`: caches played audio/photos in Cache Storage / OPFS for offline kitchen playback.
* `upload_outbox`: queues offline audio segments, OCR scans, and offline clarification answers until connectivity returns.

---

## 3. Exact TypeScript & Zod Contracts (`src/schemas/recipe.ts`)

```ts
import { z } from 'zod';

export const ProvenanceSchema = z.object({
  source: z.enum(['spoken', 'ocr', 'clarified', 'manual', 'inferred']),
  verified: z.boolean(),
});

export const IngredientSchema = z.object({
  id: z.string(),
  name_original: z.string(),
  name_display: z.string(),
  name_provenance: ProvenanceSchema,
  quantity_original: z.string().nullable(),
  quantity_provenance: ProvenanceSchema,
  unit_standard: z.string().nullable(),
  quantity_standard_value: z.number().nullable(),
  measurement_profile_id: z.string().uuid().nullable(),
  source_media_id: z.string().uuid().nullable(),
  notes: z.string().nullable(),
});

export const SensoryCueSchema = z.object({
  type: z.enum(['visual', 'aroma', 'sound', 'texture', 'taste']),
  text: z.string(),
  provenance: ProvenanceSchema,
});

export const StepSchema = z.object({
  id: z.string(),
  step_number: z.number().int().positive(),
  instruction_original: z.string(),
  instruction_display: z.string(),
  instruction_provenance: ProvenanceSchema,
  duration_sec: z.number().int().positive().nullable(),
  duration_provenance: ProvenanceSchema.nullable(),
  sensory_cues: z.array(SensoryCueSchema),
  audio_media_id: z.string().uuid().nullable(),
  audio_start_sec: z.number().nonnegative().nullable(),
  audio_end_sec: z.number().nonnegative().nullable(),
  photo_media_ids: z.array(z.string().uuid()),
});

export const ClarificationQuestionSchema = z.object({
  id: z.string(),
  priority: z.number().int().min(1).max(3),
  target_path: z.string(), // e.g., "steps[0].sensory_cues"
  gap_type: z.enum(['missing_sensory_cue', 'uncalibrated_utensil', 'ambiguous_timing_or_heat']),
  question: z.string(),
  suggested_options: z.array(z.string()).max(4),
  resolved: z.boolean(),
  answer_text: z.string().nullable(),
  answer_audio_media_id: z.string().uuid().nullable(),
});

export const ExtractionResultSchema = z.object({
  schema_version: z.literal(1),
  title: z.string(),
  original_languages: z.array(z.string()),
  story_summary: z.string().nullable(),
  servings_display: z.string().nullable(),
  prep_ahead: z.array(z.string()),
  equipment: z.array(z.string()),
  ingredients: z.array(IngredientSchema),
  steps: z.array(StepSchema),
  clarifying_questions: z.array(ClarificationQuestionSchema).max(3),
  additional_gaps: z.array(z.string()), // Surfaced in Review beyond the 3-question cap
});

// Clarification Merge Request Contract
export const ClarifyRequestSchema = z.object({
  recipe_id: z.string().uuid(),
  answers: z.array(z.object({
    question_id: z.string(),
    answer_text: z.string().nullable(),
    answer_audio_r2_key: z.string().nullable(),
    cook_unavailable: z.boolean().default(false),
  })),
});

// Free-Text-Only Translation Payload (never mutates quantities/units)
export const TranslationPayloadSchema = z.object({
  title: z.string(),
  story_summary: z.string().nullable(),
  prep_ahead: z.array(z.string()),
  equipment: z.array(z.string()),
  ingredients: z.array(z.object({
    id: z.string(),
    name_translated: z.string(),
    notes_translated: z.string().nullable(),
  })),
  steps: z.array(z.object({
    id: z.string(),
    instruction_translated: z.string(),
    sensory_cues_translated: z.array(z.string()),
  })),
});
```

---

## 4. Cloudflare Workflows AI Pipeline & Abuse Protection

1. **Auth & Protection:** Every Worker endpoint verifies the Supabase JWT, checks `vault_members` role (`owner` or `contributor`), validates a **Cloudflare Turnstile** token on capture/AI endpoints, and enforces per-vault monthly AI quotas + a global spend ceiling tracked by summing Gemini `usageMetadata` token counts in Cloudflare KV/D1.
2. **Upload & Execution (`workflows/extractRecipe.ts`):**
   * Client uploads audio segments / OCR photos directly to Worker → stored permanently in **Cloudflare R2**. Multi-file WhatsApp voice notes are stored as separate `recipe_media` rows and passed as multiple parts in one Gemini request (no client-side codec stitching).
   * Worker creates an `ai_jobs` row (`status: 'queued'`) and triggers a **Cloudflare Workflow** instance.
   * Workflow step 1: Stream media from R2 to **Gemini Files API** (Paid Tier; `GEMINI_MODEL` env var).
   * Workflow step 2: Call `generateContent` with `responseSchema` matching `ExtractionResultSchema`.
   * Workflow step 3: Validate with Zod, write draft to `recipes.working_draft_json`, log `model_id` + `prompt_version` + `usageMetadata`, and mark `ai_jobs.status = 'succeeded'`.
   * Workflow step 4 (`finally`): Explicitly call `files.delete()` on Gemini so no user media lingers on Google’s temporary file store.
3. **Audio Segment Recovery:** If a phone call or iOS backgrounding interrupts `MediaRecorder`, resuming starts a new segment (`segment_index + 1`) with its own container header and `offset_sec`. Cook Mode plays the matching segment directly and provides ±2s manual start/end trim buttons to correct Gemini timestamp drift.

---

## 5. Phased Implementation Plan (Claude Code)

### Phase 0 — Technical Spikes & Golden Eval Set (1–2 Days)
* Verify audio playback & Gemini ingestion across Chrome (`audio/webm;codecs=opus`), iOS Safari (`audio/mp4`), and WhatsApp (`audio/ogg;codecs=opus`) through a minimal Cloudflare Worker → R2 → Gemini Files API test.
* Assemble a 5-recipe golden test set (Marathi/Hindi/Telugu/Gujarati code-switched audio + 1 handwritten diary page) to verify extraction and conversational Romanization quality before building UI.

### Phase 1 — Foundation, Auth, Vault & Offline Cache
* Scaffold Vite + React + TypeScript + Tailwind + shadcn/ui + `i18next` (EN, MR, HI, TE, GU) + Vite PWA Workbox.
* Apply WCAG 2.2 AA design tokens (`#FAF6F0`, `#1C1917`, `#B84A27`, `#9A3412`) and responsive navigation (Mobile bottom tabs: **Vault, Capture, Cook, Settings**; iPad collapsible left rail).
* Deploy Supabase migrations + RLS policies, passwordless Auth, household invite links (max 6 seats), and Dexie offline read cache.

### Phase 2 — Capture Studio & Cloudflare Workflows Pipeline
* Build Capture Studio: recording + third-party AI consent screen, Screen Wake Lock, tap-toggle microphone, segmented audio recording + offline upload outbox, multi-file voice-note import, and 1–5 page OCR photo capture.
* Implement Cloudflare Workers + Workflows + R2 pipeline (`extractRecipe.ts`, `clarify.ts`), Turnstile verification, and `ai_jobs` status polling.

### Phase 3 — Provenance Review, Allergens & 5-Language Translation
* Build iPad 55/45 split-pane (phone stacked) Review & Clarification screen: ≤ 3 prioritized questions, remaining `additional_gaps`, “No Cook Available” memorial path, 2-field `{ source, verified }` badges, explicit allergen checklist (FDA Big 9 + mustard + coconut; default `not_reviewed`), and Utensil Calibration profiles.
* Implement `translate.ts` (free-text-only translation + conversational Romanization) cached in `recipe_translations` by `(recipe_id, target_lang, source_revision_number)`.

### Phase 4 — Kitchen Cook Mode, Sharing & QA
* Build Kitchen-Friendly Cook Mode (62/38 iPad split view, Wake Lock, ≥ 56×56 px buttons, inline ingredient peek, timers strictly gated on `duration_provenance.verified === true`, step audio playback with manual timestamp trim, toggleable transcripts + speed controls).
* Implement `ShareService`: formatted WhatsApp/iMessage text and read-only web link (`/share/:share_token` rendering native Indic scripts in any browser).
* Run `vitest`, `vitest-axe` (zero critical/serious violations), Playwright phone/iPad E2E, and manual VoiceOver/TalkBack audits.

---

## 6. Claude Code Starter Prompt

```text
Read docs/SPEC.md thoroughly. We are building Unmeasured as an offline-capable PWA/SPA backed by Supabase (Auth + Postgres + RLS) and Cloudflare (Workers + Workflows + R2 + Turnstile) using Gemini Paid Tier.

Before writing any code:
1. Confirm our 5-feature MVP scope, the 2-field ProvenanceSchema ({ source, verified }), the strict separation of recipes.lifecycle_status vs. ai_jobs.status, and our explicit user-confirmed allergen rules (defaulting to not_reviewed).
2. Start strictly with Phase 0: build the minimal Worker -> R2 -> Gemini Files API spike to verify Chrome WebM, Safari MP4, and WhatsApp Ogg audio ingestion and explicit Gemini file deletion.
3. Pause after Phase 0 and report codec test results before starting Phase 1.

Rules: Never hardcode GEMINI_MODEL (use env var), never use ctx.waitUntil for long extractions (use Cloudflare Workflows), never auto-infer allergens, and never let LLMs translate or mutate structured ingredient quantities.
```