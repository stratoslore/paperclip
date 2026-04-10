# 02. 하트비트 엔진 (Heartbeat Engine)

> 소스: `server/src/services/heartbeat.ts` (~3,400 lines)

하트비트 엔진은 Paperclip의 **핵심 실행 엔진**이다. 모든 에이전트 작업은 이 엔진을 통해 실행된다.

## 개념

에이전트는 **연속 실행하지 않는다**. 주기적으로 "하트비트"를 통해 깨어나고, 작업을 확인하고, 실행하고, 종료한다. 이 패턴을 통해:

- 비용을 제어할 수 있다 (무한 루프 방지)
- 여러 에이전트를 효율적으로 스케줄링할 수 있다
- 세션 상태를 하트비트 간에 유지할 수 있다
- 감사 로그를 완전하게 남길 수 있다

## 하트비트 실행 주기

```
┌──────────────────────────────────────────────────────┐
│                  Heartbeat Cycle                      │
│                                                      │
│  Timer Tick (30s) ──→ tickTimers(now)                │
│         │                                            │
│         ▼                                            │
│  wakeup_requests / scheduled_tasks 조회              │
│         │                                            │
│         ▼                                            │
│  enqueueRun()                                        │
│    ├─ heartbeat_runs 레코드 생성 (queued)            │
│    └─ 예산 사전 검증                                  │
│         │                                            │
│         ▼                                            │
│  executeRun()                                        │
│    ├─ 어댑터 결정                                     │
│    ├─ 워크스페이스 준비                                │
│    ├─ adapter.execute() ──→ AI 에이전트 실행          │
│    ├─ 결과 캡처 (stdout, stderr, exit code)          │
│    ├─ cost_events 생성                               │
│    └─ heartbeat_run 업데이트                          │
│         │                                            │
│         ▼                                            │
│  heartbeat.completed 이벤트 발행                      │
└──────────────────────────────────────────────────────┘
```

## 핵심 메서드

### `enqueueRun()`

하트비트 실행을 큐에 등록한다.

```typescript
enqueueRun({
  agentId: string;        // 실행할 에이전트 ID
  companyId: string;      // 회사 ID
  source: 'on_demand' | 'trigger' | 'wakeup' | 'retry';  // 실행 이유
  issueIds?: string[];    // 관련 이슈 ID 목록
  approvalIds?: string[]; // 관련 승인 ID 목록
}): Promise<RunEnqueueResult>
```

- `heartbeat_runs` 테이블에 `status='queued'` 레코드 생성
- 에이전트별 예산 한도 사전 검증
- 동시 실행 제한 검사 (기본 1, 최대 10)

### `executeRun()`

큐에 등록된 실행을 실제로 수행한다.

```typescript
executeRun(runId: string): Promise<RunExecutionResult>
```

실행 단계:
1. 어댑터 결정 (에이전트의 `adapterType` 기반)
2. 에이전트 설정 및 런타임 상태 조회
3. 실행 워크스페이스 구현 (git clone, 디렉토리 생성)
4. 런타임 서비스 확보 (컨테이너 시작 등)
5. `adapter.execute(config)` 호출
6. 결과 캡처 (stdout, stderr, exit code, result JSON)
7. 사용량 파싱 → `cost_events` 생성
8. `heartbeat_run` 레코드 업데이트
9. WebSocket으로 라이브 이벤트 발행

### `tickTimers()`

스케줄러에 의해 30초마다 호출된다.

```typescript
tickTimers(now: Date): Promise<{ enqueued: number }>
```

- 대기 중인 wakeup 요청 확인
- 스케줄된 루틴 트리거 확인
- 해당 에이전트들의 실행을 `enqueueRun()`으로 등록

### `reapOrphanedRuns()`

고아 실행(비정상 종료된 실행)을 정리한다.

```typescript
reapOrphanedRuns(opts?: { staleThresholdMs?: number }): Promise<void>
```

## 동시성 제어

| 설정 | 기본값 | 최대값 | 설명 |
|------|--------|--------|------|
| `maxConcurrentRuns` | 1 | 10 | 에이전트당 동시 실행 수 |

- 에이전트별 잠금으로 동시 실행 방지
- 이슈 체크아웃은 원자적 (다른 에이전트가 같은 이슈를 동시에 작업 불가)
- 예산 차감도 원자적 (이중 과금 방지)

## 세션 관리

로컬 어댑터는 하트비트 간에 세션을 유지한다.

### 세션 지원 어댑터

```
claude_local, codex_local, cursor, gemini_local, opencode_local, pi_local
```

### 세션 흐름

```
Heartbeat 1:
  에이전트 실행 → 세션 상태 생성 → agentTaskSessions에 저장

Heartbeat 2:
  이전 세션 로드 → 에이전트 실행 (컨텍스트 유지) → 세션 업데이트

...

세션 크기 초과:
  세션 압축(compaction) 트리거 → 세션 리셋
```

### 세션 테이블 (`agentTaskSessions`)

| 필드 | 설명 |
|------|------|
| `agentId` | 에이전트 ID |
| `issueId` | 이슈 ID |
| `sessionCodec` | 어댑터별 세션 상태 |
| `sessionSize` | 세션 크기 (바이트) |
| `compactedAt` | 마지막 압축 시각 |

## 실행 워크스페이스

에이전트가 작업할 파일시스템 환경을 준비한다.

### 워크스페이스 전략

| 전략 | 설명 |
|------|------|
| `git_worktree` | 기능 브랜치용 git worktree 생성 |
| `project_primary` | 프로젝트의 기본 디렉토리에서 공유 실행 |
| `task_session` | 태스크별 독립 디렉토리 |

### git_worktree 설정

```typescript
{
  type: 'git_worktree',
  baseRef?: string,           // 기본 브랜치 (예: 'main')
  branchTemplate?: string,    // 브랜치 이름 템플릿
  // ... 추가 옵션
}
```

## 환경변수 주입

에이전트 실행 시 자동으로 주입되는 환경변수:

| 변수 | 설명 |
|------|------|
| `PAPERCLIP_AGENT_ID` | 에이전트 ID |
| `PAPERCLIP_COMPANY_ID` | 회사 ID |
| `PAPERCLIP_API_URL` | Paperclip API URL |
| `PAPERCLIP_RUN_ID` | 현재 실행 ID |
| `PAPERCLIP_API_KEY` | 단기 JWT 토큰 |
| `PAPERCLIP_TASK_ID` | 트리거된 이슈 ID (있는 경우) |
| `PAPERCLIP_WAKE_REASON` | 깨어난 이유 |
| `PAPERCLIP_WAKE_COMMENT_ID` | 멘션 코멘트 ID (있는 경우) |
| `PAPERCLIP_APPROVAL_ID` | 승인 ID (승인 트리거 시) |
| `PAPERCLIP_WORKSPACE_*` | 워크스페이스 관련 변수들 |

## 실행 상태

`heartbeat_runs.status` 상태 전이:

```
queued → running → succeeded
                 → failed
                 → cancelled
                 → timed_out
```

## 비용 추적 통합

```
실행 완료
  │
  ▼
result.usageJson 파싱
  ├─ inputTokens
  ├─ outputTokens
  ├─ totalTokens
  ├─ costCents
  └─ model
  │
  ▼
cost_events 레코드 생성
  │
  ▼
예산 서비스 정책 확인
  ├─ 회사 월간: sum(costCents) this month
  ├─ 에이전트 월간: sum(costCents) for agent this month
  └─ 프로젝트 전체: sum(costCents) for project all-time
  │
  ▼
임계치 초과 시: budgetIncident 생성 → 작업 일시정지
```

## 주요 상수

| 상수 | 값 | 설명 |
|------|-----|------|
| `HEARTBEAT_SCHEDULER_INTERVAL_MS` | 30,000 | 스케줄러 틱 간격 |
| `HEARTBEAT_MAX_CONCURRENT_RUNS_DEFAULT` | 1 | 기본 동시 실행 수 |
| `HEARTBEAT_MAX_CONCURRENT_RUNS_MAX` | 10 | 최대 동시 실행 수 |
| `MAX_LIVE_LOG_CHUNK_BYTES` | 8,192 | 로그 스트리밍 청크 크기 |
