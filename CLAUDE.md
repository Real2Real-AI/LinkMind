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
- **Backend**: Python 3.11+ (검증: 3.13.12 + torch 2.6.0+cu124, NVIDIA driver 580.x), FastAPI, SQLAlchemy 2.0 async + asyncpg, pydantic-settings
- **DB**: PostgreSQL 16 (관계형 + raw 본문) + Qdrant 1.12 (벡터)
- **Embedding**: sentence-transformers (bge-m3) → Phase 2 에 TEI 로 전환
- **LLM**: OpenAI / Anthropic / Ollama (provider abstraction)
- **Frontend**: Streamlit (MVP) → Next.js (장기)
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

# OpenClaw 설치 (옵션) — 기본은 공식 install.sh (Node 자동 bootstrap)
bash scripts/install_openclaw.sh              # 권장: 개인 사용, zero-friction
bash scripts/install_openclaw.sh --npm        # 팀/CI/재현성 필요 시 (Node 22.16+ 직접 설치 필요)
bash scripts/install_openclaw.sh --source     # external/openclaw/ 에서 dev 빌드 (OpenClaw 자체 수정용)

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
scripts/                   # 실행용 .sh 만 — stepN_*.sh / install_openclaw.sh / ollama_pull.sh / slack_export.sh
backend/jobs/              # backend 모듈을 호출하는 batch (init_db / init_qdrant / backfill_* / seed_* / generate_topic_descriptions) — `python -m backend.jobs.<name>`
ai_agents/                 # LinkMind 의 client agent — telegram_inbox_watcher (NEVER §3 정신, backend 외부)
docs/                      # openclaw_integration.md, training_data_design.md
archive/                   # raw 자료 (gitignored)
volumes/                   # 컨테이너 영속 볼륨 (gitignored)
external/openclaw/         # 참조용 clone (gitignored)
```

## 11. NEVER 목록

- ❌ `.env` 파일 commit
- ❌ commit 메시지 영어 / conventional prefix
- ❌ OpenClaw 코드를 LinkMind 안에 vendor (참조는 `external/` 에서만)
- ❌ `raw_content` 를 NULL 로 두거나 변형해서 저장
- ❌ AI 분석 결과를 model/prompt 버전 없이 저장
- ❌ Telegram/Slack 봇을 LinkMind 안에 직접 만들기 (OpenClaw 위임이 기본)
- ❌ `os.environ.get(...)` 코드에서 직접 사용 (`backend.config` 경유)
- ❌ 이미지/PDF resize·compress (학습 데이터 손실)
- ❌ `--force` / `--no-verify` / `reset --hard` 같은 destructive 명령 자의로

## 12. Phase 별 로드맵

| Phase | 상태 | 핵심 |
|---|---|---|
| 1 | scaffold 완료, 실제 동작 검증 진행 중 | Postgres + Qdrant + URL ingest + Embedding + Search |
| 2 | 다음 | AI 요약/태깅, Slack export 파서, TEI 전환, feedback 테이블, dataset exporter |
| 3 | | OCR/멀티모달, 이미지 분석, RAG 고도화 |
| 4 | | **sVLL LoRA 파인튜닝** (LLaMA-Factory + Qwen2-VL 등), vLLM 서빙 |
| 5 | | Continuous training loop, 온프레미스 AI 엔진 완성 |

## 13. 현재 진행 상태 (2026-05-16 기준)

> 자세한 backlog 와 phase 별 완료 항목 / 미구현 항목은 `docs/features_backlog.md` 가
> source of truth. 여기는 큰 그림만.

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
비번 `real2real` 로 단순화.
**RAG 답변 품질 개선 (2026-05-16)** — `/ask` 의 context block 에 item 의 한국어
summary (500-1500자) + tags 추가. 이전엔 chunk snippet (300자) 만 들어가 LLM 이
자료의 깊이 활용 못 하고 일반 정의로 답하던 문제 해결. `rag_system` v3 prompt —
'자료 구체 인용 + 자료들이 다루는 측면' 두 단락 강제. 다만 `qwen2.5:14b` 모델
응답 시간 ~3분 — 더 가벼운 ask 전용 모델은 Phase 2 후반/Phase 4 (sVLL) 로.

**리팩토링 (2026-05-16)** — `scripts/` 는 .sh 만, batch python 6개 (`backfill_*`,
`seed_*`, `generate_*`, `init_db`, `init_qdrant`) 는 `backend/jobs/` 로,
telegram watcher 는 `ai_agents/` 로 (NEVER §3 정신 — backend 외부 agent). 호출은
`python -m backend.jobs.<name>`. 5 카테고리 (cpu/embedding/integration/llm/gpu)
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

**Phase C wave-2 — Slack 워크스페이스 ingest** (2026-05-16 시작점 갱신):
- ⚠️ 옛 `archive/slack_export/public_2026-05-14/` (182 채널 slackdump export) 는
  사용자가 삭제. wave-2 의 시작점은 **Slack API 직접 호출로 현재 워크스페이스
  전체 데이터 ingest**.
- 옵션 A: `bash scripts/slack_export.sh` 재 export → `backend/ingest/slack/export_parser.py`
  파싱 (검증된 패턴, 빠름. 다만 incremental sync 어려움 — 매번 full export).
- **옵션 B (권장)**: `slack_sdk` 직접 — `conversations.list` / `history` /
  `replies` 를 채널별 incremental (`oldest` = 마지막 ts). 추후 `ai_agents/`
  의 realtime watcher 도 같은 코드 베이스에서.
- `backend/ingest/slack/__init__.py` (Telegram 패턴 일관) — `SlackMessage`
  dataclass + `ingest_slack_message` + `ingest_slack_channel` + `ingest_slack_workspace`.
- thread (`thread_ts`) → reply 묶음, URL 자동 라우팅 (telegram 흐름 재사용), 첨부 download.
- 단위 테스트 + 작은 fixture (channels.list / conversations.history 응답 모사).
- 검증: 단일 채널 먼저 → 워크스페이스 전체.

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
