# Unmeasured — Engineering Specification (`docs/SPEC.md`)

**Stack:** Vite + React + TypeScript + Tailwind CSS + shadcn/ui + Vite PWA (Workbox) + `i18next` + Dexie.js (offline read cache & upload outbox) + `fflate` / `client-zip` (client-side streaming export) + Supabase (Auth + Postgres + RLS + Realtime) + Cloudflare (Workers + Workflows + R2 + Cron Triggers + Turnstile) + `@google/genai` (Gemini Paid Tier).

**Core Architectural Principles:**
1. **Cloud-backed durability, offline-capable kitchen:** Supabase Postgres + Cloudflare R2 are the canonical source of truth. Dexie.js is strictly a local offline read cache (keyed by `media_id`, never expiring URLs) and idempotent upload outbox. Sign-out wipes local caches on shared iPads—but is blocked/warned if `upload_outbox` or `recording_chunks` has unsynced items.
2. **Hybrid Auth (RLS + Named `SECURITY DEFINER` RPCs + Worker R2 Auth):** Direct client reads/writes are governed by Supabase RLS. Background Workflows (where user JWTs expire), public `/share/:token` reads, and atomic quota reserve/settle operations run through named Postgres `SECURITY DEFINER` RPCs. Users have zero direct write access to `vault_usage`. Cloudflare Worker code enforces vault membership for all R2 uploads and signed-in media fetches.
3. **Reserve-then-settle AI quotas:** Every Gemini call (`extract`, `clarify`, `translate`, and publish-time `romanize`) atomically reserves quota in Postgres *before* calling Gemini and settles with actual `usageMetadata` on both success and failure.
4. **Server-enforced provenance:** The LLM output schema omits `resolution`, `id`, `schema_version`, and app state. The server forces `resolution: 'pending'` on all AI-generated fields so the model can never self-verify durations or measurements.
5. **Explicit allergen safety:** Zero AI auto-inference of allergens or "allergen-free" status. Allergen state lives per revision snapshot (and separately on drafts) so editing a draft never alters the published recipe banner.

---

## 1. Repository Structure

```text
unmeasured/
├── docs/
│   ├── SPEC.md                     # Single source of truth for engineering
│   └── Unmeasured_PRD.md           # Product requirements & validation plan
├── .env.example                    # Required env vars & secrets (see §7)
├── supabase/
│   └── migrations/                 # Postgres schema, RLS policies, 8 SECURITY DEFINER RPCs
├── workers/                        # Cloudflare Workers + Workflows + Cron (Wrangler)
│   ├── src/
│   │   ├── index.ts                # Router, JWKS JWT verification, standard error envelope
│   │   ├── workflows/
│   │   │   └── extractRecipe.ts    # Durable Workflow: R2 -> Gemini -> Postgres RPC -> cleanup
│   │   ├── cron/
│   │   │   └── sweepGeminiFiles.ts # Hourly sweeper deleting orphaned Gemini files > 1h old
│   │   ├── routes/
│   │   │   ├── upload.ts           # Idempotent R2 media upload (Worker membership check)
│   │   │   ├── media.ts            # Signed-in GET /api/media/:media_id (Worker membership check)
│   │   │   ├── clarify.ts          # Sync field-level (target_path) patch merge + reserve/settle
│   │   │   ├── translate.ts        # Sync free-text 5-lang translation + numeric guard
│   │   │   └── share.ts            # Public /share/:token reader via SECURITY DEFINER RPC
│   │   └── lib/
│   │       ├── auth.ts             # Supabase JWKS JWT verifier
│   │       ├── gemini.ts           # @google/genai client + reserve/settle quota wrapper
│   │       └── schemas.ts          # Lean Gemini output schema + server truncation helpers
├── src/
│   ├── db/
│   │   ├── cache.ts                # Dexie offline cache (keyed by media_id), 5-10s chunks, outbox
│   │   └── sync.ts                 # Idempotent outbox flusher, "Save for Offline" prefetcher
│   ├── i18n/
│   │   ├── locales/                # Offline UI bundles (en, mr, hi, te, gu)
│   │   └── glossary/               # Regional culinary glossary (e.g., haldi/halad, hing, tadka)
│   ├── schemas/                    # Shared app Zod schemas
│   ├── services/                   # AudioRecorder (5-10s chunks + segments), WakeLock, ExportZip
│   ├── components/                 # WCAG 2.2 AA primitives, ProvenancePill, AllergenBanner, DiffModal
│   ├── pages/                      # Vault, Capture, Review, Detail, Cook, SharedView, Settings
│   └── styles/                     # Parchment (#FAF6F0) / Terracotta (#B84A27 / #9A3412) tokens
└── tests/                          # Vitest unit/axe tests + Playwright phone/iPad E2E
```

---

## 2. Role Permissions Matrix

| Role | Target Persona | Permissions |
| :--- | :--- | :--- |
| **`owner`** | The Archivist (Vault Creator) | Full read/write, capture, clarify, edit, publish, calibrate utensils, create/revoke public share links, manage invites (max 6 seats), promote `contributor` to `owner`, set Family Allergy Profile, run client-side Open JSON + Audio export, delete vault (14-day grace period) or transfer ownership before account deletion. |
| **`contributor`** | Active Family Cooks / Keepers | Capture voice/OCR, answer clarifications, edit drafts (with monotonic `draft_version` locking), publish revisions, calibrate utensils, create/revoke public share links, use Cook Mode. |
| **`learner`** | Read-Only Family Member | Browse vault, switch languages/scripts (permitted to trigger `translate` via `SECURITY DEFINER` RPC), prefetch recipes for offline Cook Mode, adjust personal audio speed (`localStorage: unmeasured:audio_speed`), copy WhatsApp text. Cannot alter heirloom originals or create/revoke public share links. |
| **Archivist-Assisted** | Elderly Keepers (No Account) | Archivist records/clarifies on their own device under a named Keeper session; consent is logged in `consents` with zero login/magic-link friction. |

---

## 3. Screen Architecture, Responsive Splits & WCAG 2.2 AA

### Visual Tokens (Verified WCAG 2.2 AA Contrast)
* **Background:** Warm parchment `#FAF6F0`; cards `#FFFDF9`; borders `#E6DDD0`.
* **Text:** Charcoal `#1C1917` on parchment (`16.2:1`); warm slate `#57534E` for metadata.
* **Accents:** Primary terracotta `#B84A27` with white text (`5.2:1`); dark terracotta `#9A3412` for small text/badges/links on parchment (`6.8:1`) and white text on `#9A3412` (`7.3:1`).
* **Non-Color Indicators:** Never rely on color alone for provenance or allergen banners—always pair icons with explicit text labels.

### Route & Responsive Layout Table
| Route | Screen | Mobile & iPad Landscape Behavior |
| :--- | :--- | :--- |
| `/vault` | Family Vault | Primary tab. Search by title/cook/tags, filter by dietary/allergen status, view Realtime `ai_jobs` progress, manage household invites (max 6 seats). |
| `/capture` | Capture Studio | Primary tab. Voice Narration (Wake Lock + ≥ 56×56 px tap-toggle controls), Archivist-Assisted Keeper mode, Multi-File Voice Note Import, and 1–5 Page OCR Scan. |
| `/recipes/:id/review` | Review & Clarify | **iPad 55/45 split view** (stacked on phone). Left: structured draft, provenance badges, explicit allergen checklist + reminder copy (*"Double-check compounded hing (wheat), ghee (milk), and nut pastes"*). Right: ≤ 3 clarifications, `additional_gaps`, source audio/OCR viewer, and 409 Diff Modal if a concurrent edit occurs. |
| `/recipes/:id` | Recipe Detail | **iPad 40/60 split view**. Published revision allergen banner, 5-language/script switcher, calibrated utensil notes, prep-ahead/equipment callouts, "Save for Offline" prefetch button, Share sheet, and Start Cook Mode CTA. |
| `/recipes/:id/cook` | Kitchen Cook Mode | Primary tab (last active recipe). Distraction-free countertop view (**iPad 62/38 split**, left nav rail hidden), Wake Lock, inline ingredient peek, `verified`-only timers, step audio playback with ±2s manual trim, and sensory cue card. |
| `/share/:token` | Public Shared View | Read-only web page served via Worker + `read_public_share` RPC; respects `include_story`, `include_allergens`, and `include_audio` (defaults `false`). |
| `/settings` | Vault & Household | Primary tab. Household invites/roles/ownership transfer, Family Allergy Profile, Utensil Calibration, Privacy/Consent log, Client-Side Open Export, and Account/Vault Deletion. |

### Mandatory WCAG 2.2 AA Guardrails
* **Touch Targets:** Minimum `56×56 px` on phones and `64×64 px` on tablets for all Capture and Cook Mode controls (`≥ 8 px` spacing).
* **Input Ergonomics:** Single-tap toggle microphones (never hold-to-speak only); `prefers-reduced-motion` support; 200% Dynamic Type reflow without horizontal scroll.
* **Screen Readers & Media:** Dynamic `lang` attributes (`en`, `mr`, `hi`, `te`, `gu`) on localized blocks for VoiceOver/TalkBack pronunciation; toggleable audio transcripts with personal playback speed controls (`0.75×`, `1×`, `1.25×` stored in `localStorage`).
* **Anti-Spam Rule:** **Never** fire `aria-live` announcements on routine background 5–10s audio chunk saves.

---

## 4. Database Schema (Supabase Postgres + RLS + `SECURITY DEFINER` RPCs)

All tables use `uuid` primary keys and `timestamptz`. Direct client access is restricted by RLS to `vault_members` where `user_id = auth.uid()`.

### Core Tables
* **`vaults`**: `id`, `name`, `owner_id`, `max_seats` (default `6`), `ui_language` (`en|mr|hi|te|gu`), `deletion_scheduled_at` (`timestamptz`, nullable — 14-day grace period), `created_at`
* **`vault_members`**: `vault_id`, `user_id`, `role` (`owner|contributor|learner`), `joined_at`
* **`vault_invites`**: `id`, `vault_id`, `token_hash`, `role`, `expires_at`, `used_by`, `used_at`
* **`vault_usage`**: `vault_id`, `billing_month` (`YYYY-MM`), `ai_calls_count`, `reserved_cents`, `settled_cents`, `input_tokens`, `output_tokens` — **zero direct user write access**; mutated exclusively by `SECURITY DEFINER` RPCs.
* **`vault_allergies`**: `id`, `vault_id`, `allergens` (`text[]` — US FDA Big 9 + `mustard`, `coconut`), `updated_at`
* **`measurement_profiles`**: `id`, `vault_id`, `display_name`, `capacity_ml`, `staple_grams_note`, `photo_r2_key`, `verified_by`, `updated_at`
* **`consents`**: `id`, `vault_id`, `recipe_id`, `person_recorded`, `recorded_by`, `consent_text_version`, `consent_audio_media_id` (`uuid`, nullable), `recorded_at` — provides auditable verbal/AI consent for Archivist-Assisted Keepers; includes a documented support/owner deletion request workflow for Keepers without accounts.
* **`recipes`**:
  * `id` (client-generated UUID), `vault_id`, `title`, `cook_name`, `recorded_by`, `original_languages` (`text[]`), `capture_mode` (`voice|ocr|voice_plus_ocr|import`)
  * `lifecycle_status` (`draft|published|archived`)
  * Denormalized published columns (updated ONLY by `publish_recipe` RPC): `prep_ahead` (`text[]`), `equipment` (`text[]`), `servings_display` (`text`), `dietary_tags` (`text[]`)
  * `current_revision` (`int`, default `0`)
  * `draft_version` (`int`, default `1` — **strictly monotonic**, never resets on re-extraction)
  * `working_draft_json` (`jsonb`, nullable — holds draft content + draft-scoped `allergen_review_status` & `confirmed_allergens`)
  * `share_token` (`uuid`, nullable), `share_settings_json` (`jsonb`: `{ include_story: false, include_allergens: true, include_audio: false }`)
  * `updated_at`, `created_at`
* **`recipe_revisions`**: `id`, `recipe_id`, `revision_number`, `snapshot_json` (`jsonb` — includes revision-scoped `allergen_review_status` & `confirmed_allergens`), `original_romanized_json` (`jsonb`, nullable — generated at publish time via reserve/settle quota), `model_id`, `prompt_version`, `schema_version`, `created_by`, `created_at`
* **`recipe_media`**: `id` (client-generated UUID), `vault_id`, `recipe_id`, `media_type` (`audio_segment|audio_clarification|photo_cooking|scan_ocr_page`), `r2_key`, `mime_type`, `segment_index`, `offset_sec`, `duration_sec`, `created_at`
* **`ai_jobs`**: `id`, `vault_id`, `recipe_id`, `job_type` (`extract`), `status` (`queued|uploading|processing|retrying|succeeded|failed`), `attempt_count`, `workflow_instance_id`, `error_message`, `token_usage_json`, `created_at`, `updated_at` *(Supabase Realtime enabled)*
* **`recipe_translations`**: `id`, `recipe_id`, `target_lang` (`en|mr|hi|te|gu`), `source_revision_number`, `free_text_payload` (`jsonb`), `romanized_payload` (`jsonb`, nullable), `model_id`, `created_at` — Unique index on `(recipe_id, target_lang, source_revision_number)`.

### Named `SECURITY DEFINER` RPCs
1. `reserve_ai_quota(vault_id, estimated_cents)` / `settle_ai_quota(vault_id, reservation_id, actual_tokens, actual_cents)`: Atomic Postgres reserve-then-settle enforcing per-vault monthly limits and global spend ceiling across all vaults.
2. `workflow_commit_extraction(job_id, recipe_id, draft_json, model_id, prompt_version)`: Allows expired-JWT Workflows to write `working_draft_json` (bumping `draft_version = draft_version + 1`) and update `ai_jobs`.
3. `save_draft(recipe_id, expected_draft_version, next_draft_json)`: Atomic compare-and-swap for Review edits; returns `409 Conflict` with current server draft if `draft_version` changed so the UI can open the Diff Modal without losing local edits.
4. `patch_draft_field(recipe_id, target_path, expected_field_hash, patch_value)`: Field-level conflict check for queued offline clarifications so unrelated edits do not stall offline answers.
5. `publish_recipe(recipe_id, expected_draft_version)`: Single atomic transaction that copies `working_draft_json` (including its allergen state) into a new `recipe_revisions` row, bumps `current_revision`, refreshes denormalized `recipes` columns, and clears `working_draft_json`.
6. `manage_share_link(recipe_id, action, settings_json)`: Restricts link creation/revocation to `owner | contributor` (blocking `learner` even at column level).
7. `read_public_share(share_token)`: Returns sanitized published revision fields respecting `share_settings_json` for anonymous `/share/:token` visitors.
8. `transfer_vault_ownership(vault_id, new_owner_id)` / `request_account_deletion()`: Prevents an `owner` from deleting their account while owning an active multi-member vault until ownership is transferred or vault deletion (14-day grace) is explicitly confirmed.

---

## 5. Exact TypeScript & Zod Contracts (`src/schemas/recipe.ts`)

```ts
import { z } from 'zod';

// App-side provenance (server ALWAYS forces resolution: 'pending' on AI output)
export const ProvenanceSchema = z.object({
  source: z.enum(['spoken', 'ocr', 'clarified', 'manual', 'inferred', 'calibrated']),
  resolution: z.enum(['pending', 'verified', 'accepted_unknown']),
});

// 1. LEAN GEMINI OUTPUT SCHEMA (No resolution, no UUIDs, no app state)
const ModelSourceEnum = z.enum(['spoken', 'ocr', 'inferred']);

export const GeminiExtractionOutputSchema = z.object({
  title: z.string(),
  original_languages: z.array(z.string()), // Index 0 is primary display language
  story_summary: z.string().nullable(),
  servings_display: z.string().nullable(),
  prep_ahead: z.array(z.string()),
  equipment: z.array(z.string()),
  ingredients: z.array(z.object({
    name_original: z.string(),
    name_display: z.string(), // Always in original_languages[0]
    name_source: ModelSourceEnum,
    quantity_original: z.string().nullable(),
    quantity_original_source: ModelSourceEnum,
    unit_standard: z.string().nullable(),
    unit_source: ModelSourceEnum,
    quantity_standard_value: z.number().nullable(), // Server sets source: 'inferred', resolution: 'pending'
    source_media_index: z.number().int().nonnegative().nullable(),
    notes: z.string().nullable(),
  })),
  steps: z.array(z.object({
    step_number: z.number().int().positive(),
    instruction_original: z.string(),
    instruction_display: z.string(), // Always in original_languages[0]
    instruction_source: ModelSourceEnum,
    duration_sec: z.number().int().positive().nullable(),
    duration_source: ModelSourceEnum.nullable(),
    sensory_cues: z.array(z.object({
      type: z.enum(['visual', 'aroma', 'sound', 'texture', 'taste']),
      text: z.string(),
      source: ModelSourceEnum,
    })),
    audio_media_index: z.number().int().nonnegative().nullable(),
    audio_start_sec: z.number().nonnegative().nullable(),
    audio_end_sec: z.number().nonnegative().nullable(),
    photo_media_indices: z.array(z.number().int().nonnegative()),
  })),
  clarifying_questions: z.array(z.object({
    priority: z.number().int().min(1).max(3),
    target_path: z.string(),
    gap_type: z.enum(['missing_sensory_cue', 'uncalibrated_utensil', 'ambiguous_timing_or_heat']),
    question: z.string(),
    suggested_options: z.array(z.string()), // Server truncates to max 4
  })), // Server truncates to top 3 by priority; overflow moved to additional_gaps
  additional_gaps: z.array(z.string()),
});

// 2. TRANSLATION PAYLOAD (ID-matched cues, numeric-preservation checked in code)
export const TranslationPayloadSchema = z.object({
  title: z.string(),
  story_summary: z.string().nullable(),
  prep_ahead: z.array(z.string()),
  equipment: z.array(z.string()),
  ingredients: z.array(z.object({
    id: z.string(),
    name_translated: z.string(),
    quantity_original_display: z.string().nullable(), // Code verifies numeric tokens match source; flags mismatch
    notes_translated: z.string().nullable(),
  })),
  steps: z.array(z.object({
    id: z.string(),
    instruction_translated: z.string(),
    sensory_cues_translated: z.array(z.object({
      id: z.string(), // Matched by ID, never array position
      text: z.string(),
    })),
  })),
});
```

---

## 6. Worker Pipeline, Media, Offline & Open Export

1. **Durable Extraction & Orphan Sweeper:**
   * `workflows/extractRecipe.ts` reserves quota via `reserve_ai_quota`, uploads R2 media to Gemini Files API, validates against `GeminiExtractionOutputSchema`, assigns server UUIDs (`id`, `sensory_cues[].id`), maps `media_index` → `recipe_media.id`, forces `resolution: 'pending'`, commits via `workflow_commit_extraction`, and settles actual `usageMetadata`.
   * On both success and failure, the Workflow calls `files.delete()` on Gemini. A scheduled hourly Cron (`cron/sweepGeminiFiles.ts`) lists and deletes any orphaned Gemini files older than 1 hour.
2. **Signed-In Media & ID-Keyed Offline Cache:**
   * Signed-in clients fetch media through `GET /api/media/:media_id` (Worker verifies JWT + `vault_members` membership, streams from R2).
   * Clicking **"Save for Offline"** prefetches all recipe audio/photos and stores the blobs in Dexie `media_cache` keyed by `media_id` (not expiring signed URLs), ensuring unplayed Cook Mode steps work in Airplane Mode.
   * **Safe Sign-Out:** Before wiping Dexie on sign-out, check `upload_outbox` and `recording_chunks`. If non-empty, block sign-out and prompt the user to upload first or explicitly discard unsynced recordings.
3. **Publish-Time Original Romanization:** When `publish_recipe` runs on a non-English recipe, the Worker triggers a lightweight Romanization pass (billed via reserve/settle) and stores `original_romanized_json` on `recipe_revisions`.
4. **Client-Side Streaming Open Export:** To avoid the 128 MB Cloudflare Worker memory limit, full vault exports (`vault.json`, `recipes/<id>.json`, `media/<id>.<ext>`) are streamed and zipped **on the client** using `fflate` / `client-zip` from authenticated media fetches.
5. **Backups & Apple 5.1.1(v) Deletion:** Cloud data is encrypted at rest in Supabase Postgres (enable Supabase PITR paid add-on before public launch) and Cloudflare R2 (scheduled daily bucket replication/backup rather than relying on S3-style versioning). Permanent vault/account deletion purges Postgres rows and all R2 objects completely.

---

## 7. Configuration, Quotas & Error Envelope

### Required Environment Variables (`.env.example`)
* **Frontend (Vite):** `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `VITE_WORKER_API_URL`, `VITE_TURNSTILE_SITE_KEY`
* **Cloudflare Worker:** `SUPABASE_URL`, `SUPABASE_JWKS_URL`, `SUPABASE_SERVICE_ROLE_KEY` (used only to invoke `SECURITY DEFINER` RPCs from Workflows/Cron), `GEMINI_API_KEY`, `GEMINI_MODEL`, `TURNSTILE_SECRET_KEY`, `R2_BUCKET`
* **JWT Verification:** Worker verifies incoming `Authorization: Bearer <jwt>` against `SUPABASE_JWKS_URL` (cached JWKS).

### Quotas & Spend Ceiling (MVP Defaults)
* **Per-Vault Monthly Quota:** 30 AI extractions, 100 clarifications/translations (~$3.00/mo ceiling per vault).
* **Global Monthly Spend Ceiling:** `$150.00` (15,000 cents) hard cap enforced atomically in `reserve_ai_quota`.

### Standard Error Response Envelope
```json
{
  "error": {
    "code": "CONFLICT_DRAFT_VERSION | QUOTA_EXCEEDED | UNAUTHORIZED | VALIDATION_FAILED",
    "message": "Human-readable explanation",
    "details": {}
  }
}
```

---

## 8. Phased Implementation Plan & Definition of Done

### Phase 0 — Technical Spikes & Golden Regression Gate (3–5 Days)
* **Deliverables:** Minimal Worker → R2 → Gemini Files API spike testing Chrome (`audio/webm;codecs=opus`), iOS Safari (`audio/mp4`), and WhatsApp (`audio/ogg;codecs=opus`) ingestion, playback, **crash-recovered chunk reassembly** (including Safari MP4 after tab kill), JS WebM→Ogg remux fallback if Gemini rejects WebM, and explicit `files.delete()` + Cron sweeper. Build a 10-recipe labeled golden test set (2 per language, with ground-truth ingredient/cue JSON) that serves as a **mandatory regression gate** whenever `GEMINI_MODEL` or `prompt_version` changes.
* **Definition of Done:** ≥ 90% ingredient recall and ≥ 85% sensory-cue fidelity on the golden set; crash-recovered chunks play cleanly on Safari & Chrome; zero orphaned Gemini files.

### Phase 1 — Foundation, Auth, Vault & Offline Cache
* **Deliverables:** Vite/React/TS/Tailwind/shadcn/`i18next` (with `i18n/glossary/`)/PWA; WCAG 2.2 AA tokens; Supabase migrations + RLS + all 8 `SECURITY DEFINER` RPCs; JWKS verification; Turnstile on sign-up/invites; Role Permissions Matrix (`owner`, `contributor`, `learner`); Dexie cache with safe sign-out guard.
* **Definition of Done:** RLS & RPC unit tests pass for all 3 roles; users cannot write `vault_usage`; sign-out blocks when outbox is non-empty and wipes cache when empty; 0 critical/serious `vitest-axe` violations.

### Phase 2 — Capture Studio & Durable Cloudflare Workflow
* **Deliverables:** Consent screen (`consents` table + Apple 5.1.1/5.1.2 third-party AI disclosure); Capture Studio (Wake Lock, 5–10s chunk crash recovery, Archivist-Assisted Keeper mode, multi-file voice-note import, 1–5 page OCR scan, idempotent outbox); Cloudflare Workflow + reserve/settle quotas + Supabase Realtime `ai_jobs`.
* **Definition of Done:** Killing tab mid-recording recovers all prior chunks; offline retries create zero duplicates; failed Gemini calls still settle token spend; server forces `resolution: 'pending'` on all AI output.

### Phase 3 — Provenance Review, Allergens & 5-Language Switching
* **Deliverables:** iPad 55/45 split-pane Review (`save_draft` RPC + 409 Diff Modal, field-level `patch_draft_field` clarify, memorial `accepted_unknown`, full provenance including `calibrated`, per-revision allergen checklist + hing/ghee/nut reminder, `publish_recipe` atomic RPC + publish-time Romanization); sync `translate.ts` (ID-matched cues + numeric-preservation guard, accessible to `learner`).
* **Definition of Done:** Concurrent draft edit opens Diff Modal with zero data loss; editing a draft does not alter the published allergen banner; numeric-preservation guard catches mutated quantities.

### Phase 4 — Kitchen Cook Mode, Sharing, Open Export & Accessibility QA
* **Deliverables:** Kitchen Cook Mode (62/38 iPad split, Wake Lock, ID-keyed "Save for Offline" prefetch, inline ingredient peek, `verified`-only timers, ±2s manual audio trim, personal speed in `localStorage`); WhatsApp text + revocable `/share/:token` (owner/contributor only, `include_audio` default `false`); client-side streaming Open JSON + Audio ZIP export; ownership transfer & 14-day grace deletion; Playwright + VoiceOver/TalkBack audit.
* **Definition of Done:** Prefetched recipe works 100% in Airplane Mode; revoked share token returns `404`; Owner cannot delete account without transferring or confirming vault deletion; WCAG 2.2 AA audit passes.

### Phase 5 — Market & Kitchen Pilot Validation (4 Weeks)
* **Recruitment:** 20 families (≥ 15 completing 3+ recipes across at least 2 generations).
* **Exit Criteria (Go/No-Go for V2):**
  * **Draft Acceptance:** ≥ 80% of captured recipes published with **< 5 minutes** of editing and **≤ 5 field edits**.
  * **Cooking Authenticity:** ≥ 70% of dishes cooked by a Learner rated **4+/5 ("authentic")** by the Keeper or a family member who knows the original dish.
  * **Differentiator Engagement:** ≥ 50% of Learners actively use sensory cues, utensil calibration notes, or script toggling.
  * **Willingness to Pay:** ≥ 30% of pilot families indicate willingness to purchase a Lifetime Family Vault Pass.

---

## 9. Claude Code Starter Prompt

```text
Read docs/SPEC.md thoroughly. We are building Unmeasured as an offline-capable PWA/SPA backed by Supabase (Auth + Postgres + RLS + SECURITY DEFINER RPCs + Realtime) and Cloudflare (Workers + Workflows + R2 + Cron + Turnstile) using Gemini Paid Tier.

Before writing any code:
1. Confirm our hybrid auth model (RLS + 8 named SECURITY DEFINER RPCs + Worker R2 auth), our reserve-then-settle quota flow, our lean GeminiExtractionOutputSchema (server forces resolution: 'pending'), and our monotonic draft_version + field-level clarify conflict rules.
2. Start strictly with Phase 0: build the minimal Worker -> R2 -> Gemini Files API spike verifying Chrome WebM, Safari MP4, and WhatsApp Ogg ingestion, crash-recovered chunk reassembly, JS WebM->Ogg fallback if needed, explicit files.delete(), and the hourly orphan sweeper.
3. Pause after Phase 0 and report codec, crash-recovery, and golden-set accuracy results before starting Phase 1.

Rules: Never hardcode GEMINI_MODEL, never use Cloudflare KV/D1, never let users write vault_usage directly, never auto-infer allergens, and never let LLMs mutate numeric quantities during translation.
```