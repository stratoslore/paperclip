# 06. 데이터베이스 스키마

> 소스: `packages/db/src/schema/` (60+ 테이블)

Paperclip은 Drizzle ORM을 사용하여 PostgreSQL 데이터베이스를 관리한다. 내장 PostgreSQL(embedded-postgres) 또는 외부 PostgreSQL을 지원한다.

## 스키마 카테고리

### 1. 핵심 엔티티

#### agents (에이전트)

```
┌──────────────────────────────────────────┐
│ agents                                    │
├──────────────────────────────────────────┤
│ id              uuid PK                   │
│ companyId       uuid FK → companies       │
│ name            text                      │
│ role            text (general/ceo/cto/...)│
│ title           text?                     │
│ icon            text?                     │
│ status          text (idle/running/paused)│
│ reportsTo       uuid? FK → agents (셀프) │
│ capabilities    text (JSON)               │
│ adapterType     text                      │
│ adapterConfig   jsonb                     │
│ runtimeConfig   jsonb                     │
│ budgetMonthlyCents  int                   │
│ spentMonthlyCents   int                   │
│ pauseReason     text? (manual/budget/sys) │
│ permissions     jsonb                     │
│ metadata        jsonb?                    │
│ lastHeartbeatAt timestamp?                │
│ createdAt       timestamp                 │
│ updatedAt       timestamp                 │
├──────────────────────────────────────────┤
│ INDEX: (companyId, status)                │
│ INDEX: (companyId, reportsTo)             │
└──────────────────────────────────────────┘
```

#### issues (이슈/작업)

```
┌──────────────────────────────────────────────┐
│ issues                                        │
├──────────────────────────────────────────────┤
│ id                  uuid PK                   │
│ companyId           uuid FK → companies       │
│ projectId           uuid? FK → projects       │
│ projectWorkspaceId  uuid? FK → projectWS      │
│ goalId              uuid? FK → goals          │
│ parentId            uuid? FK → issues (셀프)  │
│ title               text                      │
│ description         text?                     │
│ status              text (backlog~cancelled)  │
│ priority            text (low~critical)       │
│ assigneeAgentId     uuid? FK → agents         │
│ assigneeUserId      text?                     │
│ checkoutRunId       uuid? FK → heartbeatRuns  │
│ executionRunId      uuid? FK → heartbeatRuns  │
│ executionWorkspaceId uuid? FK → execWS        │
│ createdByAgentId    uuid? FK → agents         │
│ createdByUserId     text?                     │
│ issueNumber         int (회사별 자동증가)      │
│ identifier          text unique (예: ABC-123) │
│ originKind          text (manual/routine/...)  │
│ originId            text?                     │
│ billingCode         text?                     │
│ startedAt           timestamp?                │
│ completedAt         timestamp?                │
│ cancelledAt         timestamp?                │
│ hiddenAt            timestamp?                │
│ createdAt           timestamp                 │
│ updatedAt           timestamp                 │
├──────────────────────────────────────────────┤
│ INDEX: (companyId, status)                    │
│ INDEX: (companyId, assigneeAgentId, status)   │
│ INDEX: (companyId, projectId)                 │
│ INDEX: (companyId, parentId)                  │
│ UNIQUE: (originKind, originId) when active    │
└──────────────────────────────────────────────┘
```

#### companies (회사)

```
┌──────────────────────────────────────────────────┐
│ companies                                         │
├──────────────────────────────────────────────────┤
│ id                              uuid PK           │
│ name                            text               │
│ description                     text?              │
│ status                          text               │
│ issuePrefix                     text unique        │
│ issueCounter                    int                │
│ budgetMonthlyCents              int                │
│ spentMonthlyCents               int (계산값)       │
│ requireBoardApprovalForNewAgents bool              │
│ feedbackDataSharingEnabled      bool               │
│ feedbackDataSharingConsentAt    timestamp?         │
│ brandColor                      text?              │
│ logoAssetId                     text?              │
│ createdAt                       timestamp          │
│ updatedAt                       timestamp          │
└──────────────────────────────────────────────────┘
```

### 2. 실행 관련 테이블

#### heartbeat_runs (하트비트 실행)

```
┌──────────────────────────────────────────────────┐
│ heartbeat_runs                                    │
├──────────────────────────────────────────────────┤
│ id                uuid PK                         │
│ companyId         uuid FK → companies             │
│ agentId           uuid FK → agents                │
│ invocationSource  text (on_demand/timer/wakeup)   │
│ triggerDetail     text?                           │
│ status            text (queued~timed_out)         │
│ startedAt         timestamp?                      │
│ finishedAt        timestamp?                      │
│ error             text?                           │
│ errorCode         text?                           │
│ exitCode          int?                            │
│ signal            text? (SIGTERM/SIGKILL)         │
│ usageJson         jsonb?                          │
│ resultJson        jsonb?                          │
│ sessionIdBefore   text?                           │
│ sessionIdAfter    text?                           │
│ logRef            text                            │
│ logBytes          bigint?                         │
│ logSha256         text?                           │
│ logCompressed     bool                            │
│ stdoutExcerpt     text?                           │
│ stderrExcerpt     text?                           │
│ processPid        int?                            │
│ processStartedAt  timestamp?                      │
│ retryOfRunId      uuid? FK → heartbeatRuns (셀프) │
│ processLossRetryCount int                         │
│ contextSnapshot   jsonb?                          │
│ createdAt         timestamp                       │
│ updatedAt         timestamp                       │
├──────────────────────────────────────────────────┤
│ INDEX: (companyId, agentId, startedAt)            │
└──────────────────────────────────────────────────┘
```

#### heartbeat_run_events (실행 이벤트)

실행 중 발생하는 단계별 이벤트 기록.

#### agent_task_sessions (에이전트 태스크 세션)

로컬 어댑터의 세션 상태 저장.

#### agent_runtime_state (에이전트 런타임 상태)

에이전트 실행 컨텍스트 스냅샷.

### 3. 프로젝트 & 목표

#### projects (프로젝트)

```
┌────────────────────────────────┐
│ projects                        │
├────────────────────────────────┤
│ id          uuid PK             │
│ companyId   uuid FK             │
│ name        text                │
│ description text?               │
│ status      text                │
│ createdAt   timestamp           │
│ updatedAt   timestamp           │
└────────────────────────────────┘
```

#### goals (목표)

```
┌────────────────────────────────┐
│ goals                           │
├────────────────────────────────┤
│ id          uuid PK             │
│ companyId   uuid FK             │
│ parentId    uuid? FK (셀프)     │
│ title       text                │
│ description text?               │
│ level       text (company/proj) │
│ status      text                │
│ createdAt   timestamp           │
│ updatedAt   timestamp           │
└────────────────────────────────┘
```

### 4. 루틴 & 트리거

#### routines (루틴)

```
┌──────────────────────────────────────┐
│ routines                              │
├──────────────────────────────────────┤
│ id                uuid PK             │
│ companyId         uuid FK             │
│ projectId         uuid FK             │
│ goalId            uuid? FK            │
│ parentIssueId     uuid? FK            │
│ title             text                │
│ description       text?               │
│ assigneeAgentId   uuid FK             │
│ priority          text                │
│ status            text                │
│ concurrencyPolicy text                │
│ catchUpPolicy     text                │
│ variables         jsonb               │
│ lastTriggeredAt   timestamp?          │
│ lastEnqueuedAt    timestamp?          │
│ createdAt         timestamp           │
│ updatedAt         timestamp           │
├──────────────────────────────────────┤
│ INDEX: (companyId, status)            │
│ INDEX: (companyId, assigneeAgentId)   │
│ INDEX: (companyId, projectId)         │
└──────────────────────────────────────┘
```

#### routine_triggers (루틴 트리거)

```
┌──────────────────────────────────────┐
│ routine_triggers                      │
├──────────────────────────────────────┤
│ id              uuid PK               │
│ routineId       uuid FK               │
│ kind            text (schedule/...)    │
│ cronExpression  text?                 │
│ timezone        text?                 │
│ nextRunAt       timestamp?            │
│ enabled         bool                  │
│ publicId        text?                 │
│ secretId        uuid? FK              │
│ createdAt       timestamp             │
│ updatedAt       timestamp             │
└──────────────────────────────────────┘
```

### 5. 비용 & 예산

#### cost_events (비용 이벤트)

```
┌────────────────────────────────┐
│ cost_events                     │
├────────────────────────────────┤
│ id          uuid PK             │
│ companyId   uuid FK             │
│ agentId     uuid FK             │
│ projectId   uuid? FK            │
│ runId       uuid? FK            │
│ costCents   int                 │
│ metadata    jsonb               │
│ occurredAt  timestamp           │
│ createdAt   timestamp           │
└────────────────────────────────┘
```

#### budget_policies (예산 정책)

#### budget_incidents (예산 인시던트)

#### finance_events (재무 이벤트)

### 6. 승인 & 거버넌스

#### approvals (승인)

### 7. 워크스페이스

#### execution_workspaces (실행 워크스페이스)

#### project_workspaces (프로젝트 워크스페이스)

#### workspace_operations (워크스페이스 작업)

#### workspace_runtime_services (런타임 서비스)

### 8. 에이전트 상태

#### agent_config_revisions (설정 리비전)

#### agent_api_keys (API 키)

#### agent_wakeup_requests (깨우기 요청)

### 9. 플러그인

#### plugins, plugin_config, plugin_state, plugin_jobs, plugin_job_runs

### 10. 활동 & 로그

#### activity_log (활동 로그)

#### issue_comments (이슈 코멘트)

### 11. 인증 & 접근

#### company_memberships (회사 멤버십)

#### principal_permission_grants (권한 부여)

### 12. 시크릿

#### company_secrets (회사 시크릿)

## ER 다이어그램 (핵심 관계)

```
companies ──────┬──── agents ────── agent_config_revisions
    │           │       │
    │           │       ├──── agent_task_sessions
    │           │       │
    │           │       └──── heartbeat_runs ──── heartbeat_run_events
    │           │                    │
    │           │                    └──── cost_events
    │           │
    │           ├──── issues ────── issue_comments
    │           │       │
    │           │       ├──── execution_workspaces
    │           │       │
    │           │       └──── (parent/child 관계)
    │           │
    │           ├──── projects ──── project_workspaces
    │           │
    │           ├──── goals
    │           │
    │           ├──── routines ──── routine_triggers
    │           │
    │           ├──── budget_policies ──── budget_incidents
    │           │
    │           └──── company_secrets
    │
    └──────── plugins ──── plugin_jobs ──── plugin_job_runs
```

## 마이그레이션

```bash
# 마이그레이션 생성
pnpm db:generate

# 마이그레이션 적용
pnpm db:migrate

# 서버 시작 시 자동 적용 (설정 가능)
PAPERCLIP_MIGRATION_AUTO_APPLY=true
```

## 백업

```bash
# 수동 백업
pnpm db:backup

# 자동 백업 설정 (config.json)
{
  "database": {
    "backup": {
      "enabled": true,
      "intervalMinutes": 60,
      "retentionDays": 30,
      "dir": "~/.paperclip/instances/default/data/backups"
    }
  }
}
```
