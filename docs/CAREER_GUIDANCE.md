# Career Guidance Chatbot

An integrated, authenticated career assistant. It is an addition to the existing
application, not a replacement: interview generation, answer evaluation, RAG,
NVIDIA question generation and all existing data are unchanged.

**Status (as of 2026-11-21):** implemented and live — NVIDIA-backed chatbot.
See `docs/ai_mock_interviewer_architecture.mmd` for the architecture and
`docs/ai_mock_interviewer_roadmap.mmd` for the roadmap.

## Architecture

```
app/routes/career_guidance.py          Flask blueprint: auth + JSON API
app/career_guidance/
    services/context_service.py        user-scoped candidate context (reuses build_candidate_context)
    services/prompt_service.py         dedicated system/intent prompts (separate from interview prompts)
    services/chat_service.py           conversation CRUD, ownership gates, AI calls
app/models/career_guidance.py          CareerConversation + CareerMessage
app/templates/career_guidance/chat.html
app/static/css/career_guidance.css
app/static/js/career_guidance.js
migrations/versions/a1b2c3d4e5f6_phase7_career_guidance_chat.py
```

Wiring: blueprint registered in `app/__init__.py`; one import line in
`app/models/__init__.py`; one navigation link in `app/templates/base.html`.

## Request / response flow

1. Browser loads `GET /career-guidance/` (login required). Server renders chat UI with the
   user's conversation list and CSRF token.
2. `POST /career-guidance/conversations` creates an empty conversation.
3. `POST /career-guidance/conversations/<id>/messages` with `{"content": "..."}`:
   - validates length/non-empty;
   - verifies `conversation.user_id == current_user.id`;
   - stores the user message, derives the title from the first user message;
   - builds bounded candidate context plus the last 8 messages;
   - calls the existing NVIDIA `AIService` (no fallback, so failures are surfaced);
   - stores the assistant reply and returns both messages as JSON.
4. `GET /conversations`, `GET /conversations/<id>`, `DELETE /conversations/<id>` support the UI.

Errors are JSON `{"ok": false, "error": {"code":..., "message":...}}` with 400/404/413/502/503.

## NVIDIA provider reuse

`get_ai_service(current_app._get_current_object())` returns the app-bound service created by
`register_ai_service()`. The chatbot calls `service.complete(..., use_fallback=False)` so a
generic deterministic fallback is never presented as career advice. No API key, provider class,
base URL or model configuration is duplicated.

## User context

`context_service.build_chat_context(user_id)` reuses `build_candidate_context()` and adds
bounded role/education/projects/certifications fields. It queries only the authenticated
user's `CandidateProfile`, latest `Resume` and latest `JobDescription`. Missing data is marked
as missing. Context is capped (~6,000 characters) and the full resume text is never passed.
Users can correct profile/resume data through the existing profile and resume review pages;
the chatbot does not write to those records.

## Prompts

`prompt_service.py` owns the system prompt and five capability instruction blocks
(career paths, skill gap, roadmap, projects, interview preparation) plus a deterministic
keyword-based intent selector. Candidate data is rendered in a clearly delimited untrusted
data block. The prompt forbids fabricated credentials, guaranteed outcomes and unsupported
claims about current job openings.

## Conversation management

Two new tables (`career_conversations`, `career_messages`) with:
- `user_id` ownership on every row; every operation filters by `(id, user_id)`;
- UTC timestamps; `role` restricted by a CHECK constraint to `user`/`assistant`;
- cascading message deletion;
- composite indexes `(user_id, updated_at)` and `(conversation_id, id)` only;
- no automatic purging in v1.

Migration: `a1b2c3d4e5f6` (down revision `f1a2b3c4d5e6`), fully reversible. Apply only to a
test/staging database first with `DATABASE_URL` pointing at that database.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/career-guidance/` | Chat page |
| GET | `/career-guidance/conversations` | List own conversations (JSON) |
| POST | `/career-guidance/conversations` | New conversation |
| GET | `/career-guidance/conversations/<id>` | Own conversation + messages |
| DELETE | `/career-guidance/conversations/<id>` | Delete own conversation |
| POST | `/career-guidance/conversations/<id>/messages` | Send message / receive reply |

## Testing

```bash
# MySQL test database (never production)
set TEST_DATABASE_URL=mysql+pymysql://<user>:<password>@localhost:3306/intervia_test?charset=utf8mb4
.venv\Scripts\python.exe -m pytest tests/test_career_guidance.py -q
.venv\Scripts\python.exe -m pytest -q
```

Tests use the existing mocked NIM provider (`FakeOpener` / `make_provider`) and cover auth,
validation, provider failures, context isolation, ownership, follow-up continuity and
regression checks for existing routes, interview setup, baseline evaluation and RAG helpers.

## Known limitations

- Requires a configured NVIDIA NIM model for replies; otherwise a controlled error is shown.
- Skill evidence comes from profile keywords and keyword-extracted resume facts; listed skills
  are not verified proficiency.
- Intent detection is keyword-based, not a learned classifier.
- No web/job-board access: the assistant cannot verify current openings or salaries.
- No automatic retention policy in v1; conversations persist until the user deletes them.
