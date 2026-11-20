# Unmeasured — Product Requirements Document

**Tagline:** *Her recipes. Her voice. Your family’s legacy.*  
**Architecture:** Offline-Capable PWA / SPA backed by Supabase (Auth + Postgres) & Cloudflare (Workflows + R2)  
**Companion Engineering Spec:** `docs/SPEC.md`

---

## 1. Executive Summary & Strategic Wedge

**Unmeasured** preserves the cooking knowledge that lives in family members’ voices, hands, habits, handwritten diaries, and sensory judgment—not just standardized ingredient lists. Across every culinary tradition, family recipes are passed down through intuitive phrases such as *"one katori,"* *"a handful,"* *"until the raw smell goes away,"* or *"when the oil separates,"* or are scribbled as shorthand in worn notebooks.

> **The Strategic Wedge:** The durable family recipe archive that preserves how a dish is actually made—the cook’s original voice, native language, handwritten notes, calibrated household utensils, and sensory checkpoints—connected to a cookable, multi-device family vault.

To validate real-world kitchen value without over-engineering, the MVP focuses on **five core capabilities** backed by durable cloud storage from Day 1:

1. **Durable Multi-Device Family Vault:** Passwordless sign-in (Apple, Google, or Magic Link) and 1-tap WhatsApp-friendly household invite links (up to 6 seats) backed by Supabase Postgres and Cloudflare R2—eliminating browser eviction and letting Learners living away from home cook immediately. Elderly Keepers can record via **Archivist-Assisted Capture** with zero account setup.
2. **Dual-Entry Voice & Handwritten OCR Capture:** Resilient foreground audio recording (Screen Wake Lock, 5–10s crash-recovery chunks, and segment offsets), multi-file WhatsApp voice-note import, and 1–5 page handwritten diary scanning with idempotent offline queueing.
3. **Asynchronous Cloud AI Pipeline & Targeted Clarification:** Durable Cloudflare Workflows orchestrating Gemini Paid Tier extraction (streamed via Supabase Realtime), synchronous field-level clarification (≤ 3 questions + remaining gaps + *"No Cook Available"* memorial path), and explicit deletion of temporary Gemini files plus an hourly orphan sweeper.
4. **Complete Provenance & Explicit Allergen Safety:** Every ingredient field (`name`, `quantity_original`, `quantity_standard`, `unit`), step instruction, duration, and manual audio trim carries `{ source, resolution }` (`pending | verified | accepted_unknown`), with the server forcing `pending` on all AI output. Allergens (US FDA Big 9 + mustard + coconut) are strictly user-confirmed per revision—never AI-inferred—and unreviewed recipes explicitly display *"Allergens Not Reviewed."*
5. **5-Language Kitchen Experience & Native Sharing:** Offline UI localization (with a regional culinary glossary) and cached free-text translation across English, Marathi, Hindi, Telugu, and Gujarati (including Original Native/Romanized toggling), Kitchen-Friendly Cook Mode with explicit *"Save for Offline"* prefetch, and instant sharing via formatted WhatsApp text or revocable read-only web links.

---

## 2. Target Personas & Role Permissions

* **The Keeper (Elderly / Primary Family Cook):** Cooks by intuition, voice, and habit; often prefers speaking in Marathi, Hindi, Telugu, or Gujarati and dislikes logins or small UI controls. Uses **Archivist-Assisted Capture** (no account required) or the **`contributor`** role on their own device.
* **The Archivist (Vault `owner`):** Adult child or family organizer who records Keepers during visits, scans old diaries, calibrates household utensils, manages household invites, and safeguards the family archive.
* **The Learner (`learner` — Read-Only):** Next-generation family member living away from home who cooks from the vault on a phone or iPad, relies on sensory cues, calibrated utensil equivalents, and Romanized/English views, and should not accidentally alter heirloom originals.

| Role | Target User | Permissions |
| :--- | :--- | :--- |
| **`owner`** | The Archivist | Full read/write, capture, clarify, publish, manage invites (max 6 seats), promote `contributor` to `owner`, set Family Allergy Profile, create/revoke public share links, run client-side Open JSON + Audio export, delete vault (14-day grace period) or transfer ownership before account deletion. |
| **`contributor`** | Active Family Cooks / Keepers | Capture voice/OCR, answer clarifications, edit drafts, publish revisions, calibrate utensils, create/revoke public share links, use Cook Mode. |
| **`learner`** | Read-Only Family Member | Browse vault, switch languages/scripts (permitted to trigger translations), prefetch recipes for offline Cook Mode, adjust personal audio speed, copy WhatsApp text. Cannot edit originals or create/revoke public share links. |
| **Archivist-Assisted** | Elderly Keepers (No Account) | Archivist records/clarifies on their own device under a named Keeper session with logged consent—zero login or magic-link friction. |

---

## 3. MVP Scope & V2 Roadmap

| In Scope for MVP (Phases 0–4) | Deferred to V2 (Post-Pilot Validation) |
| :--- | :--- |
| • Passwordless Auth + Turnstile on sign-up/invites + Family Vault (max 6 seats) with Role Permissions Matrix.<br>• Durable cloud storage (Supabase Postgres + Cloudflare R2) with Dexie offline cache (keyed by `media_id`), 5–10s chunk crash recovery, idempotent outbox, and safe sign-out guard.<br>• Voice Narration (Wake Lock), Archivist-Assisted Keeper capture (with `consents` log), multi-file WhatsApp voice-note import, and 1–5 page OCR scan.<br>• Async Cloudflare Workflow for extraction (Realtime progress, reserve-then-settle Postgres quotas, Gemini file cleanup + hourly sweeper); sync routes for field-level clarify & translate.<br>• Complete `{ source, resolution }` provenance across name, original quantity, inferred standard quantity, calibrated utensils, unit, steps, durations, and manual ±2s trims; memorial `accepted_unknown` path.<br>• Per-revision explicit allergen checklist (FDA Big 9 + mustard + coconut) defaulting to `not_reviewed` + hing/ghee/nut paste reminder.<br>• 5-language UI (`i18next` + culinary glossary) + free-text translation (numeric-preservation guard) + publish-time Original Romanization.<br>• Kitchen-Friendly Cook Mode (Wake Lock, explicit *"Save for Offline"* prefetch, inline ingredient peek, `verified`-only timers, manual audio trim).<br>• Sharing via WhatsApp text or revocable `/share/:token` web link (`owner`/`contributor` only, `include_audio` default `false`) + client-side Open JSON + Audio ZIP export. | • Native iOS/iPadOS & Android store packaging via Capacitor (background native timers, App Attest / Play Integrity, In-App Purchase).<br>• Server-side Heirloom PDF & printed hardcover cookbook generation (headless browser HarfBuzz shaping).<br>• Smart Culinary Scaling (`0.5×–4×`) and AI *"Cook for a Crowd"* (`> 4×`) batching advice.<br>• Recipe forking, *"I Made This"* cook logs, and full-text offline MiniSearch across scripts.<br>• Hands-free voice navigation in Cook Mode (*"Next step"*, *"Repeat"*).<br>• Zero-install Remote Capture Links (*"Request a Recipe"*) for parents abroad.<br>• Experimental on-device / local WebGPU AI adapters. |

---

## 4. Core Workflows, Privacy & Durability

* **Consent & Two-Party Recording Protection:** Before first capture, display explicit consent covering verbal recording permission and third-party AI processing (Gemini Paid Tier, Apple 5.1.1/5.1.2). Archivist-Assisted Keeper sessions log auditable consent in the `consents` table, with a documented deletion request path for Keepers without accounts.
* **Conflict-Free Multi-Device Editing:** Drafts use a strictly monotonic `draft_version` with an atomic `save_draft` RPC and a Diff Modal on `409 Conflict` so no edits are lost. Queued offline clarifications use field-level (`target_path`) conflict checks so unrelated edits do not stall offline answers.
* **Vault Ownership & Apple 5.1.1(v) Deletion:** An `owner` cannot delete their account while owning an active multi-member vault until they transfer ownership to a `contributor` or explicitly confirm vault deletion (subject to a 14-day grace period). Permanent deletion purges Postgres rows and all R2 objects completely.
* **Backups & Open Export Guarantee:** Cloud data is encrypted at rest in Supabase Postgres (with Supabase PITR enabled before public launch) and Cloudflare R2 (scheduled daily bucket backup/replication). Owners can download a documented **Open JSON + Audio (.zip)** archive built on the client at any time so family heritage outlives any single vendor.

---

## 5. Visual System, Screen Architecture & WCAG 2.2 AA

Unmeasured uses a warm editorial palette verified for **WCAG 2.2 Level AA**: charcoal body text `#1C1917` on parchment `#FAF6F0` (`16.2:1`), dark terracotta `#9A3412` for small text/links on parchment (`6.8:1`), and white text on primary terracotta `#B84A27` (`5.2:1`). Never rely on color alone for provenance or allergen banners—always pair icons with explicit text labels.

* **Core Screens:** `/vault` (Family Vault), `/capture` (Capture Studio), `/recipes/:id/review` (iPad 55/45 split), `/recipes/:id` (iPad 40/60 split), `/recipes/:id/cook` (iPad 62/38 split, nav rail hidden), `/share/:token` (Public Shared View), and `/settings` (Vault & Household).
* **Accessibility Guardrails:** Minimum `56×56 px` (`64×64 px` on tablets) touch targets in Capture and Cook Mode; single-tap toggle microphones; 200% Dynamic Type reflow; dynamic `lang` attributes for VoiceOver/TalkBack; toggleable audio transcripts with personal playback speed controls; `prefers-reduced-motion` support; and zero `aria-live` announcements on routine 5–10s chunk saves.

---

## 6. Monetization, Phased Delivery & Phase 5 Pilot Gate

**Bursty-Use Monetization Fit:** Family recipe capture is inherently bursty (recording 10–20 dishes during a holiday visit, then cooking from them for years). Unmeasured will test a **Lifetime Family Vault Pass** (backed by our documented Open JSON + Audio export guarantee) alongside a V2 **Printed Heritage Cookbook** keepsake add-on.

### Phased Delivery Plan
1. **Phase 0 (3–5 Days) — Technical Spikes & Golden Regression Gate:** Verify Chrome WebM, Safari MP4, and WhatsApp Ogg ingestion, crash-recovered chunk reassembly, JS WebM→Ogg fallback, explicit `files.delete()`, and hourly orphan sweeper against a 10-recipe labeled golden set (mandatory regression gate on model/prompt changes).
2. **Phase 1 — Foundation, Auth, Vault & Offline Cache:** Scaffold Vite/React/TS/Tailwind/`i18next` (with culinary glossary)/PWA; deploy Supabase schema, RLS + 8 `SECURITY DEFINER` RPCs, JWKS auth, Turnstile on sign-up/invites, and Dexie cache with safe sign-out guard.
3. **Phase 2 — Capture Studio & Durable Cloudflare Workflow:** Build consent flow (`consents` table), Capture Studio (Wake Lock, 5–10s crash-recovery chunks, Archivist-Assisted mode, idempotent outbox), and Cloudflare Workflow + reserve/settle quotas + Realtime `ai_jobs`.
4. **Phase 3 — Provenance Review, Allergens & 5-Language Switching:** Build split-pane Review (`save_draft` + 409 Diff Modal, field-level clarify, memorial `accepted_unknown`, full provenance, per-revision allergen checklist, `publish_recipe` RPC + publish-time Romanization) and sync free-text translation with numeric-preservation guard.
5. **Phase 4 — Kitchen Cook Mode, Sharing, Open Export & Accessibility QA:** Build Cook Mode (ID-keyed *"Save for Offline"* prefetch, `verified`-only timers, ±2s manual trim), WhatsApp text + revocable `/share/:token` (`owner`/`contributor` only), client-side streaming Open JSON + Audio export, ownership transfer, and full VoiceOver/TalkBack audit.
6. **Phase 5 (4 Weeks) — Market & Kitchen Pilot Validation:** Recruit **20 families** (≥ 15 completing 3+ recipes across 2+ generations) to evaluate the Go/No-Go gate before V2 investment:

| Pilot Metric (Phase 5 Gate) | Pass Threshold | Measurement Definition |
| :--- | :--- | :--- |
| **Draft Acceptance** | **≥ 80%** of captured recipes | Published with **< 5 minutes** of editing and **≤ 5 field edits**. |
| **Cooking Authenticity** | **≥ 70%** of dishes cooked | Rated **4+/5 ("authentic")** by the Keeper or a family member who knows the original dish. |
| **Differentiator Engagement** | **≥ 50%** of Learners | Actively use sensory cues, utensil calibration notes, or script toggling during Cook Mode. |
| **Willingness to Pay** | **≥ 30%** of pilot families | Express intent to purchase a Lifetime Family Vault Pass. |