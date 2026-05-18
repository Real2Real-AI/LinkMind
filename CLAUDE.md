# CLAUDE.md

이 파일은 LinkMind 저장소에서 Claude Code 가 작업할 때 자동 로드되는 가이드라인이다. 매 세션마다 같은 컨텍스트를 반복 설명할 필요 없도록 핵심만 압축해서 둔다.

> 자세한 내용은 `README.md`, `docs/agent_architecture.md`, `docs/training_data_design.md` 참고.

---

## 1. 프로젝트의 진짜 목표

**LinkMind 자체는 수단**이다. 최종 목표는 **사용자가 누적한 데이터로 sVLL(small Vision-Language LLM)을 LoRA 파인튜닝해서 온프레미스 personalized AI 엔진**을 만드는 것. 그 엔진을 **지속적으로 재학습**(continuous training loop)하는 게 장기 비전.

→ LinkMind 의 모든 설계 결정은 "이게 학습 데이터를 보존/구조화/내보내는 데 도움이 되는가?" 라는 질문을 통과해야 한다.

**배포 전략**: self-host 가 우선이고 기본 사용 모드. 장기적으로 OSS (AGPL v3) 공개 + hosted SaaS 옵션 (§14 참조). 단, **사용자 데이터로 운영자의 공통 모델을 학습하는 것은 절대 금지** — personal LoRA 는 "사용자 본인 데이터로 본인 모델만" 이 유일한 형태.

## 2. 데이터 5대 원칙 (절대 위반 금지)

| 원칙 | 의미 | 강제 위치 |
|---|---|---|
| Raw-first | 원본 텍스트/파일 무손실 보존 | `items.raw_content NOT NULL` |
| Provenance | source_type/source_url/source_id/hash 추적 | schema NOT NULL 제약 |
| Idempotent | 동일 자료 중복 저장 금지 | `UNIQUE(source_type, raw_content_hash)` |
| Versioned analysis | 요약/태깅에 model/prompt 버전 기록 | `summary_model`, `embedding_model` 컬럼 |
| Loss-less storage | 이미지/PDF resize/compress 금지 | `attachments.file_hash` 그대로 |

분석 결과(summary, embedding) 는 재생성 가능하지만 raw 가 깨지면 복구 불가. **항상 raw 를 먼저 저장하고 분석은 그 후.**

## 3. 시스템 아키텍처: 단일 self-contained 시스템

LinkMind 는 **self-contained personal AI engine** — backend + agent + UI 를 한 저장소에서 같이 유지. 외부 client agent (openclaw / hermes-agent 등) 에 의존하지 않는다. self-host 한 방에 다 따라온다.

### 모듈 구조

- **`backend/`** — HTTP API (`/ingest`, `/search`, `/ask`, `/graph`), DB, embedding, LLM provider, ingest 모듈
- **`ai_agents/`** — 여러 채널의 inbox/gateway daemon. backend HTTP API 호출. ChannelAgent ABC 로 추상화 (Phase 2.5)
  - 현재: `telegram_inbox_watcher`
  - 단계적 확장 (Phase 3+, 사용자 채널 사용 빈도에 따라): slack, whatsapp, discord 등
- **`frontend/`** — Streamlit MVP (Phase 1-2)
- **`frontend_v2/`** — graph UI (cytoscape.js + vanilla JS + SSE, Phase 2.5+)

모든 모듈은 같은 venv + 같은 Postgres + 같은 Qdrant 공유. 단일 docker compose, 단일 배포 단위.

### 외부 프로젝트는 "벤치마킹 참조" 전용

`external/{openclaw,hermes-agent,hermes-webui}/` 는 gitignored clone. **셋 다 MIT 라이센스** 이므로 AGPL v3 와 호환 — 코드 vendor 가능하다 (단 LICENSE/copyright notice 보존 필수). 다만 실무 권장:

- **부분 코드 vendor** OK — license attribution 보존, 출처 주석 필수 (예: `# Adapted from hermes-agent/agent/skills.py (MIT) — Copyright (c) 2025 Nous Research`)
- **통째 fork** 비권장 — 의존성 무거움, 우리 구조와 안 맞음, 업스트림 추적 부담
- **일반적으로는 idea/UX 패턴 차용 후 자체 구현** — 가장 가볍고 유지보수 쉬움
- **vendor 한 코드는 LinkMind repo 안에 복사** — `external/` 는 gitignored 이고 언제든 삭제될 수 있으므로 source path 로 import 절대 X. 복사 후 LinkMind 코드 트리에서 자족적으로 동작해야.
- **라이센스 우회를 위해 함수명/변수명만 바꾸는 행위 금지** — 법적으로 derivative work 인정됨 (cosmetic 변형은 우회 불가) + MIT 는 attribution 만으로 완전 자유라 우회 자체가 불필요. 정직한 attribution 이 합법 + 안전 + 평판.

→ "from scratch 가 비효율적이면 적극 vendor (attribution 보존), 가벼우면 재작성" 의 사용자 판단.

흡수한/흡수할 패턴:
- **hermes-agent multi-channel gateway** → `ai_agents/` 의 ChannelAgent ABC + telegram/slack/whatsapp/discord 단계적 추가 (Phase 2.5 base, Phase 3+ 실제 채널)
- **hermes-agent `plugins/` 모양** → `backend/ingest/` 의 가벼운 정리 (auto dispatcher 명확화). ABC 강제까진 over-engineering.
- **hermes-agent 자가학습 (auto-skills)** → 자동 prompt/ingester 개선 (Phase 3+)
- **hermes-webui 세 패널 + SSE + vanilla JS** → `frontend_v2/` (Phase 2.5)
- **openclaw onboard daemon** → 향후 systemd/launchd 등록 helper (Phase 3+)

### LLM provider 책임 분리

LLM provider 추상화는 backend 의 책임. `ai_agents/` 모듈은 LLMProvider 를 직접 호출하지 않는다 — 필요하면 backend HTTP `/ask` 통해서. 이유: agent 가 직접 LLM 부르면 model/prompt 버전 추적이 두 곳으로 갈라짐 (§2 Versioned analysis 원칙 위반).

## 4. 기술 스택 / 환경

- **OS**: Ubuntu, **GPU**: NVIDIA RTX 4090 (CUDA), **Docker**: nvidia-container-toolkit
- **Backend**: Python 3.11+ (검증: 3.13.12 + torch 2.6.0+cu124, NVIDIA driver 580.x), FastAPI, SQLAlchemy 2.0 async + asyncpg, pydantic-settings
- **DB**: PostgreSQL 16 (관계형 + raw 본문) + Qdrant 1.12 (벡터)
- **Embedding**: sentence-transformers (bge-m3) → Phase 2 에 TEI 로 전환
- **LLM**: OpenAI / Anthropic / Ollama (provider abstraction)
- **Frontend**: Streamlit (Phase 1-2 MVP, Settings/Search 탭 유지) + **Next.js 14 App Router + TypeScript + Tailwind + shadcn/ui + cytoscape.js** (Phase 2.5+, graph UI 부터 SaaS path 까지 일관)
- **Object storage**: 로컬 FS → MinIO (Phase 2)
- **Python 환경**: **venv** (conda 아님). 이유: 시스템 의존성은 Docker 가 격리하고, Python 패키지는 전부 표준 pip — conda 의 강점이 안 살음. 학습(Phase 3 sVLL)용 conda env 는 그 시점에 별도 생성해 책임 분리.

설정은 **모두 `env/dev.env` 환경변수** 로 관리. 코드에 비밀값/하드코딩 절대 금지. `os.environ` 직접 접근 금지 — 항상 `backend.config.get_settings()` 를 통한다.

## 5. 코드 스타일

- **Python typing 필수**, `from __future__ import annotations`
- **async/await 우선** (블로킹 호출은 `asyncio.to_thread`)
- **Pydantic schema** 로 외부 인터페이스 정의 (`backend/schemas/`)
- **FastAPI router 구조** (`backend/api/<feature>.py`)
- **함수 단위 분리 + 서비스 단위 모듈화**, 지나친 OOP 지양
- **주석은 한국어 OK**, 충분히 작성 (특히 "왜 이렇게 했는지" 가 중요한 곳)
- 변수/함수 이름은 영어 + snake_case 유지

## 6. MVP 원칙: 과한 추상화 금지

- 디자인 패턴 / generic architecture / 미래 가정 기능 **금지**
- 동작하는 MVP > Clean Architecture
- 단, **재배포·서버 이전·온프레미스 설치·SaaS 화 가능 구조**는 처음부터 유지 (env, 볼륨, compose, healthcheck 분리)
- 새 기능 제안 시 "MVP 에 정말 필요한가?" 를 먼저 묻고, 아니면 Phase 2+ 로 미룬다

## 7. Git / Commit 규칙

- **모든 commit 메시지는 한국어로 작성**. 영문 conventional prefix(`feat:`/`fix:`/`chore:`) 사용 금지.
  - 예: `"초기 scaffold: ..."`, `"수정: ..."`, `"리팩토링: ..."`
  - 본문도 한국어. 코드 식별자, 명령어, 외부 시스템명(Postgres, Qdrant, OpenClaw) 은 원문 유지
- `git push --force`, `git reset --hard` 는 명시적 지시 없으면 금지
- `--no-verify` 등 hook 우회 금지
- `.env`, `volumes/`, `external/`, `archive/`, `__pycache__/` 는 절대 commit 금지 (.gitignore 로 처리됨)

## 8. 자주 쓰는 명령어

셋업 스크립트는 `stepN_setup_*` / `stepN_check_*` 쌍 패턴을 따른다. 각 step setup 직후 같은 번호의 check 로 sanity 확인:

```bash
# step1: Python 베이스 환경 (.venv + torch cu124 + requirements)
bash scripts/step1_install_base_env.sh
source .venv/bin/activate
bash scripts/step1_check_base_env.sh

# step2_1: 호스트에 Docker + NVIDIA Container Toolkit 설치 (sudo, 한 번만)
bash scripts/step2_1_install_docker.sh          # docker-ce + compose v2 + nvidia-container-toolkit
# 그룹 적용 위해 새 셸 또는 'newgrp docker'
bash scripts/step2_1_check_docker.sh            # docker / compose / nvidia runtime / hello-world

# step2_2: LinkMind 인프라 (Postgres + Qdrant + Ollama + OpenWebUI)
bash scripts/step2_2_setup_infra.sh             # docker compose up + healthy 대기 (step2_1_check 사전 호출)
bash scripts/step2_2_check_infra.sh             # 4개 서비스 연결성 검증
# Phase 2 (TEI + MinIO):  bash scripts/step2_2_setup_infra.sh --phase2

# step3: Ollama 모델 (env/dev.env 의 OLLAMA_MODEL pull)
bash scripts/step3_setup_ollama.sh            # qwen2.5:7b pull + 동작 검증
bash scripts/step3_check_ollama.sh            # API + 모델 존재 + generate dry run
bash scripts/ollama_pull.sh qwen2.5:14b       # 다른 모델 추가 받기 (서브 유틸)
bash scripts/ollama_chat.sh "안녕"             # 한 번의 채팅 테스트 (서브 유틸)

# step4: Qdrant 컬렉션 (bge-m3 1.4GB 첫 다운로드)
python -m backend.jobs.init_qdrant
bash scripts/step4_check_qdrant.sh            # 컬렉션 + vector dim 일치

# step5: 백엔드 + 프론트
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
streamlit run frontend/app.py
# (또는 한 셸에서 동시 기동)
bash scripts/step5_run_dev.sh             # 백그라운드, --stop / --status / --foreground

# 테스트 — 카테고리별 (자세히는 9번 Testing 정책)
bash scripts/tests/total/run_all_local.sh      # 5 카테고리 다 (로컬)
bash scripts/tests/total/run_ci_simulation.sh  # CI 가 도는 것만 (push 전 점검)
bash scripts/tests/ci/step1_cpu.sh             # 4s, default suite
bash scripts/tests/ci/step2_embedding.sh       # 15s, MiniLM CPU
bash scripts/tests/local/step5_gpu.sh          # 10s, CUDA 필요

# URL 하나 수동 수집
python -m backend.ingest.url https://arxiv.org/abs/2401.01234

# ai_agents — Telegram inbox watcher (현재). Phase 3+ 에 slack/whatsapp/discord 추가
python -m ai_agents.telegram_inbox_watcher --daemon       # 백그라운드 listen + 자동 ingest
python -m ai_agents.telegram_inbox_watcher --backfill      # 옛 메시지 일괄 처리

# (옵션) 외부 client 사용 — openclaw 별도 띄워서 LinkMind API client 로 쓰고 싶을 때만
# 자세한 셋업은 docs/agent_architecture.md §1 참고. 기본 사용엔 불필요.
# bash scripts/install_openclaw.sh

# Slack 데이터 import — slackdump (비공개 채널/DM 포함). 자세히는 docs/slack_setup.md
slackdump workspace list
slackdump workspace new -token "$SLACK_USER_TOKEN" -cookie "$SLACK_D_COOKIE" hkkim
slackdump export -workspace hkkim -type standard -files=true \
    -o archive/slack_export/full_$(date +%Y-%m-%d)
```

## 9. Testing 정책 — 새 함수 추가 시 동반 작성 필수

**핵심 규칙**: 새 기능 / 함수를 추가하거나 기존 함수의 동작이 바뀌면 같은 PR/commit
안에서 단위 테스트도 함께 작성하거나 갱신한다. 회귀 방지 + CI 보장.

### 카테고리 5 종 (마커)

| 마커 | 위치 | 어디서 도는가 | 비고 |
|---|---|---|---|
| `cpu` (마커 없음) | `tests/*.py` | CI + 로컬 (가장 빠름, ≈4s) | pure unit + mock + fixture |
| `embedding` | `tests/embedding/` | CI + 로컬 | 가벼운 MiniLM-L6-v2 (~80MB), CPU 가능 |
| `integration` | `tests/integration/` | backend live (로컬), CI 에선 skip | FastAPI e2e — fixture 가 backend 미가동 시 pytest.skip |
| `llm` | `tests/llm/` | Ollama live (로컬), CI 에선 skip | 실 LLM 호출 sanity — fixture 가 미가동 시 skip |
| `gpu` | `tests/gpu/` | 로컬 (RTX 4090) 전용 | CUDA device 강제, CI 자동 deselect |

### 새 함수 추가 시 결정 흐름

1. **pure 함수 (DB/네트워크 없음)** → `tests/` 직접 추가, 마커 없음. 기존
   `test_external_ids.py` / `test_pdf_abstract.py` 같은 형태.
2. **외부 모듈 호출 (httpx, yt_dlp, GitHub API …)** → monkeypatch 로 가짜 응답
   주는 mock test. `tests/test_github_ingest_mocked.py` 패턴 참고.
3. **DB/Postgres 가 필요** → `tests/integration/` + `@pytest.mark.integration`.
   backend fixture 로 응답 contract 만 검증 (데이터 내용 X — DB 상태 의존성 피함).
4. **sentence-transformers / 임베딩** → `tests/embedding/` + `@pytest.mark.embedding`.
   가벼운 MiniLM 로. bge-m3 같은 무거운 production 모델 X.
5. **CUDA 강제** → `tests/gpu/` + `@pytest.mark.gpu`. torch.cuda.is_available()
   체크 fixture 로 GPU 없으면 skip.
6. **실 LLM 호출** → `tests/llm/` + `@pytest.mark.llm`. Ollama health 체크
   fixture + 짧은 호출 (max_tokens=10).

### 테스트 스크립트도 동기화

새 카테고리에 첫 테스트를 추가했으면 `scripts/tests/` 의 해당 스크립트가
자동으로 잡아준다 (marker 기반). 새로운 카테고리 자체를 만들면:
- `pytest.ini` 에 marker 등록
- `scripts/tests/_lib.sh` 에 `run_<category>()` 함수 추가
- `scripts/tests/ci/` 또는 `scripts/tests/local/` 에 stepN 스크립트
- `scripts/tests/total/run_all_local.sh` 및 `run_ci_simulation.sh` 에 호출 추가
- `scripts/tests/README.md` 표 갱신

### CI 와의 매핑

GitHub Actions (`.github/workflows/ci.yml`) 는 `pytest -m "not gpu"` — GPU 만
deselect. 나머지는 시도하되 환경 미충족 시 fixture skip. CI 가 cpu + embedding +
integration + llm 카테고리에 한해 회귀 잡음.

### 명령

```bash
bash scripts/tests/total/run_all_local.sh       # 5 카테고리 다 (로컬)
bash scripts/tests/total/run_ci_simulation.sh   # CI 가 도는 것만
.venv/bin/pytest -m '' tests/                   # marker 무시 전체
.venv/bin/pytest -m embedding tests/            # 단일 카테고리
```


## 10. 디렉토리 구조 요점

```
backend/
├─ api/         # FastAPI routers (health, ingest, search, ask, graph, settings, files)
├─ config.py    # pydantic-settings 환경설정 (모든 env 진입점)
├─ db/          # connection.py, repository.py, schema.sql
├─ embedding/   # base.py, local.py, factory.py, qdrant_store.py
├─ llm/         # base.py, factory.py, openai/claude/ollama provider
├─ ingest/      # source 별 (url/, slack/, telegram/, pdf/, github/, arxiv/, youtube/)
│              #   + auto dispatcher (host 기반 라우팅)
├─ schemas/     # Pydantic 요청·응답 모델
├─ storage/     # local.py (raw 파일 무손실 보존)
├─ utils/       # chunking, hashing, external_ids
├─ jobs/        # batch (init_db / init_qdrant / backfill_* / seed_* / generate_*) — `python -m backend.jobs.<name>`
└─ main.py      # FastAPI 진입점

ai_agents/                 # LinkMind 자체 multi-channel gateway (§3)
├─ base.py                 #   ChannelAgent ABC (setup / listen / on_message → ingest)
├─ telegram_inbox_watcher.py    # 현재 (ChannelAgent 상속)
├─ slack_inbox_watcher.py       # Phase 3+
├─ whatsapp_inbox_watcher.py    # Phase 3+
└─ discord_inbox_watcher.py     # Phase 3+

frontend/app.py            # Streamlit MVP (Phase 1-2)
frontend_v2/               # graph UI — cytoscape.js + vanilla JS + SSE (Phase 2.5+)
├─ index.html
└─ static/                 # JS/CSS modules

compose/docker-compose.*   # docker compose 정의
env/                       # dev.env(gitignored) / dev.env.example
scripts/                   # 실행용 .sh 만 — stepN_*.sh / install_openclaw.sh / ollama_pull.sh / slack_export.sh
docs/                      # agent_architecture.md, training_data_design.md, features_backlog.md, slack_setup.md, telegram_setup.md
archive/                   # raw 자료 (gitignored)
volumes/                   # 컨테이너 영속 볼륨 (gitignored)
external/{openclaw,hermes-agent,hermes-webui}/  # gitignored 벤치마킹 참조 clone (코드 import 금지)
```

## 11. NEVER 목록

- ❌ `.env` 파일 commit
- ❌ commit 메시지 영어 / conventional prefix
- ❌ license 호환 안 되는 외부 코드를 LinkMind 에 vendor (GPL → AGPL 호환, MIT/Apache → AGPL 호환, AGPL→AGPL 호환, BSL/proprietary → 금지).
- ⚠️ MIT/Apache 코드 vendor 시 LICENSE 파일 + copyright notice + 출처 주석 (`# Adapted from <repo>/<file> (MIT) — Copyright (c) ...`) 필수. `external/` 의 clone 자체는 gitignored 유지.
- ❌ `raw_content` 를 NULL 로 두거나 변형해서 저장
- ❌ AI 분석 결과를 model/prompt 버전 없이 저장
- ❌ `os.environ.get(...)` 코드에서 직접 사용 (`backend.config` 경유)
- ❌ 이미지/PDF resize·compress (학습 데이터 손실)
- ❌ `--force` / `--no-verify` / `reset --hard` 같은 destructive 명령 자의로
- ❌ **사용자 데이터로 운영자의 공통 모델 학습** (privacy 침해 + GDPR/PIPA 위반 + training data extraction attack). personal LoRA 는 사용자 본인 데이터로 본인 모델만 (§14 Privacy 5원칙).
- ❌ `ai_agents/` 모듈이 backend LLMProvider 를 직접 호출 — HTTP `/ask` 경유 (§3 책임 분리 + §2 Versioned analysis).
- ❌ hosted SaaS (Phase 6+): cross-tenant 데이터 노출 (RLS / tenant_id 필터 누락)
- ❌ hosted SaaS: 사용자 데이터를 동의 없이 외부 LLM API (OpenAI/Anthropic) 에 전송 — BYOK 또는 명시적 약관 동의한 경우만.

## 12. Phase 별 로드맵

| Phase | 상태 | 핵심 |
|---|---|---|
| 1 | 완료 | Postgres + Qdrant + URL ingest + Embedding + Search |
| 2 | 진행 중 | AI 요약/태깅, Slack export 파서, TEI 전환, feedback 테이블, dataset exporter |
| 2.5 | 진행 중 | Topic 그래프 + `ai_agents/` ChannelAgent ABC (hermes-agent 영감) + graph mini web UI (hermes-webui 영감, cytoscape.js) + modality-aware item viewer |
| 3 | | OCR/멀티모달, 이미지 분석, RAG 고도화, `ai_agents/` 실제 채널 확장 (slack/whatsapp/discord), 자가학습 (auto prompt/ingester 개선) |
| 4 | | **sVLL LoRA 파인튜닝** (LLaMA-Factory + Qwen2-VL 등), vLLM 서빙 — self-host 또는 hosted enterprise tier 옵션 |
| 5 | | Continuous training loop, 온프레미스 AI 엔진 완성 |
| 6 (선택) | OSS → hosted SaaS | AGPL v3 공개, Next.js + Auth.js + Stripe, multi-tenant, BYOK. §14 참조 |

## 13. 현재 진행 상태 (2026-05-18 기준)

> 자세한 backlog 와 phase 별 완료 항목 / 미구현 항목은 `docs/features_backlog.md` 가
> source of truth. 여기는 큰 그림만.

### Phase 2.5 wave-3 (2026-05-18, 진행 중) — 단일 self-contained AI engine 전환

오늘 작업 단계 (각 step 마다 commit):
- ✅ **Day 1: §3 재정의** — "LinkMind ↔ OpenClaw" 두 시스템 모델 폐기, 단일
  self-contained 시스템 + external/ 는 벤치마킹 참조. §14 신규 (AGPL v3 + Privacy
  5원칙 + SaaS Phase 6 path). docs/agent_architecture.md 신규.
- ✅ **Day 2-3: ChannelAgent ABC** — `ai_agents/base.py` + telegram_inbox_watcher
  를 TelegramChannelAgent 로 리팩토링. 11 tests 신규.
- ✅ **Day 4: items 스키마 확장** — user_notes / user_notes_updated_at /
  is_read / read_at 컬럼 + idx_items_unread partial index. migrate_schema.py
  idempotent runner. GET/PATCH /items/{id} API. **user_notes 변경 시 BackgroundTask
  로 LLM 키워드 추출 → items.tags 자동 병합** (한국어 자유 문체 지원). 23 tests 신규.
- ✅ **Day 5/1: 다양 포맷 텍스트 추출 통합 모듈** — `backend/ingest/document/`
  (PDF 재사용 + DOCX/PPTX/TXT/MD). python-docx + python-pptx + charset-normalizer
  추가. 한국어 cp949 인코딩 우선. 17 tests 신규.
- ✅ **Day 5/2: 텔레그램 첨부 자동 ingest** — `ingest_document()` 진입점 추가
  (PDF ingest 패턴 재사용 + 모든 포맷). TelegramMessage 에 `attachments` 필드,
  ingest_telegram_message 의 첨부 분기, caption → user_notes 자동 저장,
  ai_agents/telegram_inbox_watcher 가 Telethon `download_media` 호출 → 임시
  디렉토리 → ingest 후 tmp 정리, ChannelAgent.is_ingest_successful 확장 —
  attachments 까지 모두 error 없어야만 메시지 삭제 (사용자 가드레일 "하나라도
  누락이면 삭제 X"). 5 attachments 케이스 신규. 177 tests 통과.
- ⏳ Day 5/3: backend/ingest 정리 (auto dispatcher 명확화 — 짧게)
- ⏳ Day 6-9: graph backend endpoint (cytoscape JSON)
- ⏳ Day 10-13: Next.js 14 graph UI + modality viewer
- ⏳ Day 14: end-to-end 통합

### 구현 완료 (현재 main 브랜치)

**Phase 1** — Postgres + Qdrant 인프라 + URL ingest + 검색 + RAG.
**Phase 2 first wave (2026-05-15)** — Settings UI + DB-backed runtime settings +
한국어 prompt 강제 (`summary_system` v3) + 4종 ingest (url/youtube/github/pdf) +
PDF 원본 attachments 보존 + `/files/{hash}` inline 서빙.
**Phase 2 second wave (2026-05-16)** — `ingest --force` 옵션 (url/github/pdf/youtube)
+ GitHub raw_body hash 안정화 + PDF abstract regex 보강 + PDF figure 추출
(pymupdf `get_images`) + YouTube 영상/playlist 썸네일 attachments + Streamlit
force 체크박스.
**Phase 2.5 — Topic 그룹핑 (2026-05-16)** — `topics` + `item_topics` 스키마 +
`backend/utils/external_ids.py` (arxiv/doi/github/yt/ytpl 추출+정규화) + 각
ingester 가 ingest 시 `auto_link_topics` 호출 + `/topics/*` API 4개 + Streamlit
Topics 탭 + Search 결과에 topic 칩 + `backend.jobs.generate_topic_descriptions`
(자식 item summary 합성). 검증: arxiv:2106.09685 (LoRA paper+code), arxiv:2511.20343
(AMB3R paper+code+project page 3 modality) 자동 그룹핑.
**Testing 인프라 (2026-05-16)** — pytest marker 5종 (cpu/embedding/integration/
llm/gpu) + `tests/{embedding,integration,llm,gpu}/` 디렉토리 + 실 PDF fixture
2개 + `tests/resources/test_urls.json` + `scripts/tests/{ci,local,total}/` +
GitHub Actions CI + `requirements-test.txt` lightweight. 총 95 tests 통과
(cpu 83 / embedding 3 / integration 4 / llm 2 / gpu 3).
**ENV cleanup (2026-05-16)** — LLM 런타임 선호 (provider/model) 를 env 에서 제거,
DB `app_settings` + UI Settings 탭 만 진실. 인프라 위치/시크릿만 env. Postgres
`python -m backend.jobs.<name>`. 5 카테고리 (cpu/embedding/integration/llm/gpu)
비번 `real2real` 로 단순화.
**RAG 답변 품질 개선 (2026-05-16)** — `/ask` 의 context block 에 item 의 한국어
summary (500-1500자) + tags 추가. 이전엔 chunk snippet (300자) 만 들어가 LLM 이
자료의 깊이 활용 못 하고 일반 정의로 답하던 문제 해결. `rag_system` v3 prompt —
'자료 구체 인용 + 자료들이 다루는 측면' 두 단락 강제. 다만 `qwen2.5:14b` 모델
응답 시간 ~3분 — 더 가벼운 ask 전용 모델은 Phase 2 후반/Phase 4 (sVLL) 로.

**리팩토링 (2026-05-16)** — `scripts/` 는 .sh 만, batch python 6개 (`backfill_*`,
`seed_*`, `generate_*`, `init_db`, `init_qdrant`) 는 `backend/jobs/` 로,
telegram watcher 는 `ai_agents/` 로 (NEVER §3 정신 — backend 외부 agent). 호출은
모두 PASS, 135 tests.
**Phase C wave-1 — Telegram inbox (2026-05-16)** — `LinkMind-Inbox` 텔레그램
채널에 URL/메모 던지면 자동 ingest 되는 풀 흐름. `backend/ingest/telegram/` 모듈
(파서 + ingest_telegram_message + export 파서) + `ai_agents/telegram_inbox_watcher.py`
(Telethon daemon, --daemon idempotent, --backfill, --restart). 핵심 동작:
URL 자동 host 라우팅 + topic auto-link / URL 없는 메모는 source_type='telegram'
note 저장 / **ingest 성공 시 채널에서 메시지 자동 삭제** (inbox 패턴 — 처리
안 된 것만 남음). 사용자 환경 (RTX 4090 + qwen2.5:7b) 에서 listen + backfill +
삭제 모든 단계 라이브 검증. 동반 fix: GitHub README raw HTML strip
(`_clean_readme_html`), PDF Title placeholder 거름 (`_extract_pdf_title`).

### 다음에 할 일

**짧은 follow-up** ✅ 완료 (2026-05-16):
- ✅ amber 3 modality description 검증 — 3 item (code + paper + project page) 다 반영 확인
- ✅ Streamlit 의 manual link UI 에 selectbox + 새 slug 직접 입력 fallback (autocomplete)

**Phase 2.5 후속** ✅ 완료 / ⏸ 외부 API 종료로 보류 (2026-05-16):
- ✅ 검색 결과에 같은 topic 의 다른 modality item 인라인 노출 — expander 안에 role + url + 첫 줄 요약
- ✅ arxiv API 시드 — `backend.jobs.seed_arxiv_metadata` 로 11개 arxiv:* topic 모두 title/authors/published/summary 보강 (RoBERTa/DeBERTa/Adapter/Prefix-Tuning 등 자동 paper 제목 채워짐). `tags` 에 `arxiv-seeded` 마커
- ⏸ paperswithcode slug → github_repo 자동 연결 — **외부 API 종료**. paperswithcode.com 이 Hugging Face 로 이전됐고, HF papers API 에는 github_repo 매핑이 없음. 보류.

**Phase C wave-2 — Slack 워크스페이스 일회성 backfill** (2026-05-16 갱신):
- ⚠️ 옛 `archive/slack_export/public_2026-05-14/` 는 사용자가 삭제. **사용자가
  Slack 구독을 곧 해제 — 내일부터는 Slack 안 씀** (외출 전 알림). 즉 wave-2 는
  **일회성 backfill** 만 의미.
- 기존 slackdump 셋업은 **그대로 유지** (alias `hkkim`, token/cookie, scripts/
  slack_export.sh, docs/slack_setup.md). 어제 검증된 환경 — 재 export 가 가장 빠름.
- **진행 순서**:
  1. `bash scripts/slack_export.sh` 로 새 export (workspace 전체, 자동으로
     `archive/slack_export/full_<date>/` + `latest` symlink 생성)
  2. `backend/ingest/slack/__init__.py` + `export_parser.py` 작성 — slackdump
     standard 포맷 파싱 (channel 별 디렉토리, 날짜별 JSON, thread/files 포함)
  3. `python -m backend.ingest.slack <export_dir>` 로 일괄 ingest — URL 자동
     라우팅 (telegram 흐름 재사용) + thread 묶음 + 첨부 download
  4. 단위 테스트 + 작은 fixture (channel 디렉토리 + 한 JSON 파일 모사)
  5. 검증: 단일 채널 먼저 (예: `공부-컴퓨터비전`) → 워크스페이스 전체
- 모듈 구조는 Telegram 패턴 일관 — `SlackMessage` dataclass + `ingest_slack_message`
  (단일) + `ingest_slack_export` (전체 export 폴더). 향후 다른 Slack 워크스페이스
  처리 또는 다시 쓸 때 재사용 가능.
- slack_sdk 직접 호출은 over-engineering 으로 보류 (incremental sync 필요 X).

**Phase 2.5 후속 (선택)**:
- 검색 결과에 같은 topic 의 다른 modality 도 인라인 노출 → ✅ 완료
- arxiv API 시드 → ✅ 완료
- paperswithcode slug → ⏸ 외부 API 종료로 보류

**`/ask` 자체 AI (Phase 2 후반 / Phase 4)** — 현재 qwen2.5:14b 가 RAG 응답 ~3분
이라 UX 좋지 않음. 옵션:
- ask 전용 더 작은 모델 (qwen2.5:7b 또는 더 작은 instruct 모델) 분리 — Settings 의
  ask 용 별도 model 필드 추가하면 ingest 와 ask 가 다른 모델 사용 가능
- streaming response (Streamlit 의 첫 토큰부터 표시) — UX 만 개선
- 궁극적으로 sVLL LoRA 파인튜닝 (Phase 4) — 사용자 데이터 학습한 작은 모델로
  ask 까지 처리 — 학습 데이터 self-loop 의 완성

**Phase 3-5 (CLAUDE.md §12 로드맵)**:
- AI 카테고리/태깅 강화, feedback 테이블, dataset exporter (JSONL)
- TEI 임베딩 전환, MinIO object storage
- sVLL LoRA 파인튜닝 (LLaMA-Factory + Qwen2-VL), vLLM 서빙
- Continuous training loop

---

## 14. License & Privacy & SaaS path

### License — AGPL v3 (OSS 공개 시점에 적용)

- **AGPL v3** — self-host 자유 (개인/회사 무제한), 단 변형해서 SaaS 로 재판매 시 변경 사항 공개 의무. Plausible/Cal.com/n8n 채택 모델.
- `LICENSE` 파일은 OSS 공개 시점 (Phase 6-B) 에 추가. 그 전 private repo 단계에선 보류.
- AWS / Notion 등이 LinkMindCloud 만들어 재판매하는 시나리오를 차단 — Elasticsearch 가 MIT 에서 SSPL 로 옮긴 이유.

### Privacy 5원칙 (hosted SaaS 진입 시 적용)

1. **사용자 데이터로 공통 모델 학습 절대 금지** — §1, §11. personal LoRA 는 사용자 본인 데이터로 본인 모델만. 이게 LinkMind 의 핵심 차별점.
2. **Tenant isolation** — 모든 DB query 에 `tenant_id` 필터 + Postgres RLS (Row Level Security). cross-tenant search/ask 절대 X.
3. **LLM 호출 동의** — 사용자가 BYOK (Bring Your Own Key) 했거나 명시적 약관 동의한 경우만 외부 API (OpenAI/Anthropic) 에 사용자 데이터 전송.
4. **삭제 권리 (GDPR / 한국 PIPA)** — 계정 삭제 요청 시 30일 내 Postgres + Qdrant + R2 모든 데이터 영구 삭제. 모델 학습 결과 (만약 있다면) 도 함께 제거.
5. **사용자 ingest 책임** — 사용자가 합법적으로 접근 권한 있는 URL 만. hosted 에선 PDF 직접 업로드 비활성 (저작권 위험).

### SaaS path 단계 (시기상조 — 1년+ 후 예정)

- **Phase 6-A** (현재): self-host 완성도. 본인 사용 + 데이터 누적.
- **Phase 6-B** (6+개월 후): OSS 공개 (AGPL v3), ProductHunt / HackerNews / LinkedIn 런칭.
- **Phase 6-C** (9-12개월 후): closed beta hosted — Fly.io / Vercel / Cloudflare R2, Google OAuth, **BYOK 강제** (운영자 LLM 비용 0), waitlist 50명.
- **Phase 6-D** (12-15개월 후): freemium 출시 — Free (월 한도 + BYOK) / Pro $9/월 / Team $19/사용자/월. Founding member 영구 50% 할인.
- **Phase 6-E** (15+개월 후): Enterprise tier — 격리 GPU + personal LoRA + SSO + audit log.

### hosted vs self-host 기능 분리

- **hosted only**: web UI, multi-tenant auth, Stripe 결제, 운영자가 제공하는 Telegram bot
- **self-host only**: PDF 직접 업로드, personal sVLL LoRA fine-tuning, 자유로운 client 선택 (openclaw/hermes-agent/자체 봇)
- **공통**: URL/YouTube/GitHub ingest, search, ask, graph UI, topic 그래프

---

## 13.1. 과거 기록 — Phase 2 첫 wave 완료 (2026-05-15)

### 2026-05-15 오늘 추가 — Phase 2 첫 wave 완료 ✅

기존 Phase 1 (URL ingest + 검색 + RAG) 위에 다음을 한 세션에 통합:

**LLM 모델 + 한국어 강제**
- Ollama 기본 모델: `qwen2.5:7b` → `exaone3.5:7.8b` 검증 후, 사용자 결정으로 `qwen2.5:14b` 채택 (영어 본문 요약 품질 더 우수, RTX 4090 24GB OK)
- `summary_system` prompt **v3** 활성 — "출력 무조건 한국어, 영어 본문도 번역, 예시 키워드 베껴쓰기 금지, 최소 10 bullet + 끝줄 5-10 해시태그"
- `rag_system` prompt v1 — 한국어 답변 + 영어 기술 용어 (SLAM, Transformer, LiDAR …) 원문 보존
- `_generate_and_save_summary` 의 user message 에 "한국어로 요약하라" prefix 추가 (system 만으로 못 끌리는 모델 방어)

**DB 기반 런타임 설정 + prompt 버전 히스토리**
- 새 테이블: `app_settings` (key/value), `prompts` (name/version/content/is_active history — Versioned analysis 원칙)
- `backend/runtime_settings.py` — DB ↔ in-memory 캐시, lifespan 에서 `seed_and_load()` 호출
- `backend/api/settings.py` — `GET/PUT /settings/llm`, `GET /settings/llm/models` (ollama `/api/tags`), `GET/POST /settings/prompts/{name}` + version 활성화
- Streamlit **Settings 탭** — provider/model dropdown + prompt textarea + 새 버전 저장 + 히스토리 조회

**URL ingest 강화 (논문/article 친화)**
- 페이지의 **abstract** 가 있으면 본문 전체 대신 abstract 만 LLM 입력으로 (citation_abstract meta, arxiv `<blockquote class="abstract">`, og:description 순)
- **페이지 keywords** 추출: citation_keywords meta, arxiv subject classes, JSON-LD keywords
- LLM 응답 끝줄의 `#tag` 해시태그 파싱 + 페이지 keywords + dedup → `items.tags` (최대 10)
- abstract 도 `_SUMMARY_INPUT_LIMIT` (8000자) 로 cap — 긴 abstract (플레이리스트 영상 목록) timeout 방지

**해시태그 검색**
- `/search` 가 query 의 `#tag` 토큰 자동 분리 (텍스트는 비고 태그만이면 Postgres GIN 인덱스로 최신순, 혼합이면 Qdrant + tag filter)
- `_generate_and_save_summary` 후 Qdrant chunk payload 의 tags 도 `set_payload_for_item_chunks` 로 갱신 → 벡터 검색 결과도 tag 로 필터 가능

**새 ingest source 4종 + dispatcher**
- `POST /ingest/auto` — host 자동 분류 (youtube.com/youtu.be → youtube, github.com → github, *.pdf → pdf, 나머지 → url)
- `POST /ingest/youtube` — 단일 영상 (yt-dlp 메타 + youtube-transcript-api 자막 ko→en, 자막 없으면 description + `#no-transcript`) / 플레이리스트 (flat-list 영상 메타 요약, source_type=`youtube_playlist`)
- `POST /ingest/github` — REST API (`/repos/{owner}/{repo}`, `/readme`, `/languages`), README base64 decode, paper link 탐지 (arxiv/doi/paperswithcode), **라이센스 SPDX 해시태그 강제 주입** (예: `#MIT`, `#Apache-2.0`, `#GPL-3.0`, 없으면 `#no-license`)
- `POST /ingest/pdf` (URL) + `POST /ingest/pdf/upload` (multipart) — pypdf → pymupdf fallback, NUL byte sanitize, abstract 자동 탐지, 원본 PDF 는 `volumes/archive/<yyyy>/<mm>/<hash[:2]>/<hash>` 에 loss-less 보존, attachments 테이블에 등록
- `backend/schemas/models.py` SourceType 에 `youtube_playlist` 추가
- 새 ingest 들은 모두 url ingest 의 helper (`ExtractedDoc`, `_generate_and_save_summary`, `_embed_and_index`) 재사용 — 한국어 요약/태그 동일 흐름

**파일 서빙**
- `GET /files/{file_hash}` — attachments 의 raw 파일 inline 응답 (PDF viewer 가 브라우저에서 열림). path traversal 방어 위해 hex SHA-256 64자만 허용
- PDF ingest 시 local 파일 (tempfile 포함) 이면 `source_url` 을 `/files/{file_hash}` path-only 로 저장. Streamlit 이 `/`로 시작하면 API_BASE 와 결합

**검증된 데이터 (오늘 추가)**
- arxiv 2106.09685 (LoRA) URL ingest — tags 9개 (`#Transformer`, `#LowRankAdaptation`, `#PyTorch` …)
- GitHub microsoft/LoRA — tags 10개 (MIT license tag 우선 보장은 코드 fix 반영됨)
- YouTube 단일 영상 `PYr-LSOf2OY` (Gaussian Splatting) — 자막 없는 영상, description 으로 요약, `#no-transcript` 라벨
- YouTube 플레이리스트 `PL5Q2soXY2Zi9...` (ETH Spring 2025) — 영상 50+개 flat list, abstract cap fix 후 backfill 로 1010자 한국어 요약 + 10 tags
- PDF 3건 (SLAM Multi-Camera / FAST-LIVO2 / LiDAR Teach-Radar Repeat) — 각각 39 / 131 / 103 chunks, 한국어 요약 + 해시태그, 원본 PDF `volumes/archive/` 보존

### 이번 세션에서 잡은 결함 (모두 commit 됨)

- **plyaylist 요약 timeout** — `_generate_and_save_summary` 가 abstract 길이 무제한이라 영상 50+개 목록 (~20K자) 으로 LLM 호출하면 빈 응답. abstract 도 `_SUMMARY_INPUT_LIMIT (8000)` cap.
- **빈 exception 메시지** — httpx timeout 등에서 `str(e)` 가 빈 문자열인 경우 로그가 무의미. `type(e).__name__: %s` 로 보강.
- **PDF NUL byte** — pypdf 가 가끔 `0x00` 을 텍스트에 포함, Postgres `CharacterNotInRepertoireError`. `_sanitize_text` 로 제거.
- **GitHub license 잘림** — paper_keywords 의 순서가 `[topics, primary_lang, license_tag]` 라 topics 10개로 _TAG_MAX 차서 license 잘림. `[license_tag, primary_lang, has-paper-link, *topics]` 로 우선순위 변경.
- **영문 본문 요약이 영어로** — qwen2.5:14b 가 system prompt 의 "한국어만" 보다 본문 언어에 끌림. v3 prompt + user message prefix 두 곳에 한국어 강제.
- **backfill prompt_version=seed-fallback** — 별도 프로세스인 `backend.jobs.backfill_summary` 가 runtime_settings 캐시 미적재 상태에서 prompt 가져와 `seed-fallback` 라벨 기록. main 초입에 `await runtime_settings.seed_and_load()` 추가.
- **lru_cache 무효화** — Settings 에서 model 바꿔도 `get_llm_provider` 캐시가 옛 인스턴스 반환. runtime_settings 변경 시 `cache_clear()` 호출하도록 wiring.

### 완료 — Phase 1 동작 검증 풀체인 통과 ✅

step1 ~ step5 셋업 완주 + 실제 데이터로 URL ingest → bge-m3 임베딩 → Qdrant 벡터 검색 → Ollama 한국어 요약 자동 생성 → Streamlit Search 탭에서 item 단위 dedup + 요약 표시까지 검증.

검증된 데이터:
- arxiv 2401.01234 (cure semiparametric additive hazard) → item 1건, chunks 3개, 한국어 3-5 bullet 요약
- arxiv 1706.03762 (Attention Is All You Need) → 자동 흐름으로 item + 요약 함께 저장

이외 완료:
- ✅ 전체 scaffold + git origin/main push (모든 commit)
- ✅ OpenClaw 설치 정책 확정 (install.sh 기본)
- ✅ Slack 토큰/쿠키 + slackdump workspace alias 등록 완료
  - slackdump alias: **`hkkim`** (캐시: `~/.cache/slackdump/hkkim.bin`)
  - 워크스페이스: 알콩이달콩이 (T06PXGA7LE7, `w1710672365-sjj477000.slack.com`)
  - `archive/slack_export/full_2026-05-14/` 는 폐기 — 내일 `bash scripts/slack_export.sh` 로 재수집 예정
- ✅ `env/dev.env` 주요 키 결정 (POSTGRES_PASSWORD 랜덤 40자, `DEFAULT_LLM_PROVIDER=ollama`, `OLLAMA_MODEL=qwen2.5:7b`, `HF_HUB_OFFLINE=1`)
- ✅ `docs/slack_setup.md`, `scripts/slack_export.sh` 작성

### 셋업 중 발견한 견고성 fix (모두 commit 반영됨)

- **step2_1_install_docker.sh** — "이미 설치됨" 분기에 `systemctl restart docker` 누락 → 추가 (daemon.json 만 갱신하고 reload 안 하던 버그)
- **step2_1_check_docker.sh** — daemon.json 엔 nvidia 등록됐는데 docker daemon 미반영 케이스 self-heal (sudo restart 자동 시도). `docker info` template `{{range .Runtimes}}{{.Name}}{{end}}` 가 docker 29.x 에서 빈 출력 → `{{json .Runtimes}}` 로 변경. `--no-pull`/`--no-nvidia` 처럼 사용자 명시 skip 은 ⚠️ → ℹ️
- **backend/embedding/qdrant_store.py** — `client.search()` 가 qdrant-client v1.12+ 에서 제거됨 → `client.query_points()` 로 마이그레이션
- **backend/embedding/local.py** — `get_sentence_embedding_dimension` → `get_embedding_dimension` (sentence-transformers v5+ rename, FutureWarning 제거)
- **backend/ingest/url/__main__.py** — CLI 진입점 분리 (패키지에 `__init__.py` 의 `if __name__ == "__main__":` 는 `python -m` 으로 안 도는 표준 동작)
- **backend/config.py** — `effective_ollama_base_url` 추가 (Qdrant 와 동일 패턴, host 셸에서 도는 ingest 가 `ollama:11434` DNS 해결 실패하던 문제). `HF_HUB_OFFLINE` 자동 export — 모델 캐시 후 매 startup HF Hub HEAD 요청 + 토큰 경고 차단
- **backend/api/search.py** — Qdrant 가 chunk 단위 색인이라 같은 item 의 여러 chunk 가 결과로 도배되던 문제 → item_id 별 최고 score chunk 1개만 유지 (overfetch + dedup)
- **backend/ingest/url/__init__.py** — Ollama 로 한국어 3-5 bullet 요약 생성 → `items.summary` 저장 (실패해도 raw + embedding 은 영향 없음)
- **frontend/app.py** — `summary` 우선 표시, 없으면 `snippet` fallback ("요약 미생성 — raw 본문 일부" 캡션)
- **backend.jobs.backfill_summary** — `summary IS NULL` row 들에 retroactive 한국어 요약 (Phase 2 의 다른 ingest 에도 재사용)

### 평소 dev 실행 흐름 (3개 셸)

인프라 컨테이너 (Postgres/Qdrant/Ollama/OpenWebUI) 는 이미 docker compose 로 떠 있다고 가정. 죽었으면 `bash scripts/step2_2_setup_infra.sh` 로 재기동. backend uvicorn / frontend streamlit 은 host 셸에서 직접 (FastAPI/Streamlit reload 가 dev 편함).

```bash
# 셸 1 — 백엔드 (FastAPI, port 8000, --reload 로 코드 변경 자동 적용)
source .venv/bin/activate
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000

# 셸 2 — 프론트 (Streamlit, port 8501)
source .venv/bin/activate
streamlit run frontend/app.py
# 브라우저: http://localhost:8501

# 셸 3 — 명령 (ingest, backfill, REPL 등)
source .venv/bin/activate

# URL ingest (자동으로 한국어 요약까지)
python -m backend.ingest.url https://arxiv.org/abs/2401.01234

# 요약 누락 row backfill (옛 데이터, 또는 prompt 버전 올린 후)
python -m backend.jobs.backfill_summary            # summary IS NULL 인 row 만
python -m backend.jobs.backfill_summary --force    # 모든 item 재요약

# 검색 (CLI)
curl -sX POST localhost:8000/search -H 'content-type: application/json' \
     -d '{"query":"survival analysis cure model","top_k":3}' | jq

# 헬스
curl -s localhost:8000/health | jq
```

### 처음부터 셋업 (새 머신/재배포 시)

stepN_setup → stepN_check 쌍 패턴:

1. **step1 — Python 환경** (venv + torch cu124 + requirements)
   ```bash
   bash scripts/step1_install_base_env.sh
   source .venv/bin/activate
   bash scripts/step1_check_base_env.sh
   ```
   옵션: `--recreate` (clean), `--cpu` (GPU 없는 환경), `--cuda-version=126`.

2. **step2_1 — Docker + NVIDIA Container Toolkit** (sudo, 한 번만)
   ```bash
   bash scripts/step2_1_install_docker.sh
   newgrp docker        # 또는 새 셸/SSH 재로그인
   bash scripts/step2_1_check_docker.sh
   ```

3. **step2_2 — LinkMind 인프라 컨테이너**
   ```bash
   bash scripts/step2_2_setup_infra.sh        # docker compose up + healthy 대기
   bash scripts/step2_2_check_infra.sh        # 4개 서비스 연결성 검증
   ```

4. **step3 — Ollama 모델 pull**
   ```bash
   bash scripts/step3_setup_ollama.sh         # qwen2.5:7b
   bash scripts/step3_check_ollama.sh
   ```

5. **step4 — Qdrant 컬렉션** (bge-m3 1.4GB 첫 다운로드)
   ```bash
   python -m backend.jobs.init_qdrant
   bash scripts/step4_check_qdrant.sh
   ```

6. **step5 — 위 "평소 dev 실행 흐름" 그대로**

### (옛) "다음 세션 작업" — 모두 2026-05-16 wave 에서 완료

> 아래 항목들은 2026-05-15 시점에 미구현이었으나, 다음 날 2026-05-16 wave 에서
> 모두 처리됨. 현재 살아있는 backlog 는 `docs/features_backlog.md` 와 §13 머리말의
> "다음에 할 일" 을 참조.

| 옛 항목 | 처리 |
|---|---|
| SLAM/FAST-LIVO2/LiDAR PDF backfill --force | ✅ `backend.jobs.backfill_summary` 의 PDF abstract 재추출 보강 + 3 PDF v3 한국어 재요약 |
| `/files/{hash}` 검증 | ✅ 200 inline PDF + 400/404 에러 케이스 정상 |
| microsoft/LoRA `#MIT` tag 보강 | ✅ `ingest --force` 옵션 신설 + GitHub `raw_body` 안정화 (stars/forks 제거, sorted topics) → idempotent 보장 후 force re-ingest |
| YouTube 영상 썸네일 attachments | ✅ `_pick_best_thumbnail` + role='thumbnail' (video + playlist) |
| PDF figure 추출 | ✅ pymupdf `get_images()` + xref dedup + 200×200 미만 skip → role='figure' |
| PDF abstract regex 보강 | ✅ em-dash/colon/uppercase 라벨 + 'SUPPLEMENTARY MATERIAL'/'I NTRODUCTION' (글자 사이 공백) 종결 + 라벨 없는 fallback. 8 unit tests. |
| Slack export ingest | ⏳ 미착수 — Phase C 로 이월. backlog 참조. |
| AI 카테고리/feedback/dataset exporter/TEI/MinIO | ⏳ 미착수 — Phase 2 후반 / Phase 3. backlog 참조. |

원칙 (이번 wave 에서도 유지): 각 단계에서 에러 나면 회피하지 말고 root cause 파악.
2026-05-15 결함 (playlist abstract cap, PDF NUL byte, GitHub license 순서, qwen
영어 누출, backfill prompt 캐시 미적재) 와 2026-05-16 결함 (GitHub raw_body 안의
stars/forks 가 force 매칭을 깨던 idempotent 위반, slug-with-slash 의 FastAPI path
split, 'I NTRODUCTION' 글자 사이 공백 미인식) 도 다 그렇게 잡힘.
