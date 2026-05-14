# CLAUDE.md

이 파일은 LinkMind 저장소에서 Claude Code 가 작업할 때 자동 로드되는 가이드라인이다. 매 세션마다 같은 컨텍스트를 반복 설명할 필요 없도록 핵심만 압축해서 둔다.

> 자세한 내용은 `README.md`, `docs/openclaw_integration.md`, `docs/training_data_design.md` 참고.

---

## 1. 프로젝트의 진짜 목표

**LinkMind 자체는 수단**이다. 최종 목표는 **사용자가 누적한 데이터로 sVLL(small Vision-Language LLM)을 LoRA 파인튜닝해서 온프레미스 personalized AI 엔진**을 만드는 것. 그 엔진을 **지속적으로 재학습**(continuous training loop)하는 게 장기 비전.

→ LinkMind 의 모든 설계 결정은 "이게 학습 데이터를 보존/구조화/내보내는 데 도움이 되는가?" 라는 질문을 통과해야 한다.

## 2. 데이터 5대 원칙 (절대 위반 금지)

| 원칙 | 의미 | 강제 위치 |
|---|---|---|
| Raw-first | 원본 텍스트/파일 무손실 보존 | `items.raw_content NOT NULL` |
| Provenance | source_type/source_url/source_id/hash 추적 | schema NOT NULL 제약 |
| Idempotent | 동일 자료 중복 저장 금지 | `UNIQUE(source_type, raw_content_hash)` |
| Versioned analysis | 요약/태깅에 model/prompt 버전 기록 | `summary_model`, `embedding_model` 컬럼 |
| Loss-less storage | 이미지/PDF resize/compress 금지 | `attachments.file_hash` 그대로 |

분석 결과(summary, embedding) 는 재생성 가능하지만 raw 가 깨지면 복구 불가. **항상 raw 를 먼저 저장하고 분석은 그 후.**

## 3. 시스템 경계: LinkMind ↔ OpenClaw

- **LinkMind** = backend knowledge OS. HTTP API (`/ingest`, `/search`, `/ask`) 만 노출.
- **OpenClaw** = frontend agent. 사용자랑 Telegram/Slack/Discord 등에서 대화하고, LinkMind HTTP API 호출.
- **두 시스템은 코드 차원에서 분리**. OpenClaw 가 깨지거나 사라져도 LinkMind 는 영향 없음. 다른 client(Claude Desktop, Cursor, n8n, 자체 봇)로 교체 가능.
- OpenClaw 코드를 LinkMind 저장소에 vendor 하지 않는다. `external/openclaw/` 는 gitignored 참조용 clone.
- OpenClaw 를 `LLMProvider` 추상화에 넣지 않는다 — 그건 client 이지 LLM provider 가 아니다.

## 4. 기술 스택 / 환경

- **OS**: Ubuntu, **GPU**: NVIDIA RTX 4090 (CUDA), **Docker**: nvidia-container-toolkit
- **Backend**: Python 3.11+, FastAPI, SQLAlchemy 2.0 async + asyncpg, pydantic-settings
- **DB**: PostgreSQL 16 (관계형 + raw 본문) + Qdrant 1.12 (벡터)
- **Embedding**: sentence-transformers (bge-m3) → Phase 2 에 TEI 로 전환
- **LLM**: OpenAI / Anthropic / Ollama (provider abstraction)
- **Frontend**: Streamlit (MVP) → Next.js (장기)
- **Object storage**: 로컬 FS → MinIO (Phase 2)

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

```bash
# 인프라 (Postgres + Qdrant + Ollama + OpenWebUI)
docker compose --env-file env/dev.env -f compose/docker-compose.dev.yml up -d

# Phase 2 서비스 (TEI, MinIO) 도 함께
docker compose --env-file env/dev.env -f compose/docker-compose.dev.yml --profile phase2 up -d

# Python 환경 (최초 1회)
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pip install --index-url https://download.pytorch.org/whl/cu124 torch  # CUDA 빌드 별도

# Qdrant 컬렉션 사전 생성 (옵션)
python scripts/init_qdrant.py

# 백엔드 + 프론트
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
streamlit run frontend/app.py

# URL 하나 수동 수집
python -m backend.ingest.url https://arxiv.org/abs/2401.01234

# OpenClaw 설치 (옵션) — 기본은 공식 install.sh (Node 자동 bootstrap)
bash scripts/install_openclaw.sh              # 권장: 개인 사용, zero-friction
bash scripts/install_openclaw.sh --npm        # 팀/CI/재현성 필요 시 (Node 22.16+ 직접 설치 필요)
bash scripts/install_openclaw.sh --source     # external/openclaw/ 에서 dev 빌드 (OpenClaw 자체 수정용)
```

## 9. 디렉토리 구조 요점

```
backend/
├─ api/         # FastAPI routers (health, ingest, search, ask)
├─ config.py    # pydantic-settings 환경설정 (모든 env 진입점)
├─ db/          # connection.py, repository.py, schema.sql
├─ embedding/   # base.py, local.py, factory.py, qdrant_store.py
├─ llm/         # base.py, factory.py, openai/claude/ollama provider
├─ ingest/      # source 별 (url/, slack/, telegram/, pdf/, github/, arxiv/, youtube/)
├─ schemas/     # Pydantic 요청·응답 모델
├─ storage/     # local.py (raw 파일 무손실 보존)
├─ utils/       # chunking, hashing
└─ main.py      # FastAPI 진입점

frontend/app.py            # Streamlit MVP
compose/docker-compose.*   # docker compose 정의
env/                       # dev.env(gitignored) / dev.env.example
scripts/                   # init_qdrant.py, init_db.py, install_openclaw.sh
docs/                      # openclaw_integration.md, training_data_design.md
archive/                   # raw 자료 (gitignored)
volumes/                   # 컨테이너 영속 볼륨 (gitignored)
external/openclaw/         # 참조용 clone (gitignored)
```

## 10. NEVER 목록

- ❌ `.env` 파일 commit
- ❌ commit 메시지 영어 / conventional prefix
- ❌ OpenClaw 코드를 LinkMind 안에 vendor (참조는 `external/` 에서만)
- ❌ `raw_content` 를 NULL 로 두거나 변형해서 저장
- ❌ AI 분석 결과를 model/prompt 버전 없이 저장
- ❌ Telegram/Slack 봇을 LinkMind 안에 직접 만들기 (OpenClaw 위임이 기본)
- ❌ `os.environ.get(...)` 코드에서 직접 사용 (`backend.config` 경유)
- ❌ 이미지/PDF resize·compress (학습 데이터 손실)
- ❌ `--force` / `--no-verify` / `reset --hard` 같은 destructive 명령 자의로

## 11. Phase 별 로드맵

| Phase | 상태 | 핵심 |
|---|---|---|
| 1 | scaffold 완료, 실제 동작 검증 진행 중 | Postgres + Qdrant + URL ingest + Embedding + Search |
| 2 | 다음 | AI 요약/태깅, Slack export 파서, TEI 전환, feedback 테이블, dataset exporter |
| 3 | | OCR/멀티모달, 이미지 분석, RAG 고도화 |
| 4 | | **sVLL LoRA 파인튜닝** (LLaMA-Factory + Qwen2-VL 등), vLLM 서빙 |
| 5 | | Continuous training loop, 온프레미스 AI 엔진 완성 |

## 12. 현재 미완 작업 (다음 세션에 우선 처리)

Phase 1 의 "실제 동작 검증" 까지 가려면 다음이 필요. 모두 사용자가 직접 해야 하는 일이며 코드 변경은 없음:

1. **`env/dev.env` 실제 값 채우기**
   - `POSTGRES_PASSWORD` (현재 `CHANGE_ME_BEFORE_FIRST_RUN`)
   - `DATABASE_URL` / `DATABASE_URL_LOCAL` 의 비밀번호도 동일하게 교체
   - `OPENAI_API_KEY` (없으면 LLM 호출 시 에러; embedding/검색만 쓸 거면 비워둬도 OK)
2. **Python 환경**
   ```bash
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   pip install --index-url https://download.pytorch.org/whl/cu124 torch  # CUDA 빌드 별도
   ```
3. **인프라 컨테이너 기동**
   ```bash
   docker compose --env-file env/dev.env -f compose/docker-compose.dev.yml up -d
   # Postgres 첫 부팅 시 backend/db/schema.sql 자동 import 됨 (compose 의 entrypoint mount)
   ```
4. **Qdrant 컬렉션 생성**
   ```bash
   python scripts/init_qdrant.py  # bge-m3 1.4GB 첫 다운로드 발생
   ```
5. **백엔드 + 프론트 기동**
   ```bash
   uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
   streamlit run frontend/app.py
   ```
6. **첫 URL 수집 동작 확인**
   ```bash
   python -m backend.ingest.url https://arxiv.org/abs/2401.01234
   curl -X POST localhost:8000/search -H 'content-type: application/json' \
        -d '{"query":"transformer","top_k":3}' | jq
   ```

각 단계에서 에러가 나면 그 자리에서 해결 — 회피하지 말고 root cause 파악. 특히 1번 (env) 과 3번 (Postgres healthcheck) 은 첫 부팅에서 자주 막힘.
