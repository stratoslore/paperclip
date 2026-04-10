# 실행 정책: 검토 및 승인 워크플로

Paperclip의 실행 정책 시스템은 작업이 적절한 수준의 감독 하에 완료되도록 보장합니다. 에이전트가 작업을 넘기는 것을 기억하는 것에 의존하는 대신, **런타임이** 검토 및 승인 단계를 자동으로 **적용**합니다.

## 개요

실행 정책은 실행자가 작업을 마친 후 무엇이 일어나야 하는지를 정의하는 이슈의 선택적 구조화 객체입니다. 세 가지 적용 레이어를 지원합니다:

| 레이어 | 목적 | 범위 |
|---|---|---|
| **코멘트 필수** | 모든 에이전트 실행은 이슈에 코멘트를 게시해야 함 | 런타임 불변 (항상 켜짐) |
| **검토 단계** | 검토자가 품질/정확성을 확인하고 변경을 요청할 수 있음 | 이슈별, 선택 사항 |
| **승인 단계** | 매니저/이해관계자가 최종 승인을 내림 | 이슈별, 선택 사항 |

이 레이어들은 조합됩니다. 이슈는 검토만, 승인만, 둘 다 순차적으로, 또는 둘 다 없이(코멘트 필수 안전장치만) 가질 수 있습니다.

## 데이터 모델

### 실행 정책 (이슈 필드: `executionPolicy`)

```ts
interface IssueExecutionPolicy {
  mode: "normal" | "auto";
  commentRequired: boolean;       // always true, enforced by runtime
  stages: IssueExecutionStage[];  // ordered list of review/approval stages
}

interface IssueExecutionStage {
  id: string;                                 // auto-generated UUID
  type: "review" | "approval";                // stage kind
  approvalsNeeded: 1;                         // multi-approval is not supported yet
  participants: IssueExecutionStageParticipant[];
}

interface IssueExecutionStageParticipant {
  id: string;
  type: "agent" | "user";
  agentId?: string | null;    // set when type is "agent"
  userId?: string | null;     // set when type is "user"
}
```

참가자는 에이전트 또는 이사회 사용자일 수 있습니다. 각 단계에는 여러 참가자가 있을 수 있으며, 런타임은 원래 실행자를 제외하면서 명시적으로 요청된 담당자를 우선하여 첫 번째 적격 참가자를 선택합니다.

### 실행 상태 (이슈 필드: `executionState`)

이슈가 현재 정책 워크플로에서 어디에 위치하는지 추적합니다:

```ts
interface IssueExecutionState {
  status: "idle" | "pending" | "changes_requested" | "completed";
  currentStageId: string | null;
  currentStageIndex: number | null;
  currentStageType: "review" | "approval" | null;
  currentParticipant: IssueExecutionStagePrincipal | null;
  returnAssignee: IssueExecutionStagePrincipal | null;
  completedStageIds: string[];
  lastDecisionId: string | null;
  lastDecisionOutcome: "approved" | "changes_requested" | null;
}
```

### 실행 결정 (테이블: `issue_execution_decisions`)

모든 검토/승인 작업의 감사 추적:

```ts
interface IssueExecutionDecision {
  id: string;
  companyId: string;
  issueId: string;
  stageId: string;
  stageType: "review" | "approval";
  actorAgentId: string | null;
  actorUserId: string | null;
  outcome: "approved" | "changes_requested";
  body: string;              // required comment explaining the decision
  createdByRunId: string | null;
  createdAt: Date;
}
```

## 워크플로

### 정상 경로: 검토 + 승인

```
┌──────────┐    executor     ┌───────────┐   reviewer    ┌───────────┐   approver    ┌──────┐
│  todo     │───completes───▶│ in_review  │───approves───▶│ in_review │───approves───▶│ done │
│ (Coder)  │    work         │ (QA)      │               │ (CTO)     │               │      │
└──────────┘                 └───────────┘               └───────────┘               └──────┘
```

1. **이슈 생성** 시 검토 단계(예: QA)와 승인 단계(예: CTO)를 지정하는 `executionPolicy`를 포함합니다.
2. **실행자가 작업**을 `in_progress` 상태에서 수행합니다.
3. **실행자가 `done`으로 전환** -- 런타임이 이를 가로챕니다:
   - 상태가 `in_review`로 변경됩니다 (`done`이 아님)
   - 이슈가 첫 번째 검토자에게 재할당됩니다
   - `executionState`가 검토 단계에서 `pending`으로 진입합니다
4. **검토자가 검토**하고 코멘트와 함께 `done`으로 전환합니다:
   - 결정 기록이 생성됩니다: `{ outcome: "approved" }`
   - 이슈는 `in_review` 상태를 유지하고 승인자에게 재할당됩니다
   - `executionState`가 승인 단계로 진행합니다
5. **승인자가 승인**하고 코멘트와 함께 `done`으로 전환합니다:
   - 결정 기록이 생성됩니다: `{ outcome: "approved" }`
   - `executionState.status`가 `completed`가 됩니다
   - 이슈가 실제 `done` 상태에 도달합니다

### 변경 요청 흐름

```
┌───────────┐   reviewer requests   ┌─────────────┐   executor    ┌───────────┐
│ in_review  │───changes────────────▶│ in_progress  │───resubmits──▶│ in_review │
│ (QA)      │                       │ (Coder)      │               │ (QA)      │
└───────────┘                       └──────────────┘               └───────────┘
```

1. **검토자가 변경을 요청**하고 무엇을 변경해야 하는지 설명하는 코멘트와 함께 `done` 이외의 상태(일반적으로 `in_progress`)로 전환합니다.
2. 런타임이 자동으로:
   - 상태를 `in_progress`로 설정합니다
   - 원래 실행자(`returnAssignee`에 저장)에게 재할당합니다
   - `executionState.status`를 `changes_requested`로 설정합니다
3. **실행자가 변경**하고 다시 `done`으로 전환합니다.
4. 런타임이 **동일한 검토 단계**(처음부터가 아님)로, 동일한 검토자에게 다시 라우팅합니다.
5. 검토자가 승인할 때까지 이 루프가 계속됩니다.

### 정책 변형

**검토만** (승인 단계 없음):
```json
{
  "stages": [
    { "type": "review", "participants": [{ "type": "agent", "agentId": "qa-agent-id" }] }
  ]
}
```
실행자 완료 → 검토자 승인 → done.

**승인만** (검토 단계 없음):
```json
{
  "stages": [
    { "type": "approval", "participants": [{ "type": "user", "userId": "manager-user-id" }] }
  ]
}
```
실행자 완료 → 승인자 서명 → done.

**복수 검토자/승인자:**
각 단계는 여러 참가자를 지원합니다. 런타임은 자기 검토를 방지하기 위해 원래 실행자를 제외하고 한 명을 선택합니다.

## 코멘트 필수 안전장치

검토 단계와 독립적으로, 모든 이슈 바인딩 에이전트 실행은 코멘트를 남겨야 합니다. 이는 런타임 수준에서 적용됩니다:

1. **실행 완료** -- 런타임이 에이전트가 이 실행에 대한 코멘트를 게시했는지 확인합니다.
2. **코멘트가 없으면**: `issueCommentStatus`가 `retry_queued`로 설정되고, 에이전트가 `missing_issue_comment` 이유로 한 번 더 깨어납니다.
3. **재시도 후에도 코멘트가 없으면**: `issueCommentStatus`가 `retry_exhausted`로 설정됩니다. 더 이상 재시도하지 않습니다. 실패가 기록됩니다.
4. **코멘트가 게시되면**: `issueCommentStatus`가 `satisfied`로 설정되고 코멘트 ID에 연결됩니다.

이를 통해 에이전트가 작업을 완료하지만 무슨 일이 있었는지 흔적을 남기지 않는 무음 완료를 방지합니다.

### 실행 수준 추적 필드

| 필드 | 설명 |
|---|---|
| `issueCommentStatus` | `satisfied`, `retry_queued`, 또는 `retry_exhausted` |
| `issueCommentSatisfiedByCommentId` | 요구 사항을 충족한 코멘트에 연결 |
| `issueCommentRetryQueuedAt` | 재시도 깨어남이 예약된 타임스탬프 |

## 접근 제어

- **활성 검토자/승인자**(실행 상태의 `currentParticipant`)만 현재 단계를 진행하거나 거부할 수 있습니다.
- 참가자가 아닌 사람이 이슈를 전환하려 하면 `422 Unprocessable Entity` 오류를 받습니다.
- 승인과 변경 요청 모두 **코멘트가 필수**입니다 -- 비어있거나 공백만 있는 코멘트는 거부됩니다.

## API 사용법

### 이슈 생성 시 실행 정책 설정

```bash
POST /api/companies/{companyId}/issues
{
  "title": "Implement feature X",
  "assigneeAgentId": "coder-agent-id",
  "executionPolicy": {
    "mode": "normal",
    "commentRequired": true,
    "stages": [
      {
        "type": "review",
        "participants": [
          { "type": "agent", "agentId": "qa-agent-id" }
        ]
      },
      {
        "type": "approval",
        "participants": [
          { "type": "user", "userId": "cto-user-id" }
        ]
      }
    ]
  }
}
```

생략하면 단계 ID와 참가자 ID가 자동 생성됩니다. 단계 내 중복 참가자는 자동으로 중복 제거됩니다. 유효한 참가자가 없는 단계는 제거됩니다. 유효한 단계가 남지 않으면 정책은 `null`로 설정됩니다.

### 기존 이슈의 실행 정책 업데이트

```bash
PATCH /api/issues/{issueId}
{
  "executionPolicy": { ... }
}
```

검토가 진행 중일 때 정책이 제거(`null`)되면 실행 상태가 지워지고 이슈는 원래 실행자에게 반환됩니다.

### 단계 진행 (검토자/승인자가 승인)

활성 검토자 또는 승인자가 코멘트와 함께 이슈를 `done`으로 전환합니다:

```bash
PATCH /api/issues/{issueId}
{
  "status": "done",
  "comment": "Reviewed — implementation looks correct, tests pass."
}
```

런타임이 이것이 워크플로를 완료하는지 다음 단계로 진행하는지 결정합니다.

### 변경 요청

활성 검토자가 코멘트와 함께 `done`이 아닌 상태로 전환합니다:

```bash
PATCH /api/issues/{issueId}
{
  "status": "in_progress",
  "comment": "Button alignment is off on mobile. Please fix the flex container."
}
```

런타임이 자동으로 원래 실행자에게 재할당합니다.

## UI

### 새 이슈 대화상자

새 이슈를 생성할 때, 담당자 선택기 옆에 **Reviewer**와 **Approver** 버튼이 나타납니다. 클릭하면 참가자 피커가 열리며 다음이 표시됩니다:
- "No reviewer" / "No approver" (해제)
- "Me" (현재 사용자)
- 전체 에이전트 및 이사회 사용자 목록

선택 사항이 `executionPolicy.stages` 배열을 자동으로 구성합니다.

### 이슈 속성 패널

기존 이슈의 경우, 속성 패널에 편집 가능한 **Reviewer**와 **Approver** 필드가 표시됩니다. 단계별로 여러 참가자를 추가할 수 있습니다. 변경 사항은 API를 통해 이슈의 `executionPolicy`에 저장됩니다.

## 설계 원칙

1. **런타임 적용, 프롬프트 의존 아님.** 에이전트가 작업을 넘기는 것을 기억할 필요가 없습니다. 런타임이 상태 전환을 가로채고 그에 따라 라우팅합니다.
2. **반복적, 종료적이 아님.** 검토는 루프(변경 요청 → 수정 → 재검토)이지 일회성 게이트가 아닙니다. 시스템은 재제출 시 동일한 단계로 돌아갑니다.
3. **유연한 역할.** 참가자는 에이전트 또는 사용자일 수 있습니다. 모든 조직에 "QA"가 있는 것은 아닙니다 -- 검토자/승인자 패턴은 동료 검토, 매니저 승인, 컴플라이언스 점검, 또는 모든 다자간 워크플로에 충분히 범용적입니다.
4. **감사 가능.** 모든 결정은 행위자, 결과, 코멘트, 실행 ID와 함께 기록됩니다. 전체 검토 이력은 이슈별로 조회 가능합니다.
5. **단일 실행 불변 유지.** 검토 깨어남과 코멘트 재시도는 이슈당 한 번에 하나의 에이전트 실행만 활성화될 수 있다는 기존 제약을 준수합니다.
