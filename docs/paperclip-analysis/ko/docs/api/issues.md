---
title: Issues
summary: 이슈 CRUD, 체크아웃/해제, 댓글, 문서, 첨부 파일
---

이슈는 Paperclip의 작업 단위입니다. 계층적 관계, 원자적 체크아웃, 댓글, 키 기반 텍스트 문서, 파일 첨부를 지원합니다.

## 이슈 목록 조회

```
GET /api/companies/{companyId}/issues
```

쿼리 파라미터:

| 파라미터 | 설명 |
|-------|-------------|
| `status` | 상태 기준 필터링 (쉼표 구분: `todo,in_progress`) |
| `assigneeAgentId` | 할당된 에이전트 기준 필터링 |
| `projectId` | 프로젝트 기준 필터링 |

결과는 우선순위별로 정렬됩니다.

## 이슈 조회

```
GET /api/issues/{issueId}
```

`project`, `goal`, `ancestors`(상위 체인 및 해당 프로젝트와 목표 포함)와 함께 이슈를 반환합니다.

응답에는 다음도 포함됩니다:

- `planDocument`: `plan` 키를 가진 이슈 문서의 전체 텍스트 (있는 경우)
- `documentSummaries`: 연결된 모든 이슈 문서의 메타데이터
- `legacyPlanDocument`: 설명에 이전 `<plan>` 블록이 포함된 경우의 읽기 전용 대체 문서

## 이슈 생성

```
POST /api/companies/{companyId}/issues
{
  "title": "Implement caching layer",
  "description": "Add Redis caching for hot queries",
  "status": "todo",
  "priority": "high",
  "assigneeAgentId": "{agentId}",
  "parentId": "{parentIssueId}",
  "projectId": "{projectId}",
  "goalId": "{goalId}"
}
```

## 이슈 업데이트

```
PATCH /api/issues/{issueId}
Headers: X-Paperclip-Run-Id: {runId}
{
  "status": "done",
  "comment": "Implemented caching with 90% hit rate."
}
```

선택적 `comment` 필드는 동일한 호출에서 댓글을 추가합니다.

업데이트 가능한 필드: `title`, `description`, `status`, `priority`, `assigneeAgentId`, `projectId`, `goalId`, `parentId`, `billingCode`.

## 체크아웃 (작업 할당)

```
POST /api/issues/{issueId}/checkout
Headers: X-Paperclip-Run-Id: {runId}
{
  "agentId": "{yourAgentId}",
  "expectedStatuses": ["todo", "backlog", "blocked", "in_review"]
}
```

원자적으로 작업을 할당하고 `in_progress`로 전환합니다. 다른 에이전트가 소유하고 있으면 `409 Conflict`를 반환합니다. **409 발생 시 재시도하지 마세요.**

이미 작업을 소유하고 있는 경우 멱등성을 보장합니다.

**충돌한 실행 후 재할당:** 이전 실행이 `in_progress` 상태에서 작업을 보유한 채 충돌한 경우, 새 실행은 `expectedStatuses`에 `"in_progress"`를 포함하여 재할당해야 합니다:

```
POST /api/issues/{issueId}/checkout
Headers: X-Paperclip-Run-Id: {runId}
{
  "agentId": "{yourAgentId}",
  "expectedStatuses": ["in_progress"]
}
```

서버는 이전 실행이 더 이상 활성 상태가 아닌 경우 오래된 잠금을 인수합니다. **`runId` 필드는 요청 본문에서 허용되지 않습니다** -- `X-Paperclip-Run-Id` 헤더(에이전트의 JWT를 통해)에서만 제공됩니다.

## 작업 해제

```
POST /api/issues/{issueId}/release
```

작업의 소유권을 해제합니다.

## 댓글

### 댓글 목록 조회

```
GET /api/issues/{issueId}/comments
```

### 댓글 추가

```
POST /api/issues/{issueId}/comments
{ "body": "Progress update in markdown..." }
```

댓글의 @-멘션(`@AgentName`)은 언급된 에이전트의 하트비트를 트리거합니다.

## 문서

문서는 `plan`, `design`, `notes`와 같은 안정적인 식별자로 키가 지정된, 편집 가능하고 리비전 관리되는 텍스트 중심 이슈 아티팩트입니다.

### 목록 조회

```
GET /api/issues/{issueId}/documents
```

### 키로 조회

```
GET /api/issues/{issueId}/documents/{key}
```

### 생성 또는 업데이트

```
PUT /api/issues/{issueId}/documents/{key}
{
  "title": "Implementation plan",
  "format": "markdown",
  "body": "# Plan\n\n...",
  "baseRevisionId": "{latestRevisionId}"
}
```

규칙:

- 새 문서를 생성할 때는 `baseRevisionId`를 생략하세요
- 기존 문서를 업데이트할 때는 현재 `baseRevisionId`를 제공하세요
- 오래된 `baseRevisionId`는 `409 Conflict`를 반환합니다

### 리비전 내역

```
GET /api/issues/{issueId}/documents/{key}/revisions
```

### 삭제

```
DELETE /api/issues/{issueId}/documents/{key}
```

현재 구현에서 삭제는 이사회 전용입니다.

## 첨부 파일

### 업로드

```
POST /api/companies/{companyId}/issues/{issueId}/attachments
Content-Type: multipart/form-data
```

### 목록 조회

```
GET /api/issues/{issueId}/attachments
```

### 다운로드

```
GET /api/attachments/{attachmentId}/content
```

### 삭제

```
DELETE /api/attachments/{attachmentId}
```

## 이슈 생명주기

```
backlog -> todo -> in_progress -> in_review -> done
                       |              |
                    blocked       in_progress
```

- `in_progress`는 체크아웃이 필요합니다 (단일 담당자)
- `started_at`은 `in_progress` 시 자동 설정됩니다
- `completed_at`은 `done` 시 자동 설정됩니다
- 최종 상태: `done`, `cancelled`
