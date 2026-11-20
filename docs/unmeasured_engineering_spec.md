# Unmeasured — Engineering Specification (`docs/SPEC.md`)

**Stack:** Vite + React + TypeScript + Tailwind CSS + shadcn/ui + Vite PWA (Workbox) + `i18next` + Dexie.js (offline read cache & upload outbox) + Supabase (Auth + Postgres + RLS + Realtime) + Cloudflare (Workers + Workflows + R2 + Turnstile) + `@google/genai` (Gemini Paid Tier).

**Core Architectural Principles:**
1. **Cloud-backed durability, offline-capable kitchen:** Supabase Postgres + Cloudflare R2 (with object versioning enabled) are the canonical source of truth. Dexie.js is strictly a local offline read cache and idempotent upload outbox—cleared on sign-out to protect privacy on shared family iPads.
2. **Single authorization & quota source:** The Cloudflare Worker forwards the user's Supabase JWT so **Supabase RLS** enforces all permissions in one place (no service-role drift). Quota and spend counters live in Postgres with atomic SQL increments—no Cloudflare KV or D1. Turnstile is restricted to sign-up and invite acceptance.
3. **Durable async extraction, sync lightweight routes:** Long-running audio/OCR extraction runs in a **Cloudflare Workflow** tracked via `ai_jobs` (streamed to the client via **Supabase Realtime**). Temporary Gemini Files API uploads are explicitly deleted upon completion. Clarification and translation are synchronous Worker routes.
4. **Complete provenance with explicit memorial states:** Every ingredient (`name`, `quantity_original`, `quantity_standard`, `unit`), step instruction, duration, and audio trim carries `{ source, resolution }` where `resolution` distinguishes `pending`, `verified`, and `accepted_unknown`.
5. **Explicit allergen safety:** Zero AI auto-inference of allergens or "allergen-free" status. Unreviewed recipes explicitly display `not_reviewed`.

---

## 1. Repository Structure

```text
unmeasured/
├── docs/
│   └── SPEC.md                     # This specification
├── supabase/
│   └── migrations/                 # Postgres schema, RLS policies, atomic quota RPCs
├── workers/                        # Cloudflare Workers + Workflows (Wrangler)
│   ├── src/
│   │   ├── index.ts                # API router (Supabase JWT pass-through, Turnstile on auth/invite)
│   │   ├── workflows/
│   │   │   └── extractRecipe.ts    # Durable Cloudflare Workflow: R2 -> Gemini -> Postgres -> cleanup
│   │   ├── routes/
│   │   │   ├── upload.ts           # Idempotent R2 media upload (client-generated UUID keys)
│   │   │   ├── clarify.ts          # Synchronous targeted-patch merge for <= 3 answers
│   │   │   ├── translate.ts        # Synchronous free-text 5-language translation + Romanization
│   │   │   └── share.ts            # Public /share/:token reader with short-lived signed R2 URLs
│   │   └── lib/
│   │       ├── gemini.ts           # @google/genai client, usageMetadata atomic Postgres tracker
│   │       └── schemas.ts          # Shared Zod contracts & safe truncation helpers
├── src/
│   ├── db/
│   │   ├── cache.ts                # Dexie offline read cache, 5–10s audio chunks & upload outbox
│   │   └── sync.ts                 # Idempotent outbox flusher, "Save for Offline" prefetcher
│   ├── i18n/                       # Offline UI bundles (en, mr, hi, te, gu)
│   ├── schemas/                    # Shared Zod schemas (mirrored with workers)
│   ├── services/                   # AudioRecorder (5–10s chunks + segments), WakeLock, ShareService
│   ├── components/                 # WCAG 2.2 AA primitives, ProvenancePill, AllergenBanner
│   ├── pages/                      # Vault, Capture, Review, Detail, Cook, SharedView, Settings
│   └── styles/                     # Parchment (#FAF6F0) / Terracotta (#B84A27 / #9A3412) tokens
└── tests/                          # Vitest unit/axe tests + Playwright phone/iPad E2E
```

---

## 2. Role Permissions Matrix

| Role | Target Persona | Permissions |
| :--- | :--- | :--- |
| **`owner`** | The Archivist (Vault Creator) | Full read/write, capture, clarify, publish, manage invites (max 6 seats), configure Family Allergy Profile, export full Open JSON + Audio archive, delete vault/account. |
| **`contributor`** | Active Family Cooks / Keepers with own device | Capture voice/OCR, answer clarifications, edit drafts, publish revisions, calibrate utensils, use Cook Mode, share recipes. |
| **`cook`** | The Learner (Read-Only) | Browse vault, switch languages/scripts, save recipes for offline Cook Mode, adjust personal audio playback speed, share recipes. Cannot alter heirloom originals. |
| **Archivist-Assisted** | Elderly Keepers (No Account Needed) | Archivist records/clarifies on their own device using a dedicated **"Recording [Keeper Name]"** session—zero login or magic-link friction for grandparents. |

---

## 3. Database Schema (Supabase Postgres + RLS)

All tables use `uuid` primary keys, `timestamptz`, and RLS scoped to `vault_members` where `user_id = auth.uid()`.

### Core Tables
* **`vaults`**: `id`, `name`, `owner_id`, `max_seats` (default `6`), `ui_language` (`en|mr|hi|te|gu`), `created_at`
* **`vault_members`**: `vault_id`, `user_id`, `role` (`owner|contributor|cook`), `joined_at`
* **`vault_invites`**: `id`, `vault_id`, `token_hash`, `role`, `expires_at`, `used_by`, `used_at`
* **`vault_usage`**: `vault_id`, `billing_month` (`YYYY-MM`), `ai_extractions_count`, `input_tokens`, `output_tokens`, `estimated_usd_cents` — updated via atomic `ON CONFLICT ... DO UPDATE` RPC; enforces per-vault quotas and global spend ceiling.
* **`vault_allergies`**: `id`, `vault_id`, `allergens` (`text[]` — FDA Big 9 + `mustard`, `coconut`), `updated_at`
* **`measurement_profiles`**: `id`, `vault_id`, `display_name`, `capacity_ml`, `staple_grams_note`, `photo_r2_key`, `verified_by`, `updated_at`
* **`recipes`**:
  * `id` (client-generated UUID for offline idempotency), `vault_id`, `title`, `cook_name`, `recorded_by`, `original_languages` (`text[]`), `capture_mode` (`voice|ocr|voice_plus_ocr|import`)
  * `lifecycle_status` (`draft|published|archived`)
  * `allergen_review_status` (`not_reviewed|reviewed|needs_recheck`), `confirmed_allergens` (`text[]`), `dietary_tags` (`text[]`)
  * `prep_ahead` (`text[]`), `equipment` (`text[]`), `servings_display` (`text`)
  * `current_revision` (`int`), `draft_version` (`int`, default `1` — **optimistic concurrency / compare-and-swap** for `working_draft_json`)
  * `working_draft_json` (`jsonb`, nullable)
  * `share_token` (`uuid`, nullable — revocable/regenerable), `share_settings_json` (`jsonb`: `{ include_story, include_allergens, include_audio }`)
  * `updated_at`, `created_at`
* **`recipe_revisions`**: `id`, `recipe_id`, `revision_number`, `snapshot_json` (`jsonb`), `original_romanized_json` (`jsonb`, nullable — enables Native/Romanized toggle on the Original view without redundant `mr->mr` translation), `model_id`, `prompt_version`, `schema_version`, `created_by`, `created_at`
* **`recipe_media`**: `id` (client-generated UUID idempotency key), `vault_id`, `recipe_id`, `media_type` (`audio_segment|audio_clarification|photo_cooking|scan_ocr_page`), `r2_key`, `mime_type`, `segment_index`, `offset_sec`, `duration_sec`, `created_at`
* **`ai_jobs`**: `id`, `vault_id`, `recipe_id`, `job_type` (`extract`), `status` (`queued|uploading|processing|retrying|succeeded|failed`), `attempt_count`, `workflow_instance_id`, `error_message`, `token_usage_json`, `created_at`, `updated_at` *(Supabase Realtime enabled)*
* **`recipe_translations`**: `id`, `recipe_id`, `target_lang` (`en|mr|hi|te|gu`), `source_revision_number`, `free_text_payload` (`jsonb`), `romanized_payload` (`jsonb`, nullable), `model_id`, `created_at` — Unique index on `(recipe_id, target_lang, source_revision_number)`.

### Dexie Offline Cache (`src/db/cache.ts`)
* `cached_recipes`: mirrors published `recipes` + `recipe_translations` for offline Cook Mode.
* `media_cache`: stores audio/photos prefetched via an explicit **"Save for Offline"** button on Recipe Detail (so unplayed Cook Mode steps work offline).
* `recording_chunks`: persists **5–10 second audio chunks** locally within each recording segment to survive browser crashes or iOS tab kills.
* `upload_outbox`: queues offline recipes, media segments, and clarification answers using client-generated UUIDs as idempotency keys.
* **Shared iPad Privacy Rule:** Signing out immediately wipes all Dexie tables and Cache Storage entries for the previous user.

---

## 4. Exact TypeScript & Zod Contracts (`src/schemas/recipe.ts`)

```ts
import { z } from 'zod';

export const ProvenanceSchema = z.object({
  source: z.enum(['spoken', 'ocr', 'clarified', 'manual', 'inferred']),
  resolution: z.enum(['pending', 'verified', 'accepted_unknown']),
});

// Stored / App Ingredient Schema (real UUIDs mapped server-side from Gemini media_index)
export const IngredientSchema = z.object({
  id: z.string(),
  name_original: z.string(),
  name_display: z.string(),
  name_provenance: ProvenanceSchema,
  quantity_original: z.string().nullable(),
  quantity_original_provenance: ProvenanceSchema,
  unit_standard: z.string().nullable(),
  unit_provenance: ProvenanceSchema,
  quantity_standard_value: z.number().nullable(),
  quantity_standard_provenance: ProvenanceSchema, // Always source: 'inferred' unless manually calibrated
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
  duration_provenance: ProvenanceSchema.nullable(), // Timers enabled ONLY when resolution === 'verified'
  sensory_cues: z.array(SensoryCueSchema),
  audio_media_id: z.string().uuid().nullable(),
  audio_start_sec: z.number().nonnegative().nullable(),
  audio_end_sec: z.number().nonnegative().nullable(),
  audio_trim_provenance: ProvenanceSchema.nullable(), // Set to { source: 'manual', resolution: 'verified' } on +/-2s trim
  photo_media_ids: z.array(z.string().uuid()),
});

export const ClarificationQuestionSchema = z.object({
  id: z.string(),
  priority: z.number().int().min(1).max(3),
  target_path: z.string(),
  gap_type: z.enum(['missing_sensory_cue', 'uncalibrated_utensil', 'ambiguous_timing_or_heat']),
  question: z.string(),
  suggested_options: z.array(z.string()), // Truncated to max 4 server-side if Gemini exceeds limit
  resolved: z.boolean(),
  answer_text: z.string().nullable(),
  answer_audio_media_id: z.string().uuid().nullable(),
});

// Gemini Extraction Output Contract (uses 0-based media_index, never invented UUIDs)
export const GeminiExtractionOutputSchema = z.object({
  schema_version: z.literal(1),
  title: z.string(),
  original_languages: z.array(z.string()),
  story_summary: z.string().nullable(),
  servings_display: z.string().nullable(),
  prep_ahead: z.array(z.string()),
  equipment: z.array(z.string()),
  ingredients: z.array(IngredientSchema.omit({ source_media_id: true }).extend({
    source_media_index: z.number().int().nonnegative().nullable(),
  })),
  steps: z.array(StepSchema.omit({ audio_media_id: true, photo_media_ids: true, audio_trim_provenance: true }).extend({
    audio_media_index: z.number().int().nonnegative().nullable(),
    photo_media_indices: z.array(z.number().int().nonnegative()),
  })),
  clarifying_questions: z.array(ClarificationQuestionSchema), // Server truncates to top 3 by priority; moves overflow to additional_gaps
  additional_gaps: z.array(z.string()),
});

// Synchronous Clarification Patch Request (optimistic lock via expected_draft_version)
export const ClarifyRequestSchema = z.object({
  recipe_id: z.string().uuid(),
  expected_draft_version: z.number().int().positive(),
  answers: z.array(z.object({
    idempotency_key: z.string().uuid(),
    question_id: z.string(),
    answer_text: z.string().nullable(),
    answer_audio_media_id: z.string().uuid().nullable(),
    cook_unavailable: z.boolean().default(false), // Sets target resolution to 'accepted_unknown'
  })),
});

// Synchronous Free-Text-Only Translation Payload (never mutates numeric quantities/units)
export const TranslationPayloadSchema = z.object({
  title: z.string(),
  story_summary: z.string().nullable(),
  prep_ahead: z.array(z.string()),
  equipment: z.array(z.string()),
  ingredients: z.array(z.object({
    id: z.string(),
    name_translated: z.string(),
    quantity_original_display: z.string().nullable(), // Display-only translation of phrases like "दोन चमचे"
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

## 5. Worker Pipeline, Public Sharing & Open Export

1. **Durable Extraction (`workflows/extractRecipe.ts`):**
   * Client uploads 5–10s-backed audio segments and OCR photos to `routes/upload.ts` using client-generated UUIDs (`ON CONFLICT (id) DO NOTHING`). Multi-file WhatsApp voice notes are stored as separate `recipe_media` rows and passed as multiple parts in one Gemini request.
   * Workflow streams media from R2 to Gemini Files API (Paid Tier; `GEMINI_MODEL` env var), runs extraction, maps `media_index` → real `recipe_media.id` UUIDs, truncates `clarifying_questions` to top 3 (moving any overflow to `additional_gaps`), writes `working_draft_json` with `draft_version = 1`, increments `vault_usage` atomically in Postgres, and updates `ai_jobs.status = 'succeeded'`.
   * In a `finally` block, the Workflow calls `files.delete()` on Gemini so no user media lingers on Google's temporary store.
2. **Targeted Patch Clarifications (`routes/clarify.ts`):** Synchronous route that verifies `expected_draft_version` (returning `409 Conflict` if a relative edited the draft concurrently) and applies targeted field patches rather than overwriting unrelated edits.
3. **Public Share Route (`routes/share.ts`):** Serves `/share/:share_token` without requiring login, respects `share_settings_json` (`include_story`, `include_allergens`, `include_audio`), mints short-lived (15-minute) signed R2 URLs for audio/photos if enabled, and allows the Owner to revoke or regenerate `share_token` at any time.
4. **Privacy, Durability & Open Export:**
   * **Encryption & Retention:** Encryption at rest via Supabase Postgres and Cloudflare R2 (with R2 object versioning enabled). Vault or account deletion cascades across Postgres and deletes all R2 objects, satisfying **Apple 5.1.1(v)** in-app account deletion.
   * **Documented Open Export Format:** Owners can download a standard `.zip` archive containing `vault.json` (documented open schema), `recipes/<id>.json`, and original `media/<id>.<ext>` audio/photo files—ensuring family heritage outlives any single vendor.

---

## 6. Phased Implementation Plan & Definition of Done

### Phase 0 — Technical Spikes & Golden Eval Set (3–5 Days)
* **Deliverables:** Minimal Worker → R2 → Gemini Files API spike testing Chrome (`audio/webm;codecs=opus`), iOS Safari (`audio/mp4`), and WhatsApp (`audio/ogg;codecs=opus`) playback + ingestion + explicit `files.delete()`. Assemble a 10-recipe golden test set across English, Marathi, Hindi, Telugu, and Gujarati (including handwritten diary scans).
* **Definition of Done (Pass/Fail Gate):** ≥ 90% ingredient recall and ≥ 85% step/sensory-cue fidelity across the golden set; all 3 audio codecs play cleanly on iOS Safari & Chrome; zero orphaned Gemini files.

### Phase 1 — Foundation, Auth, Vault & Offline Cache
* **Deliverables:** Vite + React + TypeScript + Tailwind + shadcn/ui + `i18next` + Vite PWA; WCAG 2.2 AA tokens; Supabase migrations + RLS + atomic `vault_usage` RPC; passwordless Auth + Turnstile on sign-up/invites; Role Permissions Matrix; Dexie offline cache with sign-out wipe.
* **Definition of Done:** All 3 roles (`owner`, `contributor`, `cook`) verified by RLS unit tests; sign-out clears 100% of Dexie/Cache Storage on shared tablets; `vitest-axe` reports 0 critical/serious violations.

### Phase 2 — Capture Studio & Durable Cloudflare Workflow
* **Deliverables:** Consent screen (recording + Apple 5.1.1/5.1.2 third-party AI disclosure); Capture Studio with Wake Lock, 5–10s chunk crash recovery, Archivist-Assisted Keeper mode, multi-file voice-note import, 1–5 page OCR scan, idempotent offline outbox, and Cloudflare Workflow + Supabase Realtime `ai_jobs` status.
* **Definition of Done:** Killing the browser tab mid-recording recovers all prior 5–10s chunks; offline capture syncs once without duplicate `recipe_media` rows; `ai_jobs` updates stream in real time.

### Phase 3 — Provenance Review, Allergens & 5-Language Switching
* **Deliverables:** iPad 55/45 split-pane Review & Clarification (`draft_version` compare-and-swap, ≤ 3 questions + `additional_gaps`, “No Cook Available” `accepted_unknown` path, full provenance badges, explicit allergen checklist defaulting to `not_reviewed`, Utensil Calibration); synchronous `translate.ts` + Original Romanization.
* **Definition of Done:** Concurrent draft edit returns `409 Conflict` without data loss; translation never alters numeric quantities; editing an ingredient resets `allergen_review_status` to `needs_recheck`.

### Phase 4 — Kitchen Cook Mode, Sharing, Open Export & Pilot Gate
* **Deliverables:** Kitchen Cook Mode (62/38 iPad split, Wake Lock, explicit "Save for Offline" prefetch, inline ingredient peek, `verified`-only timers, ±2s manual audio trim); WhatsApp text + revocable `/share/:token` web link with privacy toggles; Open JSON + Audio Export; full Playwright + VoiceOver/TalkBack audit.
* **Definition of Done:** Prefetched recipe plays all audio/photos in Airplane Mode; revoked share token returns `404`; pilot passes exit metrics (≥ 80% drafts accepted, < 5 min editing, ≥ 70% Learner first-try cooking success, and ≥ 50% active use of sensory cues or script toggling).

---

## 7. Claude Code Starter Prompt

```text
Read docs/SPEC.md thoroughly. We are building Unmeasured as an offline-capable PWA/SPA backed by Supabase (Auth + Postgres + RLS + Realtime) and Cloudflare (Workers + Workflows + R2 + Turnstile) using Gemini Paid Tier.

Before writing any code:
1. Confirm our Role Permissions Matrix (owner, contributor, cook, plus Archivist-Assisted Keeper capture), our ProvenanceSchema ({ source, resolution }), our draft_version compare-and-swap locking, and our client-UUID idempotency rules.
2. Start strictly with Phase 0: build the minimal Worker -> R2 -> Gemini Files API spike verifying Chrome WebM, Safari MP4, and WhatsApp Ogg ingestion, media_index mapping, and explicit Gemini files.delete().
3. Pause after Phase 0 and report codec and golden-set accuracy results before starting Phase 1.

Rules: Never hardcode GEMINI_MODEL (use env var), never use Cloudflare KV/D1 (use Postgres atomic RPCs), forward the user's Supabase JWT so RLS is the single auth source, never auto-infer allergens, and never let LLMs mutate numeric quantities during translation.
```