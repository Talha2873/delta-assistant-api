# Delta Assistant API

Django backend for an AI consulting assistant — a conversational agent that advises inbound
visitors on their business problem while scoring how qualified the lead is and tracking which
stage of the conversation it has reached.

Powers the chat widget on [delta-platform](https://github.com/Talha2873/delta-platform)
([live](https://delta-frontend-nine.vercel.app)).

---

## The problem

A contact form captures a name and an email and tells you nothing about whether the person on
the other end is worth calling back. Most chat widgets are worse — a canned decision tree that
visitors abandon in two clicks.

I wanted a widget that is actually useful to the visitor: it asks about their business, gives
real advice, and never pitches. The qualification happens as a side effect of a genuinely
helpful conversation rather than as an interrogation. By the time someone has explained their
revenue problem and asked about budget, the system knows more than a form would have captured,
and the visitor got something out of it either way.

---

## How it works

```
React chat widget
      |  POST /api/chat/  { messages: [...] }
      v
Django view (chat_view)
      |
      +-- 1. validate and normalise the message history
      +-- 2. clamp to the last 20 turns
      +-- 3. score the lead from conversation signals
      +-- 4. derive the stage from the score
      +-- 5. inject score + stage into the system prompt
      v
OpenAI Responses API (gpt-4o-mini)
      |
      v
{ reply, lead_score, stage }
```

**Stage-aware prompting.** The assistant's behaviour is steered by appending the current lead
score and stage to the system prompt on every call. The same base persona behaves differently
at `DISCOVERY` than at `CLOSING` without needing separate prompts or a state machine, because
the model is told where in the conversation it is.

**Lead scoring** is deliberately deterministic — keyword signals across the user's turns, each
weighted and capped at 100:

| Signal | Weight |
|---|---|
| Mentions a business, company or startup | 10 |
| Mentions customers, sales or revenue | 20 |
| Mentions price, cost or budget | 15 |
| Expresses urgency (urgent, ASAP, soon) | 20 |
| Expresses intent (start, hire, ready) | 20 |

Scores map to stages: `DISCOVERY` (<30), `DIAGNOSIS` (<60), `SOLUTION` (<80), `CLOSING` (80+).

Using rules rather than the model for scoring is intentional: the score is reproducible, costs
nothing, can't hallucinate a qualified lead, and is trivially auditable when a score looks wrong.

**Stateless by design.** Conversation history lives with the client and arrives with each
request. The server holds no session, which keeps the deployment a single process with nothing
to persist — appropriate for a widget where conversations are short and disposable.

---

## Input handling

The view is the trust boundary and treats the request body as hostile:

- Malformed JSON returns `400` rather than raising
- Every message is checked for shape — `role` must be `user` or `assistant`, `content` must be a non-empty string — and anything else is dropped, so a client cannot inject a `system` turn and rewrite the assistant's instructions
- History is clamped to the last 20 messages, bounding both token spend and prompt-stuffing attempts
- An empty message list after filtering returns `400`
- OpenAI failures are logged and returned as `502` with a generic message; the upstream error is never echoed to the client

---

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Framework | Django 5.2 | Familiar, and the admin comes free when this grows a database |
| AI | OpenAI Responses API, `gpt-4o-mini` | Cheapest model that holds a consulting persona across 20 turns |
| CORS | django-cors-headers | Widget is served from a different origin |
| Config | python-dotenv | Keeps the API key out of the codebase |
| Server | Gunicorn | Production WSGI |

---

## API

### `POST /api/chat/`

**Request**

```json
{
  "messages": [
    { "role": "user", "content": "I run a small e-commerce store and sales are flat." },
    { "role": "assistant", "content": "What does your current conversion rate look like?" },
    { "role": "user", "content": "About 1%. We need to fix this soon." }
  ]
}
```

**Response** `200`

```json
{
  "reply": "A 1% conversion rate on e-commerce suggests...",
  "lead_score": 50,
  "stage": "DIAGNOSIS"
}
```

**Errors**

| Status | Meaning |
|---|---|
| `400` | Malformed JSON, or no valid messages after filtering |
| `405` | Method other than POST |
| `502` | OpenAI unavailable or erroring |
| `500` | Unhandled server error |

---

## Getting started

### Prerequisites

- Python 3.12+
- An OpenAI API key

### Run

```bash
git clone https://github.com/Talha2873/delta-assistant-api.git
cd delta-assistant-api

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env        # add your key
python manage.py migrate
python manage.py runserver
```

Test it:

```bash
curl -X POST http://localhost:8000/api/chat/ \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"I run a startup and need more leads."}]}'
```

---

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `OPENAI_API_KEY` | yes | OpenAI API key |
| `DJANGO_SECRET_KEY` | yes | Django signing key — generate a fresh one per deployment |
| `DEBUG` | no | Defaults to `False`; only `True` locally |
| `ALLOWED_HOSTS` | yes | Comma-separated hostnames |
| `CORS_ALLOWED_ORIGINS` | yes | Comma-separated front-end origins |

---

## Project structure

```
Backend/
├── Backend/            settings, urls, wsgi/asgi
├── DeltaAssistant/
│   ├── views.py        chat_view — validation, scoring, staging, OpenAI call
│   └── urls.py         /api/chat/
└── manage.py
```

---

## Known limitations

- **Conversations are not persisted.** Lead scores are computed and returned but never stored, so there is no record of who was qualified. This is the main thing standing between this and something a business could actually run on.
- **Lead scoring is English keyword matching** and will misread paraphrase ("what's the damage?" scores nothing for budget).
- **No rate limiting.** The endpoint is unauthenticated and calls a paid API — it needs throttling before facing real traffic.
- **No tests.**
- **`SECRET_KEY` and `DEBUG` are currently hardcoded** in settings rather than read from the environment.

## Roadmap

- [ ] Move `SECRET_KEY`, `DEBUG` and `ALLOWED_HOSTS` to environment variables
- [ ] Persist conversations and lead scores; add a Django admin view of qualified leads
- [ ] Rate limiting per IP
- [ ] Tests covering message validation, scoring boundaries and the OpenAI failure path
- [ ] Structured tool/function calling so the assistant can book a consultation directly
- [ ] Email notification when a conversation crosses into `CLOSING`
