# 03. 핵심 서비스 분석

Paperclip의 비즈니스 로직은 `server/src/services/` 디렉토리의 서비스 모듈에 집중되어 있다. 66개 이상의 서비스 파일이 존재하며, 여기서는 핵심 서비스를 분석한다.

## Issues Service

> 소스: `server/src/services/issues.ts` (~2,000 lines)

이슈(작업/티켓) 라이프사이클 관리. Paperclip의 핵심 작업 단위.

### 이슈 상태 머신

```
backlog → todo → in_progress → in_review → done
                     │              │
                     ▼              ▼
                  blocked       cancelled
```

| 상태 | 설명 |
|------|------|
| `backlog` | 미처리 백로그 |
| `todo` | 할당됨, 작업 대기 |
| `in_progress` | 에이전트가 작업 중 |
| `in_review` | 리뷰 대기 |
| `blocked` | 차단됨 |
| `done` | 완료 |
| `cancelled` | 취소 |

### 핵심 기능

```typescript
issueService(db) = {
  // CRUD
  list(companyId, filters: IssueFilters): Promise<Issue[]>,
  getById(id): Promise<Issue | null>,
  create(companyId, data: IssueCreateInput): Promise<Issue>,
  update(id, patch: Partial<Issue>): Promise<Issue | null>,
  delete(id): Promise<Issue | null>,

  // 체크아웃/릴리스 (작업 잠금)
  checkout(id, agentId, runId): Promise<CheckoutResult>,
  release(id): Promise<void>,

  // 코멘트
  listComments(issueId, filters?): Promise<IssueComment[]>,
  addComment(issueId, data): Promise<IssueComment>,

  // 인박스
  archiveInbox(userId, issueId): Promise<void>,
  getInboxBadgeCounts(userId): Promise<InboxCounts>,

  // 하트비트 컨텍스트
  getHeartbeatContext(companyId, issueId): Promise<HeartbeatContext>,
}
```

### 이슈 필터

```typescript
interface IssueFilters {
  status?: string;              // 상태 필터
  assigneeAgentId?: string;     // 담당 에이전트
  projectId?: string;           // 프로젝트
  parentId?: string;            // 부모 이슈 (서브태스크)
  labelId?: string;             // 라벨
  originKind?: string;          // 생성 원본 ('manual', 'routine_execution')
  q?: string;                   // 전문 검색
}
```

### 이슈 식별자

각 이슈에는 회사별 고유 식별자가 부여된다:
- 형식: `{회사접두사}-{번호}` (예: `ABC-123`)
- 회사 `issuePrefix` + 자동 증가 `issueCounter`

### 체크아웃 메커니즘

에이전트가 이슈를 작업하려면 반드시 체크아웃해야 한다:
- 원자적 잠금으로 동시 작업 방지
- `checkoutRunId`로 잠금 소유자 추적
- 하트비트 종료 시 자동 릴리스

### 이슈 관계

- **부모/자식**: `parentId`를 통한 서브태스크 구조
- **프로젝트**: `projectId`로 프로젝트 소속
- **목표**: `goalId`로 목표 연결
- **실행 워크스페이스**: `executionWorkspaceId`로 작업 환경 연결
- **생성자**: 사용자 또는 에이전트가 생성 가능

---

## Agents Service

> 소스: `server/src/services/agents.ts` (~600 lines)

에이전트 라이프사이클 및 설정 관리.

### 에이전트 역할

| 역할 | 설명 |
|------|------|
| `general` | 범용 에이전트 |
| `ceo` | 최고경영자 역할 |
| `cto` | 기술 총괄 |
| `engineer` | 엔지니어 |
| `designer` | 디자이너 |
| `marketer` | 마케터 |
| (커스텀) | 사용자 정의 역할 |

### 에이전트 상태

| 상태 | 설명 |
|------|------|
| `idle` | 대기 중 |
| `running` | 실행 중 |
| `paused` | 일시 정지 |

### 일시 정지 사유

| 사유 | 설명 |
|------|------|
| `manual` | 수동 일시 정지 |
| `budget` | 예산 초과 |
| `system` | 시스템에 의한 정지 |

### 설정 리비전 추적

에이전트 설정 변경 시 히스토리가 기록된다:

```typescript
// 추적되는 필드
name, role, title, adapterType, adapterConfig, runtimeConfig, budgetMonthlyCents

// 리비전 메타데이터
{
  createdByAgentId?: string;    // 변경한 에이전트
  createdByUserId?: string;     // 변경한 사용자
  source?: string;              // 변경 출처
  rolledBackFromRevisionId?: string;  // 롤백 원본
}
```

- 이전 리비전으로 롤백 가능
- 변경 감사 추적 완전 지원

### API 키 관리

에이전트별 JWT 형식의 API 키 발급:
- 하트비트 실행 시 단기 JWT 자동 발급
- 에이전트가 Paperclip API를 호출할 때 사용

---

## Budgets Service

> 소스: `server/src/services/budgets.ts` (~1,000 lines)

비용 제어 및 예산 정책 관리.

### 예산 정책 구조

```typescript
interface BudgetPolicy {
  companyId: string;
  scopeType: 'company' | 'agent' | 'project';  // 적용 범위
  scopeId: string;
  metric: 'billed_cents';                        // 측정 기준
  windowKind: 'month' | 'lifetime';              // 기간 유형
  thresholds: BudgetThreshold[];                  // 임계치 목록
}

interface BudgetThreshold {
  type: 'hard_stop' | 'warning';  // 하드 스톱 vs 경고
  amount: number;                  // 금액 (센트)
}
```

### 적용 범위

| 범위 | 설명 |
|------|------|
| `company` | 회사 전체 월간/전체 예산 |
| `agent` | 에이전트별 월간 지출 한도 |
| `project` | 프로젝트별 전체 기간 예산 |

### 임계치 유형

| 유형 | 동작 |
|------|------|
| `hard_stop` | 한도 도달 시 즉시 작업 중단 |
| `warning` | 경고 알림 발생, 작업 계속 |

### 예산 적용 흐름

```
실행 전: 예산 사전 검증
  ├─ hard_stop 도달 → 실행 거부
  └─ 통과 → 실행 허용
         │
실행 후: 비용 기록 및 재평가
  ├─ cost_event 생성
  ├─ 예산 상태 재계산
  └─ 임계치 초과 시 budgetIncident 생성
       ├─ hard_stop → 에이전트 일시정지
       └─ warning → 알림 발생
```

### 인시던트 관리

예산 임계치 초과 시 인시던트가 생성된다:

```typescript
interface BudgetIncident {
  policyId: string;
  thresholdType: 'hard_stop' | 'warning';
  observedAmount: number;      // 관찰된 금액
  resolvedAt?: Date;           // 해결 시각
  resolutionAction?: string;   // 해결 조치
}
```

---

## Goals Service

> 소스: `server/src/services/goals.ts` (~80 lines)

목표/OKR 관리. 경량화된 서비스.

### 목표 계층

```
Company Goal (최상위)
  ├─ Project Goal
  │   ├─ Nested Goal
  │   └─ Nested Goal
  └─ Project Goal
      └─ Nested Goal
```

### 목표 레벨

| 레벨 | 설명 |
|------|------|
| `company` | 회사 수준 목표 (루트) |
| `project` | 프로젝트 수준 목표 |

### 목표 상태

| 상태 | 설명 |
|------|------|
| `active` | 활성 |
| `paused` | 일시 중지 |
| `archived` | 보관 |

### Goal Alignment (목표 정렬)

모든 이슈(작업)는 `goalId`를 통해 목표에 연결된다. 에이전트는 하트비트 컨텍스트에서 자신의 작업이 어떤 목표에 기여하는지 확인할 수 있다. 이를 통해:

- 에이전트가 "무엇을" 하는지뿐 아니라 "왜" 하는지 알 수 있다
- 상위 목표 변경 시 하위 작업에 자동으로 컨텍스트 전파
- 목표 달성 추적 및 보고

---

## Companies Service

> 소스: `server/src/services/companies.ts` (~500 lines)

회사(조직) 관리. Paperclip의 최상위 격리 단위.

### 멀티 컴퍼니

하나의 Paperclip 인스턴스에서 여러 회사를 운영할 수 있다:
- 완전한 데이터 격리
- 독립적인 에이전트, 이슈, 프로젝트, 예산
- 하나의 컨트롤 플레인으로 포트폴리오 관리

### 회사 스키마

```typescript
interface Company {
  id: string;
  name: string;
  description?: string;
  status: 'active' | 'paused' | 'archived';
  issuePrefix: string;            // 이슈 접두사 (예: 'ABC')
  issueCounter: number;           // 이슈 번호 카운터
  budgetMonthlyCents: number;     // 월간 예산 (센트)
  spentMonthlyCents: number;      // 이번 달 사용액 (계산값)
  requireBoardApprovalForNewAgents: boolean;  // 신규 에이전트 승인 필요
  feedbackDataSharingEnabled: boolean;
  brandColor?: string;            // 브랜드 색상
  logoAssetId?: string;           // 로고 에셋
}
```

### 이슈 접두사 로직

회사명에서 자동 생성:
- 대문자 변환, 비알파벳 제거, 첫 3자
- 충돌 시 접미사 추가 (예: ABC → ABCA → ABCAA)
- 유니크 제약 조건

### 회사 포터빌리티

`company-portability` 서비스를 통해 전체 회사 설정을 내보내기/가져오기 가능:
- 조직 구조, 에이전트 설정, 스킬
- 시크릿 스크러빙
- 충돌 처리

---

## Approvals Service

> 소스: `server/src/services/approvals.ts`

거버넌스 승인 관리.

### 승인 대상

| 유형 | 설명 |
|------|------|
| 에이전트 채용 | 새 에이전트 생성 시 보드 승인 |
| 설정 변경 | 에이전트 설정 변경 시 승인 |
| 전략 변경 | 주요 전략 변경 시 승인 |

### 승인 흐름

```
에이전트/시스템이 승인 요청 생성
  │
  ▼
보드 멤버(인간)에게 알림
  │
  ▼
승인 / 거부 / 수정 후 승인
  │
  ▼
승인 시: 관련 작업 진행
거부 시: 관련 작업 취소
```

---

## 기타 주요 서비스

| 서비스 | 파일 | 설명 |
|--------|------|------|
| `workspace-runtime` | 실행 워크스페이스 라이프사이클 (git clone, 런타임 서비스) |
| `company-skills` | 회사별 스킬 설치/관리 |
| `company-portability` | 회사 설정 임포트/엑스포트 |
| `feedback` | 에이전트 피드백 수집/관리 |
| `finance` | 재무 이벤트 및 메트릭 |
| `secrets` | 시크릿 암호화/복호화 관리 |
| `cron` | 크론 스케줄 관리 |
| `live-events` | WebSocket 실시간 이벤트 발행 |
| `plugin-*` (14개) | 플러그인 로더, 워커, 이벤트, 잡 등 |
| `access` | 접근 제어 및 권한 관리 |
| `activity` | 활동 로그 |
