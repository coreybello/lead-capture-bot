Lead Capture & Instant Response Bot (Agentic Workflow)
LLM-powered FastAPI service that replies to new inquiries in seconds, answers FAQs in your brand tone, inserts a booking link, and logs an audit trail to Postgres. Owner gets email/SMS alerts. Ships as a small Docker stack.

✨ Features (MVP)
Intake: HTTP webhook or Gmail/IMAP poller

Brains: LLM reply using your top 10 FAQs + tone + safe fallback

Conversion: Auto-insert Calendly or Google Appointment link

Delivery: Email via SendGrid, owner SMS via Twilio

Reliability: Celery + Redis for async send/retries; dedupe & idempotency

Observability: Full audit logs in Postgres; /healthz, /readyz, /metrics

Tuning: 14-day tweak window for FAQs, tone, and templates (no redeploy)

Stack: Python, FastAPI, Postgres, Celery, Redis, OpenAI API, SendGrid, Twilio, Docker

🚀 Quick Start (5 minutes)
Clone & env

bash
Copy
Edit
git clone https://github.com/your-org/lead-capture-instant-response-bot.git
cd lead-capture-instant-response-bot
cp .env.example .env          # fill in keys below
Bring it up

bash
Copy
Edit
docker compose up -d --build
# First run DB migrations (inside the api container)
docker compose exec api alembic upgrade head
Smoke test

bash
Copy
Edit
# Health
curl -s http://localhost:8000/healthz

# Send a sample lead
curl -X POST http://localhost:8000/v1/intake/webhook \
  -H "Authorization: Bearer $ADMIN_BEARER_TOKEN" -H "Content-Type: application/json" \
  -d '{
    "event_id":"evt_123",
    "email":"prospect@example.com",
    "first_name":"Jordan",
    "subject":"Do you integrate with HubSpot?",
    "message":"Curious about price and timeline."
  }'
Open docs
Visit http://localhost:8000/docs (Swagger UI)

⚙️ Configuration
Create a .env with at least:

env
Copy
Edit
# Core
DB_URL=postgresql+psycopg://postgres:postgres@db:5432/leadbot
REDIS_URL=redis://redis:6379/0
ADMIN_BEARER_TOKEN=change-me-very-secret

# Providers
OPENAI_API_KEY=sk-...
SENDGRID_API_KEY=SG....
OWNER_ALERT_EMAIL=owner@example.com

TWILIO_ACCOUNT_SID=ACxxxxxxxx
TWILIO_AUTH_TOKEN=xxxxxxxx
TWILIO_FROM_NUMBER=+15551234567
OWNER_ALERT_SMS_TO=+15557654321

# Booking
BOOKING_PROVIDER=calendly           # calendly | gcal
BOOKING_LINK=https://calendly.com/your-link/intro-call

# Intake: Gmail/IMAP poller (optional)
POLLER_ENABLED=false                 # set true to enable
POLLER_INTERVAL_SECONDS=60
IMAP_HOST=imap.gmail.com
IMAP_PORT=993
IMAP_USERNAME=inbox@example.com
IMAP_PASSWORD=app-specific-password
# OR Gmail OAuth if you wire it:
# GMAIL_CLIENT_ID=...
# GMAIL_CLIENT_SECRET=...
# GMAIL_REFRESH_TOKEN=...
# GMAIL_EMAIL=inbox@example.com

# Behavior
DEDUPE_WINDOW_MIN=10
QUIET_HOURS=22:00-07:00             # owner SMS quiet hours (optional)
LOG_LEVEL=INFO
🧱 Docker Compose (starter)
yaml
Copy
Edit
services:
  api:
    build: .
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000
    env_file: .env
    ports: [ "8000:8000" ]
    depends_on: [ db, redis ]
  worker:
    build: .
    command: celery -A app.celery_app worker --loglevel=INFO
    env_file: .env
    depends_on: [ api, db, redis ]
  beat:
    build: .
    command: celery -A app.celery_app beat --loglevel=INFO
    env_file: .env
    depends_on: [ worker ]
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: leadbot
    ports: [ "5432:5432" ]
    volumes: [ "pgdata:/var/lib/postgresql/data" ]
  redis:
    image: redis:7
    ports: [ "6379:6379" ]
volumes:
  pgdata:
First run only: docker compose exec api alembic upgrade head

📡 API Endpoints (MVP)
POST /v1/intake/webhook — receive new lead
Body: {event_id, email, first_name?, last_name?, subject?, message, metadata?}
Auth: Authorization: Bearer <ADMIN_BEARER_TOKEN>

GET /healthz — liveness checks DB/Redis/queue

GET /readyz — readiness (migrations applied)

GET /metrics — Prometheus counters/histograms

Admin (secured):

GET/PUT /v1/admin/settings (booking link, thresholds, quiet hours)

GET/POST/PUT/DELETE /v1/admin/faqs

GET/PUT /v1/admin/tone

GET/POST /v1/admin/templates

POST /v1/admin/rollback (config version)

API docs: OpenAPI/Swagger at /docs

🧠 Reply Logic (high level)
Intake (webhook or poller) → normalize → idempotency check

Dedupe within window (thread & body fingerprint)

Retrieve top-k FAQ snippets (pgvector or in-memory)

LLM generate concise, on-brand answer with booking CTA

Render Jinja template → send via SendGrid

Owner alert via Twilio SMS + email

Persist full audit trail (payloads, prompt, response, receipts)

Guardrails: low temperature, snippet-only policy, safe fallback (“book a call”) if uncertain.

🗃️ Data Model (simplified)
leads (email, names, source)

threads (subject, external thread id)

messages (IN/OUT, body_text/html, provider ids)

outbound (channel, status, attempts, receipts, latency)

events (dedupe, errors, decisions; jsonb payload)

faqs, templates, tone_configs, settings, healthchecks

Migrations via Alembic.

🔍 Observability
/metrics exports: intake_total, dedupe_suppressed_total, llm_latency_ms, send_success_total, e2e_seconds (p50/p95)

Structured logs (JSON), request ids, optional PII redaction

Heartbeats via Celery beat; alert if missed >5m

🔐 Security & Privacy
Bearer token on admin + webhook; serve behind TLS (proxy or gateway)

Never log secrets; optional PII redaction at rest (emails/phones)

GDPR delete by email lookup (simple admin script planned)

☁️ Cloud-Native (no code changes; see /deploy docs)
AWS: API Gateway + Lambda (FastAPI adapter) + RDS (Postgres/pgvector) + SES/SNS + CloudWatch

GCP: Cloud Run + Cloud SQL (Postgres) + Pub/Sub + Cloud Tasks + Cloud Monitoring

🧪 Testing
Unit: dedupe, template rendering, retrieval, prompt composer

Integration: webhook→LLM→SendGrid; IMAP→retry; Twilio alerts

Perf: 100 concurrent leads, P95 < 90s E2E
Run tests:

bash
Copy
Edit
docker compose exec api pytest -q
🗺️ Roadmap (vNext)
Lead scoring/routing, multi-brand support

Human-in-the-loop approve/edit before send

A/B testing for subjects & CTAs

Multilingual replies

Live calendar holds (propose 2–3 times via API)

🤝 Contributing
PRs welcome! Please:

Open an issue describing the change

Include tests and docs updates

Follow conventional commit style where possible

📄 License
MIT — see LICENSE (feel free to change for your org).
