# PetSmile — Project Plan & Software Framework Design

*A swipe-based, AI-assisted pet adoption platform connecting shelters with adopters*

---

## 1. Product Overview

PetSmile lets shelters list adoptable pets with photos, descriptions, and minimum adoption requirements. Adopters browse pets through a filtered, swipe-based interface. A "match" occurs when an adopter swipes right on a pet **and** the adopter's profile satisfies that pet's minimum requirements. After a match, the adopter can schedule a live video chat with the shelter to meet the pet remotely, and if that goes well, book an in-person visit — all inside the app.

AI is used throughout to remove friction on both sides: shelters get listings drafted for them from photos and a few notes; adopters get natural-language search and smarter recommendations. Every AI output that describes a real animal is a **draft requiring shelter confirmation** — nothing auto-publishes.

**Platforms:** Web (responsive) + iOS + Android.

**Primary users:**

| Role | Needs |
|---|---|
| Adopter | Discover pets, filter, swipe, chat, schedule visits |
| Shelter staff | Post/manage pets fast (AI-assisted), set requirements, review matches, manage calendar, run video calls |
| Platform admin | Verify shelters, moderate content, analytics |

**Key differentiators:**
1. **Requirement pre-screening** — a right-swipe only becomes a match if the adopter's home profile (has cats? has kids? yard? experience?) passes the pet's minimum requirements, saving shelters from unqualified inquiries.
2. **AI-assisted listing** — a shelter can publish a complete, well-written pet profile in under 3 minutes from photos and a few bullet points. Shelter effort is the platform's biggest adoption risk; AI attacks it directly.

---

## 2. Core Feature Set

### 2.1 Adopter side
1. **Onboarding & home profile** — location, housing type, yard, other pets (species/count), children (ages), work schedule, pet experience. This profile powers requirement screening.
2. **Discovery feed** — card stack of pets within a radius; filters for species, breed, age range, size, sex, energy level, distance.
3. **Natural-language search (v1.1)** — "calm older cat that's okay alone during workdays" is parsed by an LLM into structured filter parameters.
4. **Swipe interaction** — right = interested, left = pass, tap for full profile (photo gallery, video, description, medical/behavioral notes, requirements shown transparently).
5. **Match list & scheduling** — matched pets appear in a list; adopter picks a video-chat slot from the shelter's published availability, later books an in-person visit.
6. **In-app video call** — join the scheduled live chat directly from the app.
7. **Notifications** — new match, appointment reminders, pet status changes.

### 2.2 Shelter side
1. **Shelter account & verification** — registration with documents; admin approves before listing.
2. **AI-assisted pet listing** — staff upload photos and jot rough notes; the system then:
   - picks the best photo as the card cover and flags blurry/dark shots (vision model),
   - pre-fills species, breed guess, size, and color from photos for staff to confirm,
   - drafts the warm, adopter-facing description from the staff notes (LLM),
   - staff review, edit, and approve before anything goes live.
3. **Requirement builder with AI extraction** — staff can write requirements in plain language ("she doesn't do well with young kids or cats"); an LLM maps this to structured rules (`no_children_under: 6`, `other_cats: false`) which staff confirm. Structured rules power the matching engine; the original free text stays visible to adopters.
4. **Match inbox with AI briefs** — matched adopters shown with a privacy-scoped profile summary; before a video call, staff see a one-paragraph AI brief ("Sarah, apartment with fenced yard, one senior dog, first-time cat owner — meets all of Milo's requirements") *(v1.1)*.
5. **Availability calendar** — staff publish video-chat and visit slots; app handles booking, rescheduling, cancellation, reminders.
6. **Video call console** — start/join calls, mark outcomes, leave notes. AI call summarization into outcome notes *(v2)*.

### 2.3 Matching logic

```
right-swipe received
  → evaluate adopter profile against pet.requirements (rules engine)
      ├─ fails any hard rule → polite "not eligible" (reason shown), no match created
      └─ passes → Match created
            ├─ shelter auto-accept ON  → scheduling unlocked immediately
            └─ shelter auto-accept OFF → pending shelter approval → then scheduling
```

Requirements are structured key/value rules evaluated automatically. AI extraction (2.2 #3) is the bridge between how shelters naturally write and what the engine can evaluate — free text in, confirmed structured rules out.

Ranking: MVP orders the deck with simple heuristics (distance, freshness, longest-listed boost). Once swipe volume exists, a learning-to-rank model personalizes deck order *(v2)*.

### 2.4 AI capability roadmap

| Capability | Type | Phase |
|---|---|---|
| Pet description generation from staff notes | LLM, single call | **MVP** |
| Requirement extraction (free text → structured rules) | LLM, single call | **MVP** |
| Content moderation (listing text + images) | Moderation/vision API | **MVP** |
| Best-photo selection, quality flags, attribute pre-fill | Vision model | v1.1 |
| Natural-language search → filters | LLM parsing | v1.1 |
| Pre-call match briefs for shelters | LLM summarization | v1.1 |
| Adoption Q&A assistant (answers from pet profile + shelter FAQ, escalates when unsure) | RAG | v2 |
| Deck ranking / recommendations | Learning-to-rank on swipe data | v2 (needs traffic) |
| Video-call note summarization | Transcription + LLM | v2 |

**Guardrails (apply to all of the above):**
- Human-in-the-loop: AI text about a real animal's health, behavior, or requirements is always a draft a staff member approves. This is both a quality and a liability decision — a hallucinated "good with kids" is unacceptable.
- Every generation is stored with prompt, model, and version for auditability.
- The Q&A assistant answers only from the pet's confirmed profile and shelter FAQ; when it doesn't know, it says so and hands off to the shelter.
- Adopter PII is never sent to model APIs beyond what a feature strictly needs; match briefs use the same privacy-scoped fields the shelter already sees.

---

## 3. System Architecture

```
                 ┌────────────────────────────────────────────┐
                 │                 Clients                    │
                 │  Web (React/Next.js)   iOS/Android (RN)    │
                 └──────────────┬─────────────────────────────┘
                                │ HTTPS / WSS
                 ┌──────────────▼─────────────┐
                 │        API Gateway         │  auth, rate limit, routing
                 └─┬──────┬─────────┬───────┬─┘
        ┌──────────┤      │         │       ├──────────────┐
┌───────▼──────┐ ┌─▼──────▼──┐ ┌────▼─────┐ ┌▼───────────┐ ┌▼──────────┐
│ Core API     │ │ Matching  │ │Scheduling│ │ Media Svc  │ │ AI Service│
│ (users, pets,│ │ (feed,    │ │(calendar,│ │ (upload,   │ │ (descrip- │
│ shelters)    │ │ swipes,   │ │ bookings,│ │ resize,    │ │ tions,    │
│              │ │ rules eng)│ │reminders)│ │ CDN)       │ │ rule extr,│
│              │ │           │ │          │ │            │ │ moderation│
└───────┬──────┘ └─────┬─────┘ └──┬───────┘ └──┬─────────┘ └──┬────────┘
        │              │          │            │              │
   ┌────▼──────────────▼──────────▼────┐  ┌────▼─────┐  ┌─────▼────────┐
   │ PostgreSQL (+PostGIS)  │  Redis   │  │ S3 + CDN │  │ LLM/Vision   │
   └───────────────────────────────────┘  └──────────┘  │ APIs         │
                                                        │ (Anthropic + │
   Cross-cutting: Notification service (FCM/APNs/email) │ vision/mod.) │
                  Video service (Twilio Video / LiveKit)└──────────────┘
                  Auth (JWT + refresh; OAuth social login)
```

**Recommended approach:** start as a **modular monolith** (one deployable, clean module boundaries matching the services above) and split into services only when scale demands it. The **AI Service is a module from day one** with a narrow interface (`generateDescription`, `extractRules`, `moderate`, later `summarizeMatch`, `parseSearch`) so models and vendors can be swapped without touching product code. AI calls run async via the job queue where latency allows (moderation, briefs) and synchronously with streaming where UX demands it (description drafting in the listing editor).

### 3.1 Technology choices

| Layer | Choice | Rationale |
|---|---|---|
| Web frontend | React + Next.js + TypeScript | SEO for pet listings, SSR, shared TS types |
| Mobile | React Native (Expo) | One codebase for iOS/Android; shares logic/types with web |
| Backend | Node.js + NestJS (TypeScript) | End-to-end TypeScript, modular structure fits monolith→services path |
| AI — text | Anthropic API (Claude) | Description drafting, rule extraction, briefs, NL search parsing; structured outputs (JSON) for rule extraction |
| AI — vision/moderation | Vision-capable model + moderation API (e.g. Claude vision, AWS Rekognition for image moderation) | Photo quality/attributes; listing & profile screening |
| Database | PostgreSQL + PostGIS | Relational fit; geo queries for radius search |
| Cache/queues | Redis (cache, dedupe) + SQS/BullMQ | Feed caching, notification fan-out, async AI jobs |
| Media | S3 + CloudFront, on-upload image pipeline | Photo-heavy app |
| Video calls | Twilio Video or LiveKit Cloud | Managed WebRTC — never in-house for MVP |
| Search/feed | Postgres first; OpenSearch in v2 | Avoid premature infra |
| Notifications | FCM + APNs + SendGrid | Push + email reminders |
| Auth | JWT access/refresh, Apple/Google sign-in | Store requirements |
| Infra | AWS (ECS Fargate), Terraform, GitHub Actions | Standard, hiring-friendly |
| Monitoring | Sentry + CloudWatch/Grafana; AI eval logging | Error + perf visibility; track edit-distance on AI drafts as a quality metric |

**Monorepo layout (Turborepo/Nx):**

```
/apps
  /web            Next.js
  /mobile         React Native (Expo)
  /api            NestJS  (modules: core, matching, scheduling, media, ai)
/packages
  /shared-types   DTOs, enums, requirement rule schema (zod)
  /prompts        versioned prompt templates + eval fixtures for the AI module
  /ui             shared design tokens/components where practical
  /api-client     generated typed client from OpenAPI
```

### 3.2 Data model (core entities)

```
users(id, role[adopter|shelter_staff|admin], email, auth..., created_at)
adopter_profiles(user_id FK, location geog, housing_type, has_yard,
                 other_pets jsonb, children jsonb, experience_level, ...)
shelters(id, name, verification_status, location geog, contact, ...)
shelter_staff(user_id FK, shelter_id FK, role)

pets(id, shelter_id FK, name, species, breed, age_months, size, sex,
     energy_level, description, description_source[ai_draft|staff_edited|manual],
     status[available|pending|adopted|hidden], created_at, updated_at)
pet_media(id, pet_id FK, type[photo|video], url, sort_order,
          quality_score, is_cover bool)
pet_requirements(id, pet_id FK, rule_key, operator, value, is_hard_rule bool,
                 source[ai_extracted|manual], confirmed_by FK nullable)
     -- e.g. ('other_cats','eq',false,true), ('adopter_age','gte',21,true)

ai_generations(id, type[description|rule_extraction|brief|moderation|search_parse],
               subject_id, model, prompt_version, input_hash, output jsonb,
               status[draft|approved|rejected|edited], reviewed_by FK, created_at)

swipes(id, user_id FK, pet_id FK, direction, created_at)  UNIQUE(user_id, pet_id)
matches(id, user_id FK, pet_id FK, shelter_id FK,
        status[pending_shelter|active|closed], matched_at)

availability_slots(id, shelter_id FK, type[video|visit], start, end, capacity)
appointments(id, match_id FK, slot_id FK, type, status[booked|completed|
             cancelled|no_show], video_room_id, outcome_notes)

notifications(id, user_id FK, type, payload jsonb, read_at)
```

### 3.3 Key API surface (REST, versioned `/v1`)

```
POST /auth/register|login|refresh
GET/PUT /me/profile

GET  /feed?filters...              # paginated card deck, geo + filter aware
POST /search/parse {query}         # v1.1: NL query → structured filters
POST /pets/:id/swipe {direction}   # returns {matched: bool, match?}

GET  /matches                      # adopter or shelter scoped
POST /matches/:id/approve|decline  # shelter
GET  /matches/:id/brief            # v1.1: AI pre-call brief (shelter only)

CRUD /shelters/:id/pets            # staff only
POST /pets/:id/media (presigned S3 upload)
POST /pets/:id/ai/describe {notes} # streaming draft description
POST /pets/:id/ai/extract-rules {text}  # → proposed structured rules
POST /pets/:id/requirements/confirm     # staff approves AI-proposed rules
CRUD /pets/:id/requirements

CRUD /shelters/:id/slots
POST /matches/:id/appointments {slot_id}
POST /appointments/:id/cancel|reschedule
POST /appointments/:id/video-token # short-lived token for Twilio/LiveKit room

WS   /ws                           # match events, appointment updates
```

### 3.4 Feed, swipe & AI pipeline notes

- Feed query: PostGIS radius filter + attribute filters + `NOT EXISTS` in swipes + status = available; cache page cursors in Redis.
- Swipe writes are idempotent (unique constraint) and evaluated synchronously against the rules engine so the "It's a match!" moment is instant. The rules engine itself is deterministic — AI only *proposes* rules; it never sits in the hot matching path.
- Listing pipeline: photos upload → moderation + quality scoring job → staff notes → streamed description draft → staff edit/approve → publish. Moderation failures block publish and notify staff.
- Prompt templates live in `/packages/prompts`, versioned, with a small eval fixture set (sample notes → expected rule extractions) run in CI so prompt changes don't silently regress extraction accuracy.
- Track % of AI drafts approved without edits and average edit distance — the north-star quality metrics for the AI listing features.

### 3.5 Security, privacy & compliance

- Shelter verification workflow before any listing goes live.
- Adopter profile shared with a shelter **only after a match**, and only requirement-relevant fields; the same scoping applies to AI-generated briefs.
- Data minimization to model APIs: no adopter contact info or exact address in any prompt; use a vendor with a no-training-on-inputs policy (e.g. Anthropic API).
- AI content policy: health/behavior claims require staff approval (enforced by `ai_generations.status`); audit trail retained.
- Media upload scanning + AI image moderation before publish.
- Video calls: access via short-lived room tokens tied to the appointment; no recordings in MVP.
- Adopters must be 18+; age gate at signup.
- Standard: TLS everywhere, encrypted at rest, rate limiting, audit log for shelter actions, GDPR/CCPA-style data export & deletion (including associated `ai_generations`).

---

## 4. Project Plan

### Phase 0 — Discovery & Design (Weeks 1–3)
Interviews with 3–5 shelters — validate requirement screening, scheduling, **and the AI listing flow** (show a clickable prototype of notes-to-description; measure whether staff trust and would use it). Competitive review, finalize MVP scope, UX wireframes and design system. Technical spikes: video SDK, RN/Next.js code sharing, and a prompt-engineering spike for description generation + rule extraction against real shelter-written text.

### Phase 1 — MVP Build (Weeks 4–15)

| Sprint | Focus |
|---|---|
| 1–2 | Monorepo, CI/CD, auth, user/shelter models, shelter verification (manual admin) |
| 3–4 | Pet CRUD + media pipeline, adopter profile onboarding |
| 5 | AI service module: description drafting (streaming) + moderation pipeline |
| 6 | Requirement builder + AI rule extraction with staff confirmation flow |
| 7–8 | Feed with filters + geo, swipe mechanics, rules engine, match creation |
| 9–10 | Availability calendar, video-chat booking, Twilio/LiveKit integration |
| 11 | In-person visit booking, notifications (push/email), reminders |
| 12 | Web polish, mobile builds, admin dashboard basics, prompt eval suite in CI, load/security testing, beta prep |

**MVP cut line:** AI = description drafting, rule extraction, and moderation only (photo scoring, NL search, briefs deferred to v1.1); no in-app text chat (video + email only); no payments; English only; Postgres-only search; shelter auto-accept only.

### Phase 2 — Pilot (Weeks 16–21)
Launch with 3–5 partner shelters in one metro area. Success metrics: median time-to-publish a listing <5 min, ≥70% of AI description drafts approved with light or no edits, ≥60% of matches book a video chat, ≥30% of video chats convert to visits, shelter weekly-active retention. Collect real staff free-text to expand the rule taxonomy and extraction eval set — the messy cases will surface here.

### Phase 3 — v1 Public Launch (Weeks 22–31)
Shelter approval flow, in-app messaging, photo quality scoring + attribute pre-fill, natural-language search, pre-call match briefs, eligibility badges in feed, saved searches & alerts, app store launch, shelter analytics dashboard, second metro.

### Phase 4 — v2 (post-launch roadmap)
Adoption Q&A assistant (RAG over pet profiles + shelter FAQs), learning-to-rank deck ordering on accumulated swipe data, video-call summarization into outcome notes, multi-language support, foster-program workflows, application forms & e-signatures, post-adoption follow-ups, donations, OpenSearch-backed discovery.

### Team (MVP)

1 product/founder, 1 designer, 2 full-stack TS engineers, 1 mobile-leaning engineer, part-time QA + DevOps. AI features use API-based models with prompt engineering — no ML hires needed until the v2 ranking work. (~4.5 FTE for ~15 weeks.)

### Top risks & mitigations

| Risk | Mitigation |
|---|---|
| Shelters won't maintain listings/calendars | AI-assisted listing makes posting <5 min; pilot co-design; CSV/Petfinder import later |
| AI hallucinates health/behavior claims | Hard rule: all AI text is a draft requiring staff approval; audit trail; prompt eval suite in CI |
| Rule extraction misreads shelter free text | Staff confirmation step on every extracted rule; original text always displayed; extraction accuracy tracked against eval set |
| Requirement rules too rigid for real cases | Hard vs. soft rules; shelter approval override; free-text notes |
| AI API cost creep | Cache by input hash; draft-once/edit-forever flow; per-shelter rate limits; cheap model tiers for moderation |
| Video infra complexity | Managed SDK (Twilio/LiveKit), never custom WebRTC for MVP |
| Two-sided cold start | Launch city-by-city; seed supply first (shelters), then adopters |
| Ghosting/no-shows | Reminders, confirm-24h-before flow, no-show tracking |

---

## 5. Immediate Next Steps

1. Confirm MVP cut line and draft the initial `rule_key` taxonomy with 2–3 shelters.
2. Run the prompt spike: collect 20–30 real shelter-written pet notes and requirement texts, build the first description + extraction prompts, and score them — this de-risks the two MVP AI features before sprint 5.
3. Decide Twilio vs. LiveKit (cost spike).
4. Wireframe the AI listing editor, swipe deck, requirement builder, and booking flow first — they carry the product.
