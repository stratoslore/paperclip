# Paperclip 기능 정밀 분석

## 목차

| 문서 | 내용 |
|------|------|
| [00-overview.md](./00-overview.md) | 전체 개요 및 아키텍처 |
| [01-architecture.md](./01-architecture.md) | 프로젝트 구조 및 아키텍처 상세 |
| [02-heartbeat-engine.md](./02-heartbeat-engine.md) | 하트비트 엔진 (핵심 실행 엔진) |
| [03-services.md](./03-services.md) | 핵심 서비스 분석 (Issues, Agents, Budgets, Goals, Companies) |
| [04-adapters.md](./04-adapters.md) | 에이전트 어댑터 시스템 |
| [05-plugin-system.md](./05-plugin-system.md) | 플러그인 SDK 및 시스템 |
| [06-database-schema.md](./06-database-schema.md) | 데이터베이스 스키마 (60+ 테이블) |
| [07-cli-commands.md](./07-cli-commands.md) | CLI 명령어 |
| [08-configuration.md](./08-configuration.md) | 설정 및 환경변수 |
| [09-skills-system.md](./09-skills-system.md) | 스킬 시스템 및 에이전트 API |
| [10-routines.md](./10-routines.md) | 루틴 (스케줄 기반 반복 실행) |
| [11-local-setup.md](./11-local-setup.md) | 로컬 실행 및 커스텀 가이드 |
| [12-i18n.md](./12-i18n.md) | i18n (국제화) 아키텍처 |

---

## Paperclip이란?

Paperclip은 **"제로 휴먼 기업(zero-human company)"을 위한 오픈소스 오케스트레이션 플랫폼**이다.

> "OpenClaw이 직원이라면, Paperclip은 회사다."

Node.js 서버 + React UI로 구성되며, AI 에이전트 팀을 오케스트레이션하여 비즈니스를 운영한다. 어떤 에이전트든(Claude Code, Codex, Cursor, Gemini 등) 투입할 수 있고, 목표 설정, 예산 관리, 전략 승인을 하나의 대시보드에서 수행한다.

### 핵심 컨셉

```
┌─────────────────────────────────────────────────────┐
│                    Paperclip                         │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │  Company  │  │  Company  │  │  Company  │  ...    │
│  │          │  │          │  │          │          │
│  │ CEO(AI)  │  │ CEO(AI)  │  │ CEO(AI)  │          │
│  │  ├─CTO   │  │  ├─Eng   │  │  ├─PM    │          │
│  │  ├─Eng   │  │  └─Mkt   │  │  └─Dev   │          │
│  │  └─Mkt   │  │          │  │          │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│                                                     │
│  ┌─────────────────────────────────────────┐        │
│  │  Heartbeat Engine (30초 주기)            │        │
│  │  → 에이전트 깨우기 → 작업 체크아웃       │        │
│  │  → 실행 → 결과 기록 → 비용 추적         │        │
│  └─────────────────────────────────────────┘        │
│                                                     │
│  Adapters: Claude | Codex | Cursor | Gemini | ...   │
│  Storage:  PostgreSQL (embedded/external)            │
│  Auth:     local_trusted | authenticated             │
└─────────────────────────────────────────────────────┘
```

### 3단계 운영 모델

| 단계 | 설명 | 예시 |
|------|------|------|
| **01** | 목표 정의 | "AI 메모 앱으로 월 $1M MRR 달성" |
| **02** | 팀 구성 | CEO, CTO, 엔지니어, 디자이너, 마케터 — 어떤 봇이든, 어떤 프로바이더든 |
| **03** | 승인 및 실행 | 전략 검토, 예산 설정, 실행, 대시보드 모니터링 |

### 핵심 기능 요약

| 기능 | 설명 |
|------|------|
| **Bring Your Own Agent** | 어떤 에이전트든, 어떤 런타임이든 하나의 조직도에 통합 |
| **Goal Alignment** | 모든 작업이 회사 미션으로 추적. 에이전트는 "무엇"과 "왜"를 안다 |
| **Heartbeats** | 에이전트가 스케줄에 따라 깨어나 작업을 확인하고 실행 |
| **Cost Control** | 에이전트별 월간 예산. 한도 도달 시 자동 정지 |
| **Multi-Company** | 하나의 배포로 여러 회사 운영. 완전한 데이터 격리 |
| **Ticket System** | 모든 대화 추적. 모든 결정 설명. 불변 감사 로그 |
| **Governance** | 채용 승인, 전략 오버라이드, 에이전트 일시정지/종료 가능 |
| **Org Chart** | 계층, 역할, 보고 라인 — 에이전트에게 상사, 직함, 직무설명서 부여 |
| **Plugin System** | 지식 베이스, 커스텀 트레이싱, 큐 등 확장 가능 |
| **Routines** | Cron 기반 반복 작업 자동 실행 |

### 기술 스택

| 레이어 | 기술 |
|--------|------|
| Server | Express 5, Node.js 20+, TypeScript |
| Database | PostgreSQL (embedded-postgres 또는 외부) + Drizzle ORM |
| UI | React 19, Vite 6, TailwindCSS 4, Radix UI |
| Auth | better-auth |
| Realtime | WebSocket (ws) |
| CLI | Commander.js, @clack/prompts |
| Testing | Vitest, Playwright |
| Package Manager | pnpm 9.15+ (monorepo workspaces) |

### 라이선스

MIT (c) 2026 Paperclip
