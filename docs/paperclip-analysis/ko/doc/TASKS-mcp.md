# 작업 관리 MCP 인터페이스

Paperclip 작업 관리 시스템의 함수 계약. 에이전트(및 외부 도구)가 MCP를 통해
사용할 수 있는 작업을 정의합니다. 기본 데이터 모델은
[TASKS.md](./TASKS.md)를 참조하세요.

모든 작업은 JSON을 반환합니다. ID는 UUID입니다. 타임스탬프는 ISO 8601입니다.
이슈 식별자(예: `ENG-123`)는 이슈 `id`가 필요한 모든 곳에서 사용할 수 있습니다.

---

## 이슈

### `list_issues`

워크스페이스의 이슈를 목록 조회 및 필터링합니다.

| 매개변수          | 타입     | 필수 | 참고                                                                                            |
| ----------------- | -------- | ---- | ----------------------------------------------------------------------------------------------- |
| `query`           | string   | 아니오 | 제목과 설명에서 자유 텍스트 검색                                                                |
| `teamId`          | string   | 아니오 | 팀별 필터                                                                                       |
| `status`         | string   | 아니오 | 특정 워크플로 상태별 필터                                                                       |
| `stateType`       | string   | 아니오 | 상태 카테고리별 필터: `triage`, `backlog`, `unstarted`, `started`, `completed`, `cancelled`      |
| `assigneeId`      | string   | 아니오 | 담당자(에이전트 ID)별 필터                                                                      |
| `projectId`       | string   | 아니오 | 프로젝트별 필터                                                                                 |
| `parentId`        | string   | 아니오 | 상위 이슈별 필터(하위 이슈 반환)                                                                |
| `labelIds`        | string[] | 아니오 | 이 레이블 전부를 가진 이슈만 필터                                                               |
| `priority`        | number   | 아니오 | 우선순위별 필터 (0-4)                                                                           |
| `includeArchived` | boolean  | 아니오 | 보관된 이슈 포함. 기본값: false                                                                 |
| `orderBy`         | string   | 아니오 | `created`, `updated`, `priority`, `due_date`. 기본값: `created`                                 |
| `limit`           | number   | 아니오 | 최대 결과 수. 기본값: 50                                                                        |
| `after`           | string   | 아니오 | 다음 페이지 커서                                                                                |
| `before`          | string   | 아니오 | 이전 페이지 커서                                                                                |

**반환값:** `{ issues: Issue[], pageInfo: { hasNextPage, endCursor, hasPreviousPage, startCursor } }`

---

### `get_issue`

ID 또는 식별자로 단일 이슈를 조회하며, 모든 관계를 확장하여 반환합니다.

| 매개변수  | 타입   | 필수 | 참고                                              |
| --------- | ------ | ---- | -------------------------------------------------- |
| `id`      | string | 예   | UUID 또는 사람이 읽을 수 있는 식별자 (예: `ENG-123`) |

**반환값:** 다음을 포함하는 전체 `Issue` 객체:

- `state` (확장된 WorkflowState)
- `assignee` (확장된 Agent, 설정된 경우)
- `labels` (확장된 Label[])
- `relations` (확장된 관련 이슈를 포함한 IssueRelation[])
- `children` (하위 이슈 요약: id, identifier, title, state, assignee)
- `parent` (요약, 하위 이슈인 경우)
- `comments` (Comment[], 최신순)

---

### `create_issue`

새 이슈를 생성합니다.

| 매개변수      | 타입     | 필수   | 참고                                          |
| ------------- | -------- | ------ | --------------------------------------------- |
| `title`       | string   | 예     |                                               |
| `teamId`      | string   | 예     | 이슈가 속하는 팀                              |
| `description` | string   | 아니오 | Markdown                                      |
| `status`     | string   | 아니오 | 워크플로 상태. 기본값: 팀의 기본 상태         |
| `priority`    | number   | 아니오 | 0-4. 기본값: 0 (없음)                         |
| `estimate`    | number   | 아니오 | 포인트 추정치                                 |
| `dueDate`     | string   | 아니오 | ISO 날짜                                      |
| `assigneeId`  | string   | 아니오 | 할당할 에이전트                               |
| `projectId`   | string   | 아니오 | 연결할 프로젝트                               |
| `milestoneId` | string   | 아니오 | 프로젝트 내 마일스톤                          |
| `parentId`    | string   | 아니오 | 상위 이슈 (하위 이슈로 만듦)                  |
| `goalId`      | string   | 아니오 | 연결된 목표/목적                              |
| `labelIds`    | string[] | 아니오 | 적용할 레이블                                 |
| `sortOrder`   | number   | 아니오 | 뷰 내 정렬 순서                               |

**반환값:** 계산된 필드(`identifier`, `createdAt` 등)가 포함된 생성된 `Issue` 객체.

**부수 효과:**

- `parentId`가 설정된 경우, 명시적으로 제공되지 않으면 상위 이슈의 `projectId`를 상속
- `identifier`는 팀 키 + 다음 시퀀스 번호로 자동 생성

---

### `update_issue`

기존 이슈를 업데이트합니다.

| 매개변수      | 타입     | 필수   | 참고                                         |
| ------------- | -------- | ------ | -------------------------------------------- |
| `id`          | string   | 예     | UUID 또는 식별자                             |
| `title`       | string   | 아니오 |                                              |
| `description` | string   | 아니오 |                                              |
| `status`     | string   | 아니오 | 새 워크플로 상태로 전환                      |
| `priority`    | number   | 아니오 | 0-4                                          |
| `estimate`    | number   | 아니오 |                                              |
| `dueDate`     | string   | 아니오 | ISO 날짜, 또는 `null`로 초기화               |
| `assigneeId`  | string   | 아니오 | 에이전트 ID, 또는 `null`로 할당 해제         |
| `projectId`   | string   | 아니오 | 프로젝트 ID, 또는 `null`로 프로젝트에서 제거 |
| `milestoneId` | string   | 아니오 | 마일스톤 ID, 또는 `null`로 초기화            |
| `parentId`    | string   | 아니오 | 재배치, 또는 `null`로 독립 이슈로 승격       |
| `goalId`      | string   | 아니오 | 목표 ID, 또는 `null`로 연결 해제             |
| `labelIds`    | string[] | 아니오 | 모든 레이블을 **교체** (추가가 아님)         |
| `teamId`      | string   | 아니오 | 다른 팀으로 이동                             |
| `sortOrder`   | number   | 아니오 | 뷰 내 정렬 순서                              |

**반환값:** 업데이트된 `Issue` 객체.

**부수 효과:**

- `status`를 `started` 카테고리 상태로 변경하면 `startedAt`이 설정됨 (아직 설정되지 않은 경우)
- `status`를 `completed`로 변경하면 `completedAt`이 설정됨
- `status`를 `cancelled`로 변경하면 `cancelledAt`이 설정됨
- 하위 이슈 자동 닫기가 활성화된 상태에서 `completed`/`cancelled`로 이동하면 열린 하위 이슈가 완료 처리됨
- `teamId`를 변경하면 식별자가 재할당됨 (예: `ENG-42` → `DES-18`); 기존 식별자는 `previousIdentifiers`에 보존

---

### `archive_issue`

이슈를 소프트 보관합니다. `archivedAt`을 설정합니다. 삭제하지 않습니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `id`      | string | 예   |

**반환값:** `{ success: true }`

---

### `list_my_issues`

특정 에이전트에게 할당된 이슈를 목록 조회합니다. `assigneeId`가 미리 채워진
`list_issues`의 편의 래퍼입니다.

| 매개변수    | 타입   | 필수   | 참고                           |
| ----------- | ------ | ------ | ------------------------------ |
| `agentId`   | string | 예     | 이슈를 조회할 에이전트         |
| `stateType` | string | 아니오 | 상태 카테고리별 필터           |
| `orderBy`   | string | 아니오 | 기본값: `priority`             |
| `limit`     | number | 아니오 | 기본값: 50                     |

**반환값:** `list_issues`와 동일한 형태.

---

## 워크플로 상태

### `list_workflow_states`

팀의 워크플로 상태를 카테고리별로 그룹화하여 목록 조회합니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `teamId`  | string | 예   |

**반환값:** `{ states: WorkflowState[] }` -- 카테고리별(triage, backlog, unstarted, started, completed, cancelled), 그리고 각 카테고리 내에서 `position`별로 정렬됨.

---

### `get_workflow_state`

이름 또는 ID로 워크플로 상태를 조회합니다.

| 매개변수  | 타입   | 필수 | 참고               |
| --------- | ------ | ---- | ------------------ |
| `teamId`  | string | 예   |                    |
| `query`   | string | 예   | 상태 이름 또는 UUID |

**반환값:** 단일 `WorkflowState` 객체.

---

## 팀

### `list_teams`

워크스페이스의 모든 팀을 목록 조회합니다.

| 매개변수  | 타입   | 필수   |
| --------- | ------ | ------ | -------------- |
| `query`   | string | 아니오 | 이름별 필터    |

**반환값:** `{ teams: Team[] }`

---

### `get_team`

이름, 키, 또는 ID로 팀을 조회합니다.

| 매개변수  | 타입   | 필수 | 참고                          |
| --------- | ------ | ---- | ----------------------- |
| `query`   | string | 예   | 팀 이름, 키, 또는 UUID  |

**반환값:** 단일 `Team` 객체.

---

## 프로젝트

### `list_projects`

워크스페이스의 프로젝트를 목록 조회합니다.

| 매개변수          | 타입    | 필수   | 참고                                                                            |
| ----------------- | ------- | ------ | ------------------------------------------------------------------------------- |
| `teamId`          | string  | 아니오 | 이 팀의 이슈를 포함하는 프로젝트로 필터                                         |
| `status`          | string  | 아니오 | 상태별 필터: `backlog`, `planned`, `in_progress`, `completed`, `cancelled`       |
| `includeArchived` | boolean | 아니오 | 기본값: false                                                                   |
| `limit`           | number  | 아니오 | 기본값: 50                                                                      |
| `after`           | string  | 아니오 | 커서                                                                            |

**반환값:** `{ projects: Project[], pageInfo }`

---

### `get_project`

이름 또는 ID로 프로젝트를 조회합니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `query`   | string | 예   |

**반환값:** `milestones[]`와 상태 카테고리별 이슈 수를 포함하는 단일 `Project` 객체.

---

### `create_project`

| 매개변수      | 타입   | 필수   |
| ------------- | ------ | ------ |
| `name`        | string | 예     |
| `description` | string | 아니오 |
| `summary`     | string | 아니오 |
| `leadId`      | string | 아니오 |
| `startDate`   | string | 아니오 |
| `targetDate`  | string | 아니오 |

**반환값:** 생성된 `Project` 객체. 상태 기본값은 `backlog`.

---

### `update_project`

| 매개변수      | 타입   | 필수   |
| ------------- | ------ | ------ |
| `id`          | string | 예     |
| `name`        | string | 아니오 |
| `description` | string | 아니오 |
| `summary`     | string | 아니오 |
| `status`      | string | 아니오 |
| `leadId`      | string | 아니오 |
| `startDate`   | string | 아니오 |
| `targetDate`  | string | 아니오 |

**반환값:** 업데이트된 `Project` 객체.

---

### `archive_project`

프로젝트를 소프트 보관합니다. `archivedAt`을 설정합니다. 삭제하지 않습니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `id`      | string | 예   |

**반환값:** `{ success: true }`

---

## 마일스톤

### `list_milestones`

| 매개변수    | 타입   | 필수 |
| ----------- | ------ | ---- |
| `projectId` | string | 예   |

**반환값:** `{ milestones: Milestone[] }` -- `sortOrder`별 정렬.

---

### `get_milestone`

ID로 마일스톤을 조회합니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `id`      | string | 예   |

**반환값:** 상태 카테고리별 이슈 수를 포함하는 단일 `Milestone` 객체.

---

### `create_milestone`

| 매개변수      | 타입   | 필수   |
| ------------- | ------ | ------ |
| `projectId`   | string | 예     |
| `name`        | string | 예     |
| `description` | string | 아니오 |
| `targetDate`  | string | 아니오 |
| `sortOrder`   | number | 아니오 | 프로젝트 내 정렬 순서 |

**반환값:** 생성된 `Milestone` 객체.

---

### `update_milestone`

| 매개변수      | 타입   | 필수   |
| ------------- | ------ | ------ |
| `id`          | string | 예     |
| `name`        | string | 아니오 |
| `description` | string | 아니오 |
| `targetDate`  | string | 아니오 |
| `sortOrder`   | number | 아니오 | 프로젝트 내 정렬 순서 |

**반환값:** 업데이트된 `Milestone` 객체.

---

## 레이블

### `list_labels`

팀에서 사용 가능한 레이블을 목록 조회합니다(워크스페이스 수준 레이블 포함).

| 매개변수  | 타입   | 필수   | 참고                                      |
| --------- | ------ | ------ | ----------------------------------------- |
| `teamId`  | string | 아니오 | 생략하면 워크스페이스 레이블만 반환        |

**반환값:** `{ labels: Label[] }` -- 레이블 그룹별로 묶이고, 그룹에 속하지 않는 레이블은 별도로 표시.

---

### `get_label`

이름 또는 ID로 레이블을 조회합니다.

| 매개변수  | 타입   | 필수 | 참고                |
| --------- | ------ | ---- | ------------------ |
| `query`   | string | 예   | 레이블 이름 또는 UUID |

**반환값:** 단일 `Label` 객체.

---

### `create_label`

| 매개변수      | 타입   | 필수   | 참고                                |
| ------------- | ------ | ------ | ----------------------------------- |
| `name`        | string | 예     |                                     |
| `color`       | string | 아니오 | Hex 색상. 생략 시 자동 할당         |
| `description` | string | 아니오 |                                     |
| `teamId`      | string | 아니오 | 워크스페이스 수준 레이블은 생략     |
| `groupId`     | string | 아니오 | 상위 레이블 그룹                    |

**반환값:** 생성된 `Label` 객체.

---

### `update_label`

| 매개변수      | 타입   | 필수   |
| ------------- | ------ | ------ |
| `id`          | string | 예     |
| `name`        | string | 아니오 |
| `color`       | string | 아니오 |
| `description` | string | 아니오 |

**반환값:** 업데이트된 `Label` 객체.

---

## 이슈 관계

### `list_issue_relations`

이슈의 모든 관계를 목록 조회합니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `issueId` | string | 예   |

**반환값:** `{ relations: IssueRelation[] }` -- 각각 확장된 `relatedIssue` 요약(id, identifier, title, state)을 포함.

---

### `create_issue_relation`

두 이슈 간의 관계를 생성합니다.

| 매개변수         | 타입   | 필수 | 참고                                           |
| ---------------- | ------ | ---- | ---------------------------------------------- |
| `issueId`        | string | 예   | 원본 이슈                                      |
| `relatedIssueId` | string | 예   | 대상 이슈                                      |
| `type`           | string | 예   | `related`, `blocks`, `blocked_by`, `duplicate` |

**반환값:** 생성된 `IssueRelation` 객체.

**부수 효과:**

- `duplicate`는 원본 이슈를 취소 상태로 자동 전환
- A->B에 대해 `blocks`를 생성하면 B가 A에 `blocked_by`임을 암시적으로 의미 (양쪽 이슈 조회 시 양방향 모두 표시)

---

### `delete_issue_relation`

두 이슈 간의 관계를 제거합니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `id`      | string | 예   |

**반환값:** `{ success: true }`

---

## 댓글

### `list_comments`

이슈의 댓글을 목록 조회합니다.

| 매개변수  | 타입   | 필수   | 참고        |
| --------- | ------ | ------ | ----------- |
| `issueId` | string | 예     |             |
| `limit`   | number | 아니오 | 기본값: 50  |

**반환값:** `{ comments: Comment[] }` -- 스레드 형식 (최상위 댓글에 중첩된 `children` 포함).

---

### `create_comment`

이슈에 댓글을 추가합니다.

| 매개변수   | 타입   | 필수   | 참고                                  |
| ---------- | ------ | ------ | ------------------------------------- |
| `issueId`  | string | 예     |                                       |
| `body`     | string | 예     | Markdown                              |
| `parentId` | string | 아니오 | 기존 댓글에 답글 (스레드)             |

**반환값:** 생성된 `Comment` 객체.

---

### `update_comment`

댓글 본문을 업데이트합니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `id`      | string | 예   |
| `body`    | string | 예   |

**반환값:** 업데이트된 `Comment` 객체.

---

### `resolve_comment`

댓글 스레드를 해결됨으로 표시합니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `id`      | string | 예   |

**반환값:** `resolvedAt`이 설정된 업데이트된 `Comment`.

---

## 이니셔티브

### `list_initiatives`

| 매개변수  | 타입   | 필수   | 참고                             |
| --------- | ------ | ------ | -------------------------------- |
| `status`  | string | 아니오 | `planned`, `active`, `completed` |
| `limit`   | number | 아니오 | 기본값: 50                       |

**반환값:** `{ initiatives: Initiative[] }`

---

### `get_initiative`

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `query`   | string | 예   |

**반환값:** 확장된 `projects[]`(상태 및 이슈 수를 포함하는 요약)를 포함하는 단일 `Initiative` 객체.

---

### `create_initiative`

| 매개변수      | 타입     | 필수   |
| ------------- | -------- | ------ |
| `name`        | string   | 예     |
| `description` | string   | 아니오 |
| `ownerId`     | string   | 아니오 |
| `targetDate`  | string   | 아니오 |
| `projectIds`  | string[] | 아니오 |

**반환값:** 생성된 `Initiative` 객체. 상태 기본값은 `planned`.

---

### `update_initiative`

| 매개변수      | 타입     | 필수   |
| ------------- | -------- | ------ |
| `id`          | string   | 예     |
| `name`        | string   | 아니오 |
| `description` | string   | 아니오 |
| `status`      | string   | 아니오 |
| `ownerId`     | string   | 아니오 |
| `targetDate`  | string   | 아니오 |
| `projectIds`  | string[] | 아니오 |

**반환값:** 업데이트된 `Initiative` 객체.

---

### `archive_initiative`

이니셔티브를 소프트 보관합니다. `archivedAt`을 설정합니다. 삭제하지 않습니다.

| 매개변수  | 타입   | 필수 |
| --------- | ------ | ---- |
| `id`      | string | 예   |

**반환값:** `{ success: true }`

---

## 요약

| 엔티티        | list | get | create | update | delete/archive |
| ------------- | ---- | --- | ------ | ------ | -------------- |
| Issue         | x    | x   | x      | x      | archive        |
| WorkflowState | x    | x   | --     | --     | --             |
| Team          | x    | x   | --     | --     | --             |
| Project       | x    | x   | x      | x      | archive        |
| Milestone     | x    | x   | x      | x      | --             |
| Label         | x    | x   | x      | x      | --             |
| IssueRelation | x    | --  | x      | --     | x              |
| Comment       | x    | --  | x      | x      | resolve        |
| Initiative    | x    | x   | x      | x      | archive        |

**총 35개 작업**

워크플로 상태와 팀은 관리자가 설정하며, MCP를 통해 생성되지 않습니다.
MCP는 주로 에이전트가 작업을 관리하기 위한 것입니다: 이슈 생성, 상태 업데이트,
관계와 댓글을 통한 조율, 프로젝트 컨텍스트 파악.
