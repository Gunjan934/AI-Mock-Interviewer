# Product Requirements Document: AI Mock Interviewer Platform

**Document status:** Draft for product and engineering review  
**Version:** 1.0  
**Author:** Manus AI  
**Date:** 27 September 2026  
**Primary audience:** Product, engineering, design, data, security, and QA teams

> **Implementation status (as of 2026-11-21 — verified in this repository):**
> - **Phases 1–2 — IMPLEMENTED:** NVIDIA NIM connectivity playground
>   (`/ai/playground`), dynamic NVIDIA-generated interview questions with
>   backend validation/dedupe and question-bank fallback, and the AI
>   interview agent (`FOLLOW_UP`/`CLARIFY`/`NEXT_QUESTION`/`FINISH`, capped
>   follow-ups, bank fallback). See
>   `docs/ai_mock_interviewer_architecture.mmd`.
> - **Phase 3 — AI answer evaluation — IMPLEMENTED:** `ai_v1` structured
>   rubric evaluation (correctness/relevance/completeness/clarity/practical
>   understanding; backend-computed weighted overall); `baseline_v1`
>   preserved as the fallback evaluator.
> - **Phase 4 — ML evaluator — IMPLEMENTED, disabled by default:** `app/ml`
>   (TF-IDF + LinearSVC, `ml_v1`). See `docs/ML_EVALUATOR_LINEAR_SVC.md`.
> - **Phase 6 — RAG — foundation implemented, disabled by default:** `app/rag`
>   (ChromaDB + sentence-transformers). See `docs/PHASE6_RAG.md`.
> - **Phase 7 — career guidance chat — IMPLEMENTED:** NVIDIA-backed
>   `/career-guidance/` chatbot.
> - **Voice (ASR/TTS) — NOT implemented** (planned).
>
> Verified test status: `pytest` — **127 passed, 1 failed** (the live test
> `tests/test_phase2_live.py::test_live_question_generation` fails because the
> model response was not a JSON array; live behavior is not yet fully
> verified). See `PROJECT_DESCRIPTION.md` for the authoritative verified
> implementation notes (Sections 4–19).

## 1. Product definition

The AI Mock Interviewer is a web platform that helps candidates prepare for interviews through realistic, role-specific practice sessions. It combines an adaptive AI interviewer, resume and job-description analysis, interview preparation plans, answer coaching, and a performance dashboard.

The product must support text-first interviewing in the MVP and be designed for low-latency voice interviewing in the next phase. It should provide useful coaching without pretending to be a hiring decision-maker. Scores are coaching signals, not assessments of employability or personality.

### 1.1 Problem statement

Candidates often prepare with generic question lists and receive little feedback on whether their answers are relevant, specific, concise, and supported by evidence. Existing tools commonly separate resume editing, interview practice, and analytics. The product should connect these workflows so that practice is grounded in the candidate’s target role, resume, experience level, and previous performance.

### 1.2 Product goals

1. Generate an interview plan from a target role, job description, seniority, interview type, and candidate profile.
2. Conduct realistic mock interviews with adaptive follow-up questions.
3. Analyze answers using explainable criteria such as relevance, structure, evidence, clarity, technical correctness, and completeness.
4. Help users improve their resume for a target job without fabricating experience.
5. Provide a dashboard that shows progress by competency, question type, role, and session over time.
6. Support secure, observable, replaceable NVIDIA model integrations.
7. Give users actionable next steps after every session, not only a score.

### 1.3 Non-goals for the first release

- Automated hiring, candidate rejection, or employment recommendations.
- Guaranteed resume ranking or ATS-pass claims.
- Fabricating work history, education, certifications, projects, or achievements.
- Live assistance during a real employer interview.
- Biometric identity verification or emotion recognition.
- Medical, psychological, or personality diagnosis.
- A marketplace for recruiters in the MVP.

## 2. Target users and use cases

### 2.1 Primary personas

**Candidate.** A student, career switcher, experienced professional, or job seeker who wants structured practice and measurable improvement.

**Career coach or educator.** A professional who wants to review a candidate’s sessions and assign practice plans, subject to explicit candidate consent.

**Platform administrator.** An internal operator who manages question banks, rubrics, feature flags, model configuration, abuse reports, and system health.

### 2.2 Core user journeys

#### Journey A: First-time onboarding

1. User creates an account and verifies email.
2. User selects target role, seniority, location, interview type, and preferred language.
3. User uploads a resume as PDF or DOCX and optionally pastes a job description.
4. The platform extracts structured profile data and shows a review screen.
5. User confirms or edits skills, projects, experience, and target competencies.
6. The platform generates a preparation plan and recommends a first mock interview.

#### Journey B: Text mock interview

1. User chooses a role, duration, mode, and difficulty.
2. The platform creates a session plan containing question categories and rubric dimensions.
3. The interviewer asks one question at a time.
4. The user submits a text answer or uses voice when enabled.
5. The AI evaluates the answer, decides whether a follow-up is needed, and continues within the session budget.
6. The platform ends with a summary, score breakdown, model-generated feedback, and recommended drills.

#### Journey C: Resume builder

1. User selects a target job.
2. The platform compares the resume against the job description.
3. It identifies missing or weakly evidenced keywords and competencies.
4. It suggests rewrites grounded in user-provided facts.
5. User accepts, edits, or rejects suggestions.
6. The platform exports a versioned resume as PDF and DOCX in a later release; MVP can export structured text/Markdown or PDF.

#### Journey D: Progress dashboard

1. User opens the dashboard.
2. The platform shows session history, competency trend, strongest areas, weakest areas, answer-quality distribution, and completion streak.
3. User drills into a competency to see source answers, feedback, and recommended practice.
4. User starts a targeted drill generated from the weakest measured competency.

## 3. Scope by release

### Release 1: MVP

- Email/password authentication and optional social login abstraction.
- Candidate profile, target roles, resume upload, job-description input.
- Resume parsing into editable structured data.
- Text mock interviews for behavioral, technical conceptual, and HR questions.
- Question bank with role, seniority, competency, and difficulty metadata.
- Adaptive follow-up questions using the candidate’s answer and session rubric.
- Answer scoring with a versioned rubric and explanation.
- Session summary and basic dashboard.
- Admin console for question and rubric management.
- NVIDIA NIM LLM integration through an internal provider adapter.
- Basic safety, PII redaction, prompt-injection handling, audit logging, and rate limiting.

### Release 2: Voice and richer preparation

- Browser microphone capture and streaming ASR.
- Spoken AI interviewer with TTS.
- Transcript with timestamps and editable corrections.
- Delivery signals such as filler-word count, pause distribution, pace, and answer duration. These are descriptive signals only and must not infer confidence, honesty, emotion, or personality.
- Personalized daily preparation plan.
- Resume export to PDF/DOCX.
- Saved answer library and STAR story builder.

### Release 3: Coaching and team features

- Coach/educator workspaces with consent-based sharing.
- Cohort analytics with anonymization.
- Custom rubrics and question packs.
- Organization-level SSO, retention policies, and private deployment options.
- Experimentation platform for model and prompt evaluation.

## 4. Functional requirements

### 4.1 Authentication and account management

- The system shall support registration, login, logout, password reset, email verification, and session revocation.
- Passwords shall be hashed using Argon2id or an equivalent modern password hashing scheme.
- Access tokens shall be short-lived. Refresh tokens shall be rotated and revocable.
- The system shall support account deletion, export, and configurable retention.
- A user shall be able to manage preferred language, timezone, accessibility preferences, and notification settings.

### 4.2 Candidate profile and target role

A candidate profile shall include:

- Name and preferred name.
- Email and optional phone number.
- Location and work authorization fields only when explicitly supplied.
- Education, experience, skills, projects, achievements, certifications, and links.
- Target role, industry, seniority, interview type, and target companies as optional fields.
- Preferred language and answer mode.

The product shall display extracted data for confirmation before using it in generation. Any field produced by parsing or AI suggestion must have provenance: `user_entered`, `document_extracted`, or `ai_suggested`.

### 4.3 Resume ingestion and builder

The ingestion pipeline shall:

1. Accept PDF and DOCX with a configurable file-size limit.
2. Malware-scan the file before processing.
3. Extract text and document metadata.
4. Preserve page and section references where possible.
5. Parse the resume into a normalized schema.
6. Detect possible PII and store only what is necessary for the user’s workflow.
7. Present a review and correction UI.

The builder shall never add a factual claim unless the user provides or confirms the underlying fact. Suggestions must be represented as edits with before/after text, rationale, source facts, and confidence.

### 4.4 Job-description analysis

The system shall extract:

- Required and preferred skills.
- Responsibilities.
- Years of experience.
- Role level.
- Domain and tools.
- Behavioral competencies.
- Interview signals such as system design, coding, case study, or stakeholder management.

The system shall compare job requirements with the confirmed candidate profile and classify each requirement as `evidenced`, `partially_evidenced`, `not_evidenced`, or `unclear`. It shall avoid stating that the candidate is unqualified.

### 4.5 Interview configuration

A user shall be able to choose:

- Target role and job description.
- Session type: behavioral, technical, case, HR, mixed, or custom.
- Seniority: entry, associate, mid, senior, lead, or executive.
- Difficulty: beginner, standard, challenging.
- Duration or maximum number of questions.
- Language.
- Text or voice mode, when enabled.
- Whether hints are allowed.
- Whether the user wants feedback after each answer or only at the end.

The system shall show the selected rubric and expected session length before starting.

### 4.6 Adaptive interviewer

For each session, the interviewer shall:

- Ask a question grounded in the session plan.
- Avoid repeating recently asked questions unless repetition is intentional practice.
- Use the user’s confirmed resume and job description as context.
- Ask follow-ups when the answer is incomplete, vague, inconsistent, or missing evidence.
- Stop follow-ups when the answer satisfies the rubric or the session budget is reached.
- Keep questions within the selected topic and difficulty.
- Never request secrets, passwords, confidential employer information, or protected personal information that is not necessary.
- Handle an explicit skip, pause, restart, or end-session command.

### 4.7 Answer evaluation

Every evaluated answer shall use a rubric version. The default behavioral rubric is:

- Relevance to the question: 0–5.
- Structure and clarity: 0–5.
- Specific evidence or examples: 0–5.
- Ownership and actions: 0–5.
- Outcome or measurable result: 0–5.
- Concision and completeness: 0–5.

Technical answers may add:

- Correctness: 0–5.
- Trade-off reasoning: 0–5.
- Assumptions and edge cases: 0–5.
- Communication of implementation detail: 0–5.

The evaluator shall return a score, evidence spans or excerpts, missing elements, strengths, and one to three prioritized improvement actions. Scores must be calculated by the application from structured rubric dimensions where possible, rather than trusting an unvalidated total supplied by the LLM.

### 4.8 Performance dashboard

The dashboard shall provide:

- Total completed sessions and average session score.
- Trend by week and month.
- Scores by competency and interview type.
- Question difficulty distribution.
- Average answer length and duration.
- Follow-up rate and skipped-question rate.
- Strengths and improvement areas.
- Recommended next practice session.
- Comparison against the user’s own prior sessions, not a public ranking in MVP.

Each chart shall link to underlying sessions and answers. Users shall be able to export their data.

### 4.9 Admin and operations

The admin console shall support:

- CRUD for questions, question variants, competencies, rubrics, and prompt templates.
- Publishing workflow: draft, review, published, retired.
- Model-provider configuration by environment.
- Prompt and rubric versioning.
- Session and inference observability.
- User report review and moderation actions.
- Feature flags for voice, export, and experimental models.
- Audit log access with role-based permissions.

## 5. Technical architecture

### 5.1 Actual technology stack (as built, 2026-11-21)

| Layer | Actual technology | Status |
| --- | --- | --- |
| Web client | Jinja2 templates + Bootstrap 5 (CDN) + custom CSS/JS + Inter font | Implemented |
| API layer | Flask 3.1 (blueprints, thin routes, Flask-WTF CSRF) | Implemented |
| Business logic | `app/services`: evaluation_service, interview_service, resume_service, jd_service, knowledge_service, candidate_context; `app/nlp` | Implemented |
| AI stack | `app/ai`: `AIService` facade, `NvidiaProvider` (stdlib `urllib`), versioned `prompts.registry`, `question_gen`, `agent`, `answer_eval`, typed `exceptions` | Implemented (Phases 1–3) |
| Database | MySQL via Flask-SQLAlchemy + PyMySQL; Flask-Migrate/Alembic (6 revisions) | Implemented (primary; SQLite only as backup/test fixture) |
| Auth | Flask-Login + Werkzeug hashing + Flask-WTF CSRF | Implemented |
| Optional ML evaluator | `app/ml` — TF-IDF + LinearSVC (`ml_v1`) | Implemented, disabled by default |
| Optional RAG | `app/rag` — ChromaDB + Sentence Transformers | Foundation implemented, disabled by default |
| Voice/speech | — (NVIDIA Speech NIM/Riva referenced for future scope) | Not implemented |

Dependencies (requirements.txt): Flask 3.1.2, Flask-SQLAlchemy 3.1.1, Flask-Migrate 4.1.0 (Alembic), Flask-Login 0.6.3, Flask-WTF 1.2.2, python-dotenv 1.1.1, pypdf 6.0.0, PyMySQL 1.1.1 (+cryptography), scikit-learn/pandas/numpy/scipy/joblib (Phase 4 ML), chromadb/sentence-transformers (Phase 6 RAG), pytest 8.4.1.

### 5.2 Actual architecture

Flask is the only application framework. Topology (as built):

```text
Browser (Jinja2 + Bootstrap 5 + vanilla JS)
  → Flask application factory (app/__init__.py create_app)
      → Flask blueprints (app/routes): auth, profile, resume, jd, questions,
        interview, dashboard, ai, knowledge, career_guidance, errors
      → Business logic (app/services): evaluation_service, interview_service,
        resume_service, jd_service, knowledge_service, candidate_context
      → AI stack (app/ai): AIService → NvidiaProvider → NVIDIA NIM
      → Optional: app/ml (LinearSVC inference), app/rag (ChromaDB retrieval)
      → Optional: app/career_guidance services (NVIDIA chat)
      → SQLAlchemy models (app/models) → MySQL (Flask-SQLAlchemy)
```

The classic Flask/Django split is not used (see §17.5). Full architecture diagram: `docs/ai_mock_interviewer_architecture.mmd`.

### 5.3 NVIDIA integration (as built)

`app/ai/providers/nvidia.py` implements the live provider: `POST https://integrate.api.nvidia.com/v1/chat/completions` (OpenAI-compatible), `Authorization: Bearer NVIDIA_API_KEY` (from `.env`), model from `NVIDIA_CHAT_MODEL`, `response_format: {"type": "json_object"}` in JSON mode. Uses stdlib `urllib` (no SDK). Retries on 408/429/5xx with exponential backoff; strict response parsing. Typed errors (`app/ai/exceptions.py`): `not_configured` (503), `authentication_failed` (500), `rate_limited` (429), `timeout` (504), `connection_failed` (502), `invalid_response` (502), `validation_failed` (502). The API key never appears in responses or logs (credential-free `health()` snapshot with `has_api_key` boolean). The playground endpoint `/ai/playground` (blueprint `app/routes/ai.py`) is authenticated, CSRF-protected, and never changes interview questions or scores.

Live connectivity status: `tests/test_phase2_live.py::test_live_question_generation` calls the live NIM; in this environment the model returns a JSON response that fails structured validation ("Expected a JSON array from the model.") — live behavior is not yet fully verified.

### 5.4 Speech — not implemented (planned)

Voice (ASR/TTS) is out of scope for the current build. NVIDIA Speech NIM/Riva remains referenced only as a future option; no speech code paths exist.

### 5.5 Guardrails (as built + planned)

Application-level guards are implemented: strict JSON validation and schema validation in `app/ai/client.py`; validated, deduped, user-scoped generation in `app/ai/services/question_gen.py`; backend-validated agent decisions in `app/ai/services/agent.py` (per-question follow-up caps, rejection of malformed messages); untrusted-data wrapping in `app/ai/prompts/registry.py`; ownership-filtered routes. NVIDIA NeMo Guardrails remains a planned future layer; no guardrails wiring exists in the current code.

### 5.6 Retrieval and context policy (as built)

Phase 6 RAG (`app/rag`, disabled by default via `RAG_ENABLED=0`): when enabled, documents (PDF/TXT/MD/DOCX, validated uploads) are ingested — text cleaning, paragraph-preferred chunking with overlap and stable IDs, status tracking (PENDING/PROCESSING/COMPLETED/FAILED) — into ChromaDB (`instance/vector_store`, collection `interview_knowledge`, cosine space). Embeddings via sentence-transformers `all-MiniLM-L6-v2` (batch-normalized). Retrieval: query embedding, Top-K + similarity threshold, dedupe, `NO_RELEVANT_CONTEXT` when below threshold. Grounded helpers: `rag_interview_question` (question generation) and `rag_answer_feedback` (feedback) attach results with source attribution to a bounded context (≤6000 chars). Retrieved content is treated as untrusted reference data. UI: `/knowledge/` dashboard + upload + detail/search + RAG playground. See `docs/PHASE6_RAG.md` and `docs/ai_mock_interviewer_architecture.mmd`.

## 6. API specification

All endpoints shall be versioned under `/api/v1`. Responses shall include `request_id`. Errors shall follow a consistent shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The session duration must be between 5 and 60 minutes.",
    "details": {"field": "duration_minutes"},
    "request_id": "req_123"
  }
}
```

### 6.1 Authentication

```http
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
POST /api/v1/auth/password-reset/request
POST /api/v1/auth/password-reset/confirm
GET  /api/v1/me
PATCH /api/v1/me
DELETE /api/v1/me
```

### 6.2 Profiles and documents

```http
GET    /api/v1/profile
PATCH  /api/v1/profile
POST   /api/v1/resumes
GET    /api/v1/resumes
GET    /api/v1/resumes/{resume_id}
PATCH  /api/v1/resumes/{resume_id}
DELETE /api/v1/resumes/{resume_id}
POST   /api/v1/resumes/{resume_id}/parse
POST   /api/v1/job-descriptions
GET    /api/v1/job-descriptions/{job_description_id}
POST   /api/v1/job-descriptions/{job_description_id}/analyze
```

Example resume upload response:

```json
{
  "id": "res_01J...",
  "status": "queued",
  "filename": "resume.pdf",
  "content_type": "application/pdf",
  "created_at": "2026-09-27T12:30:00Z"
}
```

### 6.3 Interview sessions

```http
POST /api/v1/interview-sessions
GET  /api/v1/interview-sessions
GET  /api/v1/interview-sessions/{session_id}
POST /api/v1/interview-sessions/{session_id}/start
POST /api/v1/interview-sessions/{session_id}/answers
POST /api/v1/interview-sessions/{session_id}/skip
POST /api/v1/interview-sessions/{session_id}/pause
POST /api/v1/interview-sessions/{session_id}/resume
POST /api/v1/interview-sessions/{session_id}/end
GET  /api/v1/interview-sessions/{session_id}/report
```

Create-session request:

```json
{
  "target_role": "Backend Python Developer",
  "job_description_id": "jd_01J...",
  "resume_id": "res_01J...",
  "session_type": "behavioral",
  "seniority": "mid",
  "difficulty": "standard",
  "duration_minutes": 25,
  "feedback_mode": "after_each_answer",
  "language": "en",
  "mode": "text"
}
```

Submit-answer request:

```json
{
  "question_id": "q_01J...",
  "answer_text": "In my last project...",
  "client_started_at": "2026-09-27T12:40:00Z",
  "client_submitted_at": "2026-09-27T12:42:15Z",
  "idempotency_key": "ans_01J..."
}
```

Submit-answer response:

```json
{
  "answer_id": "ans_01J...",
  "evaluation_status": "completed",
  "evaluation": {
    "rubric_version": "behavioral_v1",
    "dimensions": {
      "relevance": {"score": 4, "max": 5, "evidence": ["...implemented a queue..."]},
      "structure": {"score": 3, "max": 5, "evidence": []},
      "evidence": {"score": 4, "max": 5, "evidence": ["reduced processing time by 30%"]}
    },
    "strengths": ["Uses a concrete example with a measurable outcome."],
    "gaps": ["Clarify the personal action versus the team action."],
    "next_actions": ["Retry using Situation, Task, Action, Result headings."],
    "overall_score": 3.8
  },
  "next_question": {
    "id": "q_01J...",
    "text": "What trade-off did you consider when choosing that design?",
    "type": "follow_up"
  }
}
```

### 6.4 Dashboard and preparation

```http
GET  /api/v1/dashboard/overview?range=30d
GET  /api/v1/dashboard/competencies?range=90d
GET  /api/v1/dashboard/trends?metric=overall_score&range=90d
POST /api/v1/preparation-plans
GET  /api/v1/preparation-plans/current
POST /api/v1/drills
GET  /api/v1/drills/{drill_id}
```

### 6.5 Internal AI orchestration endpoints

These endpoints shall be private-network-only or service-authenticated:

```http
POST /internal/ai/interview/question
POST /internal/ai/interview/evaluate
POST /internal/ai/resume/parse
POST /internal/ai/job/analyze
POST /internal/ai/feedback/summarize
POST /internal/ai/safety/check
GET  /internal/ai/health
```

Internal requests must include a service token, `request_id`, user consent state, model policy, and data classification. The public client must never call the NVIDIA endpoint directly.

## 7. Data model

Core tables/entities:

- `users`: identity, status, consent timestamps, created/updated timestamps.
- `profiles`: candidate profile fields and preferred settings.
- `resumes`: file metadata, storage key, parse status, checksum, retention status.
- `resume_versions`: normalized profile JSON, source provenance, version number.
- `job_descriptions`: raw text, normalized requirements, source and status.
- `target_roles`: role, seniority, industry, interview types.
- `competencies`: canonical competency taxonomy.
- `questions`: question text, type, competency, role, level, difficulty, status.
- `question_variants`: localized or model-generated variants linked to a canonical question.
- `rubrics`: versioned scoring dimensions, weights, descriptions, and examples.
- `interview_sessions`: configuration snapshot, status, start/end, model policy, rubric version.
- `interview_turns`: question, answer, transcript, sequence, timing, safety status.
- `evaluations`: structured scores, evidence, feedback, evaluator model, prompt version.
- `preparation_plans`: goals, milestones, generated drills, status.
- `dashboard_snapshots`: optional precomputed aggregates for performance.
- `ai_requests`: provider, model alias, latency, token usage, status, hashed user/session reference.
- `audit_events`: actor, action, object type/id, result, timestamp, IP/device metadata where lawful.

Use relational columns for queryable fields and JSONB for versioned model payloads. All AI-generated content must be linked to `model`, `prompt_version`, `input_source_ids`, and `created_at`.

## 8. AI prompt and evaluation contracts

### 8.1 Question generation contract

The model shall return only JSON matching:

```json
{
  "question": "string",
  "question_type": "behavioral|technical|case|follow_up|hr",
  "competency": "string",
  "difficulty": "beginner|standard|challenging",
  "intent": "string",
  "expected_signals": ["string"],
  "follow_up_triggers": ["string"]
}
```

### 8.2 Evaluation contract

```json
{
  "rubric_version": "string",
  "dimensions": [
    {
      "key": "relevance",
      "score": 0,
      "max_score": 5,
      "evidence": ["short quoted span or empty"],
      "reason": "string"
    }
  ],
  "strengths": ["string"],
  "gaps": ["string"],
  "improvement_actions": ["string"],
  "needs_follow_up": true,
  "follow_up_reason": "string"
}
```

The API shall reject scores outside allowed bounds, missing rubric dimensions, unsupported question types, unsupported claims, and evidence that is not present in the answer unless clearly marked as an inference. A second validation pass shall calculate the overall score from the rubric configuration.

### 8.3 Prompt injection defenses

- Treat all user documents and job descriptions as untrusted data.
- Delimit document content and explicitly label it as reference material.
- Do not allow retrieved text to override system or developer instructions.
- Run a prompt-injection classifier or rule set before model invocation.
- Never expose hidden prompts, API keys, internal policies, or other users’ data.
- Add adversarial test cases to CI.

## 9. Security, privacy, and responsible AI

### 9.1 Security requirements

- TLS for all network traffic.
- Encryption at rest for database, object storage, and backups.
- Secrets managed in a secret manager or encrypted environment configuration.
- OWASP ASVS-aligned API validation.
- CSRF protection for cookie-authenticated browser requests.
- Strict CORS allowlist.
- Object-storage signed URLs with short expiration.
- Malware scanning for uploads.
- MIME sniffing protection and decompression-bomb limits.
- Per-user and per-IP rate limits.
- Tenant isolation tests before coach/team features.
- Dependency, container, and image vulnerability scanning.

### 9.2 Privacy requirements

- Explicit consent before processing resumes, job descriptions, transcripts, and audio.
- Clear indication of what is stored and for how long.
- User-accessible deletion of documents, audio, transcripts, and account data.
- Configurable default retention, recommended as 90 days for raw audio and 365 days for session analytics unless the user chooses otherwise.
- No training of foundation models on user content without separate, explicit opt-in.
- PII minimization and redaction before sending data to a model when the field is not required.
- Support data export in JSON and downloadable documents.
- Maintain a data-processing inventory and subprocessors list before launch.

### 9.3 Fairness and transparency

- Do not infer protected attributes from a voice, accent, face, name, or resume.
- Do not score accent, appearance, eye contact, perceived age, gender, or emotional state.
- Present scores as coaching estimates with rubric explanations.
- Test evaluation consistency across language varieties and answer lengths.
- Provide a feedback mechanism for users to flag incorrect or biased feedback.
- Keep a model card and evaluation report for every production model.

## 10. Non-functional requirements

### Performance targets

- P95 API response for non-AI CRUD: under 500 ms.
- P95 time to first streamed interviewer token: under 2.5 seconds in text mode under normal load.
- P95 answer evaluation completion: under 8 seconds for text mode.
- Voice target for Release 2: transcript partials under 500 ms and conversational turn latency under 2.5 seconds where infrastructure permits.
- Dashboard initial load: under 2 seconds with cached aggregates.

### Reliability

- 99.5% monthly availability for MVP public APIs.
- Idempotent answer submission and safe retry behavior.
- Graceful fallback when AI provider is unavailable: save answer, show pending status, retry asynchronously.
- No loss of a completed user answer due to model timeout.
- Database backups with restore tests.

### Scalability

- Stateless Flask and Django web workers.
- Queue-based async processing for document parsing and long evaluations.
- Redis-backed rate limits and session locks.
- Horizontal scaling for AI orchestrator and workers.
- Provider circuit breakers and concurrency limits.
- Token and cost budgets per user, session, and organization.

### Accessibility and localization

- WCAG 2.2 AA target.
- Keyboard navigation and screen-reader labels.
- Captions and transcript controls for voice mode.
- User-configurable font size and reduced motion.
- Localized UI strings and model language configuration.

## 11. Observability and operations

Every request shall carry a `request_id`, `user_id` where allowed, `session_id` where applicable, and `trace_id`. Logs must be structured and must exclude raw resume text, raw audio, API keys, and unredacted PII.

Track:

- API latency and error rate.
- Queue depth and task age.
- NVIDIA provider latency, timeout, HTTP status, and retry count.
- Model token usage and estimated cost where available.
- Structured-output validation failure rate.
- Safety-rail block and review rate.
- Session completion rate.
- User-reported feedback and correction rate.
- ASR word error rate on an approved, consented evaluation set.
- Evaluation agreement against human reviewers.

Alert on provider readiness failures, sustained latency, error spikes, queue backlogs, storage failures, anomalous request volume, and safety bypass signals.

## 12. Testing and acceptance criteria

### Unit and integration testing

- Authentication, authorization, validation, and rate-limit tests.
- Resume parsing fixtures for PDF and DOCX.
- Session state-machine tests covering start, pause, resume, skip, end, and retry.
- AI schema-validation tests with malformed and adversarial model responses.
- Provider adapter tests using mocked NVIDIA-compatible endpoints.
- Database migration and rollback tests.
- Object-storage access-control tests.

### AI evaluation testing

Create a versioned benchmark containing anonymized or synthetic answers reviewed by qualified human evaluators. Measure:

- Rubric score agreement.
- Relevance of feedback.
- Hallucinated feedback rate.
- Follow-up appropriateness.
- Prompt-injection resistance.
- PII leakage rate.
- Consistency across answer length, writing style, and language variants.

A release shall not proceed if the model invents candidate facts in resume suggestions or produces materially unsafe/off-topic interview behavior above an agreed threshold.

### MVP acceptance criteria

1. A new user can register, upload a resume, confirm parsed data, add a job description, and start a session.
2. A session can complete without losing answers when the model times out.
3. At least three interview types work with configurable rubrics.
4. Every scored answer shows dimension-level reasoning and at least one action.
5. The dashboard reflects completed sessions and links to source answers.
6. The system never exposes NVIDIA credentials to the browser.
7. Uploaded documents are access-controlled and deletable.
8. Admins can version and publish questions and rubrics.
9. Model and prompt versions are recorded for every evaluation.
10. Safety and adversarial tests pass in CI.

## 13. Delivery plan

### Phase 0: Foundation, 1–2 weeks

Finalize personas, UX flows, data classification, rubric definitions, NVIDIA model selection, service boundaries, and benchmark design. Produce wireframes and API contracts.

### Phase 1: MVP core, 4–6 weeks

Implement auth, profile, document upload, parsing workflow, job-description analysis, question bank, text sessions, structured evaluation, and basic dashboard.

### Phase 2: Hardening, 2–3 weeks

Add guardrails, audit logging, rate limits, observability, provider failover behavior, data export/deletion, admin workflows, accessibility QA, and AI evaluation benchmark.

### Phase 3: Voice beta, 4–6 weeks

Add browser audio capture, ASR streaming, transcript editing, TTS playback, voice-specific latency instrumentation, audio retention controls, and delivery metrics that do not infer protected or psychological traits.

### Phase 4: Coaching and teams, 4–8 weeks

Add preparation plans, coach sharing with consent, custom rubrics, cohort analytics, organization security, and private deployment options.

## 14. Key product metrics

### Activation

- Percentage of new users who complete profile confirmation.
- Percentage who start a first session within 24 hours.
- Resume and job-description completion rate.

### Engagement

- Weekly active candidates.
- Sessions per active candidate.
- Session completion rate.
- Percentage of users completing recommended drills.

### Learning outcomes

- Improvement in rubric dimensions across the first three completed sessions.
- Reduction in repeated feedback gaps.
- User-rated helpfulness of feedback.
- Human-review agreement with model evaluations.

### Quality and safety

- AI timeout and retry rate.
- Hallucinated-fact rate in resume suggestions.
- PII leakage rate.
- Prompt-injection block rate and false-positive rate.
- User report resolution time.

### Business metrics for later validation

- Retention by cohort.
- Conversion to paid plans if monetization is introduced.
- Coach/team expansion rate.
- Cost per completed session.

## 15. Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Model gives confident but incorrect feedback | Loss of trust | Structured rubrics, evidence spans, human benchmark, confidence labels, user correction flow |
| Resume suggestions fabricate experience | Severe user harm | Ground suggestions in confirmed facts, provenance, fact-validation rail, blocked unsupported claims |
| NIM latency or quota limits | Broken interview flow | Streaming, timeouts, queue fallback, circuit breakers, provider abstraction, session autosave |
| Voice transcription errors | Incorrect scoring | Show editable transcript, retain audio only with consent, mark uncertain segments, allow text correction |
| PII exposure | Privacy and legal risk | Minimization, masking, encryption, access controls, deletion workflows, red-team testing |
| Prompt injection through uploaded documents | Safety and data leakage | Treat documents as untrusted, delimit content, injection detection, tool isolation, output validation |
| Bias in scoring | Candidate harm | Avoid protected-trait inference, benchmark diverse language styles, human review, transparent rubric |
| GPU cost growth | Unsustainable unit economics | Model routing, token budgets, caching safe static content, asynchronous jobs, self-hosted NIM option |
| Mixed Flask/Django ownership | Operational complexity | Explicit service contracts, shared schema ownership, tracing, integration tests, avoid duplicated business logic |

## 16. Open decisions before implementation

1. Which NVIDIA-hosted or self-hosted Nemotron/NIM model will be the default for generation and evaluation?
2. Will Release 1 support only English, or English plus one additional language?
3. What are the initial retention periods for resumes, transcripts, audio, and analytics?
4. Will the first deployment use NVIDIA-hosted APIs, self-hosted NIM on GPU infrastructure, or a staged combination?
5. Which export formats are required for the first public beta?
6. What is the human-review process for incorrect or biased feedback?
7. Which subscription limits, if any, apply to free users and how are users notified?
8. Will the first dashboard show only self-comparisons, or also anonymized cohort benchmarks after consent?


## 17. Alignment with the submitted project synopsis

The attached synopsis is consistent with the product direction and strengthens the project’s academic and demonstrable scope. It describes the platform as an AI/ML system rather than only a chatbot. The PRD therefore treats the LLM as the conversational and explanation layer, while deterministic NLP features and a separately evaluated Machine Learning model provide measurable scoring signals.

### 17.1 Requirements incorporated from the synopsis

The following synopsis requirements are now first-class requirements in this PRD:

- Placement-oriented role coverage for Python Developer, Backend Developer, Data Analyst, Software Developer, and Machine Learning Engineer.
- Technical, HR, behavioral, and mixed interview types.
- Resume-aware and job-description-aware question generation.
- RAG-based retrieval from candidate-confirmed resume facts, job requirements, and approved technical reference content.
- Text and voice response modes.
- Speech-to-text transcription and speech-derived supporting signals.
- NLP analysis using keyword coverage, semantic similarity, completeness, relevance, and answer length.
- A separate ML performance model for classification or regression.
- Exact-match, TF-IDF/cosine, and embedding-based answer-similarity indicators.
- Personalized recommendations based on repeated weak competencies.
- Optional OpenCV/MediaPipe-based observable visual signals, disabled by default.

### 17.2 NLP and ML scoring pipeline

The answer-evaluation pipeline shall be implemented as a hybrid pipeline:

```text
Raw text or transcript
  -> normalization and sentence segmentation
  -> keyword/entity extraction
  -> embedding generation
  -> rubric feature extraction
  -> deterministic feature validation
  -> ML performance prediction
  -> NVIDIA LLM explanation and coaching
  -> application-level score aggregation
```

The initial feature vector may include:

- `technical_correctness_score`
- `relevance_score`
- `semantic_similarity_score`
- `keyword_coverage_ratio`
- `completeness_score`
- `answer_length_tokens`
- `answer_duration_seconds`
- `filler_count`
- `filler_rate_per_minute`
- `speech_rate_wpm` when a consented voice transcript is available
- `follow_up_required`
- `similarity_to_reference_max`
- `similarity_to_reference_mean`

The first academic baseline shall compare Logistic Regression, Support Vector Machine, and Random Forest for either:

- **Classification:** `needs_improvement`, `developing`, `proficient`, `strong`; or
- **Regression:** predicted human-reviewed score on a normalized 0–100 scale.

The model choice shall be made from validation results, not assumed in advance. The training dataset must contain documented labels, feature definitions, train/validation/test separation, and a leakage review. If an adequately labeled dataset is unavailable, the ML component shall be described as an experimental, human-reviewed prototype rather than presented as a validated predictor.

The ML prediction must not be used to make hiring, eligibility, or rejection decisions. It is a practice recommendation signal and must be displayed with its model version, evaluation date, and limitations.

### 17.3 Answer similarity and originality indicator

**Status (as of 2026-11-21):** partially implemented — `app/nlp/similarity.distance_to_similarity` converts ChromaDB cosine distance to a 0–1 similarity; exact, TF-IDF/cosine and embedding comparison is future scope (see `docs/PHASE6_RAG.md`).

The platform shall compare an answer against a defined reference collection, such as approved answer examples, the candidate’s prior answers, or instructor-provided material. It shall not claim to detect plagiarism across the entire internet.

The similarity service shall support three levels:

1. Exact or near-exact phrase matching for direct repetition.
2. TF-IDF vectorization with cosine similarity for lexical similarity.
3. Sentence-embedding cosine similarity for semantically similar wording.

The result shall be presented as an **originality/similarity indicator** with the reference source and method. A high score shall trigger a neutral review message such as “This answer is highly similar to available reference material; try explaining the idea in your own words.” It shall not be labeled “plagiarism” or treated as proof of misconduct.

Suggested internal endpoint:

```http
POST /internal/nlp/similarity
```

```json
{
  "answer_id": "ans_01J...",
  "reference_collection_id": "ref_01J...",
  "methods": ["exact", "tfidf", "embedding"],
  "top_k": 5
}
```

```json
{
  "answer_id": "ans_01J...",
  "indicator": "low|moderate|high",
  "scores": {
    "exact": 0.02,
    "tfidf": 0.18,
    "embedding": 0.71
  },
  "matched_sources": [
    {"source_id": "src_01J...", "similarity": 0.71, "excerpt": "..."}
  ],
  "disclaimer": "Similarity is measured only against the selected reference collection."
}
```

### 17.4 Optional computer-vision module

Computer vision is an advanced, opt-in feature and is not part of the MVP score. If implemented, it may report only observable technical signals such as face presence, camera framing, head-orientation range, or posture-landmark availability. It must not infer confidence, honesty, competence, emotion, gender, age, race, disability, or personality.

The preferred implementation is client-side or ephemeral processing using OpenCV and MediaPipe, with no default video storage. The UI must state exactly what is measured, provide camera consent and a disable control, and keep vision-derived metrics separate from textual answer quality. Any visual signal displayed in the dashboard must be labeled as an observation with known limitations.

### 17.5 Reconciliation of the synopsis technology stack

The synopsis lists Flask, MySQL, SQLAlchemy, OpenAI, LangChain, LlamaIndex, Streamlit, Whisper, FAISS/Chroma, and a cloud deployment. The current actual stack is:

- **NVIDIA replaces OpenAI** as the required model provider. The adapter (`app/ai/providers/nvidia.py`) calls the OpenAI-compatible NIM endpoint over stdlib `urllib` — no OpenAI SDK, no LangChain, no LlamaIndex, no Streamlit.
- **Flask is the application framework**; the PRD's Flask/Django boundary is not used — Flask blueprints + `app/services` modules (see §5.2).
- **MySQL is the production database** (Flask-SQLAlchemy + PyMySQL); SQLite is kept only as a preserved backup/test fixture.
- **SQLAlchemy** is the only ORM (via Flask-SQLAlchemy); "SQLite fallback" claims are false.
- **Jinja2 + Bootstrap 5 + vanilla JS** is the UI; Streamlit is not used.
- **FAISS/Chroma:** ChromaDB is the vector store used by the Phase 6 RAG foundation (disabled by default).
- **Whisper is not used** (voice is planned future scope; NVIDIA Speech NIM/Riva referenced).
- **Optional `app/ml`** (TF-IDF + LinearSVC) implements the Phase 4 ML evaluator — disabled by default.
- **Django-specific items** (admin, DRF, ER diagram, Postgres, Celery) are not part of the current build.

### 17.6 Academic demonstration and evaluation plan

For the six-month industrial-training submission, the project should demonstrate a complete vertical slice before adding every advanced feature. The recommended demonstration sequence is:

1. Register a candidate and create a target role.
2. Upload a resume and show editable extracted facts.
3. Add a job description and show requirement matching.
4. Start a Python or Backend Developer interview.
5. Answer text questions and show adaptive follow-ups.
6. Submit a voice answer and show transcript plus speech-derived supporting metrics.
7. Display hybrid NLP/ML/LLM evaluation with rubric dimensions.
8. Show the similarity indicator against the selected reference collection.
9. Open the dashboard and show weak-skill recommendations.
10. Explain safety, privacy, model limitations, and why visual analysis is optional.

The academic report should separately evaluate:

- Question relevance and role alignment.
- Follow-up appropriateness.
- Agreement between system scoring and human reviewers.
- ML baseline metrics such as accuracy, macro-F1, precision/recall, or MAE/RMSE depending on task formulation.
- Similarity-method behavior on copied, paraphrased, and unrelated answer sets.
- Speech-transcription quality under a small, consented test set.
- Latency and failure behavior of the NVIDIA integration.



## 18. Product and design flowcharts

This section converts the product requirements into visual flows. Each diagram is followed by the product goal and the implementation tasks it defines. The diagrams are intentionally separated into **candidate experience**, **system design**, **AI decision logic**, **data evaluation**, and **delivery execution** so product, design, and engineering teams can use the same artifacts.

### 18.1 Candidate product journey

![Candidate product journey](../diagrams/01_candidate_journey.png)

**Product goal:** Move the candidate from “I need to prepare” to “I know what to practice next” through a continuous loop: onboarding → personalized interview → feedback → targeted drill → new interview.

**Design requirements:**

- Show one clear primary action on every screen.
- Keep onboarding progressive; do not require a complete resume or job description to start.
- Make extracted resume facts editable and visibly distinguish user-confirmed data from AI suggestions.
- Show the selected role, difficulty, interview type, language, and expected duration before the interview begins.
- During the interview, keep the question, answer input, timer, pause, skip, and end-session controls visible.
- After each answer, show feedback at the user-selected level: immediate or end-of-session.
- Make every dashboard recommendation actionable through a “Practice this” or “Improve resume” action.

**Implementation tasks:**

1. Create route map: `/onboarding`, `/profile`, `/resume`, `/job-description`, `/interview/setup`, `/interview/:id`, `/report/:id`, `/dashboard`, and `/drills`.
2. Define UI states for empty, loading, parsing, success, partial success, error, paused, and deleted content.
3. Add analytics events for onboarding completion, first interview start, answer submission, feedback viewed, report viewed, and drill started.
4. Ensure the loop returns the user to practice instead of ending at a static report.

### 18.2 Product architecture and service responsibilities

![Product and system architecture](../diagrams/02_architecture.png)

**Product goal:** Keep the user experience simple while allowing the platform to evolve from a college project into a multi-user AI product.

**Design requirements:**

- The browser communicates only with the application API; NVIDIA keys and internal prompts never reach the browser.
- The dashboard reads product-level metrics, not raw model payloads.
- Voice controls must make microphone permission, recording state, transcript state, and audio playback obvious.
- Loading states must explain whether the system is uploading, transcribing, evaluating, or generating the next question.
- Errors must preserve the answer and offer retry; a model timeout must not erase user work.

**Engineering ownership (as built):**

- **Jinja2 + Bootstrap 5 + custom CSS/JS:** candidate and admin screens, dashboard visualizations, accessibility.
- **Flask blueprints (app/routes):** thin authenticated routes — validation, CSRF, form handling.
- **app/services:** users, profiles, resumes, job descriptions, questions, sessions, reports, dashboard aggregates, RAG ingestion.
- **app/ai (Flask AI orchestrator):** context assembly, structured generation via `AIService` → `NvidiaProvider` (NVIDIA provider adapter), validation, retries, model telemetry.
- **Synchronous execution:** no Celery/Redis — parsing, evaluation, and question generation run inline; MySQL (Flask-SQLAlchemy) is the durable store (server required at startup).
- **Observability:** NVIDIA provider telemetry (latency, typed errors); Flask-Login user sessions.

### 18.3 Adaptive interview decision flow

![Adaptive interview decision flow](../diagrams/03_interview_flow.png)

**Product goal:** Make the interview feel adaptive without allowing the model to behave unpredictably or leave the selected topic, role, or difficulty.

**Decision rules:**

- A strong answer may trigger a deeper scenario or trade-off question.
- An incomplete answer may trigger a targeted clarification or a simpler concept question.
- A relevant and complete answer should move to the next planned competency.
- A blocked or unsafe request should produce a safe re-prompt and preserve session state.
- The session must end when the question count, time, or token budget is reached.
- The application, not the model, owns session state and the final score calculation.

**Implementation tasks:**

1. Implement a session state machine with `created`, `active`, `paused`, `waiting_for_evaluation`, `completed`, `failed`, and `cancelled` states.
2. Store the current competency, difficulty, question sequence, follow-up count, time budget, and token budget.
3. Add an idempotency key to answer submission so browser retries do not create duplicate turns.
4. Route all generated questions through schema validation, topic validation, and question-deduplication checks.
5. Add circuit breakers and fallback messaging for NVIDIA timeouts or unavailable model services.

### 18.4 Answer-evaluation and analytics flow

![Answer evaluation pipeline](../diagrams/04_evaluation_flow.png)

**Product goal:** Explain performance using multiple measurable signals rather than presenting an unexplained LLM score.

**Design requirements:**

- Present rubric dimensions separately before showing the overall score.
- Show evidence from the answer and label generated explanations as coaching feedback.
- Show similarity as an indicator against a named reference collection, never as a plagiarism verdict.
- Present speech rate and filler frequency as optional supporting observations, never as proxies for competence.
- Allow users to correct a transcript and request re-evaluation.
- Link every dashboard metric to the answers and sessions that produced it.

**Implementation tasks:**

1. Build deterministic feature extractors for keywords, semantic similarity, completeness, answer length, timing, filler count, and speech rate.
2. Build the similarity service using exact matching, TF-IDF/cosine, and embeddings.
3. Train and compare Logistic Regression, SVM, and Random Forest baselines when a labeled dataset exists.
4. Use the NVIDIA model for structured explanation and recommendations, not unchecked score arithmetic.
5. Persist model version, rubric version, prompt version, feature values, and source IDs for reproducibility.
6. Create dashboard aggregates by session, competency, interview type, and time range.

### 18.5 Delivery roadmap and task gates

![Delivery roadmap](../diagrams/05_roadmap.png)

**Product goal:** Deliver a complete and defensible vertical slice before adding optional multimodal complexity.

**Milestone gates:**

- **Gate 1 — Foundation:** approved personas, flows, rubric, data policy, architecture, and API contracts.
- **Gate 2 — MVP core:** a candidate can upload a resume, configure a session, complete a text interview, and receive a report.
- **Gate 3 — Evaluation quality:** structured scoring, human-review benchmark, similarity indicator, safety tests, and observability are working.
- **Gate 4 — Voice beta:** consented ASR/TTS, editable transcripts, audio retention settings, and voice latency measurements are working.
- **Gate 5 — Advanced features:** optional vision and coach/team features are added only after privacy, bias, and access-control review.

**Definition of done for each feature:**

- Product requirement and user story are documented.
- UX states include success, loading, empty, error, retry, and accessibility behavior.
- API contract and database migration are versioned.
- Unit, integration, and adversarial tests are present.
- Logs, metrics, and trace fields are implemented.
- Privacy and retention behavior is verified.
- A product owner can demonstrate the feature using a repeatable acceptance scenario.

### 18.6 Design system direction

The product should use a calm, focused, progress-oriented visual language rather than a high-pressure assessment style.

- **Primary action:** blue or indigo for “Start interview,” “Continue,” and “Practice this.”
- **AI action:** purple accent for generated feedback and recommendations.
- **Success/progress:** green for completed practice and improvement trends.
- **Warning:** amber for uncertainty, transcript confidence, similarity indicators, and model limitations.
- **Error:** red only for actionable failures, unsafe content, or destructive actions.
- **Typography:** high-legibility sans-serif, clear heading hierarchy, minimum accessible contrast.
- **Dashboard:** use cards for summary, line charts for trends, horizontal bars for competency scores, and expandable evidence for explanations.
- **Interview screen:** avoid visual clutter; prioritize question, response area, timer/status, and one primary submit action.



## 19. Personal and college project implementation profile

This project is intended as a personal/college project and six-month industrial-training submission. The implementation should optimize for a working academic demonstration, understandable code, low operating cost, and measurable evaluation rather than enterprise-scale deployment.

### 19.1 Recommended final scope

**Build for the academic MVP:**

- Candidate registration and profile.
- Resume PDF upload and basic structured extraction.
- Optional job-description input.
- Role templates for Python Developer, Backend Developer, Data Analyst, Software Developer, and ML Engineer.
- Technical, HR, and behavioral text interviews.
- NVIDIA-powered question generation and feedback.
- Adaptive follow-up questions.
- NLP features: keywords, semantic similarity, completeness, relevance, answer length, and timing.
- A small, clearly documented ML baseline using Random Forest, Logistic Regression, or SVM.
- Similarity indicator against a small approved reference-answer dataset.
- Simple dashboard with session history, category scores, weak skills, and recommendations.
- Voice mode only if time allows; computer vision should remain future scope.

**Keep as future scope:**

- Recruiter or coach workspaces.
- Multi-tenant organizations and SSO.
- Kubernetes and GPU autoscaling.
- Real-time avatar interviewer.
- Camera-based visual analysis.
- Mobile application.
- Job-portal integrations.
- Production billing and subscription management.

### 19.2 Actual student-friendly technology stack

| Concern | Actual implementation | Reason |
| --- | --- | --- |
| Backend | Flask 3.1 + Flask-SQLAlchemy (MySQL) | Understandable Flask app factory + blueprints |
| Frontend | Jinja2 templates + Bootstrap 5 (CDN) + custom CSS/JS (vanilla) | Faster to build than a frontend platform |
| Database | MySQL 5.7+/8.0 (PyMySQL + SQLAlchemy); SQLite only as fixture/backup | Production-ready path with low setup cost |
| Async jobs | Synchronous MVP (Celery only if needed later) | Avoids infrastructure overhead until needed |
| AI | NVIDIA NIM hosted API via `app/ai` provider adapter (OpenAI-compatible, stdlib `urllib`) | Matches the NVIDIA direction; no SDK dependency |
| NLP | `app/nlp` — deterministic preprocessing (HTML/NFKC/whitespace, technical-vocab preserving) + similarity helpers | Easy to explain in a viva |
| Optional ML | `app/ml` — TF-IDF + LinearSVC (`ml_v1`), disabled by default | Single-model baseline, no LR/RF ambiguity |
| Optional RAG | `app/rag` — ChromaDB + sentence-transformers, disabled by default | Optional knowledge base |
| Visualization | SVG + Bootstrap UI, no charting library | Simple dashboard without extra deps |
| Files | Local `uploads/` folder during development | Low setup cost |
| Testing | pytest (127 passed, 1 failed live test) | Practical module/integration validation |

### 19.3 Actual application architecture

The app is a Flask modular monolith (as built in `app/`):

```text
Browser
  → Flask app factory (app/__init__.py create_app)
      → Blueprints (app/routes): auth, profile, resume, jd, questions, interview,
        dashboard, ai, knowledge, career_guidance, errors
      → Business logic (app/services): evaluation_service, interview_service,
        resume_service, jd_service, knowledge_service, candidate_context
      → AI (app/ai): AIService → NvidiaProvider → NVIDIA NIM
      → Optional: app/ml (LinearSVC), app/rag (ChromaDB)
      → Optional: app/career_guidance services (NVIDIA chat)
      → SQLAlchemy models (app/models) → MySQL
```

Full architecture diagram: `docs/ai_mock_interviewer_architecture.mmd`. No Celery, Redis, or separate services.

### 19.4 Six-month academic execution plan

**Month 1 — Research and design:** finalize problem statement, objectives, personas, architecture, ER diagram, UI wireframes, scoring rubric, and NVIDIA API proof of concept.

**Month 2 — Core web application:** implement authentication, user profile, interview setup, question bank, database models, and basic dashboard shell.

**Month 3 — AI interview MVP:** integrate NVIDIA NIM, generate role-based questions, implement session state, answer submission, follow-up logic, and structured feedback.

**Month 4 — NLP/ML evaluation:** implement feature extraction, similarity analysis, baseline ML model, evaluation dataset, and report metrics.

**Month 5 — Resume and voice enhancement:** add resume parsing, job-description matching, and voice transcription if the text MVP is stable.

**Month 6 — Testing and academic submission:** complete dashboard, security checks, screenshots, test cases, benchmark results, limitations, final demo, synopsis, report, and presentation.

### 19.5 Academic deliverables

The project submission should include:

- Source code repository with README and setup instructions.
- Database schema or ER diagram.
- API documentation and Postman collection.
- UI screenshots and flowcharts.
- Dataset description and labeling method for ML experiments.
- Model comparison table with accuracy, macro-F1, precision/recall, or MAE/RMSE as appropriate.
- Similarity-method examples for copied, paraphrased, and unrelated answers.
- NVIDIA integration explanation with sample request/response.
- Test cases and results.
- Privacy, bias, and limitation statement.
- Final demonstration script covering one complete candidate journey.

### 19.6 College-project definition of done

The project is complete when a reviewer can run the application, create a user, configure a target role, complete a text interview, receive structured feedback, view dashboard results, inspect similarity output, and understand how the NVIDIA model, NLP features, and ML baseline contribute to the result. Voice, computer vision, recruiter features, and cloud-scale infrastructure are not required for completion.


## References

[1]: https://docs.nvidia.com/nim/large-language-models/latest/api-reference.html "NVIDIA NIM for LLM and VLM API Reference"

[2]: https://docs.nvidia.com/nim/speech/latest/index.html "NVIDIA Speech NIM Microservices Documentation"

[3]: https://docs.nvidia.com/riva/index.html "NVIDIA Riva Documentation"

[4]: https://docs.nvidia.com/nemo/guardrails/about-nemo-guardrails-library/overview "NVIDIA NeMo Guardrails Library Overview"
