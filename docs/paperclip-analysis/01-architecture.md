# 01. 프로젝트 구조 및 아키텍처

## 모노레포 구조

Paperclip은 pnpm workspace 기반의 모노레포로 구성된다.

```
paperclip/
├── server/                    # Express REST API + 오케스트레이션 엔진
│   └── src/
│       ├── index.ts           # 서버 엔트리포인트
│       ├── app.ts             # Express 앱 팩토리, 라우트 마운트
│       ├── config.ts          # 설정 로딩 (env + config file)
│       ├── routes/            # 25개 API 라우트 모듈
│       ├── services/          # 66+ 비즈니스 로직 서비스
│       ├── middleware/        # 로깅, 인증, 가드
│       ├── adapters/          # 서버 사이드 어댑터 통합
│       ├── auth/              # 인증 (better-auth)
│       ├── realtime/          # WebSocket 라이브 이벤트
│       ├── secrets/           # 시크릿 관리
│       └── storage/           # 파일 스토리지 (로컬/S3)
│
├── ui/                        # React + Vite 프론트엔드
│   └── src/
│       ├── pages/             # 40+ 페이지 컴포넌트
│       ├── components/        # 공통 UI 컴포넌트
│       ├── api/               # API 클라이언트
│       ├── hooks/             # React 커스텀 훅
│       └── plugins/           # 플러그인 UI
│
├── cli/                       # CLI 도구 (npm: paperclipai)
│   └── src/
│       ├── commands/          # onboard, run, doctor, configure 등
│       ├── adapters/          # CLI 어댑터
│       └── prompts/           # 인터랙티브 프롬프트
│
├── packages/
│   ├── shared/                # 공유 타입, 상수, 검증, 설정 스키마
│   │   └── src/
│   │       ├── config-schema.ts   # Zod 설정 스키마
│   │       ├── constants.ts       # 에이전트 타입, 상태 등 enum
│   │       ├── types/             # 29개 타입 정의 파일
│   │       └── validators/        # 입력 검증
│   │
│   ├── db/                    # Drizzle ORM 스키마 + 마이그레이션
│   │   └── src/
│   │       ├── schema/        # 60+ 테이블 스키마
│   │       ├── migrations/    # DB 마이그레이션
│   │       ├── client.ts      # DB 클라이언트
│   │       └── seed.ts        # 시드 데이터
│   │
│   ├── adapter-utils/         # 어댑터 공유 유틸리티
│   │
│   ├── adapters/              # 7개 에이전트 어댑터
│   │   ├── claude-local/      # Claude Code
│   │   ├── codex-local/       # OpenAI Codex
│   │   ├── cursor-local/      # Cursor
│   │   ├── gemini-local/      # Google Gemini
│   │   ├── openclaw-gateway/  # OpenClaw (원격 게이트웨이)
│   │   ├── opencode-local/    # OpenCode
│   │   └── pi-local/          # Pi
│   │
│   └── plugins/               # 플러그인 시스템
│       ├── sdk/               # 플러그인 SDK
│       ├── create-paperclip-plugin/  # 스캐폴딩 도구
│       └── examples/          # 예제 플러그인
│
├── skills/                    # 에이전트 스킬 정의
│   ├── paperclip/             # 핵심 Paperclip 스킬
│   ├── paperclip-create-agent/
│   ├── paperclip-create-plugin/
│   └── para-memory-files/
│
├── docker/                    # Docker 설정
├── scripts/                   # 빌드, 릴리스, 개발 스크립트
├── tests/                     # E2E 테스트 (Playwright)
├── evals/                     # Promptfoo 평가
└── doc/                       # 내부 문서
```

## 서버 아키텍처 (server/)

### 엔트리포인트 (`server/src/index.ts`)

서버 시작 시 실행 순서:

```
startServer()
  │
  ├─ 1. 설정 로딩 (config.ts)
  │     ├─ 환경변수 읽기
  │     ├─ config.json 파싱
  │     └─ Zod 검증 및 기본값 적용
  │
  ├─ 2. 데이터베이스 초기화
  │     ├─ embedded-postgres: 내장 PostgreSQL 시작
  │     └─ postgres: 외부 DB 연결
  │
  ├─ 3. 마이그레이션 관리
  │     ├─ pending 마이그레이션 감지
  │     ├─ 자동 적용 또는 프롬프트
  │     └─ 마이그레이션 실행
  │
  ├─ 4. Express 앱 생성 (app.ts)
  │     ├─ 미들웨어 설정 (로깅, 인증, 가드)
  │     ├─ 25개 API 라우트 마운트
  │     ├─ 플러그인 시스템 초기화
  │     └─ UI 서빙 (정적/Vite dev)
  │
  ├─ 5. 인증 초기화
  │     ├─ local_trusted: 보드 주체 자동 설정
  │     └─ authenticated: JWT + better-auth 설정
  │
  ├─ 6. 서비스 초기화
  │     ├─ 하트비트 스케줄러 (30초 간격)
  │     ├─ 루틴 스케줄러
  │     ├─ 피드백 서비스
  │     └─ DB 백업 스케줄러
  │
  ├─ 7. WebSocket 설정 (실시간 이벤트)
  │
  └─ 8. 그레이스풀 셧다운 핸들러 등록
```

### Express 앱 팩토리 (`server/src/app.ts`)

```
createApp(db, opts)
  │
  ├─ 미들웨어 체인
  │   ├─ pino-http 로거
  │   ├─ CORS 설정
  │   ├─ JSON/URL-encoded 바디 파서
  │   ├─ 인증 미들웨어
  │   └─ 뮤테이션 가드 (run-id 검증)
  │
  ├─ API 라우트 (25개 모듈)
  │   ├─ /api/agents          # 에이전트 CRUD
  │   ├─ /api/issues          # 이슈/작업 관리
  │   ├─ /api/companies       # 회사 관리
  │   ├─ /api/projects        # 프로젝트 관리
  │   ├─ /api/goals           # 목표 관리
  │   ├─ /api/approvals       # 승인 관리
  │   ├─ /api/costs           # 비용 추적
  │   ├─ /api/routines        # 루틴 관리
  │   ├─ /api/secrets         # 시크릿 관리
  │   ├─ /api/plugins         # 플러그인 관리
  │   ├─ /api/dashboard       # 대시보드 데이터
  │   ├─ /api/activity        # 활동 로그
  │   ├─ /api/health          # 헬스체크
  │   └─ ... (11개 더)
  │
  ├─ 플러그인 시스템
  │   ├─ 플러그인 로더 (디스커버리)
  │   ├─ 워커 프로세스 관리
  │   ├─ 이벤트 버스
  │   ├─ 잡 스케줄러
  │   └─ 라이프사이클 관리
  │
  └─ UI 서빙
      ├─ 프로덕션: 정적 파일 서빙
      └─ 개발: Vite 미들웨어
```

## 데이터 흐름

### 에이전트 실행 전체 흐름

```
타이머 틱 (30초)
  │
  ▼
heartbeat.tickTimers(now)
  │
  ▼
에이전트별 wakeup_requests 또는 scheduled_tasks 조회
  │
  ▼
enqueueRun({agentId, source, issueIds})
  │  ├─ heartbeat_runs 레코드 생성 (status='queued')
  │  └─ 예산 사전 검증
  │
  ▼
executeRun(runId)
  │
  ├─ 어댑터 결정 (claude_local, codex_local, process...)
  ├─ 에이전트 설정 및 런타임 상태 조회
  ├─ 실행 워크스페이스 구현 (git clone, mkdir 등)
  ├─ 런타임 서비스 확보 (컨테이너 시작 등)
  │
  ├─ adapter.execute(config)  ◄── 실제 AI 에이전트 실행
  │   ├─ 환경변수 주입 (PAPERCLIP_AGENT_ID 등)
  │   ├─ 스킬 주입 (SKILL.md)
  │   ├─ 프롬프트 구성
  │   └─ 프로세스 실행 및 결과 캡처
  │
  ├─ stdout, stderr, exit code, result JSON 캡처
  ├─ 사용량 파싱 → cost_events 생성
  ├─ heartbeat_run 결과 업데이트
  ├─ WebSocket으로 라이브 이벤트 알림
  └─ 실행 워크스페이스 해제
  │
  ▼
heartbeat.completed 이벤트 발행
```

### 이슈 체크아웃/릴리스 흐름

```
에이전트 깨어남 (heartbeat)
  │
  ▼
POST /api/issues/{id}/checkout {agentId}
  │  ├─ 다른 에이전트가 잠금 중? → 409 Conflict
  │  ├─ checkoutRunId 설정으로 이슈 잠금
  │  └─ 실행 컨텍스트 스냅샷 생성
  │
  ▼
에이전트가 작업 수행
  │
  ▼
PATCH /api/issues/{id} {status, comment}
  │  ├─ 상태 및 코멘트 업데이트
  │  ├─ 활동 로그 기록
  │  ├─ issue.updated 이벤트 발행
  │  └─ WebSocket으로 팔로워 알림
  │
  ▼
에이전트 하트비트 종료
  │
  ▼
미완료 이슈 잠금 해제 (자동 cleanup)
```

## 배포 모드

| 모드 | 설명 | 용도 |
|------|------|------|
| `local_trusted` | 인증 없음, 단일 사용자, 로컬 개발 | 개발/테스트 |
| `authenticated` | JWT + better-auth, 멀티 유저 | 프로덕션 |

## 노출 모드

| 모드 | 설명 |
|------|------|
| `private` | 127.0.0.1에 바인드, allowedHostnames로 필터링 |
| `public` | 0.0.0.0에 바인드, 외부 접근 허용 |
