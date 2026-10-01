# sandboxd 전수조사 분석 & 활용 정리 (한국어)

> 이 문서는 `bmshin94/sandboxd` 레포지토리를 전수조사하여
> **① 정체 ② 작동 원리 ③ 설치·사용법 ④ 활용 방향 ⑤ 수익화 전략**을
> 한국어로 정리한 자료입니다.
>
> - **원본(업스트림) 레포:** https://github.com/tastyeffectco/sandboxd
> - **이 포크:** https://github.com/bmshin94/sandboxd
> - **공식 문서/데모:** https://sandboxd.io · https://sandboxd.io/demo/
> - **분석 기준 버전:** v0.3.20 (베타 0.x) · **라이선스:** MIT
> - **작성일:** 2026-10-01

---

## 목차

1. [sandboxd란 무엇인가](#1-sandboxd란-무엇인가)
2. [레포지토리 구조 전수조사](#2-레포지토리-구조-전수조사)
3. [작동 원리](#3-작동-원리)
4. [기술적 하이라이트 4가지](#4-기술적-하이라이트-4가지)
5. [쉬운 비유로 이해하기](#5-쉬운-비유로-이해하기)
6. [설치 및 사용법](#6-설치-및-사용법)
7. [Q&A — 자주 묻는 7가지](#7-qa--자주-묻는-7가지)
8. [수익화 아이디어 8선](#8-수익화-아이디어-8선)
9. [리스크 체크리스트](#9-리스크-체크리스트)
10. [실행 로드맵](#10-실행-로드맵)

---

## 1. sandboxd란 무엇인가

### 한 줄 정의

> **Lovable / Bolt / v0 / Replit 처럼 "프롬프트를 넣으면 AI가 앱을 만들어주는 서비스"의
> 엔진을, 내 서버에서 직접 돌리는 오픈소스 구현체.**

HTTP 요청 한 번으로 다음 3단계가 자동 수행된다.

1. **격리된 전용 컨테이너**를 띄운다 (자체 파일시스템 + 리소스 상한)
2. 그 안에서 **AI 코딩 에이전트**가 프롬프트대로 코드를 작성한다
3. 앱에 **라이브 프리뷰 URL**을 발급한다

유휴 샌드박스는 자동으로 잠들고(`docker stop`) 다음 요청에 깨어나므로,
**앱 1개당 VM 1대가 아니라 평범한 서버 1대에 앱 수십 개**를 올릴 수 있다.

### 규모 (실측)

| 항목 | 수치 |
|---|---|
| Go 소스 | **198 파일 / 약 34,900줄** |
| 콘솔(React/TS) | 41 파일 (`.ts` 22 + `.tsx` 19) |
| `/v1` API 엔드포인트 | **53개** (`docs/openapi.yaml`) |
| 런타임 자동탐지 레시피 | **43개** YAML |
| 앱 스토어 카탈로그 | **80+ 개** (`console/src/catalog*.ts`) |
| DB 마이그레이션 | 23개 `.sql` |
| 설계 문서 | 13개 (`docs/`) + ADR |

### 기술 스택

| 레이어 | 기술 |
|---|---|
| 컨트롤 플레인 | **Go 1.22** (단일 바이너리, `docker` CLI 직접 호출 — SDK 미사용) |
| 상태 저장소 | **SQLite (WAL)** — 유일한 진실의 원천 |
| 엣지 라우터 | **Traefik** (Docker 라벨 프로바이더) |
| 웹 콘솔 | **React 18 + Vite 5 + TypeScript 5** + CodeMirror 6 + xterm.js |
| 샌드박스 내부 | `runtimed` (Go, 컨테이너 1번 프로세스) |
| 의존성 | websocket, go-sqlite3, ulid, prometheus, crypto, yaml — **단 8개** |

> 의도적으로 작게 설계됨: **Kubernetes 없음, 별도 DB 없음, 메시지 큐 없음.**

---

## 2. 레포지토리 구조 전수조사

### 최상위

| 경로 | 정체 | 역할 |
|---|---|---|
| `control-plane/` | **심장 (Go)** | 전체 코드의 약 90%. Docker를 조종 |
| `console/` | 웹 UI (React) | **순수 `/v1` 클라이언트.** 없어도 엔진은 동작 |
| `image/` | 샌드박스 베이스 이미지 | Dockerfile + 템플릿 5종 + PHP/Ruby 추가 레이어 |
| `traefik/` | 엣지 설정 | 프리뷰 라우팅 + wake 캐치올(`dynamic/wake.yml`) |
| `docs/` | 설계 문서 13개 | `openapi.yaml`, `agent-auth.md`, `isolation.md`, `gvisor.md` 등 |
| `deploy/` | VPS 원클릭 | `cloud-init.yaml`, `bootstrap.sh`, `DEPLOY.md` |
| `scripts/` | 보조 스크립트 | 릴리스/테스트 하네스 |
| `install.sh` | 설치 | 멱등. Docker 체크 → 이미지 빌드 → `compose up -d` |
| `upgrade.sh` | 업그레이드 | **DB 백업 → 헬스체크 → 실패 시 자동 롤백** |
| `uninstall.sh` | 제거 | `--images` / `--data` / `--all` / `--yes` |
| `console-login.sh` | 로그인 조회 | `--reset-password`로 비번 복구 |
| `docker-compose.yml` | 스택 정의 | 서비스 3개: `traefik`, `sandboxd`, `console`(프로필) |
| `.env.example` | 설정 카탈로그 | **26개 키** 전부 주석 설명 포함 |

### `control-plane/` 내부

```
control-plane/
├─ cmd/
│  ├─ sandboxd/     메인 데몬 (/v1 API 서버 + 샌드박스 생명주기)
│  └─ runtimed/     샌드박스 내부 수퍼바이저 + 태스크 러너
│                   ├─ claude.go / opencode.go / codex.go  ← 에이전트 어댑터 3종
│                   ├─ agentenv.go   ← 크레덴셜형 env 스크럽
│                   ├─ manifest.go   ← sandbox.yaml 파싱
│                   ├─ process.go    ← web/workers 프로세스 감시
│                   ├─ workspace.go  ← 태스크 체크포인트(git private ref)
│                   └─ health.go     ← build_status / preview_ok / app_healthy
├─ internal/        37개 모듈 (아래)
└─ migrations/      23개 SQL 마이그레이션
```

### `internal/` 37개 모듈 (핵심)

| 모듈 | 역할 |
|---|---|
| `api/` | `/v1` HTTP 핸들러 (`handlers.go` 46KB, `v1*.go` 20여 개) |
| `docker/` | `docker` CLI 셸아웃 — SDK 미사용(의도적) |
| `store/` | SQLite(WAL) 접근 — 유일한 진실의 원천 |
| `traefik/` | 컨테이너 라벨 생성 → 프리뷰 라우터 자동 등록 |
| `reaper/` | **유휴 수거**(idle → stop) + **메모리 압박 수거** |
| `wake/` | 꺼진 샌드박스 요청 시 `docker start` + "준비 중" 페이지 |
| `authproxy/` | **크레덴셜 주입 리버스 프록시** (핵심 보안 설계) |
| `agentauth/` | 제공자별 자격증명 저장 (워크스페이스 밖, 암호화) |
| `secrets/` | 앱별 시크릿 — 암호화 저장, **쓰기 전용**(값 조회 불가) |
| `snapshot/` | 스냅샷 / 포크 / 복원 (의존성·빌드 산출물 제외) |
| `recipes/` | 43개 런타임 자동탐지 레시피 (`data/*.yaml`) |
| `preset/` | 런타임 프리셋 5종 (React·Next·Express·FastAPI·Worker) |
| `reconcile/` | 부팅 시 Docker ↔ SQLite 정합성 수렴 |
| `loopback/` | 워크스페이스 프로비저닝 (스켈레톤 시딩 + 바인드 마운트) |
| `egress/` | nftables 송신 제어 (**구현됨, OSS 빌드에서는 비활성**) |
| `gitimport/` | 공개 레포 토큰리스 클론 / 비공개 레포 암호화 PAT |
| `events/` | 앱별 활동 타임라인 (durable, newest-first) |
| `audit/` | 감사 로그 |
| `metrics/` | Prometheus 메트릭 |
| `previewhost/` | 프리뷰 호스트명 규칙 (서브도메인/플랫 스타일) |
| `cgroup/` | `memory.high` 소프트 스로틀 (opt-in) |
| `upgrade/` | 릴리스 노트 / breaking change 노출 |
| `telemetry/` | PostHog (opt-out 가능) |

### `image/` — 샌드박스 베이스 이미지

```
image/
├─ Dockerfile            베이스 이미지 (sandboxd-base:0.3.0)
├─ php/Dockerfile        PHP 레이어 (sandboxd-php:0.4.0)
├─ ruby/Dockerfile       Ruby 레이어 (sandboxd-ruby:0.4.0)
├─ templates/            프리셋 스타터 5종
│  ├─ react-standard/      index.html, package.json, pnpm-lock.yaml, src/
│  ├─ nextjs-standard/     app/layout.js, app/page.js, package.json
│  ├─ node-express-standard/
│  ├─ fastapi-standard/    main.py, requirements.txt
│  └─ worker-standard/
├─ skel/workspace/       워크스페이스 초기 스켈레톤
├─ etc/                  npmrc, pip.conf, profile.d/sandbox-env.sh
└─ verify-base.sh        이미지 검증
```

**PHP 레이어가 해금하는 앱:** Dokuwiki, FreshRSS, Grocy, Heimdall, MediaWiki,
Organizr, PrivateBin, Shlink, Vvveb, ClassicPress, Drupal
→ 실행 방식: `php -S 0.0.0.0:3000 -t <docroot>`

**Ruby 레이어가 해금하는 앱:** Redmine, Docuseal, Fizzy, Once-Campfire, Sessy

---

## 3. 작동 원리

### 전체 토폴로지

```
                     ┌── 호스트 (Docker 데몬) ──────────────────────┐
브라우저 ──:80──▶  :80│ traefik ──┬─▶ s-<id>-3000 (실행 중 샌드박스)  │
                      │           │      ▲ dev 서버 :3000            │
API/CLI  ──:9090─▶ sandboxd ──────┼──────┘                           │
                      │           └─▶ /forward-auth, /wake (캐치올)  │
                      │  SQLite (진실의 원천)                         │
                      │  리퍼: 유휴(stop) + 메모리 압박               │
                      │  workspaces/<id>/ (바인드 마운트, 영구 보존)   │
                      └──────────────────────────────────────────────┘
```

### "프롬프트 → 앱" 흐름

| 단계 | API | 내부 동작 |
|---|---|---|
| 1 | `POST /sandbox` | 하드닝된 컨테이너 생성, 워크스페이스 시딩, Traefik 라벨 부여 |
| 2 | `POST /v1/sandboxes/{id}/tasks` | **작업 전 git 체크포인트 커밋** → 에이전트 실행 |
| 3 | — | `runtimed`가 `sandbox.yaml` 읽고 `web` 프로세스 기동 |
| 4 | — | Traefik이 priority-100 라우터 발행 |
| 5 | 브라우저 | `http://s-<id>-3000.preview.localhost` → 라이브 앱 |

### 꺼진 샌드박스에 첫 요청이 왔을 때 (sleep/wake)

1. 브라우저 → `http://s-<id>-3000.preview.localhost`
2. 컨테이너가 꺼져 있어 priority-100 라우터가 없음 → **Traefik 캐치올(priority-1)** 매치
3. 요청이 `sandboxd:9000`의 wake 경로로 포워드
4. sandboxd가 **메모리 여유 확인(wake admission)** → `docker start` → 포트 폴링
5. "Spinning up your app…" 스타일 페이지 반환 (자동 새로고침)
6. 시작된 컨테이너의 라벨로 Traefik이 라우터 발행 → 다음 새로고침부터 직결

### 런타임 모델 (생성 시점 → 실행 시점 레이어링)

1. **베이스 이미지** — 도구 + `runtimed` + 프리셋 템플릿 (`docs/base-image.md`)
2. **런타임 프리셋** — 템플릿 + 생성된 `sandbox.yaml` + capabilities
3. **`sandbox.yaml`** — `web`(프리뷰 대상) 1개 + `workers` N개 + `build` + `health` + `restart_after_task`
4. **스타터/임포트** — 프리셋 템플릿 시딩 또는 Git 클론
5. **스냅샷** — 워크스페이스 상태 캡처 (`node_modules`, `.next`, `.venv`, `__pycache__` 등 제외)

### 저장 영속성

| 분류 | 위치 | stop 생존 | 재부팅 생존 |
|---|---|---|---|
| 워크스페이스 | `DATA_DIR/workspaces/<id>/` (바인드) | ✅ | ✅ |
| 컨트롤플레인 상태 | `DATA_DIR/state/sandboxd.db` | ✅ | ✅ |
| 컨테이너 쓰기 레이어 | 없음 (`--read-only`) | ❌ | ❌ |
| `/tmp`, `/var/tmp` | tmpfs | ❌ | ❌ |

> 샌드박스 내부에서 쓰기 가능한 유일한 디스크 위치는 `/home/sandbox`.

---

## 4. 기술적 하이라이트 4가지

### ① 크레덴셜이 샌드박스에 절대 들어가지 않는다 🔐

이 프로젝트의 가장 중요한 설계 결정.

```
샌드박스가 보는 것 : ANTHROPIC_BASE_URL (프록시 주소) + 더미 키
진짜 키가 있는 곳   : 컨트롤플레인 (DATA_DIR/agent-auth/<provider>/, 암호화)
키 주입 시점        : 네트워크 통과 시 프록시가 Authorization 헤더에 주입
```

결과: 에이전트가 폭주해도 **파일시스템 / 환경변수 / 스냅샷 / 로그 / 태스크 결과
어디에도 진짜 키가 존재하지 않는다.**

추가 방어: `*_KEY`, `*_TOKEN`, `*_SECRET` 형태의 환경변수는 에이전트 프로세스에서
**자동 스크럽**된다(`cmd/runtimed/agentenv.go`). 과거처럼 샌드박스 생성 시
`env`로 `ANTHROPIC_API_KEY`를 넣어도 에이전트에 닿지 않는다.

Git PAT도 동일 원칙: **네트워크 git(클론/푸시)은 호스트에서만** 실행되고
(`GIT_ASKPASS` + `0600` 파일), **로컬 git(status/diff/commit)은 샌드박스 내부**에서
크레덴셜 없이 실행된다.

### ② 모든 AI 태스크가 되돌려진다 ⏪

태스크 실행 직전 워크스페이스를 **비공개 git ref**로 커밋한다.

```
refs/sandboxd/checkpoints/<taskId>
  ↑ commit-tree + 격리된 index로 생성
  ↑ HEAD / 브랜치 히스토리 / 사용자 스테이징 영역을 전혀 건드리지 않음
```

이 체크포인트가 두 가지 역할을 한다.
- `files_changed` 계산의 **기준선**
- `POST /v1/sandboxes/{id}/tasks/{taskId}/revert` 의 **복원 지점**

스냅샷(워크스페이스 전체 동결)이나 사용자 커밋과는 명확히 구분되는 별개 메커니즘.

### ③ sleep/wake로 유휴 비용이 0 💤

두 개의 리퍼가 돈다.
- **유휴 리퍼** — 임계값(`SANDBOXD_IDLE_THRESHOLD_SECONDS`)을 넘긴 샌드박스를 `docker stop` → RAM 회수
- **압박 리퍼** — 호스트 메모리가 부족해지면 샌드박스를 정지

깨우기는 Traefik 캐치올 + wake 경로가 담당하므로 **사용자는 URL만 누르면 된다.**
이 덕분에 "유저 100명 / 앱 300개"여도 동시 활성은 10~20개뿐 → **서버 2대로 커버 가능.**

### ④ 격리를 제대로 잠근다 (단, 한계를 솔직히 명시) 🔒

```
--cap-drop=ALL
--security-opt=no-new-privileges
--read-only 루트파일시스템 (+ /tmp는 tmpfs)
--memory 하드 상한
--pids-limit
파일 디스크립터 ulimit
--userns=host (워크스페이스 소유권 결정성 확보)
```

**위협 모델이 문서에 명시되어 있다:** "인증되고 책임 있는 사용자가 자기 코드를
실행하는 상황" — 익명 적대적 멀티테넌시가 아님. 커널 CVE 기반 컨테이너 탈출은
**VM 경계가 아니라 패치로 완화**한다고 솔직히 적혀 있고, 더 강한 격리가 필요하면
신뢰 도메인별 전용 VM을 권고한다. (gVisor / Kata / Firecracker는 로드맵)

### v0.3 설계상 한계 (문서에 명시된 트레이드오프)

| 영역 | v1 선택 | 하드닝 방법 |
|---|---|---|
| 워크스페이스 디스크 | 일반 디렉터리 | **쿼터 없음** → fs/volume 레이어에서 설정 |
| 메모리 | 하드 `--memory` | `memory.high` 소프트 스로틀은 opt-in |
| 송신(egress) | **기본 허용, 로깅 없음** | 호스트 방화벽 / 프록시 추가 |
| TLS/도메인 | HTTP `*.localhost` | 와일드카드 도메인 + cert resolver |
| 스냅샷 | API 존재, **실험적** | 워크스페이스 디렉터리 복사 사용 |
| 프리뷰 포트 | **샌드박스당 1개** | 단일포트 모드 또는 내부 리버스프록시 |
| 베이스 이미지 | **인스턴스 전체에 1개** | 앱별 이미지 선택은 로드맵 |
| 데이터베이스 | **앱 내장 SQLite만** | Postgres/MySQL/Redis는 커스텀 이미지/외부 |

> 단일 포트에서 **HTTP / WebSocket 업그레이드 / SSE 모두 동작 확인됨**
> (Streamlit·Jupyter 커널 WS → 101, Reflex·NiceGUI·Chainlit socket.io → 101,
> Sanic 네이티브 WS → 101 + echo, Gradio SSE 큐 스트리밍)

---

## 5. 쉬운 비유로 이해하기

### "AI 셰어하우스" 모델

```
🏢 셰어하우스 건물 = 내 서버 1대

  🚪 프론트 데스크 (Traefik)      "301호 손님이세요? 안내할게요"
  👔 건물 관리소장 (sandboxd, Go)  방 배정 / 입주 / 퇴실 / 장부
  📒 장부 (SQLite)                "301호=김앱" 유일한 공식 기록
  🛏️ 각 방 (샌드박스 = 컨테이너)
       🤖 집사 1명 (runtimed) + 🏗️ AI 인테리어 기사 (Claude Code)
```

손님이 **"투두 앱 하나 만들어줘"** 한 마디 하면:

| 단계 | 셰어하우스 | 실제 |
|---|---|---|
| 1 | 빈 방 하나 열어줌 | `docker run` 컨테이너 생성 |
| 2 | AI 인테리어 기사 투입 | Claude Code / OpenCode 실행 |
| 3 | **작업 전 방 사진 촬영** 📸 | git 체크포인트 커밋 |
| 4 | 가구 배치 | 파일 생성·수정 |
| 5 | 집사가 조명 켬 | dev 서버 실행 |
| 6 | **방 주소 발급** 🔑 | `s-<id>-3000.preview.localhost` |

### 핵심 아이디어를 비유로

| 기술 | 비유 |
|---|---|
| **sleep/wake** | 아무도 안 쓰는 방은 불을 끔. 주소 누르면 "방 준비 중 ☕" 뜨고 다시 켜짐 |
| **authproxy** | 인테리어 기사한테 **가짜 열쇠**만 주고 "결제는 데스크에서". 방을 다 뒤져도 진짜 열쇠가 **존재하지 않음** |
| **체크포인트/revert** | 작업 직전 방 사진으로 되돌리기. 이 사진은 **몰래 숨긴 선반**(private ref)에 보관해서 내 기록을 안 건드림 |
| **앱 스토어 80+** | 처음부터 짓는 것만이 아니라 **완성 가구**도 클릭 한 번에 들여놓기 (Ghost, n8n, Grafana, Gitea, Jupyter…) |

---

## 6. 설치 및 사용법

### 준비물

- **Linux** 호스트 (macOS는 Docker Desktop으로 best-effort)
- **Docker Engine + Compose 플러그인**, **git**
- **2 vCPU / 4 GB RAM** 이면 시작 충분
- **amd64 / arm64 네이티브** (Apple Silicon, AWS Graviton 포함 — 크로스 컴파일 없음)
- Docker 실행 권한 (`docker` 그룹 또는 `sudo` — 스크립트가 자동 감지)

### 설치 (1줄)

```bash
curl -fsSL https://raw.githubusercontent.com/tastyeffectco/sandboxd/main/install.sh | bash
```

또는 이 포크를 클론해서:

```bash
git clone https://github.com/bmshin94/sandboxd.git
cd sandboxd
./install.sh
```

`install.sh`는 **멱등**하며 다음을 수행한다.

1. Docker 확인
2. `.env.example` → `.env` 복사
3. 베이스 이미지 빌드 (`sandboxd-base:0.3.0` — 첫 빌드는 수 분, 이후 캐시)
4. 컨트롤 플레인 빌드
5. 데이터 디렉터리 생성
6. `docker compose up -d` (**웹 콘솔 포함**)

완료 시 **콘솔 URL + 자동 생성된 로그인 정보**를 출력한다 (비밀번호 설정 단계 없음).

### 설치 확인

```bash
curl -s http://127.0.0.1:9090/healthz   # → ok
curl -s http://127.0.0.1:9090/readyz    # → ready
curl -s http://127.0.0.1:9090/version   # → 빌드 버전 + 커밋 (JSON, 인증 불필요)
```

### 사용법 A — 웹 콘솔 (노코드, 입문 추천)

| 주소 / 명령 | 용도 |
|---|---|
| `http://console.localhost` | 웹 콘솔 |
| `./console-login.sh` | 로그인 정보 재확인 |
| `./console-login.sh --reset-password` | 비밀번호 분실 복구 |

콘솔에서 가능한 작업 (소스 확인 기준):

- 앱 생성 / 프리셋 선택 / **앱 스토어 80+ 원클릭 설치**
- **에이전트 채팅** + 태스크 히스토리 + **태스크별 revert**
- **파일 트리 + CodeMirror 에디터** (인라인 git status/diff, lazy-load)
- **git diff 뷰어 → commit → push**
- 앱별 config / secrets 관리 (암호화, 쓰기 전용)
- 스냅샷 / 포크 / 복원
- 활동 타임라인, 프로세스별 로그
- **웹 터미널** (xterm.js, `/v1/sandboxes/{id}/terminal` WebSocket)
- 설정 조회 (수정은 생명주기 튜너블만)

### 사용법 B — API (제품에 임베드할 때)

```bash
API=http://127.0.0.1:9090

# 1. 에이전트 연결 (최초 1회)
curl -s -XPOST $API/v1/agents/claude-code/api-key -d '{"api_key":"sk-ant-..."}'

# 2. 샌드박스 생성
ID=$(curl -s -XPOST $API/sandbox -H 'content-type: application/json' \
       -d '{"ports":[3000]}' | sed -E 's/.*"id":"([^"]+)".*/\1/')

# 3. AI에게 태스크 전달
curl -s -XPOST $API/v1/sandboxes/$ID/tasks -H 'content-type: application/json' \
  -d '{"prompt":"build a todo app on port 3000","agent":"claude-code"}'

# 4. 결과 확인
#    http://s-$ID-3000.preview.localhost

# 5. 정리
curl -s -XPOST $API/v1/sandboxes/$ID/stop    # 재우기 (RAM 회수)
curl -s -XPOST $API/sandbox/$ID/purge        # 컨테이너 + 워크스페이스 완전 삭제
```

### 주요 API 엔드포인트 (전체 53개 중)

| 메서드 & 경로 | 용도 |
|---|---|
| `POST /sandbox` | 생성 (`ports`, `env`; `id` 생략 시 ULID 자동) |
| `GET /sandboxes` · `GET /sandbox/{id}` | 목록 / 단건 |
| `POST /sandbox/{id}/exec` | 명령 실행 (비대화형, TTY/stdin 없음) |
| `POST /sandbox/{id}/keepalive` | 유휴 리퍼 연기 |
| `POST /v1/sandboxes/{id}/stop` · `/start` | 정지 / 시작 |
| `DELETE /sandbox/{id}` | 컨테이너 삭제, **워크스페이스 유지** |
| `POST /sandbox/{id}/purge` | 컨테이너 + 워크스페이스 삭제 |
| `GET/PUT /v1/sandboxes/{id}/files` · `/files/content` | 워크스페이스 파일 목록/읽기/쓰기 |
| `POST /v1/sandboxes/{id}/tasks` | AI 코딩 태스크 실행 |
| `GET /v1/sandboxes/{id}/tasks/{taskId}/events` | 태스크 진행 스트림 (SSE) |
| `POST .../tasks/{taskId}/cancel` · `/revert` | 취소 / 되돌리기 |
| `GET /v1/sandboxes/{id}/processes/{name}/logs` | 프로세스별 로그 |
| `GET /v1/sandboxes/{id}/terminal` | 웹 터미널 (WebSocket) |
| `GET /v1/presets` | 런타임 프리셋 목록 |
| `GET /v1/runtime/recipes` | 런타임 탐지 레시피 |
| `POST /v1/apps` · `GET /v1/apps/{id}` | 앱 CRUD |
| `POST /v1/apps/{id}/snapshots` · `/restore` · `/fork` | 스냅샷 / 복원 / 포크 |
| `GET /v1/apps/{id}/events` | 활동 타임라인 |
| `GET/POST /v1/apps/{id}/git/status` · `/diff` · `/commit` · `/push` | Git 워크플로 |
| `GET/PUT /v1/apps/{id}/config` · `/config/{key}` | 앱 config / secrets |
| `POST /v1/agents/{provider}/api-key` · `/oauth/start` · `/oauth/finish` | 에이전트 연결 |
| `GET/POST /v1/api-keys` | sandboxd API 키 발급 |
| `POST /v1/auth/login` · `/setup` · `/password` | 콘솔 인증 |
| `GET /v1/upgrade` | 릴리스 노트 / breaking change |
| `GET /healthz` · `/readyz` · `/version` | 헬스 / 레디 / 버전 |

### 운영 명령어

```bash
docker compose logs -f sandboxd                  # 컨트롤 플레인 로그
docker compose ps                                # 스택 상태
docker compose restart sandboxd                  # 재시작
docker ps --filter label=sandboxd.managed=true   # 실행 중 샌드박스 목록

./upgrade.sh            # 업그레이드 (DB 백업 + 헬스체크 + 실패 시 자동 롤백)
./upgrade.sh --check    # 현재 버전 확인

./uninstall.sh              # 스택 정지 + 샌드박스 제거 (워크스페이스 유지)
./uninstall.sh --images     # 빌드된 이미지도 제거
./uninstall.sh --data       # 워크스페이스 + 상태 삭제 (확인 프롬프트)
./uninstall.sh --all --yes  # 전부, 프롬프트 없이
```

### `.env` 주요 설정 (전체 26개 중)

| 키 | 기본값 | 설명 |
|---|---|---|
| `HTTP_PORT` | `80` | 80번이 사용 중이면 변경 (프리뷰 URL에 포트 포함됨) |
| `SANDBOXD_API_BIND` | `127.0.0.1:9090` | API 바인딩 주소 |
| `PREVIEW_DOMAIN` | `localhost` | 프리뷰 도메인 |
| `PREVIEW_HOST_STYLE` / `PREVIEW_HOST_TAG` | — | 서브도메인 / 플랫 호스트 스타일 |
| `PREVIEW_TLS` / `PREVIEW_URL_SCHEME` | — | TLS 전환 |
| `CONSOLE_HOST` | `console.<domain>` | 콘솔 호스트명 |
| `SANDBOXD_API_AUTH_DISABLED` | `true` ⚠️ | **공개 서버에서는 반드시 `false`** |
| `SANDBOXD_API_TOKENS` | — | `name:secret` 형식 |
| `SANDBOXD_SECRETS_KEY` | — | 시크릿 암호화 키 |
| `SANDBOXD_IMAGE` | `sandboxd-base:0.3.0` | 베이스 이미지 (PHP/Ruby 레이어로 교체 가능) |
| `SANDBOXD_DEFAULT_AGENT` | `opencode` | 기본 에이전트 |
| `SANDBOXD_IDLE_THRESHOLD_SECONDS` | — | 유휴 판정 시간 |
| `SANDBOXD_SET_MEMORY_HIGH` | off | `memory.high` 소프트 스로틀 (호스트 cgroup 접근 필요) |
| `SANDBOXD_USERNS` | `host` | 비우면 데몬 기본값 |
| `SANDBOXD_DATA_DIR` / `SANDBOXD_LOG_DIR` | — | 데이터 / 로그 경로 |
| `SANDBOXD_TELEMETRY` | — | 텔레메트리 opt-out |

### 헤드리스 실행 (콘솔 없이)

```bash
SANDBOXD_CONSOLE=0 ./install.sh     # 또는 --no-console
docker compose up -d                # 콘솔 프로필 제외
docker compose --profile console up -d   # 콘솔 포함
```

### VPS 배포

```bash
# 방법 1: 서버 생성 시 user-data에 deploy/cloud-init.yaml 붙여넣기 (자동 설치)
# 방법 2: 생성된 서버에서
curl -fsSL https://raw.githubusercontent.com/tastyeffectco/sandboxd/main/deploy/bootstrap.sh | sudo bash
```

제공자별 가이드: `deploy/DEPLOY.md`

### 트러블슈팅

| 증상 | 원인 / 해결 |
|---|---|
| `readyz`가 ready 아님 / `docker info: exit status 1` | 컨트롤 플레인이 Docker 소켓 접근 불가 → `/var/run/docker.sock` 마운트 및 데몬 확인 |
| 포트 80 사용 중 | `.env`의 `HTTP_PORT` 변경 후 `docker compose up -d` (프리뷰 URL에 포트 추가) |
| `id must be a ULID` | 비-ULID `id`를 전달 → `id` 생략해서 자동 생성 |
| "Spinning up your app…" 계속 표시 | 샌드박스가 깨어나는 중이거나, 해당 포트에 아무것도 리스닝 안 함 |
| 생성 시 시딩/권한 오류 | 데몬이 `userns-remap` 사용 → `SANDBOXD_USERNS=host` 기본값 유지 |

---

## 7. Q&A — 자주 묻는 7가지

### Q1. 플러그인인가? 스킬인가? MCP인가?

**셋 다 아니다. 독립적인 서버 제품(데몬)이다.**

| 분류 | 여부 | 근거 |
|---|---|---|
| 플러그인 | ❌ | 무언가에 끼워 넣는 구조가 아님. 자체 데몬 + 자체 API |
| 스킬 (Claude Skill) | ❌ | 마크다운 지침서가 아니라 Go 바이너리 34,900줄 |
| MCP 서버 | ❌ | MCP 프로토콜 구현 코드가 레포에 전혀 없음 (전수 확인) |
| **인프라 플랫폼** | ✅ | Docker 위에서 동작하는 PaaS 엔진 |

**관계는 방향이 반대다.**

```
❌ sandboxd가 Claude에 붙는 것이 아니라
✅ Claude Code가 sandboxd 안에서 실행되는 쪽
```

`cmd/runtimed/claude.go`, `opencode.go`, `codex.go` 가 각 CLI를 기동하는 어댑터다.

**단, MCP 서버로 감쌀 수는 있다.** `/v1` 53개 엔드포인트가 깔끔한 REST이므로
얇은 MCP 브릿지를 만들면 Claude Desktop에서 직접 샌드박스를 만들고 앱을 띄울 수 있다.
(→ 수익화 아이디어 4번)

| 개념 | 비유 |
|---|---|
| MCP | Claude의 **손** (도구를 쓰게 해주는 규격) |
| 스킬 | Claude의 **지식** (방법을 알려주는 문서) |
| **sandboxd** | Claude가 들어가서 일하는 **작업장** |

### Q2. API 토큰을 꼭 써야 하나?

**세 종류가 있고 성격이 다르다.**

**① sandboxd 자체 API 토큰 → 선택 (기본 OFF)**

```env
SANDBOXD_API_AUTH_DISABLED=true    # 기본값 — 인증 없음
```

로컬 단독 사용이면 그대로 동작한다. 다만 **공개 서버라면 반드시 켜야 한다.**

```env
SANDBOXD_API_AUTH_DISABLED=false
SANDBOXD_API_TOKENS=myapp:<secret>
```

```bash
curl -H "Authorization: Bearer <secret>" $API/v1/apps
```

콘솔은 별도로 비밀번호 로그인(`/v1/auth/*`)을, 프로그램 접근은 `/v1/api-keys` 발급을 지원한다.

**② AI 모델 제공자 자격증명 → AI를 쓸 경우 필요 (3가지 선택지)**

| 방법 | 비용 | 명령 |
|---|---|---|
| **OpenCode 무료 플랜** | **0원** | 기본으로 동작 (`SANDBOXD_DEFAULT_AGENT=opencode`) |
| Anthropic API 키 | 종량제 | `POST /v1/agents/anthropic/api-key` |
| Claude 구독 OAuth (PKCE) | 구독료만 | `POST /v1/agents/claude-code/oauth/start` → `/oauth/finish` |

저장 위치는 `SANDBOXD_DATA_DIR/agent-auth/<provider>/` — **워크스페이스 밖, 암호화,
API로 조회 불가.** 샌드박스는 프록시 주소 + 더미 키만 본다.

> ⚠️ 과거 방식(샌드박스 `env`로 `ANTHROPIC_API_KEY` 주입)은 **스크럽되어 더 이상 동작하지 않는다.**
> 반드시 `/v1/agents/...` 로 연결해야 한다.

**③ Git PAT → push 할 때만 필요**

`/v1/git-credentials`에 저장(암호화, 소유자 스코프). push는 **새 브랜치로만**,
force-push 없음, PR 생성 없음. 저장된 레포 URL만 대상으로 하며
`.git/config`의 origin은 사용하지 않는다. **HTTPS + PAT만** (SSH/App/OAuth 미지원).

**정리**

| 상황 | 토큰 필요? |
|---|---|
| 로컬 단독, 앱 실행만 | ❌ 불필요 |
| 로컬 + AI 코딩 (OpenCode 무료) | ❌ 불필요 |
| AI 코딩 (Claude 사용) | ✅ 모델 자격증명 1개 |
| 공개 서버 운영 | ✅✅ sandboxd 토큰 + 모델 자격증명 |
| git push 사용 | ✅ + PAT |

### Q3. AI 에이전트 구축에 도움이 되나?

**매우 도움이 된다. 이것 자체가 "에이전트 샌드박스 레이어"다.**

AI 에이전트를 실제로 만들면 반드시 이 벽에 부딪힌다.

> "에이전트가 생성한 코드를 어디서 실행하지? 터지면? 악성이면? 결과를 어떻게 보여주지?"

이를 해결해주는 상용 서비스가 **E2B, Daytona, Modal, Fly Machines** (월 수십~수백 달러).
sandboxd가 그 자리를 **MIT 라이선스로** 채운다.

| 에이전트 개발에 필요한 것 | sandboxd가 제공하는 것 |
|---|---|
| 코드 격리 실행 | 하드닝된 컨테이너 (cap-drop ALL, read-only, 메모리 상한) |
| 결과 노출 | 프리뷰 URL 자동 발급 |
| 파일 읽기/쓰기 | `/v1/sandboxes/{id}/files` |
| 명령 실행 | `/sandbox/{id}/exec`, `/terminal` (WS) |
| **실패 롤백** | 태스크 체크포인트 + revert |
| **키 유출 방지** | authproxy + env 스크럽 |
| 상태 영속 | 워크스페이스 바인드 마운트 + 스냅샷 |
| 진행 상황 스트림 | `/v1/tasks/{id}/events` (SSE) |
| 유휴 비용 절감 | sleep/wake |
| 감사 로그 | `internal/audit` + 이벤트 타임라인 |
| 메트릭 | Prometheus (`internal/metrics`) |

**연동 패턴 4가지**

1. **코드 인터프리터 백엔드** — `POST /sandbox` → `/exec` → stdout 회수
2. **멀티에이전트 워크스페이스** — 에이전트별 샌드박스 분리로 상호 간섭 제거
3. **자가검증 루프** — 태스크 결과의 `build_status` / `preview_ok` / `app_healthy`를
   읽어 에이전트가 **스스로 검증하고 재시도**. 대부분 에이전트는 "코드 작성 완료"로
   끝나는데 여기선 정직한 실행 결과가 돌아온다.
4. **MCP 브릿지** — `/v1`을 MCP 서버로 감싸 Claude Desktop에 연결

**한계 (솔직하게)**

- 단일 호스트 (멀티 호스트 스케줄링은 "Later" 로드맵)
- 프리뷰 포트 1개 (프론트+백 분리 포트는 단일포트 모드 or 내부 리버스프록시 필요)
- DB는 앱 내장 SQLite만 (Postgres/Redis는 커스텀 이미지 또는 외부)
- 익명 악성 코드에는 부족 (VM 아님 — gVisor/Firecracker는 로드맵)
- egress 기본 허용 (nftables 코드는 존재하나 OSS 빌드에서 비활성)

→ **MVP ~ 중규모 서비스에는 충분.** 수천 테넌트 규모에서 하드닝 필요.

### Q4. React나 PHP로 만들 수 있나?

질문이 두 가지로 해석되므로 둘 다 답한다.

**A. "sandboxd 위에서 React/PHP 앱을 만들 수 있나?" → 가능하다.**

**React: 1급 지원.**
- `GET /v1/presets` → `react-standard` (React/Vite), `nextjs-standard`,
  `node-express-standard`, `fastapi-standard`, `worker-standard`
- `image/templates/react-standard/`에 `package.json` + `pnpm-lock.yaml` + `src/` 포함
- 생성 즉시 프리뷰 기동, 에이전트 수정 시 HMR로 즉시 반영
- 레시피에 Angular, Astro, Next, Nuxt, Svelte, Remix, Vite, Bun, Hono 등 포함

**PHP: 추가 런타임 레이어로 지원.**
```dockerfile
# image/php/Dockerfile — sandboxd-php:0.4.0
FROM sandboxd-base:0.3.0
RUN apt-get install -y php-cli php-sqlite3 php-mysql php-pgsql \
      php-xml php-mbstring php-gd php-curl php-zip php-intl \
      php-bcmath php-gmp php-ldap php-soap
# 실행: php -S 0.0.0.0:3000 -t <docroot>
```
```env
SANDBOXD_IMAGE=sandboxd-php:0.4.0
```
> ⚠️ 베이스 이미지는 **인스턴스 전체에 1개**다. 앱별 이미지 선택은 로드맵이므로,
> PHP를 쓸 경우 인스턴스를 PHP용으로 운영하거나 별도 인스턴스를 권장한다.
> Ruby/Rails 레이어(`image/ruby/Dockerfile`)도 동일한 방식.

**B. "sandboxd 자체를 React/PHP로 재구현할 수 있나?" → 비권장.**

| 대상 | 가능성 | 평가 |
|---|---|---|
| PHP로 컨트롤 플레인 재작성 | 기술적으로는 가능 | **비권장.** 상주 데몬 + 리퍼 루프 + WebSocket + SQLite WAL 동시성 — PHP의 요청-응답 모델과 궁합이 나쁘다 |
| Node/TS로 재작성 | 가능 | **비권장.** 34,900줄 재작성 비용 대비 이득이 없다 |
| **React로 콘솔 교체** | ✅ | **권장.** 콘솔은 순수 `/v1` 클라이언트라 통째로 교체해도 엔진 영향 0 |
| **PHP/Laravel로 제품 레이어** | ✅ | **권장.** 결제·회원·대시보드를 Laravel로 만들고 `/v1` 호출 |

**현실적인 정답 구조**

```
┌──────────────────────────────────────────┐
│  내 제품 (React / Next.js / Laravel)      │ ← 직접 만드는 부분
│  로그인 · 결제 · 요금제 · 대시보드 · 브랜딩  │
└────────────────┬─────────────────────────┘
                 │ REST /v1 (53 엔드포인트)
┌────────────────▼─────────────────────────┐
│  sandboxd (Go) — 수정 없이 그대로 사용      │ ← 업스트림 업데이트 그대로 수신
└──────────────────────────────────────────┘
```

사업적으로도 최선이다. `./upgrade.sh`로 업스트림을 계속 받으면서 제품 레이어에만 집중할 수 있다.
API 클라이언트 참고 코드는 `console/src/api.ts`에 이미 있다.

### Q5. 유튜브 강의 영상으로 제작 가능한가?

**가능하며, 소재로 매우 좋다.**

**법적:** MIT 라이선스 → 영상 제작·코드 설명·수익화 모두 자유.
README에 `"Use it, ship it, sell what you build on it"` 명시.
(의무는 아니나 설명란에 원본 레포 + MIT 표기는 매너)

**소재 경쟁력**

| 이유 | 설명 |
|---|---|
| 트렌드 정중앙 | "AI 앱빌더" + "셀프호스팅" 둘 다 화제 |
| 비주얼이 극적 | 프롬프트 → 실제 앱 등장. 썸네일 소재 확실 |
| 클릭 유발 각도 | "무료로 Lovable 만들기" |
| 경쟁 부재 | Trendshift 일일 트렌드 등재했으나 **한국어 콘텐츠 거의 없음** |
| 교육 가치 | Go 설계, Docker 격리, Traefik, 에이전트 구조 |

**추천 커리큘럼**

| EP | 제목 | 길이 | 포인트 |
|---|---|---|---|
| 0 | "월 5천원 서버에 내 Lovable 만들기" | 8분 | 훅 영상. 설치→프롬프트→앱 등장 풀샷 |
| 1 | 설치 완전정복 (Linux/맥/VPS) | 15분 | `install.sh`, 트러블슈팅 |
| 2 | 콘솔 투어 — 노코드로 앱 10개 | 18분 | 스토어 80+, Ghost·n8n |
| 3 | AI 에이전트 연결 (무료로 시작) | 14분 | OpenCode 무료 플랜 = 0원 |
| 4 | "API 키가 샌드박스에 들어가지 않는다" | 16분 | authproxy 원리 — 차별화 포인트 |
| 5 | AI가 망쳐도 되돌리기 | 12분 | 체크포인트 / revert / private ref |
| 6 | API로 내 서비스 만들기 (React) | 25분 | `/v1` 호출, 프리뷰 임베드 |
| 7 | 아키텍처 해부 — Go 3.5만줄 읽기 | 22분 | reaper / wake / traefik |
| 8 | sleep/wake로 비용 1/10 | 13분 | 유휴 수거, 압박 수거 |
| 9 | VPS 실전 배포 + TLS + 하드닝 | 20분 | cloud-init, 도메인, API 인증 ON |
| 10 | 이걸로 수익화하기 | 15분 | 8장 내용 |

**제작 팁**

- 베이스 이미지 빌드가 수 분 소요 → **미리 빌드 후 배속/컷 편집**
- 터미널 폰트 크게 (모바일 시청자 고려)
- 프리뷰가 `*.localhost`라 로컬 데모가 깔끔함
- 🚨 **콘솔 로그인 정보, API 키, `.env` 절대 노출 금지**
- 베타 0.x이므로 **영상에 기준 버전 명시** (`v0.3.20 기준`)
- 설치 실패 문의가 많을 것 → **트러블슈팅 전용 영상**이 조회수에 유리

**수익 연계 경로**

```
유튜브 (무료, 신뢰 구축)
  ├─▶ 설명란 "설치 대행 문의"      → 서비스 수익
  ├─▶ 인프런/클래스101 심화 강의    → 강의 수익
  ├─▶ 뉴스레터                     → 리스트 확보
  ├─▶ VPS 제휴 링크 (원본도 운영 중) → 제휴 수익
  └─▶ 자체 SaaS 런칭               → 구독 수익
```

### Q6. 누가 써야 하고, 누가 쓰지 말아야 하나?

README가 솔직하게 명시한다.

**✅ 적합:** 다수의 샌드박스를 **타인에게 제공**하는 경우 — AI 앱빌더, 에이전트 플랫폼,
코딩 플레이그라운드, 사용자/브랜치별 프리뷰 환경, 팀 멀티앱 호스팅

**❌ 부적합:** 본인용 컨테이너 1~2개만 필요한 경우 → 셸 스크립트나 `docker run`이 더 단순

### Q7. 이 프로젝트의 현재 상태는?

- **베타 0.x** — 1.0 전까지 breaking change 가능. 버전 핀 권장
- 롤링 릴리스: **패치 번호 증가**가 일반 릴리스, **마이너 증가**가 마일스톤
- v0.3.20 기준 **플랫폼 기능은 사실상 완비** (콘솔, 프리셋, 에이전트, git, 시크릿, 스냅샷, 이벤트, 로그)
- **0.4.x 예정:** 원클릭 배포/퍼블리시(헤드라인), 콘솔 폴리시, **egress 제어 OSS 활성화**,
  스냅샷 백엔드 하드닝(디렉터리-tar)
- **Later:** gVisor/Kata/Firecracker 격리, containerd/OCI 백엔드, 관리형 DB·사이드카,
  멀티 호스트 스케줄링, 워크스페이스별 디스크 쿼터, 앱 스토어 확장

---

## 8. 수익화 아이디어 8선

### TIER 1 — 즉시 시작, 초기 자본 0원

#### 아이디어 1. 셀프호스팅 설치·운영 대행

**판매 대상:** sandboxd를 쓰고 싶지만 Docker를 모르는 조직
(원본 프로젝트도 "Managed" 상품을 운영 중 → **검증된 모델**)

| 상품 | 가격 | 포함 |
|---|---|---|
| 기본 설치 | 30~50만원 | VPS 셋업, 설치, 도메인+TLS, 1시간 교육 |
| 보안 강화 | +30만원 | API 인증, 시크릿 키, 방화벽, egress 제어, 백업 자동화 |
| 월 관리 | 월 15~30만원 | 업그레이드, 모니터링, 장애 대응, 백업 검증 |

**시나리오:** 월 2건 설치(40만) + 관리 5곳(월 20만) = **월 180만원**
**장점:** 자본 0, 재고 0 · **리스크:** 노동집약적(시간 = 매출 상한)
**고객:** 1인 개발사, 스타트업 CTO, 사내 플랫폼팀, 교육기관

#### 아이디어 2. 콘텐츠 → 강의 → 협찬 (추천)

```
1단: 유튜브 무료 시리즈 (7장 Q5 커리큘럼) → 신뢰 + 구독자
2단: 인프런/클래스101 심화 강의 15~20만원
3단: 뉴스레터 + 제휴 + 협찬
```

| 채널 | 예상 수익 |
|---|---|
| 유튜브 애드센스 | 월 30~150만 (구독 1만 기준) |
| 온라인 강의 | 150,000원 × 월 20명 = 월 300만 |
| VPS 제휴 링크 | 전환당 $25~100 |
| 기업 협찬 | 회당 100~500만 |
| 기업 출강 | 일 100~300만 |

**장점:** 한국어 콘텐츠 선점 가능, 자산이 축적(패시브)
**리스크:** 베타 0.x라 버전 노후화 → 버전 명시 + 업데이트 영상

#### 아이디어 3. 프로토타이핑 부티크

**판매 대상:** 아이디어는 있고 검증이 필요한 클라이언트
**무기:** 프롬프트 → AI 코딩 → **3일 안에 돌아가는 라이브 URL 납품**

| 상품 | 가격 | 기간 |
|---|---|---|
| 랜딩 + 데모 프로토타입 | 150~300만 | 3~5일 |
| MVP (DB + 인증 포함) | 500~1,500만 | 2~4주 |
| 유지보수 | 월 50~150만 | — |

**차별점:** 경쟁사는 PPT/피그마를 주지만, **돌아가는 URL**을 준다
**리스크:** 기대 관리 + AI 결과물 검수 필수

#### 아이디어 4. MCP 브릿지 + 템플릿 팩

```
sandboxd-mcp (MCP 서버, Node/TS 수백 줄)
→ Claude Desktop 연결
→ Claude가 직접 샌드박스 생성 / 앱 기동 / URL 반환
```

| 상품 | 가격 |
|---|---|
| MCP 브릿지 (오픈소스) | 무료 → GitHub Sponsors |
| 프리미엄 템플릿 팩 (쇼핑몰/예약/대시보드 프리셋) | 5~15만원 |
| 한국형 레시피 팩 (토스페이·카카오·네이버 연동) | 10~30만원 |
| 설치형 원클릭 번들 | 50만원 |

**장점:** 한 번 만들면 반복 판매, 기술 난이도 낮음(REST 래핑)
**보너스:** 오픈소스 공개 → 원본 레포 기여 → 신뢰도 상승 → 1·3번 영업에 직결

### TIER 2 — 3~6개월, 제품화

#### 아이디어 5. 버티컬 AI 앱빌더 SaaS (최우선 추천)

**전략: 범용으로 Lovable과 경쟁하지 않고, 좁은 버티컬을 독점한다.**

| 버티컬 | 타깃 | 월 요금 |
|---|---|---|
| 스마트스토어 랜딩 빌더 | 쇼핑몰 셀러 | 2.9~9.9만 |
| 예약 시스템 빌더 | 미용실·학원·병원 | 3.9~12만 |
| 포트폴리오 빌더 | 디자이너·작가 | 1.9~4.9만 |
| 사내 어드민 빌더 | 중소기업 | 9.9~29만 |
| 코딩 수업 플랫폼 | 학원·학교 | 학생당 1만 |

**유닛 이코노믹스**

```
수익:  유료 100명 × 월 5만원            = 월 500만원

비용:  VPS (8vCPU/32GB) × 2대           월  20만
       AI 모델 토큰 (유저당 ~5,000원)    월  50만
       도메인/TLS/모니터링               월   5만
       ────────────────────────────────────────
       합계                             월  75만

➡️ 월 순이익 약 425만원 (마진 85%)
```

> 이 숫자는 **sleep/wake 덕분에 성립한다.** 유저 100명 = 앱 300개여도
> 동시 활성은 10~20개뿐 → 서버 2대로 커버.

| 리스크 | 대응 |
|---|---|
| AI 토큰 비용 폭주 | 요금제별 태스크 쿼터, OpenCode 무료 티어 믹스 |
| 악성 유저 | API 인증 ON, egress 제어, 메모리/PID 상한, 결제 선인증 |
| 단일 서버 한계 | 테넌트 그룹별 인스턴스 분리(샤딩) |
| 격리 한계 (VM 아님) | 신뢰 도메인별 VM 분리, gVisor 로드맵 대기 |
| 업스트림 breaking change | 버전 핀 + 스테이징에서 `./upgrade.sh` 검증 |

#### 아이디어 6. 오픈소스 앱 호스팅 서비스

**판매 대상:** "Ghost 블로그를 쓰고 싶지만 서버를 모르는" 사용자
앱 스토어 80+ 원클릭이 그대로 상품이 된다.

| 플랜 | 가격 | 내용 |
|---|---|---|
| 1앱 | 월 9,900원 | Ghost / n8n / Uptime Kuma 중 1개 + 서브도메인 |
| 3앱 | 월 24,900원 | + 자동 백업 |
| 비즈니스 | 월 79,000원 | 커스텀 도메인 + 전용 리소스 + 지원 |

**시나리오:** 고객 200명 × 평균 2만원 = 월 400만, 서버비 40만 → **순익 360만**
**장점:** **AI 토큰 비용 0원**(앱 실행만) → 마진 최고
**경쟁:** PikaPods, Elestio 존재하나 **한국어/한국 결제/한국 리전 부재** → 틈새
**리스크:** 고객 데이터 책임(백업·장애) → SLA 명확화, 백업 자동화 필수

### TIER 3 — 1년+, 자본·팀 필요

#### 아이디어 7. 에이전트 샌드박스 API (E2B 대안)

**판매 대상:** AI 에이전트 개발사에 "코드 실행 인프라"를 API로 제공
**차별점:** E2B/Daytona/Modal 대비 **한국 리전 + 한국 결제 + 한국어 지원**

```
샌드박스 실행 시간 : 시간당 30원
AI 태스크         : 건당 100원 + 토큰 실비
스토리지          : GB·월 100원
전용 인스턴스      : 월 50만원부터
```

**잠재력:** 고객 20곳 × 월 200만 = 월 4,000만
**리스크:** 멀티 호스트 스케줄링 직접 구현, 24/7 운영, VM 격리 필수

#### 아이디어 8. 코딩 교육 플랫폼

**구조:** 수강생 1명 = 샌드박스 1개 (각자 URL, 자동 채점, 강사 실시간 관찰)

| 상품 | 가격 |
|---|---|
| 개인 수강 | 월 4.9만 |
| 학원 B2B | 학생당 월 1만 (100명 = 월 100만) |
| 부트캠프 B2B | 기수당 500~2,000만 |
| 기업 교육 | 일 200~500만 |

**강점:** "수강생이 환경 설정을 못해 1일차를 날리는 문제"를 완전히 제거. 브라우저만 필요
**리스크:** 교육 콘텐츠 제작 비용 → 아이디어 2(유튜브)와 콘텐츠 재활용 필수

### 비교표

| # | 아이디어 | 초기비용 | 난이도 | 수익속도 | 상한 | 추천도 |
|---|---|---|---|---|---|---|
| 1 | 설치 대행 | 0 | 하 | 즉시 | 월 200만 | ⭐⭐⭐⭐ |
| 2 | 콘텐츠/강의 | 0 | 하 | 3개월 | 월 500만 | ⭐⭐⭐⭐⭐ |
| 3 | 프로토타이핑 | 0 | 중 | 즉시 | 월 500만 | ⭐⭐⭐⭐ |
| 4 | MCP/템플릿 | 0 | 하 | 1개월 | 월 100만 | ⭐⭐⭐ |
| 5 | 버티컬 SaaS | 300만 | 중 | 4개월 | 월 수천만 | ⭐⭐⭐⭐⭐ |
| 6 | 앱 호스팅 | 100만 | 중 | 2개월 | 월 1,000만 | ⭐⭐⭐⭐ |
| 7 | 샌드박스 API | 2,000만 | 상 | 1년 | 월 수억 | ⭐⭐ |
| 8 | 교육 플랫폼 | 500만 | 중 | 6개월 | 월 2,000만 | ⭐⭐⭐ |

---

## 9. 리스크 체크리스트

| 리스크 | 왜 중요한가 | 대응 |
|---|---|---|
| ⚠️ **베타 0.x** | breaking change 가능 | 버전 핀 + 스테이징에서 `./upgrade.sh` 검증 |
| 🚨 **API 인증 기본 OFF** | 공개 노출 시 즉시 침해 | `SANDBOXD_API_AUTH_DISABLED=false` **필수** |
| ⚠️ **컨테이너 격리 (VM 아님)** | 커널 CVE 탈출 가능성 | 신뢰 도메인별 VM 분리, 패치 자동화 |
| ⚠️ **egress 기본 허용, 로깅 없음** | 코인 채굴·스팸 악용 | 호스트 방화벽 + egress 토글 (0.4.x) |
| ⚠️ **AI 토큰 비용** | 마진 침식 | 쿼터 + 하드 리밋 + OpenCode 무료 믹스 |
| ⚠️ **단일 호스트** | 스케일 상한 | 인스턴스 샤딩 설계 선행 |
| ⚠️ **디스크 쿼터 없음** | 한 유저가 디스크 점유 | fs/volume 레이어에서 쿼터 |
| ⚠️ **프리뷰 포트 1개** | 멀티포트 앱 미지원 | 단일포트 모드 / 내부 리버스프록시 |
| ⚠️ **스냅샷 실험적** | 디렉터리 스토리지에서 불안정 | 워크스페이스 디렉터리 복사 병행 |
| ⚠️ **베이스 이미지 1개** | 앱별 런타임 선택 불가 | PHP/Ruby는 별도 인스턴스 |
| ✅ **MIT 라이선스** | 상업적 이용 자유 | 원본 표기(매너) |

---

## 10. 실행 로드맵

```
📅 1개월차   유튜브 EP0~2 + 설치 대행 오픈 (TIER 1)
             목표: 월 50~100만 · 리스크 0

📅 2~3개월   MCP 브릿지 오픈소스 공개 + 프로토타이핑 수주
             목표: 월 200~300만 · 신뢰도 구축

📅 4~6개월   버티컬 1개 선택 → SaaS MVP (아이디어 5)
             + 앱 호스팅 베타 (아이디어 6)
             목표: 월 500만

📅 7~12개월  SaaS 성장 + 교육 B2B
             목표: 월 1,000만+
```

### 최종 권고

> **아이디어 2(콘텐츠)로 시작 → 1·3(서비스)으로 현금흐름 → 5(버티컬 SaaS)로 승부.**

근거:
- 유튜브가 **마케팅 + 신뢰 + 수익**을 동시에 제공하며 리스크가 0이다
- 설치 대행·프로토타이핑이 **현금흐름**을 만들어 준다
- 그 과정에서 **어떤 버티컬이 수익성 있는지 고객이 직접 알려준다**
- 확신을 가진 뒤 SaaS를 만들면 실패 확률이 크게 낮아진다

**타이밍:** 이 프로젝트는 Trendshift 일일 트렌드에 등재되었으나
**한국어 콘텐츠가 거의 없다.** 선점 가능한 구간이다.

---

## 부록 A. 참고 링크

| 분류 | 링크 |
|---|---|
| 원본 레포 | https://github.com/tastyeffectco/sandboxd |
| 이 포크 | https://github.com/bmshin94/sandboxd |
| 공식 문서 | https://sandboxd.io |
| 라이브 데모 | https://sandboxd.io/demo/ |
| 퀵스타트 | https://sandboxd.io/quickstart |
| 핵심 개념 | https://sandboxd.io/concepts |
| 아키텍처 | https://sandboxd.io/reference/architecture |
| API 레퍼런스 | https://sandboxd.io/reference/api |
| 설정 레퍼런스 | https://sandboxd.io/reference/configuration |
| 코딩 에이전트 가이드 | https://sandboxd.io/guides/agents |
| 프로덕션 / TLS | https://sandboxd.io/guides/production-tls |
| 하드닝 | https://sandboxd.io/guides/hardening |
| 앱 실행 방법 | https://sandboxd.io/guides/apps-in-the-console |
| 로드맵 | https://sandboxd.io/roadmap |
| 디스커션 | https://github.com/tastyeffectco/sandboxd/discussions |
| 릴리스 | https://github.com/tastyeffectco/sandboxd/releases |

## 부록 B. 레포 내 필독 문서

| 파일 | 내용 |
|---|---|
| `README.md` | 전체 소개, 퀵스타트, VPS 배포 |
| `AGENTS.md` | **AI 에이전트/사람용 자급자족 운영 런북** (복붙 가능) |
| `ARCHITECTURE.md` | 컴포넌트, 런타임 모델, 격리, 저장, v0.3 트레이드오프 |
| `ROADMAP.md` | 0.3 완료 / 0.4.x 예정 / Later |
| `CHANGELOG.md` | 릴리스 히스토리 (현재 v0.3.20) |
| `SECURITY.md` | 취약점 비공개 제보 절차 |
| `CONTRIBUTING.md` | 기여 가이드 (좋은 첫 PR: 런타임 프리셋, 앱 스토어 레시피) |
| `.env.example` | 설정 키 26개 전부 주석 설명 |
| `docs/openapi.yaml` | `/v1` 엔드포인트 53개 명세 |
| `docs/agent-auth.md` | 크레덴셜 주입 프록시 상세 |
| `docs/isolation.md` · `docs/gvisor.md` | 격리 모델 / 강화 격리 |
| `docs/git-workflow.md` | Git 임포트·커밋·푸시 경계 |
| `docs/sandbox-manifest.md` | `sandbox.yaml` 스키마 |
| `docs/base-image.md` | 베이스 이미지 계약 |
| `docs/upgrading.md` | 업그레이드 절차 |
| `docs/web-framework-recipes.md` | 프레임워크별 레시피 |
| `docs/production-safety.md` | 프로덕션 안전 수칙 |
| `docs/APP-CATALOG-CONTRACT.md` | 앱 카탈로그 계약 |
| `docs/adr/0001-*.md` | 앱 카탈로그 범위 ADR |

## 부록 C. 치트시트

```bash
# 설치 / 제거 / 업그레이드
./install.sh                    # 설치 (멱등)
./upgrade.sh                    # 업그레이드 (백업+헬스체크+자동롤백)
./upgrade.sh --check            # 버전 확인
./uninstall.sh --all --yes      # 완전 제거
./console-login.sh              # 콘솔 로그인 정보
./console-login.sh --reset-password

# 헬스
curl -s http://127.0.0.1:9090/healthz    # ok
curl -s http://127.0.0.1:9090/readyz     # ready
curl -s http://127.0.0.1:9090/version

# 운영
docker compose logs -f sandboxd
docker compose ps
docker compose restart sandboxd
docker ps --filter label=sandboxd.managed=true

# 샌드박스 사이클
API=http://127.0.0.1:9090
ID=$(curl -s -XPOST $API/sandbox -d '{"ports":[3000]}' | sed -E 's/.*"id":"([^"]+)".*/\1/')
curl -s -XPOST $API/v1/sandboxes/$ID/tasks -d '{"prompt":"...","agent":"claude-code"}'
curl -s -XPOST $API/v1/sandboxes/$ID/stop
curl -s -XPOST $API/sandbox/$ID/purge

# 에이전트 연결
curl -s -XPOST $API/v1/agents/anthropic/api-key -d '{"api_key":"sk-ant-..."}'
curl -s -XPOST $API/v1/agents/claude-code/oauth/start
curl -s -XPOST $API/v1/agents/claude-code/oauth/finish

# 프리뷰 URL 형식
http://s-<sandboxId>-<port>.preview.<PREVIEW_DOMAIN>[:<HTTP_PORT>]
```

---

*이 문서는 레포지토리 전수조사(Go 198파일 · 콘솔 41파일 · docs 13종 · 레시피 43종)를
기반으로 작성되었습니다. 기준 버전 v0.3.20 · 2026-10-01*
