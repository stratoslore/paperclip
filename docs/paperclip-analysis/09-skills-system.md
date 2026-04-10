# 09. 스킬 시스템 및 에이전트 API

> 소스: `skills/paperclip/SKILL.md`, `skills/`

## 스킬 시스템 개요

스킬은 에이전트에게 런타임에 주입되는 **지시사항 + API 레퍼런스**다. 에이전트가 Paperclip과 상호작용하는 방법을 정의한다.

### 제공 스킬 목록

| 스킬 | 경로 | 설명 |
|------|------|------|
| Paperclip Core | `skills/paperclip/SKILL.md` | 핵심 하트비트 프로시저, API 레퍼런스 |
| Create Agent | `skills/paperclip-create-agent/` | 에이전트가 새 에이전트를 생성하는 스킬 |
| Create Plugin | `skills/paperclip-create-plugin/` | 플러그인 생성 스킬 |
| PARA Memory | `skills/para-memory-files/` | 지식/메모리 파일 관리 스킬 |

## Paperclip Core Skill (SKILL.md) 상세

에이전트가 하트비트 내에서 따라야 할 9단계 프로시저.

### 인증

에이전트 실행 시 자동 주입되는 환경변수로 인증한다:

```bash
# 자동 주입
PAPERCLIP_AGENT_ID      # 에이전트 ID
PAPERCLIP_COMPANY_ID    # 회사 ID
PAPERCLIP_API_URL       # API URL (예: http://localhost:3100)
PAPERCLIP_RUN_ID        # 현재 실행 ID
PAPERCLIP_API_KEY       # 단기 JWT 토큰

# 조건부 주입
PAPERCLIP_TASK_ID             # 이슈 트리거 시
PAPERCLIP_WAKE_REASON         # 깨어난 이유
PAPERCLIP_WAKE_COMMENT_ID     # 멘션 코멘트 ID
PAPERCLIP_APPROVAL_ID         # 승인 트리거 시
PAPERCLIP_APPROVAL_STATUS     # 승인 상태
```

### API 호출 규칙

```bash
# 모든 요청에 Authorization 헤더 포함
Authorization: Bearer $PAPERCLIP_API_KEY

# 뮤테이션 요청에 Run-Id 헤더 포함
X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
```

### 9단계 하트비트 프로시저

```
Step 1: 신원 확인
  GET /api/agents/me
  → 나의 역할, 설정, 보고 라인 확인

Step 2: 승인 후속 조치 (승인 트리거 시)
  GET /api/approvals/{PAPERCLIP_APPROVAL_ID}/issues
  → 승인된 작업 확인 및 처리

Step 3: 할당된 작업 확인
  GET /api/agents/me/inbox-lite
  → 나에게 할당된 이슈 목록 조회

Step 4: 작업 선택
  우선순위: in_progress > todo
  → 진행 중인 작업 먼저, 없으면 새 작업 선택

Step 5: 이슈 체크아웃
  POST /api/issues/{issueId}/checkout
  → 작업 잠금 (다른 에이전트 접근 차단)

Step 6: 컨텍스트 이해
  GET /api/issues/{issueId}/heartbeat-context
  → 이슈 상세, 프로젝트 정보, 목표 계층, 코멘트 히스토리

Step 7: 작업 수행
  → 도구와 기능을 사용하여 실제 작업 수행

Step 8: 상태 업데이트
  PATCH /api/issues/{issueId}
  → 상태 변경 + 진행 상황 코멘트 추가

Step 9: 위임 (필요 시)
  POST /api/companies/{companyId}/issues
  → 서브태스크 생성 및 다른 에이전트에게 할당
```

## 핵심 API 엔드포인트

### 에이전트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/agents/me` | 내 정보 조회 |
| GET | `/api/agents/me/inbox-lite` | 내 인박스 (할당된 이슈) |

### 이슈

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/issues/` | 이슈 목록 (필터 지원) |
| GET | `/api/issues/{id}` | 이슈 상세 |
| POST | `/api/companies/{companyId}/issues` | 이슈 생성 |
| PATCH | `/api/issues/{id}` | 이슈 수정 |
| POST | `/api/issues/{id}/checkout` | 이슈 체크아웃 |
| GET | `/api/issues/{id}/heartbeat-context` | 하트비트 컨텍스트 |
| GET | `/api/issues/{id}/comments` | 코멘트 목록 |
| POST | `/api/issues/{id}/comments` | 코멘트 추가 |

### 승인

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/approvals/{id}/issues` | 승인 관련 이슈 |

### 루틴

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/routines/` | 루틴 목록 |
| POST | `/api/routines/` | 루틴 생성 |
| PATCH | `/api/routines/{id}` | 루틴 수정 |

## 멘션 기반 트리거

에이전트가 이슈 코멘트에서 멘션(`@agent-name`)되면 깨어난다.

### 멘션 처리 규칙

```
PAPERCLIP_WAKE_COMMENT_ID가 설정된 경우:
  1. 해당 코멘트를 먼저 읽는다
  2. 코멘트가 소유권 이전을 요청하면 → 셀프 할당 (체크아웃)
  3. 코멘트가 의견만 요청하면 → 응답하되 할당하지 않음
  4. 코멘트가 소유권을 지시하지 않으면 → 셀프 할당하지 않음
```

## 차단된 태스크 중복 방지

```
차단된 태스크 작업 전:
  1. 코멘트 스레드 확인
  2. 내 마지막 코멘트가 차단 업데이트였고 새 코멘트 없음 → 건너뛰기
  3. 새 컨텍스트가 있을 때만 다시 참여
```

## 이슈 생성 시 코멘트 형식

에이전트가 상태를 업데이트할 때 권장되는 코멘트 형식:

```markdown
## Progress Update

**Status:** in_progress → in_review
**Work Done:**
- Implemented feature X
- Added tests for Y
- Updated documentation

**Next Steps:**
- Code review needed
- Deploy to staging
```

## 스킬 주입 메커니즘

### Claude Code 어댑터

SKILL.md 내용이 에이전트의 시스템 프롬프트에 주입된다.

### Codex 어댑터

`$CODEX_HOME/skills/` 디렉토리에 스킬 파일이 자동 복사된다.

### 커스텀 스킬 추가

`skills/` 디렉토리에 새 스킬 폴더를 생성하고, 회사별로 `company-skills` 서비스를 통해 설치/관리한다.
