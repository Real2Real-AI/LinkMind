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
python scripts/step4_init_qdrant.py
bash scripts/step4_check_qdrant.sh            # 컬렉션 + vector dim 일치

# step5: 백엔드 + 프론트
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
streamlit run frontend/app.py

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
scripts/                   # stepN_setup_*.sh / stepN_check_*.sh + install_openclaw.sh, ollama_pull.sh, init_db.py
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

## 12. 현재 진행 상태 (2026-05-15 기준)

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
- **scripts/backfill_summary.py** — `summary IS NULL` row 들에 retroactive 한국어 요약 (Phase 2 의 다른 ingest 에도 재사용)

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
python scripts/backfill_summary.py            # summary IS NULL 인 row 만
python scripts/backfill_summary.py --force    # 모든 item 재요약

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
   python scripts/step4_init_qdrant.py
   bash scripts/step4_check_qdrant.sh
   ```

6. **step5 — 위 "평소 dev 실행 흐름" 그대로**

### 다음 세션 작업

- **Ask 탭 (RAG) 검증** — Streamlit Ask 탭에서 "이 자료가 다루는 통계 모델이 뭐야?" 질문 → Qdrant 검색 + Ollama qwen2.5:7b 답변 + 인용. 인프라 + 코드 다 준비됨, 한 번 눌러서 검증만.
- **Phase 2 — Slack 데이터 본격 ingest**
  1. `bash scripts/slack_export.sh` 로 재수집 (자동으로 `archive/slack_export/full_<timestamp>/` + `latest` symlink)
  2. `backend/ingest/slack/export_parser.py` 작성 — slackdump standard 포맷 파싱 → LinkMind DB
     - 입력: `archive/slack_export/latest/<channel>/<yyyy-mm-dd>.json` + `attachments/`
     - 출력: items (raw_content=Slack 메시지 원문, source_type=`slack`, source_id=`<team>_<channel>_<ts>`, source_url=`https://<workspace>/archives/<channel>/p<ts>`) + chunks + Qdrant 벡터
  3. thread 댓글 (parent_message_ts → reply), 첨부 다운로드 (필요 시 SLACK_EXPORT_FILE_TOKEN)

각 단계에서 에러 나면 회피하지 말고 root cause 파악. 이번 셋업에서 잡은 결함들 (docker reload, qdrant client v1.12+ API, ollama URL host/docker 분기, HF Hub 시끄러운 로그, search dedup) 다 그렇게 잡힌 것.
