# 10. 루틴 (스케줄 기반 반복 실행)

> 소스: `server/src/services/routines.ts` (~1,400 lines)

## 개요

루틴은 **반복적으로 실행되는 예약 작업**이다. Cron 기반 스케줄, 웹훅, API 호출로 트리거된다.

## 루틴 구조

```typescript
interface Routine {
  id: string;
  companyId: string;
  projectId: string;            // 소속 프로젝트
  assigneeAgentId: string;      // 담당 에이전트
  title: string;                // 작업 제목
  description?: string;         // 작업 설명
  priority: 'low' | 'medium' | 'high';
  status: 'active' | 'paused' | 'archived';
  concurrencyPolicy: string;    // 동시 실행 정책
  catchUpPolicy: string;        // 놓친 실행 정책
  variables: RoutineVariable[]; // 동적 변수
  lastTriggeredAt?: Date;
  lastEnqueuedAt?: Date;
}
```

## 트리거 유형

| 유형 | 설명 | 예시 |
|------|------|------|
| `schedule` | Cron 표현식 기반 | `0 9 * * 1` (매주 월요일 9시) |
| `webhook` | 외부 웹훅 호출 | POST /api/routines/{id}/webhook |
| `api` | API 직접 호출 | POST /api/routines/{id}/trigger |

### 트리거 설정

```typescript
interface RoutineTrigger {
  id: string;
  routineId: string;
  kind: 'schedule' | 'webhook' | 'api';
  cronExpression?: string;      // Cron 표현식 (schedule 타입)
  timezone?: string;            // IANA 타임존 (예: 'Asia/Seoul')
  nextRunAt?: Date;             // 다음 실행 예정 시각
  enabled: boolean;
  publicId?: string;            // 웹훅 공개 ID
  secretId?: string;            // 웹훅 인증 시크릿
}
```

## 동시 실행 정책 (Concurrency Policy)

루틴의 이전 실행이 아직 진행 중일 때 새 실행을 어떻게 처리할지 결정한다.

| 정책 | 동작 |
|------|------|
| `coalesce_if_active` | 진행 중인 실행이 있으면 건너뛰기 |
| `queue_all` | 모든 실행을 큐에 등록 |
| `drop_if_active` | 진행 중이면 새 실행 폐기 |

### 예시

```
coalesce_if_active:
  09:00 트리거 → 실행 시작
  09:05 트리거 → 이전 실행 진행 중 → 건너뛰기
  09:10 트리거 → 이전 실행 완료 → 새 실행 시작

queue_all:
  09:00 트리거 → 실행 시작
  09:05 트리거 → 큐에 등록
  09:10 트리거 → 큐에 등록
  (09:00 완료 → 09:05 실행 시작 → 완료 → 09:10 실행 시작)

drop_if_active:
  09:00 트리거 → 실행 시작
  09:05 트리거 → 이전 실행 진행 중 → 폐기
  09:10 트리거 → 이전 실행 진행 중 → 폐기
```

## 놓친 실행 정책 (Catch-up Policy)

서버 다운타임 등으로 놓친 트리거를 어떻게 처리할지 결정한다.

| 정책 | 동작 |
|------|------|
| `skip_missed` | 놓친 트리거 무시 |
| `catch_up` | 모든 놓친 트리거 실행 (최대 25회) |
| `catch_up_once` | 하나의 배치로 한 번만 실행 |

## 변수 보간 (Variable Interpolation)

루틴 이슈 생성 시 동적 변수를 사용할 수 있다.

```typescript
interface RoutineVariable {
  name: string;       // 변수명
  type: 'string' | 'date' | 'number';
  value: string;      // 값 또는 템플릿
}
```

### 사용 예

루틴 제목에 변수를 포함할 수 있다:
```
"Weekly Report - {{date:YYYY-MM-DD}}"
→ "Weekly Report - 2026-04-03"
```

## 실행 흐름

```
Timer Tick
  │
  ▼
routines.tickScheduledTriggers(now)
  │
  ├─ 각 루틴의 schedule 트리거 확인
  │   ├─ cronExpression이 현재 시각(timezone) 매치?
  │   └─ nextRunAt 계산
  │
  ├─ concurrencyPolicy 확인
  │   ├─ 이전 실행 진행 중? → 정책에 따라 처리
  │   └─ 실행 가능? → 계속
  │
  ├─ 변수 보간 → 이슈 템플릿 생성
  │
  ├─ 이슈 생성 (originKind='routine_execution')
  │   ├─ title: 루틴 제목 (변수 치환됨)
  │   ├─ description: 루틴 설명
  │   ├─ assigneeAgentId: 루틴 담당 에이전트
  │   ├─ projectId: 루틴 프로젝트
  │   └─ priority: 루틴 우선순위
  │
  └─ 에이전트 인박스에 이슈 표시
      │
      ▼
  에이전트 하트비트 → 이슈 체크아웃 → 작업 수행
      │
      ▼
  routine_runs 레코드 생성 (완료/실패 기록)
```

## 루틴 사용 예시

### 일일 고객 지원 보고서

```json
{
  "title": "Daily Support Report - {{date:YYYY-MM-DD}}",
  "description": "고객 지원 통계를 수집하고 일일 보고서를 작성합니다.",
  "assigneeAgentId": "<support-agent-id>",
  "projectId": "<support-project-id>",
  "priority": "medium",
  "concurrencyPolicy": "coalesce_if_active",
  "catchUpPolicy": "skip_missed",
  "triggers": [{
    "kind": "schedule",
    "cronExpression": "0 18 * * *",
    "timezone": "Asia/Seoul"
  }]
}
```

### 주간 코드 리뷰

```json
{
  "title": "Weekly Code Review",
  "description": "지난주 커밋을 리뷰하고 코드 품질 보고서를 작성합니다.",
  "assigneeAgentId": "<cto-agent-id>",
  "priority": "high",
  "concurrencyPolicy": "drop_if_active",
  "catchUpPolicy": "catch_up_once",
  "triggers": [{
    "kind": "schedule",
    "cronExpression": "0 10 * * 1",
    "timezone": "Asia/Seoul"
  }]
}
```

### 소셜 미디어 포스팅

```json
{
  "title": "Social Media Post",
  "description": "트위터와 링크드인에 오늘의 포스트를 작성합니다.",
  "assigneeAgentId": "<marketing-agent-id>",
  "priority": "low",
  "concurrencyPolicy": "queue_all",
  "catchUpPolicy": "catch_up",
  "triggers": [{
    "kind": "schedule",
    "cronExpression": "0 9,14 * * 1-5",
    "timezone": "Asia/Seoul"
  }]
}
```

## 관련 DB 테이블

| 테이블 | 설명 |
|--------|------|
| `routines` | 루틴 정의 |
| `routine_triggers` | 트리거 설정 |
| `routine_runs` | 실행 기록 |

## 상수

| 상수 | 값 | 설명 |
|------|-----|------|
| `MAX_CATCH_UP_RUNS` | 25 | catch_up 정책 최대 실행 횟수 |
