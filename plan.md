# SENTIA Multi-Agent AI Geliştirme Sistemi — Uygulama Planı

## TL;DR
SENTIA, yazılım geliştirme süreçlerini 8 otonom AI agenta (3 çekirdek + 5 destek) dağıtan bir multi-agent orkestrasyon sistemidir. **Python/FastAPI** backend + **LangGraph** agent orkestrasyon + **Next.js 15** dashboard mimarisi ile sıfırdan inşa edilecek. LLM olarak **OpenAI + Anthropic** (agent bazlı model seçimi) kullanılacak. Altyapı servisleri **cloud** (Neon, Upstash), sandbox **E2B**, deploy **Railway + Vercel** — Docker bağımlılığı yok. Ekip çalışması **contract-first** (OpenAPI spec) + **Git branching** ile paralel yürütülecek.

---

## Mimari Genel Bakış

```
┌──────────────────────────────────────────────────────────┐
│                  DASHBOARD (Next.js 15 → Vercel)          │
│  Agent Monitoring │ Task Board │ Logs │ Chat │ Projects   │
└──────────────────────┬────────────────────────────────────┘
                       │ REST + WebSocket (SSE)
┌──────────────────────▼────────────────────────────────────┐
│           API GATEWAY (FastAPI → Railway)                  │
│  Auth │ Projects │ Tasks │ Agent API │ WebSocket Hub       │
└──────────────────────┬────────────────────────────────────┘
                       │
┌──────────────────────▼────────────────────────────────────┐
│          ORCHESTRATION LAYER (LangGraph)                   │
│                                                            │
│  ┌──────────────┐  ┌─────────────┐  ┌───────────────┐    │
│  │ CTO Agent    │  │ NLU Agent   │  │ Backend Agent │    │
│  │ (Supervisor) │  │ (Analyst)   │  │               │    │
│  │ Claude Opus  │  │ GPT-4o      │  │ Claude Sonnet │    │
│  └──────┬───────┘  └─────────────┘  └───────────────┘    │
│         │                                                  │
│  ┌──────┴───────┐  ┌─────────────┐  ┌───────────────┐    │
│  │Frontend Agent│  │Terminal Agt │  │  QA Agent     │    │
│  │ GPT-4o       │  │Claude Sonnet│  │  GPT-4o       │    │
│  └──────────────┘  └─────────────┘  └───────────────┘    │
│                                                            │
│  ┌──────────────┐  ┌─────────────┐                        │
│  │Security Agent│  │ Doc/PM Agent│                        │
│  │ Claude Opus  │  │ GPT-4o      │                        │
│  └──────────────┘  └─────────────┘                        │
└──────────────────────┬────────────────────────────────────┘
                       │
┌──────────────────────▼────────────────────────────────────┐
│                    DATA LAYER (Cloud)                       │
│  Neon (PostgreSQL) │ Upstash (Redis) │ pgvector            │
└──────────────────────┬────────────────────────────────────┘
                       │
┌──────────────────────▼────────────────────────────────────┐
│              EXECUTION SANDBOX (E2B)                       │
│  İzole container │ Timeout │ Network restricted            │
└───────────────────────────────────────────────────────────┘
```

---

## Framework Seçim Gerekçesi: Neden LangGraph?

SENTIA'nın 8 agentlık yapısı, supervisor pattern, feedback loop'lar ve dual LLM gereksinimleri değerlendirildi.

| Kriter | LangGraph | CrewAI | AutoGen |
|--------|-----------|--------|---------|
| Dual LLM (agent başına farklı model) | ✅ Native | ⚠️ Kısıtlı | ✅ Destekler |
| Supervisor pattern (CTO merkez) | ✅ Native graph node | ⚠️ Hierarchical ama kısıtlı | ⚠️ Conversation paradigması |
| Typed state yönetimi | ✅ Node'lar arası tam kontrol | ⚠️ Task bazlı, sınırlı | ⚠️ Chat history ağırlıklı |
| Feedback loop (QA ↔ Backend) | ✅ Conditional edges | ⚠️ Hack gerekir | ⚠️ Conversation döngüsü |
| Real-time streaming | ✅ astream_events | ✅ Destekler | ⚠️ Kısıtlı |
| Paralel agent execution | ✅ Native | ⚠️ Kısıtlı | ⚠️ Kısıtlı |
| Production olgunluğu | ✅ Yüksek | ⚠️ Orta | ⚠️ Orta |

**Sonuç:** LangGraph — state machine paradigması, SENTIA'nın CTO-merkezli orkestrasyon akışına birebir uyuyor.

---

## LLM Stratejisi

### Agent Bazlı Model Dağılımı

| Agent | Model | Gerekçe | Tahmini Maliyet/1K çağrı |
|-------|-------|---------|--------------------------|
| CTO / Orchestrator | **Claude Opus** | Karmaşık reasoning, planlama, risk analizi | ~$15-25 |
| NLU / Product Analyst | **GPT-4o** | Doğal dil anlama, yapılandırma | ~$5-10 |
| Backend Agent | **Claude Sonnet** | Kod üretimi, mimari kalite | ~$8-15 |
| Frontend / UI Agent | **GPT-4o** | UI/UX kod üretimi | ~$5-10 |
| Terminal / Execution | **Claude Sonnet** | Log analizi, hata çözümleme | ~$8-15 |
| QA / Code Review | **GPT-4o** | Kod kalite bilgisi | ~$5-10 |
| Security / DevSecOps | **Claude Opus** | Derin güvenlik analizi, OWASP | ~$15-25 |
| Documentation / PM | **GPT-4o** | Doküman üretimi, özetleme | ~$5-10 |

### Neden Cloud LLM (Local Değil)?
- CTO ve Security agent'lar Claude Opus seviyesi reasoning gerektiriyor — local model (7B-70B) bu kalitede çıktı veremiyor
- 32B+ local model için RTX 4090 / A100 GPU gerekli — ekipte bu donanım yoksa pratikte çalışmaz
- Aylık gerçekçi kullanımda LLM maliyeti **$20-80** — donanım yatırımından çok daha ekonomik
- LLM çağrıları zaten internet gerektiriyor, "offline çalışma" avantajı yok

### Gelecekte Local Model Seçeneği
LangGraph mimarisi model değişikliğine izin veriyor — ileride Docs agent ve basit NLU görevleri için **Ollama + Qwen2.5-Coder** eklenebilir. `llm.py` factory class'ında tek satır değişiklik yeterli.

---

## Altyapı Stratejisi: Hybrid Cloud (Docker Bağımsız)

### Servis Dağılımı

| Servis | Çözüm | Tier | Maliyet |
|--------|-------|------|---------|
| PostgreSQL | **Neon** (serverless, branch desteği) | Free | $0 |
| Redis | **Upstash** (HTTP tabanlı, serverless) | Free | $0 |
| Vektör DB | **pgvector** (Neon üzerinde) | Free (Neon dahili) | $0 |
| Sandbox (Terminal Agent) | **E2B** (isolated cloud execution) | Hobby | ~$0-10/ay |
| Backend Deploy | **Railway** (git push deploy) | Hobby | $5/ay |
| Frontend Deploy | **Vercel** (Next.js native) | Free | $0 |
| LLM API'leri | OpenAI + Anthropic | Kullanım başına | ~$20-80/ay |

**Toplam başlangıç maliyeti: ~$25-95/ay** (büyük kısmı LLM kullanımı)

### Geliştirme Ortamı (Her Ekip Üyesi)
- `.env` dosyasını doldurur (Neon, Upstash, OpenAI, Anthropic key'leri)
- Backend ekibi: `uvicorn app.main:app --reload` (local Python)
- Frontend ekibi: `next dev` (local Node.js)
- Agent ekibi: Local Python + aynı `.env` ile LLM + DB bağlantısı
- Herkes **aynı cloud DB'ye** bağlanır → senkronizasyon sorunu yok

### Docker Ne Zaman Devreye Girer?
Production deployment ihtiyacı doğduğunda veya self-hosted tercih edildiğinde. Bu plan kapsamında Docker **scope dışında**.

---

## Ekip Çalışma Modeli

### Sorumluluk Dağılımı

| Ekip / Kişi | Alan | Monorepo Klasörü | Ana Teknoloji |
|-------------|------|------------------|---------------|
| **LLM / Agent Ekibi** | LangGraph graph, agent prompts, tool'lar, LLM factory | `backend/app/agents/`, `backend/app/core/llm.py` | LangGraph, LangChain |
| **Backend Ekibi** | FastAPI routes, DB modelleri, auth, Celery, WebSocket | `backend/app/api/`, `backend/app/models/`, `backend/app/services/` | FastAPI, SQLAlchemy |
| **Frontend Ekibi** | Next.js dashboard, bileşenler, WebSocket entegrasyonu | `frontend/` | Next.js, React, shadcn/ui |
| **Infra / DevOps** | Cloud servisleri, env yönetimi, deployment | `.env.example`, `docs/` | Neon, Upstash, Railway |

### Contract-First: API Sözleşmesi (Paralel Çalışmanın Anahtarı)

1. **OpenAPI spec** (`docs/openapi.yaml`) — Backend ekibi yazar, tüm endpoint tanımları
2. **Redis event formatları** — Agent ekibi tanımlar: `agent:status:update`, `task:log:stream`
3. **WebSocket event şeması** — `{type: "agent_status", agentId, status, timestamp, data}`
4. Frontend ekibi **Prism mock server** kullanarak backend hazır olmadan geliştirir

### Git Workflow

```
main                  ← production-ready, korumalı
  └── develop         ← entegrasyon branch'i (tüm PR'lar buraya)
        ├── feat/agent-cto-supervisor       ← Agent ekibi
        ├── feat/agent-backend-agent        ← Agent ekibi
        ├── feat/backend-auth-api           ← Backend ekibi
        ├── feat/backend-websocket          ← Backend ekibi
        ├── feat/frontend-dashboard         ← Frontend ekibi
        └── feat/frontend-agent-monitoring  ← Frontend ekibi
```

**Kurallar:**
- `main` ve `develop`'a direkt push yok — PR zorunlu
- Her PR en az 1 kişi review eder
- `develop` → `main` merge haftalık veya milestone'da
- Commit mesajları: `feat:`, `fix:`, `docs:`, `refactor:` prefix'li (conventional commits)

### Ekip Koordinasyon Araçları

| İhtiyaç | Araç | Alternatif |
|---------|------|-----------|
| Görev takibi | **Linear** | Notion, Jira |
| API dokümantasyon | **Swagger UI** (FastAPI otomatik) | Redocly |
| Mock server (Frontend) | **Prism** | MSW |
| Secret paylaşımı | **`.env.example`** + güvenli kanal | Doppler |
| Kod kalite | **pre-commit hooks** (ruff, eslint, prettier) | CI'da |
| İletişim | **Discord/Slack** + GitHub PR comments | — |

---

## Teknoloji Stack'i (Tam Liste)

### Backend (Agent System)
- **Python 3.12+** — Ana dil
- **FastAPI** — REST API + WebSocket
- **LangGraph** — Multi-agent orkestrasyon, state machine, supervisor pattern
- **LangChain** — LLM provider abstraction (OpenAI + Anthropic + gelecekte Ollama)
- **Celery + Upstash Redis** — Async task queue (uzun süreli agent görevleri)
- **SQLAlchemy + Alembic** — ORM + migration (Neon PostgreSQL'e bağlı)
- **Pydantic v2** — Data validation + settings management
- **python-jose + passlib** — JWT auth + password hashing
- **E2B SDK** — Sandbox execution (Terminal Agent)

### Frontend (Dashboard)
- **Next.js 15** (App Router) + **React 19** + **TypeScript**
- **Tailwind CSS v4** + **shadcn/ui** — Profesyonel UI bileşenleri
- **Socket.io-client** veya native **EventSource (SSE)** — Real-time streaming
- **TanStack Query** — Server state management + caching
- **Zustand** — Client state management
- **Recharts** — Dashboard grafikleri / metrikler

### Veritabanı & Storage (Cloud)
- **Neon** (PostgreSQL 16, serverless) — Ana veri + pgvector (embeddings)
- **Upstash** (Redis 7, HTTP/serverless) — Cache, pub/sub, Celery broker
- **Vercel Blob** veya **Uploadthing** — Dosya depolama (üretilen kodlar, raporlar)

### Güvenlik & Analiz Araçları (Agent Tool)
- **Semgrep** — SAST (statik kod analizi)
- **Bandit** — Python güvenlik taraması
- **ESLint Security** — JS/TS güvenlik kuralları
- **Trivy** — Dependency vulnerability scanning

### Geliştirme Araçları
- **Ruff** — Python linter + formatter
- **ESLint + Prettier** — JS/TS lint + format
- **pre-commit** — Git hook'ları ile otomatik kalite kontrolü
- **pytest** — Backend test framework
- **Vitest** — Frontend test framework

---

## Proje Yapısı (Monorepo)

```
sentia/
├── .env.example                # Tüm env değişkenleri şablon (secret yok)
├── .gitignore
├── .pre-commit-config.yaml
├── README.md
│
├── docs/
│   ├── openapi.yaml            # API sözleşmesi (contract-first)
│   ├── event-schemas.md        # WebSocket + Redis event formatları
│   ├── agent-specs.md          # Agent davranış tanımları
│   ├── api-reference.md
│   └── deployment.md
│
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── alembic/versions/
│   ├── app/
│   │   ├── main.py             # FastAPI app entry, CORS, lifespan
│   │   ├── config.py           # Settings (pydantic-settings)
│   │   │
│   │   ├── api/                # --- BACKEND EKİBİ ---
│   │   │   ├── deps.py
│   │   │   └── routes/
│   │   │       ├── auth.py
│   │   │       ├── projects.py
│   │   │       ├── tasks.py
│   │   │       ├── agents.py
│   │   │       └── ws.py
│   │   │
│   │   ├── models/
│   │   │   ├── user.py
│   │   │   ├── project.py
│   │   │   ├── task.py
│   │   │   ├── agent_log.py
│   │   │   └── agent_artifact.py
│   │   │
│   │   ├── schemas/
│   │   │   ├── user.py
│   │   │   ├── project.py
│   │   │   ├── task.py
│   │   │   └── agent.py
│   │   │
│   │   ├── services/
│   │   │   ├── auth_service.py
│   │   │   ├── project_service.py
│   │   │   └── task_service.py
│   │   │
│   │   ├── core/
│   │   │   ├── database.py     # Neon PostgreSQL bağlantısı
│   │   │   ├── security.py     # JWT token, password hashing
│   │   │   ├── redis.py        # Upstash Redis bağlantısı
│   │   │   └── llm.py          # LLM Provider Factory
│   │   │
│   │   ├── agents/             # --- AGENT EKİBİ ---
│   │   │   ├── base.py
│   │   │   ├── orchestrator.py     # CTO / Supervisor
│   │   │   ├── nlu_analyst.py
│   │   │   ├── backend_agent.py
│   │   │   ├── frontend_agent.py
│   │   │   ├── terminal_agent.py
│   │   │   ├── qa_agent.py
│   │   │   ├── security_agent.py
│   │   │   ├── docs_agent.py
│   │   │   ├── graph.py            # LangGraph StateGraph
│   │   │   ├── state.py            # SharedState TypedDict
│   │   │   ├── tools/
│   │   │   │   ├── code_tools.py
│   │   │   │   ├── terminal_tools.py   # E2B execution
│   │   │   │   ├── search_tools.py     # pgvector semantik arama
│   │   │   │   ├── security_tools.py   # Semgrep, Bandit, Trivy
│   │   │   │   └── doc_tools.py
│   │   │   └── prompts/
│   │   │       ├── cto.md
│   │   │       ├── nlu.md
│   │   │       ├── backend.md
│   │   │       ├── frontend.md
│   │   │       ├── terminal.md
│   │   │       ├── qa.md
│   │   │       ├── security.md
│   │   │       └── docs.md
│   │   │
│   │   └── workers/
│   │       └── celery_app.py
│   │
│   └── tests/
│       ├── conftest.py
│       ├── test_agents/
│       ├── test_api/
│       └── test_services/
│
├── frontend/                   # --- FRONTEND EKİBİ ---
│   ├── package.json
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   └── src/
│       ├── app/
│       │   ├── layout.tsx
│       │   ├── page.tsx            # Dashboard
│       │   ├── login/page.tsx
│       │   ├── projects/
│       │   │   ├── page.tsx
│       │   │   └── [id]/page.tsx
│       │   ├── agents/page.tsx
│       │   ├── logs/page.tsx
│       │   └── chat/page.tsx
│       ├── components/
│       │   ├── ui/                 # shadcn/ui
│       │   ├── agents/
│       │   │   ├── AgentCard.tsx
│       │   │   ├── AgentStatus.tsx
│       │   │   ├── AgentTimeline.tsx
│       │   │   └── AgentGrid.tsx
│       │   ├── tasks/
│       │   │   ├── TaskBoard.tsx
│       │   │   └── TaskDetail.tsx
│       │   ├── projects/
│       │   │   └── ProjectOverview.tsx
│       │   ├── chat/
│       │   │   ├── ChatInput.tsx
│       │   │   └── ChatHistory.tsx
│       │   ├── logs/
│       │   │   ├── LogStream.tsx
│       │   │   └── LogFilter.tsx
│       │   └── layout/
│       │       ├── Sidebar.tsx
│       │       ├── Header.tsx
│       │       └── ThemeToggle.tsx
│       ├── lib/
│       │   ├── api.ts
│       │   ├── socket.ts
│       │   └── utils.ts
│       ├── hooks/
│       │   ├── useAgents.ts
│       │   ├── useTasks.ts
│       │   ├── useProjects.ts
│       │   └── useSocket.ts
│       └── stores/
│           ├── authStore.ts
│           └── agentStore.ts
│
└── scripts/
    ├── setup-env.sh
    └── seed-db.py
```

---

## Veritabanı Şeması

**users** — id (UUID), email (unique), password_hash, full_name, role (admin/user), created_at, updated_at

**projects** — id (UUID), name, description, owner_id (FK→users), status (active/archived/completed), repo_url, created_at, updated_at

**tasks** — id (UUID), project_id (FK→projects), title, description, status (pending/in_progress/review/done/failed), assigned_agent (enum), parent_task_id (FK→tasks, nullable), priority (1-5), input_data (JSONB), output_data (JSONB), created_at, updated_at

**agent_logs** — id (UUID), task_id (FK→tasks), agent_type (enum), action, input_summary, output_summary, llm_model, tokens_used (int), cost_usd (decimal), duration_ms (int), status (success/error), error_message, created_at

**agent_artifacts** — id (UUID), task_id (FK→tasks), agent_type (enum), artifact_type (code/doc/report/config/test), file_path, content (text), language, created_at

**llm_usage** — id (UUID), agent_type, model, input_tokens, output_tokens, cost_usd, created_at

---

## Uygulama Fazları

### Faz 0: Kickoff & Sözleşme
> Hedef: Ekiplerin birbirini beklemeden çalışabilmesini sağlayan ortak zemin.

1. GitHub repo oluştur, monorepo yapısını kur
2. `.env.example` — tüm env değişkenleri (NEON_DATABASE_URL, UPSTASH_REDIS_URL, OPENAI_API_KEY, ANTHROPIC_API_KEY, E2B_API_KEY, JWT_SECRET)
3. OpenAPI spec taslağı (`docs/openapi.yaml`)
4. Redis + WebSocket event formatlarını tanımla (`docs/event-schemas.md`)
5. Git branch stratejisi (main → develop → feat/*)
6. pre-commit hook'ları kur (ruff, eslint, prettier)
7. Neon, Upstash, E2B, Railway, Vercel hesapları oluştur, key'leri güvenli dağıt

**Doğrulama:** Her ekip üyesi Neon ve Upstash'e bağlanabilir. Frontend Prism mock server'ı çalıştırabilir.

---

### Faz 1: Temel Altyapı (Foundation)
> Hedef: Veritabanı, auth, API gateway ve LLM factory.

**1a. Backend iskelet** *(Backend ekibi)*
1. `pyproject.toml` — fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic-settings, python-jose, passlib, celery
2. `config.py` — pydantic-settings ile .env okuma
3. `core/database.py` — Async SQLAlchemy + Neon bağlantısı
4. `core/redis.py` — Upstash Redis bağlantısı
5. `core/security.py` — JWT token, password hash
6. SQLAlchemy modelleri: User, Project, Task, AgentLog, AgentArtifact, LlmUsage
7. Alembic init + ilk migration
8. Auth route'ları: POST /auth/register, /auth/login, /auth/refresh
9. CRUD route'ları: /projects, /tasks
10. `main.py` — FastAPI app, CORS, lifespan
11. WebSocket endpoint: bağlantı yönetimi, room sistemi

**1b. LLM Factory** *(Agent ekibi — paralel)*
1. `core/llm.py` — `get_llm(agent_type) -> BaseChatModel` factory
2. Config'den model eşleşmesi: `CTO_MODEL=claude-opus-4-5`, `BACKEND_MODEL=claude-sonnet-4-5`
3. OpenAI + Anthropic client initialization
4. Token sayımı ve maliyet hesaplama utility

**1c. Frontend iskelet** *(Frontend ekibi — paralel)*
1. Next.js 15 projesi: App Router, TypeScript, Tailwind v4
2. shadcn/ui kurulumu
3. Layout: Sidebar + Header + ThemeToggle (dark/light)
4. API client (`lib/api.ts`): Prism mock server'a bağlı
5. Auth sayfası + JWT Zustand store

**Doğrulama:**
- Backend: Swagger UI'da auth + CRUD çalışır
- Agent: `get_llm('cto').invoke('test')` → yanıt gelir
- Frontend: `next dev` → Login sayfası, mock API isteği atar

---

### Faz 2: Agent Çekirdeği (LangGraph Orchestration)
> Hedef: LangGraph state machine, CTO Supervisor, agent base class.

*(Agent ekibi)*

1. `agents/state.py` — SharedState TypedDict: `project_id, user_request, refined_request, tasks[], current_agent, messages[], artifacts[], iteration_count, max_iterations, status`
2. `agents/base.py` — BaseAgent: `run(state) -> state`, loglama, token takibi, retry (max 2)
3. `agents/orchestrator.py` — CTO / Supervisor:
   - System prompt: `prompts/cto.md`
   - Talep analizi → task listesi → agent yönlendirme
   - `Command(goto="backend_agent")` tarzı routing
4. `agents/graph.py` — LangGraph StateGraph:
   - CTO node (entry point, supervisor)
   - Her agent için node
   - Conditional edges (CTO routing kararına göre)
   - Feedback loop edges (QA → CTO → ilgili agent)
   - `max_iterations` sonsuz döngü koruması
5. Celery worker: graph execution async çalıştırma
6. API endpoints:
   - `POST /api/agents/execute` → Celery task başlat, task_id döndür
   - `GET /api/agents/status/{task_id}` → durum
   - `GET /api/agents/result/{task_id}` → sonuçlar
7. Redis pub/sub: her adımda `agent:status:update` event'i

**Doğrulama:** `POST /agents/execute {"request": "Basit REST API tasarla"}` → CTO task listesi döner, AgentLog'da kayıtlar görünür

---

### Faz 3: Çekirdek Agentlar (Core Agents)
> Hedef: NLU, Backend ve Frontend agentlarını tool'larıyla geliştirmek.

**3a. NLU / Product Analyst Agent** *(paralel)*
- Model: GPT-4o
- Prompt: Belirsizlik tespiti, eksik bilgi sorgulama, kabul kriteri üretimi
- Tool: `requirement_parser`, `acceptance_criteria_generator`
- Çıktı: `{scope, goals, acceptance_criteria[], risks[], clarification_questions[]}`

**3b. Backend Agent** *(paralel)*
- Model: Claude Sonnet
- Prompt: API tasarımı, veri modeli, servis mantığı, migration
- Tool: `code_generator`, `file_writer`, `api_designer`
- Şablonlar: FastAPI route, SQLAlchemy model, Pydantic schema

**3c. Frontend / UI Agent** *(paralel)*
- Model: GPT-4o
- Prompt: shadcn/ui + Tailwind, responsive, component tabanlı, v0.dev kalitesinde
- Tool: `component_generator`, `page_generator`
- Şablonlar: Next.js page, React component, form/tablo/dashboard

**3d.** Graph güncelleme: yeni agentları ekle, CTO routing kuralları

**Doğrulama:** "Randevu modülü oluştur" → NLU → CTO → Backend kodu + Frontend sayfası → AgentArtifact'larda görünür

---

### Faz 4: Destek Agentları (Support Agents)
> Hedef: Terminal, QA, Security, Docs agentları + feedback loop tamamlama.

**4a. Terminal / Execution Agent** *(paralel)*
- Model: Claude Sonnet
- Tool: `e2b_executor` (E2B API — npm run build, pytest, pip install)
- Tool: `log_parser`, `dependency_analyzer`
- Güvenlik: E2B izole, timeout max 60sn, network kısıtlı

**4b. QA / Code Review Agent** *(paralel)*
- Model: GPT-4o
- Tool: `code_reader`, `test_generator`
- Çıktı: `{issues: [{severity, file, line, description, suggestion}], passed: bool}`

**4c. Security / DevSecOps Agent** *(paralel)*
- Model: Claude Opus
- Tool: `semgrep_runner`, `bandit_runner`, `dependency_scanner` (Trivy)
- Çıktı: `{vulnerabilities: [{severity, type, cve, file, fix_suggestion}]}`

**4d. Documentation / PM Agent** *(paralel)*
- Model: GPT-4o
- Tool: `doc_generator`, `changelog_builder`
- Çıktı: README.md, CHANGELOG.md, sprint-summary.md

**4e. Feedback Loop:**
- QA/Security → sorun varsa CTO'ya rapor → CTO ilgili agenta düzeltme gönderir → tekrar kontrol
- `max_iterations=3` — 3 turda çözülmeyen "kalan risk" olarak raporlanır

**Doğrulama:** Tam akış: istek → NLU → CTO → Backend+Frontend → Terminal (build) → QA → Security → Docs → tüm kayıtlar AgentLog ve AgentArtifact'ta

---

### Faz 5: Dashboard (Frontend UI)
> Hedef: Profesyonel yönetim paneli.

*(Frontend ekibi — Faz 1c'den itibaren paralel, mock data ile)*

1. **Dashboard Ana Sayfa** — 8 agent durum kartı grid, aktif görev sayısı, son aktiviteler, LLM maliyet özeti
2. **Agent Monitoring** — Real-time durum (idle/working/error), timeline (adım/süre), canlı log streaming
3. **Proje Yönetimi** — Proje listesi + oluşturma, detay (dosya ağacı, görev listesi, artifact viewer)
4. **Task Board** — Kanban (Pending→In Progress→Review→Done), görev detay modal
5. **Log Viewer** — Filtrelenebilir (agent, severity, tarih), log detay drawer, CSV export
6. **Chat / İstek Girişi** — Chat arayüzü, istek geçmişi, real-time durum takibi
7. **Maliyet Dashboard** — Agent/model bazlı token kullanımı ve maliyet grafikleri (Recharts)
8. **WebSocket Entegrasyonu** — Real-time: agent status, yeni log, task update, artifact oluşturma

**Doğrulama:** Dashboard'dan istek gönder → agentları real-time izle → kodlar + raporlar panelde görünsün

---

### Faz 6: Entegrasyon, Test & Polish
> Hedef: Uçtan uca doğrulama, güvenlik, performans.

1. **E2E Test Senaryoları:**
   - Yeni modül geliştirme (randevu sistemi — tam akış)
   - Mevcut repo analizi (zip/URL upload → agent analiz)
   - Hızlı MVP çıkarma
   - Teknik borç temizliği
   - Belirsiz istek (NLU clarification sorusu sorsun)

2. **Agent Collaboration Refinement** — Feedback döngülerini optimize, edge case routing

3. **Error Handling & Resilience:**
   - LLM API timeout → retry (max 2, exponential backoff)
   - Agent crash → CTO'ya rapor, yeniden ata veya skip
   - E2B timeout → clean fail, kullanıcıya bilgi

4. **Maliyet Kontrol:**
   - Agent başına günlük token limiti
   - Dashboard'a maliyet feed'i
   - Limit aşımı uyarı sistemi

5. **Güvenlik Sertleştirmesi:**
   - Input validation (Pydantic strict mode)
   - JWT expiry + refresh flow
   - CORS whitelist (sadece Vercel domain)
   - Rate limiting
   - Prompt injection koruması (agent input sanitization)

6. **Performans:**
   - Benzer istekler için agent response caching (Redis)
   - DB query optimization (index'ler)
   - WebSocket heartbeat + reconnect

7. **Monitoring:**
   - `/health` endpoint (DB, Redis, LLM API durumu)
   - Agent metrik endpoint
   - Structured JSON logging (agent_type, task_id ile)

**Doğrulama:** 5 senaryo uçtan uca başarılı, güvenlik taraması temiz, ortalama yanıt <30sn

---

## Operasyon Akışı (Tam Döngü)

```
Kullanıcı → Dashboard Chat UI → "Randevu sistemi modülü oluştur"
    │
    ▼
API Gateway → POST /agents/execute → Celery task başlat
    │
    ▼
NLU Agent (GPT-4o) → İsteği analiz et, kabul kriterleri üret
    │
    ▼
CTO Agent (Claude Opus) → İşi görevlere böl + distribute:
    ├── Task 1: Backend API tasarımı → Backend Agent
    ├── Task 2: DB modeli → Backend Agent
    ├── Task 3: Frontend sayfaları → Frontend Agent
    └── Task 4: Dokümantasyon → Docs Agent
    │
    ▼ (paralel)
Backend Agent (Claude Sonnet) → API + model + migration kodu üret
Frontend Agent (GPT-4o) → TSX bileşenler + sayfalar üret
    │
    ▼
Terminal Agent (Claude Sonnet) → E2B'de build test (npm run build, pytest)
    │
    ▼
QA Agent (GPT-4o) → Kod review + test önerisi
    │  ↓ sorun varsa
    │  CTO → düzeltme → QA tekrar (max 3 iterasyon)
    │
    ▼
Security Agent (Claude Opus) → Semgrep + Bandit + Trivy scan
    │  ↓ vulnerability varsa
    │  CTO → düzeltme → Security tekrar scan
    │
    ▼
Docs Agent (GPT-4o) → README + changelog + sprint özeti
    │
    ▼
CTO Agent → Kapanış değerlendirmesi → tüm artifact'lar + rapor
    │
    ▼
Dashboard → Real-time güncellenir: kodlar görünür, raporlar indirilir
```

---

## Kararlar & Kapsam

### Alınan Kararlar
- **LangGraph** — Multi-agent framework (CrewAI ve AutoGen elendi)
- **Hybrid Cloud** — Docker bağımsız; Neon + Upstash + E2B + Railway + Vercel
- **Dual LLM Cloud** — OpenAI + Anthropic, agent bazlı model; local model ileride Ollama ile eklenebilir
- **Monorepo** — `sentia/` tek repo, ekip bazlı klasör alanları
- **Contract-First** — OpenAPI spec + event schema → ekipler paralel çalışır
- **LangGraph Supervisor** — CTO agent merkezi karar verici
- **E2B Sandbox** — Terminal agent için izole execution (Docker yok)
- **Feedback Loop** — QA/Security → CTO → düzeltme, max 3 iterasyon
- **Git Flow** — main → develop → feat/*, PR zorunlu, conventional commits

### Scope İçinde
- 8 agent (CTO, NLU, Backend, Frontend, Terminal, QA, Security, Docs)
- Web dashboard (agent monitoring, task board, log viewer, chat, maliyet)
- LLM maliyet takibi
- E2E test senaryoları (5 senaryo)

### Scope Dışı (Gelecek İterasyonlar)
- Docker/Kubernetes production deployment
- CI/CD pipeline (GitHub Actions)
- Multi-tenant sistemi
- Billing / ücretlendirme
- Local LLM (Ollama) entegrasyonu
- Mobile app
- Git entegrasyonu (otomatik PR açma)

---

## Dikkat Edilecek Kritik Konular

1. **Sonsuz Döngü Koruması** — `max_iterations=3` limiti zorunlu. 3 turda çözülemeyen konu "kalan risk" olarak raporlanmalı.

2. **LLM Maliyet Kontrolü** — Her agent çağrısında token + maliyet kaydı (LlmUsage tablosu). Günlük üst limit ve dashboard uyarısı.

3. **Prompt Injection** — Kullanıcı inputu doğrudan agent prompt'una ekleniyor. Sanitization + system prompt sınırları zorunlu.

4. **E2B Sandbox Limitleri** — 60sn timeout, egress kısıtlı. Büyük `npm install` için cache stratejisi gerekebilir.

5. **Ekip Senkronizasyonu** — `docs/openapi.yaml` ve `docs/event-schemas.md` değişikliklerinde tüm ekiplere bildirim zorunlu. Bu dosyalar versiyonlanmalı.
