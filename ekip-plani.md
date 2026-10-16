# SENTIA — Ekip Uygulama Rehberi

> Multi-Agent AI Geliştirme Sistemi | Başlangıç: Mart 2026

---

## Kullanılacak Teknolojiler

### Backend
- Python 3.12+
- FastAPI (REST API + WebSocket)
- LangGraph (agent orkestrasyon)
- LangChain (LLM bağlantıları)
- SQLAlchemy + Alembic (ORM + migration)
- Celery (async görevler)
- Pydantic v2
- pytest

### Frontend
- Next.js 15 (App Router) + React 19 + TypeScript
- Tailwind CSS v4 + shadcn/ui
- TanStack Query + Zustand
- Recharts
- Socket.io-client / SSE

### Veritabanı & Servisler
- **Neon** — PostgreSQL (+ pgvector)
- **Upstash** — Redis
- **E2B** — Sandbox execution
- **Railway** — Backend deploy
- **Vercel** — Frontend deploy

### LLM Modelleri
- OpenAI (GPT-4o)
- Anthropic (Claude Opus, Claude Sonnet)

### Güvenlik Araçları
- Semgrep, Bandit, Trivy

### Geliştirme Araçları
- Ruff (Python lint/format)
- ESLint + Prettier (JS/TS)
- pre-commit hooks

---

## Ekip Görev Dağılımı

### Backend Ekibi
**Çalışma alanı:** `backend/app/api/`, `backend/app/models/`, `backend/app/services/`, `backend/app/core/`

- FastAPI uygulama iskeleti (main.py, config.py)
- Neon PostgreSQL bağlantısı (async SQLAlchemy)
- Upstash Redis bağlantısı
- JWT authentication (register, login, refresh)
- SQLAlchemy modelleri (User, Project, Task, AgentLog, AgentArtifact, LlmUsage)
- Alembic migration'ları
- CRUD API'leri (/auth, /projects, /tasks, /agents)
- WebSocket endpoint (real-time streaming)
- Celery worker yapılandırması

### Agent / LLM Ekibi
**Çalışma alanı:** `backend/app/agents/`, `backend/app/core/llm.py`

- LLM Provider Factory (OpenAI + Anthropic, agent bazlı model seçimi)
- LangGraph StateGraph tanımı (supervisor pattern)
- SharedState TypedDict
- BaseAgent sınıfı (loglama, token takibi, retry)
- 8 Agent implementasyonu:
  - CTO / Orchestrator (Claude Opus) — iş dağıtımı, routing
  - NLU / Analyst (GPT-4o) — istek analizi, kabul kriterleri
  - Backend Agent (Claude Sonnet) — API + model kodu üretimi
  - Frontend Agent (GPT-4o) — TSX bileşen + sayfa üretimi
  - Terminal Agent (Claude Sonnet) — E2B ile build/test çalıştırma
  - QA Agent (GPT-4o) — kod review, test önerisi
  - Security Agent (Claude Opus) — Semgrep/Bandit/Trivy tarama
  - Docs Agent (GPT-4o) — README, changelog, sprint özeti
- Agent tool'ları (code_tools, terminal_tools, search_tools, security_tools, doc_tools)
- Agent system prompt'ları (prompts/*.md)
- Feedback loop (QA/Security → CTO → düzeltme, max 3 tur)

### Frontend Ekibi
**Çalışma alanı:** `frontend/`

- Next.js 15 proje kurulumu + shadcn/ui
- Layout (Sidebar, Header, dark/light mode)
- Auth sayfaları (Login, JWT token yönetimi)
- Dashboard ana sayfa (agent kartları, aktif görevler, maliyet özeti)
- Agent Monitoring sayfası (durum, timeline, canlı log)
- Proje Yönetimi (liste, detay, dosya ağacı, artifact viewer)
- Task Board (Kanban görünümü)
- Log Viewer (filtrelenebilir, export)
- Chat arayüzü (istek gönderme, geçmiş)
- Maliyet Dashboard (token kullanımı grafiği)
- WebSocket/SSE entegrasyonu

### DevOps
- Neon, Upstash, E2B, Railway, Vercel hesap kurulumu
- `.env.example` ve secret dağıtımı
- pre-commit hook yapılandırması
- Deploy pipeline'ları

---

## İş Akışı

### Adım 1 — Hazırlık (Herkes)
1. GitHub repo'yu klonla
2. `.env` dosyasını doldur (key'ler güvenli kanaldan paylaşılacak)
3. Python 3.12+ ve Node.js 20+ kurulu olmalı
4. `pip install -e ".[dev]"` (backend) / `npm install` (frontend)
5. Bağlantıları test et (Neon, Upstash, OpenAI, Anthropic)

### Adım 2 — Sözleşme (İlk Hafta)
- Backend ekibi `docs/openapi.yaml` (API spec) taslağını yazar
- Agent ekibi `docs/event-schemas.md` (Redis + WebSocket event formatları) yazar
- Frontend ekibi bu spec'lere göre Prism mock server kurar
- **Bu dosyalar onaylandıktan sonra herkes paralel çalışmaya başlar**

### Adım 3 — Paralel Geliştirme

| Backend Ekibi | Agent Ekibi | Frontend Ekibi |
|---------------|-------------|----------------|
| DB modelleri + migration | LLM Factory | Next.js iskelet + layout |
| Auth API (register/login) | SharedState + BaseAgent | Auth sayfası (mock API) |
| CRUD API'leri | CTO Agent + Graph | Dashboard sayfa iskeletleri |
| WebSocket endpoint | NLU + Backend + Frontend Agent | Agent Monitoring UI |
| Celery worker | Terminal + QA + Security + Docs Agent | Task Board + Log Viewer |
| — | Feedback loop | Chat arayüzü + WebSocket |

### Adım 4 — Entegrasyon
- Frontend, mock server'dan gerçek API'ye geçer
- Agent sistemi API üzerinden test edilir
- Dashboard'dan istek gönder → agentları izle → sonuçları gör (tam akış)

### Adım 5 — Test & Polish
- 5 E2E senaryo testi (yeni modül, repo analizi, MVP, teknik borç, belirsiz istek)
- Güvenlik sertleştirmesi (input validation, rate limiting, prompt injection koruması)
- Performans (caching, query optimization)
- Maliyet tracking doğrulaması

---

## Git Kuralları

```
main → develop → feat/*
```

- `main` ve `develop`'a direkt push **yasak** — PR zorunlu
- Her PR en az 1 review almalı
- Branch isimlendirme: `feat/agent-cto`, `feat/backend-auth`, `feat/frontend-dashboard`
- Commit prefix: `feat:`, `fix:`, `docs:`, `refactor:`

---

## Ortam Değişkenleri (.env)

```
NEON_DATABASE_URL=postgresql+asyncpg://...
UPSTASH_REDIS_URL=rediss://...
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
E2B_API_KEY=e2b_...
JWT_SECRET=...
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=30

# Agent Model Eşleşmeleri
CTO_MODEL=claude-opus-4
NLU_MODEL=gpt-4o
BACKEND_MODEL=claude-sonnet-4
FRONTEND_MODEL=gpt-4o
TERMINAL_MODEL=claude-sonnet-4
QA_MODEL=gpt-4o
SECURITY_MODEL=claude-opus-4
DOCS_MODEL=gpt-4o
```

---

## Proje Klasör Yapısı

```
sentia/
├── .env.example
├── .gitignore
├── .pre-commit-config.yaml
├── README.md
├── docs/
│   ├── openapi.yaml
│   ├── event-schemas.md
│   └── agent-specs.md
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── alembic/versions/
│   └── app/
│       ├── main.py
│       ├── config.py
│       ├── api/routes/         ← Backend Ekibi
│       ├── models/             ← Backend Ekibi
│       ├── schemas/            ← Backend Ekibi
│       ├── services/           ← Backend Ekibi
│       ├── core/               ← Backend + Agent Ekibi (llm.py)
│       ├── agents/             ← Agent Ekibi
│       │   ├── graph.py
│       │   ├── state.py
│       │   ├── base.py
│       │   ├── orchestrator.py
│       │   ├── nlu_analyst.py
│       │   ├── backend_agent.py
│       │   ├── frontend_agent.py
│       │   ├── terminal_agent.py
│       │   ├── qa_agent.py
│       │   ├── security_agent.py
│       │   ├── docs_agent.py
│       │   ├── tools/
│       │   └── prompts/
│       └── workers/
├── frontend/                   ← Frontend Ekibi
│   ├── package.json
│   └── src/
│       ├── app/
│       ├── components/
│       ├── lib/
│       ├── hooks/
│       └── stores/
└── scripts/
    ├── setup-env.sh
    └── seed-db.py
```

---

## Önemli Kurallar

1. **Sözleşme önce** — `openapi.yaml` ve `event-schemas.md` onaylanmadan paralel geliştirme başlamaz
2. **Feedback loop limiti** — Agent döngüsü max 3 iterasyon, sonra "kalan risk" olarak raporlanır
3. **Maliyet takibi** — Her LLM çağrısında token + maliyet kaydedilir
4. **Prompt injection** — Kullanıcı inputu sanitize edilir, system prompt'ta sınırlar net tanımlanır
5. **E2B timeout** — Sandbox komutları max 60 saniye
6. **Secret'lar** — `.env` dosyası repo'ya eklenmez, `.env.example` şablon olarak tutulur
7. **API spec değişikliği** — `docs/` altındaki sözleşme dosyaları değiştiğinde tüm ekiplere bildirim zorunlu

---

## İletişim

- Günlük sync: Discord/Slack
- Kod review: GitHub PR yorumları
- Görev takibi: Linear
- API dokümantasyon: Swagger UI (FastAPI otomatik)
