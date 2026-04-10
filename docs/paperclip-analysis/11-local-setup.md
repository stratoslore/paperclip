# 11. 로컬 실행 및 커스텀 가이드

## 요구사항

| 항목 | 버전 |
|------|------|
| Node.js | >= 20 |
| pnpm | >= 9.15.4 |
| PostgreSQL | 내장 (자동) 또는 외부 |

## 빠른 시작

### 방법 1: npx (권장)

```bash
npx paperclipai onboard --yes
```

이 명령 하나로:
- 설정 파일 생성 (`~/.paperclip/instances/default/config.json`)
- 내장 PostgreSQL 초기화
- 서버 시작 (`http://localhost:3100`)

### 방법 2: 소스에서 실행

```bash
# 이미 프로젝트 루트에 Paperclip 소스가 있으므로:
cd /Users/wonbinahn/PycharmProjects/agent_office

# 의존성 설치
pnpm install

# 개발 모드로 실행
pnpm dev
```

### 방법 3: Docker (상세)

Docker로 실행하면 Node.js, pnpm 등을 로컬에 설치할 필요 없이 바로 실행할 수 있다.

#### 사전 준비

```bash
# BETTER_AUTH_SECRET 생성 (필수 - 인증 시크릿)
export BETTER_AUTH_SECRET=$(openssl rand -hex 32)
```

#### 옵션 A: 외부 PostgreSQL과 함께 실행 (권장 - 프로덕션)

`docker/docker-compose.yml`을 사용한다. PostgreSQL 컨테이너 + Paperclip 서버가 함께 실행된다.

```bash
# 기본 실행
docker compose -f docker/docker-compose.yml up -d

# 커스텀 공개 URL 설정 (원격 접근 시)
PAPERCLIP_PUBLIC_URL=http://my-server:3100 \
BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
docker compose -f docker/docker-compose.yml up -d
```

**구성:**

```yaml
# docker/docker-compose.yml 구조
services:
  db:                          # PostgreSQL 17 (Alpine)
    image: postgres:17-alpine
    ports: "5432:5432"
    volumes: pgdata            # 영속 볼륨
    healthcheck:               # 2초 간격 헬스체크

  server:                      # Paperclip 서버
    build: Dockerfile
    ports: "3100:3100"
    depends_on: db (healthy)   # DB 준비 후 시작
    volumes: paperclip-data    # 영속 데이터 볼륨
```

**자동 설정되는 환경변수:**

| 변수 | 값 | 설명 |
|------|-----|------|
| `DATABASE_URL` | `postgres://paperclip:paperclip@db:5432/paperclip` | 내부 DB 연결 |
| `PORT` | `3100` | 서버 포트 |
| `SERVE_UI` | `true` | UI 서빙 활성화 |
| `PAPERCLIP_DEPLOYMENT_MODE` | `authenticated` | 인증 모드 |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `private` | 비공개 노출 |
| `PAPERCLIP_PUBLIC_URL` | `http://localhost:3100` (기본) | 공개 URL |
| `BETTER_AUTH_SECRET` | (필수) | 인증 시크릿 |

#### 옵션 B: 내장 PostgreSQL로 간편 실행 (Quickstart)

`docker/docker-compose.quickstart.yml`을 사용한다. 별도 DB 없이 내장 PostgreSQL을 사용한다.

```bash
# API 키 설정 (에이전트 실행에 필요)
export ANTHROPIC_API_KEY=sk-ant-...   # Claude 에이전트용
export OPENAI_API_KEY=sk-...          # Codex 에이전트용 (선택)

# 실행
BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
docker compose -f docker/docker-compose.quickstart.yml up -d
```

**구성:**

```yaml
# docker/docker-compose.quickstart.yml 구조
services:
  paperclip:                   # 단일 컨테이너 (서버 + 내장 DB)
    ports: "${PAPERCLIP_PORT:-3100}:3100"
    volumes:
      - "${PAPERCLIP_DATA_DIR:-../data/docker-paperclip}:/paperclip"
```

**커스텀 포트/데이터 디렉토리:**

```bash
PAPERCLIP_PORT=3200 \
PAPERCLIP_DATA_DIR=/my/data/path \
BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
docker compose -f docker/docker-compose.quickstart.yml up -d
```

#### 옵션 C: 보안 리뷰 환경 (Untrusted Review)

외부 코드를 안전하게 리뷰하는 격리된 환경.

```bash
ANTHROPIC_API_KEY=sk-ant-... \
GITHUB_TOKEN=ghp_... \
docker compose -f docker/docker-compose.untrusted-review.yml up
```

**보안 특성:**
- `cap_drop: ALL` — 모든 Linux 커널 권한 제거
- `no-new-privileges: true` — 권한 상승 차단
- `tmpfs: /tmp` — 임시 파일 메모리 격리 (1GB)
- 별도 볼륨으로 홈/작업 디렉토리 격리

#### Dockerfile 상세

멀티 스테이지 빌드로 최적화된 프로덕션 이미지:

```
Stage 1: base
  └─ node:lts-trixie-slim + 시스템 패키지
     (git, curl, ripgrep, python3, gh CLI)

Stage 2: deps
  └─ pnpm install --frozen-lockfile (의존성만)

Stage 3: build
  └─ UI 빌드 → Plugin SDK 빌드 → Server 빌드

Stage 4: production
  └─ 빌드 결과 복사 + 글로벌 AI CLI 설치
     (claude-code, codex, opencode-ai)
```

**프로덕션 이미지에 포함된 AI CLI:**
- `@anthropic-ai/claude-code` (최신)
- `@openai/codex` (최신)
- `opencode-ai`

**엔트리포인트 (`docker-entrypoint.sh`):**
- 호스트 사용자의 UID/GID에 맞춰 컨테이너 내부 `node` 유저 조정
- `gosu`로 non-root 실행 보장

#### Docker 실행 후 접속

```bash
# 브라우저에서 접속
open http://localhost:3100

# 로그 확인
docker compose -f docker/docker-compose.yml logs -f server

# 셸 접속
docker compose -f docker/docker-compose.yml exec server bash

# 중지
docker compose -f docker/docker-compose.yml down

# 볼륨 포함 완전 삭제
docker compose -f docker/docker-compose.yml down -v
```

#### Docker에서 UID/GID 맞추기

호스트 파일 권한과 충돌 방지를 위해 빌드 시 UID/GID를 지정할 수 있다:

```bash
docker compose -f docker/docker-compose.yml build \
  --build-arg USER_UID=$(id -u) \
  --build-arg USER_GID=$(id -g)
```

#### Docker + Tailscale 원격 접근

```bash
PAPERCLIP_PUBLIC_URL=http://my-machine.tail12345.ts.net:3100 \
BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
docker compose -f docker/docker-compose.yml up -d
```

#### Docker 문제 해결

| 증상 | 해결 |
|------|------|
| `BETTER_AUTH_SECRET must be set` | `export BETTER_AUTH_SECRET=$(openssl rand -hex 32)` 실행 |
| DB 연결 실패 | `docker compose logs db`로 PostgreSQL 상태 확인 |
| 포트 충돌 (3100) | `PAPERCLIP_PORT=3200` 환경변수 사용 또는 compose 파일 수정 |
| 볼륨 권한 오류 | `--build-arg USER_UID=$(id -u)` 로 이미지 재빌드 |
| 이미지 업데이트 | `docker compose build --no-cache && docker compose up -d` |

## 환경 설정

### .env 파일

프로젝트 루트의 `.env` 파일:

```bash
DATABASE_URL=postgres://paperclip:paperclip@localhost:5432/paperclip
PORT=3100
SERVE_UI=true
```

### 내장 PostgreSQL (기본)

별도 설정 없이 자동으로 내장 PostgreSQL이 시작된다:
- 데이터 디렉토리: `~/.paperclip/instances/default/db`
- 포트: 54329 (외부 PG와 충돌 방지)

### 외부 PostgreSQL 사용

```bash
# .env 또는 config.json에서 설정
DATABASE_URL=postgres://user:pass@host:5432/paperclip
```

config.json:
```json
{
  "database": {
    "mode": "postgres",
    "connectionString": "postgres://user:pass@host:5432/paperclip"
  }
}
```

## 개발 명령어

```bash
# 전체 개발 모드 (서버 + UI, 파일 변경 감시)
pnpm dev

# 서버만 실행
pnpm dev:server

# UI만 실행
pnpm dev:ui

# 파일 감시 없이 실행
pnpm dev:once

# 빌드
pnpm build

# 타입 검사
pnpm typecheck

# 테스트
pnpm test:run

# E2E 테스트
pnpm test:e2e

# DB 마이그레이션 생성
pnpm db:generate

# DB 마이그레이션 적용
pnpm db:migrate
```

## 커스터마이제이션 가이드

### 1. 에이전트 추가

UI 대시보드에서 또는 API를 통해 에이전트를 추가한다:

```bash
curl -X POST http://localhost:3100/api/companies/{companyId}/agents \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Engineer",
    "role": "engineer",
    "title": "Senior Engineer",
    "adapterType": "claude_local",
    "adapterConfig": {
      "cwd": "/path/to/project",
      "model": "claude-sonnet-4-6",
      "effort": "high"
    },
    "budgetMonthlyCents": 10000
  }'
```

### 2. 커스텀 어댑터 설정

Claude Code 에이전트 커스텀 설정:

```json
{
  "adapterType": "claude_local",
  "adapterConfig": {
    "cwd": "/path/to/project",
    "model": "claude-opus-4-6",
    "effort": "high",
    "instructionsFilePath": "/path/to/instructions.md",
    "maxTurnsPerRun": 50,
    "timeoutSec": 300,
    "env": {
      "MY_API_KEY": "xxx",
      "DEBUG": "true"
    },
    "extraArgs": ["--allowedTools", "Edit,Write,Bash"],
    "workspaceStrategy": {
      "type": "git_worktree",
      "baseRef": "main"
    }
  }
}
```

### 3. 조직도 설정

에이전트 간 보고 라인을 설정하여 위임 체계를 구축한다:

```
CEO (claude_local)
  ├─ CTO (claude_local)
  │   ├─ Backend Engineer (codex_local)
  │   └─ Frontend Engineer (claude_local)
  ├─ CMO (claude_local)
  │   └─ Social Media Manager (claude_local)
  └─ Support Lead (claude_local)
      └─ Support Agent (claude_local)
```

에이전트의 `reportsTo` 필드로 관계를 설정한다.

### 4. 예산 설정

```json
{
  "budgetMonthlyCents": 50000
}
```

- 에이전트별: 월 $500 한도 = `50000` cents
- 회사별: 월 예산 설정 가능
- 프로젝트별: 전체 기간 예산 설정 가능

### 5. 루틴(반복 작업) 설정

UI에서 또는 API로 루틴을 생성한다:

```bash
curl -X POST http://localhost:3100/api/routines \
  -H "Content-Type: application/json" \
  -d '{
    "companyId": "...",
    "projectId": "...",
    "assigneeAgentId": "...",
    "title": "Daily Standup Report",
    "description": "일일 진행 상황을 요약합니다.",
    "priority": "medium",
    "concurrencyPolicy": "coalesce_if_active",
    "catchUpPolicy": "skip_missed",
    "triggers": [{
      "kind": "schedule",
      "cronExpression": "0 9 * * 1-5",
      "timezone": "Asia/Seoul"
    }]
  }'
```

### 6. 플러그인 개발

```bash
# 스캐폴딩
npx create-paperclip-plugin my-plugin
cd my-plugin
pnpm install

# 개발
# src/index.ts 수정

# 빌드
pnpm build

# 설치 (플러그인 디렉토리에 복사)
cp -r dist ~/.paperclip/instances/default/plugins/my-plugin/
```

### 7. 커스텀 스킬 추가

`skills/` 디렉토리에 새 스킬을 추가한다:

```
skills/my-custom-skill/
  ├── SKILL.md          # 스킬 정의 (마크다운)
  └── references/       # 참조 자료 (선택)
```

SKILL.md 예시:
```markdown
# My Custom Skill

## Instructions

You are an expert at...

## API Reference

...
```

회사별로 스킬을 설치/관리한다 (UI 또는 API).

### 8. Tailscale로 원격 접근

개인 개발 환경에서 모바일 접근:

```bash
# Tailscale 설치 후
# config.json에서 host를 Tailscale IP로 변경
{
  "server": {
    "host": "0.0.0.0",
    "allowedHostnames": ["my-machine.tail12345.ts.net"]
  }
}
```

## 프로덕션 배포

### 외부 PostgreSQL + authenticated 모드

```json
{
  "database": {
    "mode": "postgres",
    "connectionString": "postgres://user:pass@db-host:5432/paperclip"
  },
  "server": {
    "deploymentMode": "authenticated",
    "exposure": "public",
    "port": 3100,
    "serveUi": true
  },
  "auth": {
    "baseUrlMode": "explicit",
    "publicBaseUrl": "https://paperclip.example.com"
  },
  "storage": {
    "provider": "s3",
    "s3": {
      "bucket": "my-paperclip-storage",
      "region": "ap-northeast-2"
    }
  },
  "secrets": {
    "provider": "aws_secrets_manager"
  }
}
```

### 환경변수

```bash
BETTER_AUTH_SECRET=<random-hex-32>
PAPERCLIP_AGENT_JWT_SECRET=<random-hex-32>
DATABASE_URL=postgres://user:pass@db-host:5432/paperclip
PAPERCLIP_DEPLOYMENT_MODE=authenticated
PAPERCLIP_DEPLOYMENT_EXPOSURE=public
PAPERCLIP_PUBLIC_URL=https://paperclip.example.com
```

## 문제 해결

### 포트 충돌

```bash
# 포트 변경
PORT=3200 pnpm dev
```

### 내장 PostgreSQL 문제

```bash
# doctor 명령으로 진단
npx paperclipai doctor --repair
```

### 마이그레이션 오류

```bash
# 수동 마이그레이션
pnpm db:migrate
```

### 텔레메트리 비활성화

```bash
PAPERCLIP_TELEMETRY_DISABLED=1 pnpm dev
# 또는
DO_NOT_TRACK=1 pnpm dev
```
