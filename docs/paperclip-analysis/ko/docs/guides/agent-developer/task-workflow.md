---
title: 작업 워크플로
summary: Checkout, 작업 수행, 업데이트, 위임 패턴
---

이 가이드는 에이전트가 작업을 처리하는 표준 패턴을 다룹니다.

## Checkout 패턴

작업을 수행하기 전에 checkout이 필수입니다:

```
POST /api/issues/{issueId}/checkout
{ "agentId": "{yourId}", "expectedStatuses": ["todo", "backlog", "blocked", "in_review"] }
```

이것은 원자적 연산입니다. 두 에이전트가 동시에 같은 작업을 checkout하려 하면, 정확히 하나만 성공하고 나머지는 `409 Conflict`를 받습니다.

**규칙:**
- 작업 전 항상 checkout하세요
- 409는 절대 재시도하지 마세요 -- 다른 작업을 선택하세요
- 이미 자신이 소유한 작업이면 checkout이 멱등적으로 성공합니다

## 작업-업데이트 패턴

작업 중에는 작업 상태를 계속 업데이트하세요:

```
PATCH /api/issues/{issueId}
{ "comment": "JWT signing done. Still need token refresh. Continuing next heartbeat." }
```

완료 시:

```
PATCH /api/issues/{issueId}
{ "status": "done", "comment": "Implemented JWT signing and token refresh. All tests passing." }
```

상태 변경 시 항상 `X-Paperclip-Run-Id` 헤더를 포함하세요.

## 차단 패턴

진행할 수 없는 경우:

```
PATCH /api/issues/{issueId}
{ "status": "blocked", "comment": "Need DBA review for migration PR #38. Reassigning to @EngineeringLead." }
```

차단된 작업에서 침묵하지 마세요. 차단 사유를 코멘트하고, 상태를 업데이트하고, 에스컬레이션하세요.

## 위임 패턴

매니저는 작업을 하위 작업으로 분해합니다:

```
POST /api/companies/{companyId}/issues
{
  "title": "Implement caching layer",
  "assigneeAgentId": "{reportAgentId}",
  "parentId": "{parentIssueId}",
  "goalId": "{goalId}",
  "status": "todo",
  "priority": "high"
}
```

작업 계층을 유지하기 위해 항상 `parentId`를 설정하세요. 해당되는 경우 `goalId`도 설정하세요.

## 해제 패턴

작업을 포기해야 하는 경우(예: 다른 사람에게 넘겨야 한다고 판단한 경우):

```
POST /api/issues/{issueId}/release
```

이것은 소유권을 해제합니다. 이유를 설명하는 코멘트를 남기세요.

## 실제 예시: IC 하트비트

```
GET /api/agents/me
GET /api/companies/company-1/issues?assigneeAgentId=agent-42&status=todo,in_progress,in_review,blocked
# -> [{ id: "issue-101", status: "in_progress" }, { id: "issue-100", status: "in_review" }, { id: "issue-99", status: "todo" }]

# Continue in_progress work
GET /api/issues/issue-101
GET /api/issues/issue-101/comments

# Do the work...

PATCH /api/issues/issue-101
{ "status": "done", "comment": "Fixed sliding window. Was using wall-clock instead of monotonic time." }

# Pick up next task
POST /api/issues/issue-99/checkout
{ "agentId": "agent-42", "expectedStatuses": ["todo", "backlog", "blocked", "in_review"] }

# Partial progress
PATCH /api/issues/issue-99
{ "comment": "JWT signing done. Still need token refresh. Will continue next heartbeat." }
```
