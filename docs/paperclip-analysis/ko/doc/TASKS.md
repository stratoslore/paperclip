# 작업 관리 데이터 모델

Paperclip에서 작업 추적이 어떻게 동작하는지에 대한 참고 자료입니다. 엔티티, 관계,
작업 생명주기를 관리하는 규칙을 설명합니다. 목표 모델로 작성되었으며
-- 일부는 이미 구현되었고, 일부는 향후 계획입니다.

---

## 엔티티 계층 구조

```
Workspace
  Initiatives          (로드맵 수준의 목표, 분기 단위로 진행)
    Projects           (시간 한정 산출물, 여러 팀에 걸칠 수 있음)
      Milestones       (프로젝트 내 단계)
        Issues         (작업 단위, 핵심 엔티티)
          Sub-issues   (상위 이슈 하위로 분해된 작업)
```

모든 것은 아래로 흐릅니다. 이니셔티브는 프로젝트를 포함하고, 프로젝트는
마일스톤과 이슈를 포함하며, 이슈는 하위 이슈를 가질 수 있습니다. 각 수준은
세분성을 추가합니다.

---

## 이슈 (핵심 엔티티)

이슈는 작업의 기본 단위입니다.

### 필드

| 필드          | 타입             | 필수     | 참고                                                              |
| ------------- | ---------------- | -------- | ----------------------------------------------------------------- |
| `id`          | uuid             | 예       | 기본 키                                                           |
| `identifier`  | string           | 계산됨   | 사람이 읽을 수 있는 형태, 예: `ENG-123` (팀 키 + 자동 증가 번호)  |
| `title`       | string           | 예       | 짧은 요약                                                         |
| `description` | text/markdown    | 아니오   | 전체 설명, markdown 지원                                          |
| `status`      | WorkflowState FK | 예       | 팀의 기본 상태로 기본 설정                                        |
| `priority`    | enum (0-4)       | 아니오   | 기본값 0 (없음). 우선순위 섹션 참조.                              |
| `estimate`    | number           | 아니오   | 복잡도/크기 포인트                                                |
| `dueDate`     | date             | 아니오   |                                                                   |
| `teamId`      | uuid FK          | 예       | 모든 이슈는 정확히 하나의 팀에 속함                               |
| `projectId`   | uuid FK          | 아니오   | 이슈당 최대 하나의 프로젝트                                       |
| `milestoneId` | uuid FK          | 아니오   | 이슈당 최대 하나의 마일스톤                                       |
| `assigneeId`  | uuid FK          | 아니오   | **단일 담당자.** 담당자 섹션 참조.                                |
| `creatorId`   | uuid FK          | 아니오   | 생성자                                                            |
| `parentId`    | uuid FK (self)   | 아니오   | 상위 이슈, 하위 이슈 관계용                                      |
| `goalId`      | uuid FK          | 아니오   | 연결된 목표/목적                                                  |
| `sortOrder`   | float            | 아니오   | 뷰 내 정렬 순서                                                   |
| `createdAt`   | timestamp        | 예       |                                                                   |
| `updatedAt`   | timestamp        | 예       |                                                                   |
| `startedAt`   | timestamp        | 계산됨   | 이슈가 "started" 상태에 진입한 시점                               |
| `completedAt` | timestamp        | 계산됨   | 이슈가 "completed" 상태에 진입한 시점                             |
| `cancelledAt` | timestamp        | 계산됨   | 이슈가 "cancelled" 상태에 진입한 시점                             |
| `archivedAt`  | timestamp        | 아니오   | 소프트 보관                                                       |

---

## 워크플로 상태

이슈 상태는 단순한 enum이 **아닙니다**. 팀별 명명된 상태의 집합이며,
각각 다음 고정된 **카테고리** 중 하나에 속합니다:

| 카테고리      | 용도                         | 예시 상태                       |
| ------------- | ---------------------------- | ------------------------------- |
| **Triage**    | 접수됨, 검토 필요            | Triage                          |
| **Backlog**   | 수용됨, 작업 준비 안 됨      | Backlog, Icebox                 |
| **Unstarted** | 준비됨, 아직 시작 안 함      | Todo, Ready                     |
| **Started**   | 작업 중                      | In Progress, In Review, In QA   |
| **Completed** | 완료                         | Done, Shipped                   |
| **Cancelled** | 거절 또는 폐기               | Cancelled, Won't Fix, Duplicate |

### 규칙

- 각 팀은 이러한 카테고리 내에서 자체 워크플로 상태를 정의
- 팀은 카테고리당 최소 하나의 상태를 가져야 함 (Triage는 선택 사항)
- 모든 카테고리 내에 사용자 정의 상태를 추가할 수 있음 (예: Started 내의 "In Review")
- 카테고리는 고정되어 있고 순서가 정해져 있음 -- 카테고리 _내에서_ 상태를 재정렬할 수 있지만
  카테고리 자체는 불가
- 새 이슈는 팀의 첫 번째 Backlog 상태가 기본값
- Started 상태로 이동하면 `startedAt`이 자동 설정; Completed는 `completedAt` 설정;
  Cancelled는 `cancelledAt` 설정
- 이슈를 중복으로 표시하면 자동으로 Cancelled 상태로 이동

### WorkflowState 필드

| 필드          | 타입    | 참고                                                                          |
| ------------- | ------- | ----------------------------------------------------------------------------- |
| `id`          | uuid    |                                                                               |
| `name`        | string  | 표시 이름, 예: "In Review"                                                    |
| `type`        | enum    | `triage`, `backlog`, `unstarted`, `started`, `completed`, `cancelled` 중 하나 |
| `color`       | string  | Hex 색상                                                                      |
| `description` | string  | 선택적 안내 텍스트                                                            |
| `position`    | float   | 카테고리 내 정렬 순서                                                         |
| `teamId`      | uuid FK | 각 상태는 하나의 팀에 속함                                                    |

---

## 우선순위

고정되어 있으며 사용자 정의가 불가능한 숫자 척도:

| 값    | 레이블      | 참고                                   |
| ----- | ----------- | -------------------------------------- |
| 0     | No priority | 기본값. 우선순위 뷰에서 마지막으로 정렬. |
| 1     | Urgent      | 즉시 알림을 트리거할 수 있음           |
| 2     | High        |                                        |
| 3     | Medium      |                                        |
| 4     | Low         |                                        |

척도는 의도적으로 작고 고정되어 있습니다. 더 많은 우선순위 수준을 추가하는 대신
추가 분류에는 레이블을 사용하세요.

---

## 팀

팀은 주요 조직 단위입니다. 거의 모든 것이 팀에 범위가 지정됩니다.

| 필드          | 타입   | 참고                                                           |
| ------------- | ------ | -------------------------------------------------------------- |
| `id`          | uuid   |                                                                |
| `name`        | string | 예: "Engineering"                                              |
| `key`         | string | 짧은 대문자 접두사, 예: "ENG". 이슈 식별자에 사용.             |
| `description` | string |                                                                |

### 팀 범위 지정

- 각 이슈는 정확히 하나의 팀에 속함
- 워크플로 상태는 팀별
- 레이블은 팀 범위 또는 워크스페이스 전체일 수 있음
- 프로젝트는 여러 팀에 걸칠 수 있음

우리의 맥락(AI 회사)에서 팀은 기능 영역에 매핑됩니다. 각 에이전트는
역할에 따라 팀에 소속됩니다.

---

## 프로젝트

프로젝트는 특정 시간 한정 산출물을 향해 이슈를 그룹화합니다. 여러 팀에
걸칠 수 있습니다.

| 필드          | 타입      | 참고                                                          |
| ------------- | --------- | ------------------------------------------------------------- |
| `id`          | uuid      |                                                               |
| `name`        | string    |                                                               |
| `description` | text      |                                                               |
| `summary`     | string    | 짧은 설명                                                     |
| `status`      | enum      | `backlog`, `planned`, `in_progress`, `completed`, `cancelled` |
| `leadId`      | uuid FK   | 책임을 위한 단일 소유자                                       |
| `startDate`   | date      |                                                               |
| `targetDate`  | date      |                                                               |
| `createdAt`   | timestamp |                                                               |
| `updatedAt`   | timestamp |                                                               |

### 규칙

- 이슈는 최대 하나의 프로젝트에 속함
- 프로젝트 상태는 **수동으로** 업데이트됨 (이슈 상태에서 자동 파생되지 않음)
- 프로젝트는 연결된 엔티티로 문서(스펙, 브리프)를 포함할 수 있음

---

## 마일스톤

마일스톤은 프로젝트를 의미 있는 단계로 세분화합니다.

| 필드          | 타입    | 참고                           |
| ------------- | ------- | ------------------------------ |
| `id`          | uuid    |                                |
| `name`        | string  |                                |
| `description` | text    |                                |
| `targetDate`  | date    |                                |
| `projectId`   | uuid FK | 정확히 하나의 프로젝트에 속함  |
| `sortOrder`   | float   |                                |

프로젝트 내 이슈는 선택적으로 마일스톤에 할당할 수 있습니다.

---

## 레이블 / 태그

레이블은 카테고리별 태깅을 제공합니다. 두 가지 범위에 존재합니다:

- **워크스페이스 레이블** -- 모든 팀에서 사용 가능
- **팀 레이블** -- 특정 팀으로 제한

| 필드          | 타입           | 참고                            |
| ------------- | -------------- | ------------------------------- |
| `id`          | uuid           |                                 |
| `name`        | string         |                                 |
| `color`       | string         | Hex 색상                        |
| `description` | string         | 상황별 안내                     |
| `teamId`      | uuid FK        | 워크스페이스 수준 레이블은 Null |
| `groupId`     | uuid FK (self) | 그룹화를 위한 상위 레이블       |

### 레이블 그룹

레이블은 한 단계의 중첩으로 구성할 수 있습니다 (그룹 -> 레이블):

- 그룹 내 레이블은 이슈에서 **상호 배타적** (각 그룹에서 하나만
  적용 가능)
- 그룹은 다른 그룹을 포함할 수 없음 (단일 중첩 수준만)
- 예: "Type" 그룹에 "Bug", "Feature", "Chore" 레이블 포함 -- 이슈에는
  최대 하나만 적용

### 이슈-레이블 결합

`issue_labels` 조인 테이블을 통한 다대다 관계:

| 필드      | 타입    |
| --------- | ------- |
| `issueId` | uuid FK |
| `labelId` | uuid FK |

---

## 이슈 관계 / 종속성

이슈 간 네 가지 관계 유형:

| 타입         | 의미                         | 동작                                          |
| ------------ | ---------------------------- | --------------------------------------------- |
| `related`    | 일반적인 연결                | 참조용 링크                                   |
| `blocks`     | 이 이슈가 다른 이슈를 차단   | 차단된 이슈에 플래그 표시                     |
| `blocked_by` | 이 이슈가 다른 이슈에 의해 차단 | blocks의 역방향                             |
| `duplicate`  | 이 이슈가 다른 이슈와 중복   | 중복 이슈를 Cancelled 상태로 자동 이동        |

### IssueRelation 필드

| 필드             | 타입    | 참고                                           |
| ---------------- | ------- | ---------------------------------------------- |
| `id`             | uuid    |                                                |
| `type`           | enum    | `related`, `blocks`, `blocked_by`, `duplicate` |
| `issueId`        | uuid FK | 원본 이슈                                      |
| `relatedIssueId` | uuid FK | 대상 이슈                                      |

### 규칙

- 차단 이슈가 해결되면 관계가 참조용으로 변경됨 (플래그가 녹색으로 전환)
- 중복은 단방향 (중복을 표시하는 것이지, 원본을 표시하는 것이 아님)
- 차단은 시스템 수준에서 **이행적이지 않음** (A가 B를 차단하고, B가 C를 차단해도
  A->C를 자동 차단하지 않음)

---

## 담당자

설계상 **단일 담당자 모델**입니다.

- 각 이슈는 한 번에 최대 한 명의 담당자를 가짐
- 이는 의도적: 명확한 소유권이 책임의 분산을 방지
- 여러 사람이 관여하는 협업 작업에는 서로 다른 담당자를 가진
  **하위 이슈**를 사용

우리의 맥락에서 에이전트가 담당자입니다. 이슈의 `assigneeId` FK는
`agents` 테이블을 가리킵니다.

---

## 하위 이슈 (부모/자식)

이슈는 부모/자식 중첩을 지원합니다.

- 이슈에 `parentId`를 설정하면 하위 이슈가 됨
- 하위 이슈도 자체 하위 이슈를 가질 수 있음 (다단계 중첩)
- 하위 이슈는 생성 시 상위 이슈의 **프로젝트**를 상속
  (소급 적용되지 않음), 하지만 팀, 레이블, 담당자는 상속하지 않음

### 자동 닫기

- **하위 이슈 자동 닫기**: 상위 이슈가 완료되면 나머지 하위 이슈가
  자동 완료

### 변환

- 기존 이슈를 재배치할 수 있음 (`parentId` 추가 또는 제거)
- 많은 하위 이슈를 가진 상위 이슈를 프로젝트로 "승격"할 수 있음

---

## 추정치

팀별로 구성되는 포인트 기반 추정.

### 사용 가능한 척도

| 척도       | 값                       |
| ----------- | ------------------------ |
| Exponential | 1, 2, 4, 8, 16 (+32, 64) |

추정되지 않은 이슈는 진행률/속도 계산에서 기본값 1 포인트로 간주됩니다.

---

## 댓글

| 필드         | 타입           | 참고                       |
| ------------ | -------------- | -------------------------- |
| `id`         | uuid           |                            |
| `body`       | text/markdown  |                            |
| `issueId`    | uuid FK        |                            |
| `authorId`   | uuid FK        | 사용자 또는 에이전트일 수 있음 |
| `parentId`   | uuid FK (self) | 스레드 답글용              |
| `resolvedAt` | timestamp      | 스레드가 해결된 경우       |
| `createdAt`  | timestamp      |                            |
| `updatedAt`  | timestamp      |                            |

---

## 이니셔티브

최상위 계획 구조. 전략적 목표를 향해 프로젝트를 그룹화합니다.
이니셔티브는 전략적 소유자를 가지며, 일반적으로 "완료/미완료"가 아닌 성과/OKR로 측정됩니다.

| 필드          | 타입    | 참고                             |
| ------------- | ------- | -------------------------------- |
| `id`          | uuid    |                                  |
| `name`        | string  |                                  |
| `description` | text    |                                  |
| `ownerId`     | uuid FK | 단일 소유자                      |
| `status`      | enum    | `planned`, `active`, `completed` |
| `targetDate`  | date    |                                  |

이니셔티브는 프로젝트를 포함하고(다대다) 포함된 모든 프로젝트에 대한
진행률 롤업 뷰를 제공합니다.

---

## 식별자

이슈는 사람이 읽을 수 있는 식별자를 사용합니다: `{TEAM_KEY}-{NUMBER}`

- 팀 키: 팀별로 설정되는 짧은 대문자 문자열 (예: "ENG", "DES")
- 번호: 팀별 자동 증가 정수
- 예시: `ENG-123`, `DES-45`, `OPS-7`
- 이슈가 팀 간에 이동하면 새 식별자를 받고 기존 식별자는
  `previousIdentifiers`에 보존

이는 UUID보다 사람 간 소통에 훨씬 유용합니다. 사람들은 "grab 7f3a..."가 아닌
"grab ENG-42"라고 말합니다.

---

## 엔티티 관계

```
Team (1) ----< (many) Issue
Team (1) ----< (many) WorkflowState
Team (1) ----< (many) Label (team-scoped)

Issue (many) >---- (1) WorkflowState
Issue (many) >---- (0..1) Assignee (Agent)
Issue (many) >---- (0..1) Project
Issue (many) >---- (0..1) Milestone
Issue (many) >---- (0..1) Parent Issue
Issue (1) ----< (many) Sub-issues
Issue (many) >---< (many) Labels         (via issue_labels)
Issue (many) >---< (many) Issue Relations (via issue_relations)
Issue (1) ----< (many) Comments

Project (many) >---- (0..1) Lead (Agent)
Project (1) ----< (many) Milestones
Project (1) ----< (many) Issues

Initiative (many) >---< (many) Projects  (via initiative_projects)
Initiative (many) >---- (1) Owner (Agent)
```

---

## 구현 우선순위

권장 구축 순서, 가치가 높은 순:

### 높은 가치

1. **팀** -- `teams` 테이블 + 이슈의 `teamId` FK. 사람이 읽을 수 있는
   식별자(`ENG-123`)와 팀별 워크플로 상태의 기반. 대부분의 다른 기능이
   팀 범위 지정에 의존하므로 이것을 먼저 구축.
2. **워크플로 상태** -- `workflow_states` 테이블 + 이슈의 `stateId` FK.
   카테고리 기반 상태 전환이 있는 팀별 사용자 정의 워크플로.
3. **레이블** -- `labels` + `issue_labels` 테이블. 상태 필드를 오염시키지 않는
   분류(bug/feature/chore, 영역 태그 등).
4. **이슈 관계** -- `issue_relations` 테이블. 에이전트 조율에 차단/차단됨은
   필수(에이전트 A는 에이전트 B가 완료할 때까지 시작 불가).
5. **하위 이슈** -- `issues`의 `parentId` self-FK. 에이전트가 대규모 작업을
   분해할 수 있게 함.
6. **댓글** -- `comments` 테이블. 에이전트는 설명을 덮어쓰지 않고
   이슈에 대해 소통해야 함.

### 중간 가치

7. **전환 타임스탬프** -- 이슈의 `startedAt`, `completedAt`, `cancelledAt`,
   워크플로 상태 변경 시 자동 설정. 속도 추적 및 SLA 측정 가능.

### 낮은 우선순위 (나중에)

8. **마일스톤** -- 프로젝트가 단계가 필요할 정도로 복잡해지면 유용.
9. **이니셔티브** -- 공통 전략 목표를 가진 여러 프로젝트가 생기면 유용.
10. **추정치** -- 처리량을 측정하고 용량을 예측하고 싶을 때 유용.
