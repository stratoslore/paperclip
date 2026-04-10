# CLI 참조

Paperclip CLI는 이제 다음 두 가지를 모두 지원합니다:

- 인스턴스 설정/진단 (`onboard`, `doctor`, `configure`, `env`, `allowed-hostname`)
- 컨트롤 플레인 클라이언트 작업 (issues, approvals, agents, activity, dashboard)

## 기본 사용법

개발 시 저장소 스크립트를 사용합니다:

```sh
pnpm paperclipai --help
```

처음 로컬 부트스트랩 + 실행:

```sh
pnpm paperclipai run
```

로컬 인스턴스 선택:

```sh
pnpm paperclipai run --instance dev
```

## 배포 모드

모드 분류와 설계 의도는 `doc/DEPLOYMENT-MODES.md`에 문서화되어 있습니다.

현재 CLI 동작:

- `paperclipai onboard`와 `paperclipai configure --section server`가 설정에 배포 모드를 저장
- 런타임에서 `PAPERCLIP_DEPLOYMENT_MODE`로 모드 오버라이드 가능
- `paperclipai run`과 `paperclipai doctor`는 아직 직접적인 `--mode` 플래그를 노출하지 않음

목표 동작(계획됨)은 `doc/DEPLOYMENT-MODES.md` 섹션 5에 문서화되어 있습니다.

인증/프라이빗 호스트명 허용 (예: 커스텀 Tailscale DNS):

```sh
pnpm paperclipai allowed-hostname dotta-macbook-pro
```

모든 클라이언트 명령은 다음을 지원합니다:

- `--data-dir <path>`
- `--api-base <url>`
- `--api-key <token>`
- `--context <path>`
- `--profile <name>`
- `--json`

회사 범위 명령은 `--company-id <id>`도 지원합니다.

모든 기본 로컬 상태(config/context/db/logs/storage/secrets)를 `~/.paperclip`에서 분리하려면 CLI 명령에 `--data-dir`를 사용합니다:

```sh
pnpm paperclipai run --data-dir ./tmp/paperclip-dev
pnpm paperclipai issue list --data-dir ./tmp/paperclip-dev
```

## 컨텍스트 프로파일

`~/.paperclip/context.json`에 로컬 기본값을 저장합니다:

```sh
pnpm paperclipai context set --api-base http://localhost:3100 --company-id <company-id>
pnpm paperclipai context show
pnpm paperclipai context list
pnpm paperclipai context use default
```

컨텍스트에 비밀을 저장하지 않으려면 `apiKeyEnvVarName`을 설정하고 키를 env에 보관합니다:

```sh
pnpm paperclipai context set --api-key-env-var-name PAPERCLIP_API_KEY
export PAPERCLIP_API_KEY=...
```

## 회사 명령

```sh
pnpm paperclipai company list
pnpm paperclipai company get <company-id>
pnpm paperclipai company delete <company-id-or-prefix> --yes --confirm <same-id-or-prefix>
```

예시:

```sh
pnpm paperclipai company delete PAP --yes --confirm PAP
pnpm paperclipai company delete 5cbe79ee-acb3-4597-896e-7662742593cd --yes --confirm 5cbe79ee-acb3-4597-896e-7662742593cd
```

참고:

- 삭제는 `PAPERCLIP_ENABLE_COMPANY_DELETION`에 의해 서버에서 제어됩니다.
- 에이전트 인증의 경우, 회사 삭제는 회사 범위입니다. 다른 회사가 아닌 현재 회사 ID/접두사를 사용하세요(예: `--company-id` 또는 `PAPERCLIP_COMPANY_ID` 통해).

## 이슈 명령

```sh
pnpm paperclipai issue list --company-id <company-id> [--status todo,in_progress] [--assignee-agent-id <agent-id>] [--match text]
pnpm paperclipai issue get <issue-id-or-identifier>
pnpm paperclipai issue create --company-id <company-id> --title "..." [--description "..."] [--status todo] [--priority high]
pnpm paperclipai issue update <issue-id> [--status in_progress] [--comment "..."]
pnpm paperclipai issue comment <issue-id> --body "..." [--reopen]
pnpm paperclipai issue checkout <issue-id> --agent-id <agent-id> [--expected-statuses todo,backlog,blocked]
pnpm paperclipai issue release <issue-id>
```

## 에이전트 명령

```sh
pnpm paperclipai agent list --company-id <company-id>
pnpm paperclipai agent get <agent-id>
pnpm paperclipai agent local-cli <agent-id-or-shortname> --company-id <company-id>
```

`agent local-cli`는 로컬 Claude/Codex를 Paperclip 에이전트로 수동 실행하는 가장 빠른 방법입니다:

- 새로운 장기 에이전트 API 키를 생성
- 누락된 Paperclip 스킬을 `~/.codex/skills`와 `~/.claude/skills`에 설치
- `PAPERCLIP_API_URL`, `PAPERCLIP_COMPANY_ID`, `PAPERCLIP_AGENT_ID`, `PAPERCLIP_API_KEY`에 대한 `export ...` 행을 출력

단축명 기반 로컬 설정 예시:

```sh
pnpm paperclipai agent local-cli codexcoder --company-id <company-id>
pnpm paperclipai agent local-cli claudecoder --company-id <company-id>
```

## 승인 명령

```sh
pnpm paperclipai approval list --company-id <company-id> [--status pending]
pnpm paperclipai approval get <approval-id>
pnpm paperclipai approval create --company-id <company-id> --type hire_agent --payload '{"name":"..."}' [--issue-ids <id1,id2>]
pnpm paperclipai approval approve <approval-id> [--decision-note "..."]
pnpm paperclipai approval reject <approval-id> [--decision-note "..."]
pnpm paperclipai approval request-revision <approval-id> [--decision-note "..."]
pnpm paperclipai approval resubmit <approval-id> [--payload '{"...":"..."}']
pnpm paperclipai approval comment <approval-id> --body "..."
```

## 활동 명령

```sh
pnpm paperclipai activity list --company-id <company-id> [--agent-id <agent-id>] [--entity-type issue] [--entity-id <id>]
```

## 대시보드 명령

```sh
pnpm paperclipai dashboard get --company-id <company-id>
```

## 하트비트 명령

`heartbeat run`은 이제 컨텍스트/api-key 옵션도 지원하며 공유 클라이언트 스택을 사용합니다:

```sh
pnpm paperclipai heartbeat run --agent-id <agent-id> [--api-base http://localhost:3100] [--api-key <token>]
```

## 로컬 저장소 기본값

기본 로컬 인스턴스 루트는 `~/.paperclip/instances/default`입니다:

- 설정: `~/.paperclip/instances/default/config.json`
- 내장 db: `~/.paperclip/instances/default/db`
- 로그: `~/.paperclip/instances/default/logs`
- 저장소: `~/.paperclip/instances/default/data/storage`
- 비밀 키: `~/.paperclip/instances/default/secrets/master.key`

환경 변수로 기본 홈 또는 인스턴스를 오버라이드합니다:

```sh
PAPERCLIP_HOME=/custom/home PAPERCLIP_INSTANCE_ID=dev pnpm paperclipai run
```

## 저장소 설정

저장소 제공자 및 설정을 구성합니다:

```sh
pnpm paperclipai configure --section storage
```

지원되는 제공자:

- `local_disk` (기본값; 로컬 단일 사용자 설치)
- `s3` (S3 호환 객체 저장소)
