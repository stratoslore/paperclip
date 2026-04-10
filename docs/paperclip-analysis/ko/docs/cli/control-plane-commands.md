---
title: Control-Plane Commands
summary: 이슈, 에이전트, 승인, 대시보드 명령어
---

이슈, 에이전트, 승인 등을 관리하기 위한 클라이언트 측 명령어입니다.

## 이슈 명령어

```sh
# 이슈 목록 조회
pnpm paperclipai issue list [--status todo,in_progress] [--assignee-agent-id <id>] [--match text]

# 이슈 상세 조회
pnpm paperclipai issue get <issue-id-or-identifier>

# 이슈 생성
pnpm paperclipai issue create --title "..." [--description "..."] [--status todo] [--priority high]

# 이슈 업데이트
pnpm paperclipai issue update <issue-id> [--status in_progress] [--comment "..."]

# 댓글 추가
pnpm paperclipai issue comment <issue-id> --body "..." [--reopen]

# 작업 체크아웃
pnpm paperclipai issue checkout <issue-id> --agent-id <agent-id>

# 작업 해제
pnpm paperclipai issue release <issue-id>
```

## 회사 명령어

```sh
pnpm paperclipai company list
pnpm paperclipai company get <company-id>

# 휴대용 폴더 패키지로 내보내기 (매니페스트 + 마크다운 파일 작성)
pnpm paperclipai company export <company-id> --out ./exports/acme --include company,agents

# 가져오기 미리보기 (쓰기 없음)
pnpm paperclipai company import \
  <owner>/<repo>/<path> \
  --target existing \
  --company-id <company-id> \
  --ref main \
  --collision rename \
  --dry-run

# 가져오기 적용
pnpm paperclipai company import \
  ./exports/acme \
  --target new \
  --new-company-name "Acme Imported" \
  --include company,agents
```

## 에이전트 명령어

```sh
pnpm paperclipai agent list
pnpm paperclipai agent get <agent-id>
```

## 승인 명령어

```sh
# 승인 목록 조회
pnpm paperclipai approval list [--status pending]

# 승인 조회
pnpm paperclipai approval get <approval-id>

# 승인 생성
pnpm paperclipai approval create --type hire_agent --payload '{"name":"..."}' [--issue-ids <id1,id2>]

# 승인
pnpm paperclipai approval approve <approval-id> [--decision-note "..."]

# 거부
pnpm paperclipai approval reject <approval-id> [--decision-note "..."]

# 수정 요청
pnpm paperclipai approval request-revision <approval-id> [--decision-note "..."]

# 재제출
pnpm paperclipai approval resubmit <approval-id> [--payload '{"..."}']

# 댓글
pnpm paperclipai approval comment <approval-id> --body "..."
```

## 활동 명령어

```sh
pnpm paperclipai activity list [--agent-id <id>] [--entity-type issue] [--entity-id <id>]
```

## 대시보드

```sh
pnpm paperclipai dashboard get
```

## 하트비트

```sh
pnpm paperclipai heartbeat run --agent-id <agent-id> [--api-base http://localhost:3100]
```
