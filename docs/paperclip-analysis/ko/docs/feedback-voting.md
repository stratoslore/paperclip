# 피드백 투표 - 로컬 데이터 가이드

에이전트의 응답에 **도움됨** (엄지 위) 또는 **개선 필요** (엄지 아래)로 평가하면, Paperclip은 실행 중인 인스턴스와 함께 투표를 로컬에 저장합니다. 이 가이드는 무엇이 저장되는지, 어떻게 접근하는지, 어떻게 내보내는지를 다룹니다.

## 투표 작동 방식

1. 에이전트 댓글 또는 문서 수정에서 **도움됨** 또는 **개선 필요**를 클릭합니다.
2. **개선 필요**를 클릭하면 선택적 텍스트 프롬프트가 나타납니다: _"무엇이 더 나았을 수 있나요?"_ 이유를 입력하거나 닫을 수 있습니다.
3. 동의 대화상자가 투표를 로컬에 보관할지 공유할지 묻습니다. 선택은 향후 투표를 위해 기억됩니다.

### 저장되는 내용

각 투표는 두 개의 로컬 레코드를 생성합니다:

| 레코드 | 포함 내용 |
|--------|-----------------|
| **투표** | 투표(위/아래), 선택적 이유 텍스트, 공유 선호도, 동의 버전, 타임스탬프 |
| **추적 번들** | 전체 컨텍스트 스냅샷: 투표된 댓글/수정 텍스트, 이슈 제목, 에이전트 정보, 투표 및 이유 — 피드백을 단독으로 이해하는 데 필요한 모든 것 |

모든 데이터는 로컬 Paperclip 데이터베이스에 있습니다. 명시적으로 공유를 선택하지 않으면 머신을 떠나지 않습니다.

투표가 공유로 표시되면, Paperclip은 텔레메트리 백엔드를 통해 추적 번들을 즉시 업로드하려고 시도합니다. 업로드는 전송 중 압축되어 전체 추적 번들이 게이트웨이 크기 제한 이하로 유지됩니다. 즉시 푸시가 실패하면, 추적은 나중에 플러시를 시도할 수 있는 재시도 가능한 실패 상태로 남습니다. 앱 서버는 원시 피드백 추적 번들을 객체 저장소에 직접 업로드하지 않습니다.

## 투표 보기

### 빠른 보고서 (터미널)

```bash
pnpm paperclipai feedback report
```

색상으로 구분된 요약을 표시합니다: 투표 수, 이유가 포함된 추적별 세부 정보 및 내보내기 상태.

```bash
# 설치된 CLI
paperclipai feedback report

# 다른 서버나 회사를 지정
pnpm paperclipai feedback report --api-base http://127.0.0.1:3000 --company-id <company-id>

# 보고서에 원시 페이로드 덤프 포함
pnpm paperclipai feedback report --payloads
```

### API 엔드포인트

모든 엔드포인트는 보드 사용자 접근 권한이 필요합니다 (로컬 개발에서는 자동).

**이슈의 투표 목록:**
```bash
curl http://127.0.0.1:3102/api/issues/<issueId>/feedback-votes
```

**이슈의 추적 번들 목록 (전체 페이로드 포함):**
```bash
curl 'http://127.0.0.1:3102/api/issues/<issueId>/feedback-traces?includePayload=true'
```

**회사 전체 추적 목록:**
```bash
curl 'http://127.0.0.1:3102/api/companies/<companyId>/feedback-traces?includePayload=true'
```

**단일 추적 엔벨로프 레코드 가져오기:**
```bash
curl http://127.0.0.1:3102/api/feedback-traces/<traceId>
```

**추적의 전체 내보내기 번들 가져오기:**
```bash
curl http://127.0.0.1:3102/api/feedback-traces/<traceId>/bundle
```

#### 필터링

추적 엔드포인트는 쿼리 매개변수를 허용합니다:

| 매개변수 | 값 | 설명 |
|-----------|--------|-------------|
| `vote` | `up`, `down` | 투표 방향으로 필터링 |
| `status` | `local_only`, `pending`, `sent`, `failed` | 내보내기 상태로 필터링 |
| `targetType` | `issue_comment`, `issue_document_revision` | 투표 대상으로 필터링 |
| `sharedOnly` | `true` | 사용자가 공유를 선택한 투표만 표시 |
| `includePayload` | `true` | 전체 컨텍스트 스냅샷 포함 |
| `from` / `to` | ISO 날짜 | 날짜 범위 필터 |

## 데이터 내보내기

### 파일 + zip으로 내보내기

```bash
pnpm paperclipai feedback export
```

타임스탬프가 있는 디렉토리를 생성합니다:

```
feedback-export-20260331T120000Z/
  index.json                    # 요약 통계가 포함된 매니페스트
  votes/
    PAP-123-a1b2c3d4.json      # 투표 메타데이터 (투표당 하나)
  traces/
    PAP-123-e5f6g7h8.json      # Paperclip 피드백 엔벨로프 (추적당 하나)
  full-traces/
    PAP-123-e5f6g7h8/
      bundle.json              # 추적의 전체 내보내기 매니페스트
      ...원시 어댑터 파일     # 사용 가능한 경우 codex / claude / opencode 세션 아티팩트
feedback-export-20260331T120000Z.zip
```

내보내기는 기본적으로 전체입니다. `traces/`는 Paperclip 엔벨로프를 유지하고, `full-traces/`는 더 풍부한 추적별 번들과 복구 가능한 어댑터 네이티브 파일을 포함합니다.

```bash
# 커스텀 서버 및 출력 디렉토리
pnpm paperclipai feedback export --api-base http://127.0.0.1:3000 --company-id <company-id> --out ./my-export
```

### 내보낸 추적 읽기

`traces/`의 파일을 열면 다음을 볼 수 있습니다:

```json
{
  "id": "trace-uuid",
  "vote": "down",
  "issueIdentifier": "PAP-123",
  "issueTitle": "Fix login timeout",
  "targetType": "issue_comment",
  "targetSummary": {
    "label": "Comment",
    "excerpt": "The first 80 chars of the comment that was voted on..."
  },
  "payloadSnapshot": {
    "vote": {
      "value": "down",
      "reason": "Did not address the root cause"
    },
    "target": {
      "body": "Full text of the agent comment..."
    },
    "issue": {
      "identifier": "PAP-123",
      "title": "Fix login timeout"
    }
  }
}
```

`full-traces/<issue>-<trace>/bundle.json`을 열면 캡처 노트, 어댑터 유형, 무결성 메타데이터 및 함께 작성된 원시 파일 목록을 포함한 확장된 내보내기 메타데이터를 볼 수 있습니다.

`bundle.json.files[]`의 각 항목은 경로명이 아닌 `contents` 아래에 실제 캡처된 파일 페이로드를 포함합니다. 텍스트 아티팩트는 UTF-8 텍스트로 저장되고, 바이너리 아티팩트는 base64와 `encoding` 마커를 사용합니다.

내장 로컬 어댑터는 이제 네이티브 세션 아티팩트를 더 직접적으로 내보냅니다:

- `codex_local`: `adapter/codex/session.jsonl`
- `claude_local`: `adapter/claude/session.jsonl`, 그리고 존재하는 경우 `adapter/claude/session/...` 사이드카 파일과 `adapter/claude/debug.txt`
- `opencode_local`: `adapter/opencode/session.json`, `adapter/opencode/messages/*.json`, `adapter/opencode/parts/<messageId>/*.json`, 선택적 `project.json`, `todo.json`, `session-diff.json`

## 공유 선호도

처음 투표할 때 동의 대화상자가 묻습니다:

- **로컬 보관** — 투표가 로컬에만 저장됩니다 (`sharedWithLabs: false`)
- **이 투표 공유** — 투표가 공유로 표시됩니다 (`sharedWithLabs: true`)

선호도는 회사별로 저장됩니다. 피드백 설정을 통해 언제든지 변경할 수 있습니다. "로컬 보관"으로 표시된 투표는 내보내기를 위해 대기열에 넣어지지 않습니다.

## 데이터 수명주기

| 상태 | 의미 |
|--------|---------|
| `local_only` | 로컬에 저장된 투표, 공유로 표시되지 않음 |
| `pending` | 공유로 표시됨, 로컬에 저장됨, 즉시 업로드 시도 대기 중 |
| `sent` | 성공적으로 전송됨 |
| `failed` | 전송이 시도되었지만 실패함 (예: 백엔드에 도달할 수 없거나 구성되지 않음); 이후 플러시가 백엔드 사용 가능 시 재시도 |

로컬 데이터베이스는 공유 상태와 관계없이 항상 전체 투표 및 추적 데이터를 유지합니다.

## 원격 동기화

공유로 선택한 투표는 투표 요청에서 즉시 텔레메트리 백엔드로 전송됩니다. 서버는 또한 백그라운드 플러시 워커를 유지하여 실패한 추적을 나중에 재시도할 수 있습니다. 텔레메트리 백엔드는 요청을 검증한 다음 번들을 구성된 객체 저장소에 저장합니다.

- 앱 서버 책임: 번들을 구성하고, 텔레메트리 백엔드에 POST하고, 추적 상태를 업데이트
- 텔레메트리 백엔드 책임: 요청을 인증하고, 페이로드 형태를 검증하고, 번들을 압축/저장하고, 최종 객체 키를 반환
- 재시도 동작: 실패한 업로드는 `failureReason`에 오류 메시지와 함께 `failed`로 이동하고, 워커가 이후 틱에서 재시도
- 기본 엔드포인트: 피드백 내보내기 백엔드 URL이 구성되지 않은 경우, Paperclip은 `https://telemetry.paperclip.ing`으로 폴백
- 중요한 뉘앙스: 업로드된 객체는 투표 시점의 전체 번들 스냅샷입니다. 나중에 로컬 번들을 가져오고 기본 어댑터 세션 파일이 계속 커진 경우, 로컬에서 재생성된 번들은 같은 추적에 대해 이미 업로드된 스냅샷보다 클 수 있습니다.

내보낸 객체는 쉽게 검사할 수 있는 결정적 키 패턴을 사용합니다:

```text
feedback-traces/<companyId>/YYYY/MM/DD/<exportId-or-traceId>.json
```
