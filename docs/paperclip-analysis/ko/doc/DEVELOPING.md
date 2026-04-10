# 개발

이 프로젝트는 PostgreSQL을 수동으로 설정하지 않고도 로컬 개발에서 완전히 실행할 수 있습니다.

## 배포 모드

모드 정의 및 의도된 CLI 동작은 `doc/DEPLOYMENT-MODES.md`를 참조하세요.

현재 구현 상태:

- 정식 모델: `local_trusted` 및 `authenticated` (`private/public` 노출 포함)

## 사전 요구사항

- Node.js 20+
- pnpm 9+

## 의존성 잠금 파일 정책

GitHub Actions가 `pnpm-lock.yaml`을 소유합니다.

- 풀 리퀘스트에서 `pnpm-lock.yaml`을 커밋하지 마세요.
- 풀 리퀘스트 CI는 매니페스트가 변경될 때 의존성 해결을 검증합니다.
- `master`로의 푸시는 `pnpm install --lockfile-only --no-frozen-lockfile`로 `pnpm-lock.yaml`을 재생성하고, 필요하면 커밋하고, `--frozen-lockfile`로 검증을 실행합니다.

## 개발 시작

저장소 루트에서:

```sh
pnpm install
pnpm dev
```

이것은 다음을 시작합니다:

- API 서버: `http://localhost:3100`
- UI: 개발 미들웨어 모드로 API 서버가 제공 (API와 동일 출처)

`pnpm dev`는 서버를 watch 모드로 실행하고 워크스페이스 패키지(어댑터 패키지 포함)의 변경 시 재시작합니다. 파일 감시 없이 실행하려면 `pnpm dev:once`를 사용하세요.

`pnpm dev:once`는 개발 서버를 시작하기 전에 기본적으로 보류 중인 로컬 마이그레이션을 자동 적용합니다.

`pnpm dev`와 `pnpm dev:once`는 이제 현재 저장소 및 인스턴스에 대해 멱등합니다: 일치하는 Paperclip 개발 러너가 이미 실행 중이면, Paperclip은 중복을 시작하지 않고 기존 프로세스를 보고합니다.

현재 저장소의 관리형 개발 러너를 검사하거나 중지:

```sh
pnpm dev:list
pnpm dev:stop
```

`pnpm dev:once`는 이제 백엔드 관련 파일 변경과 보류 중인 마이그레이션을 추적합니다. 현재 부트가 오래된 경우, 보드 UI에 `재시작 필요` 배너가 표시됩니다. `인스턴스 설정 > 실험적`에서 보호된 자동 재시작을 활성화할 수도 있으며, 대기/실행 중인 로컬 에이전트 실행이 완료될 때까지 기다린 후 개발 서버를 재시작합니다.

Tailscale/프라이빗 인증 개발 모드:

```sh
pnpm dev --tailscale-auth
```

이것은 `authenticated/private`로 개발을 실행하고 서버를 프라이빗 네트워크 접근을 위해 `0.0.0.0`에 바인딩합니다.

추가 프라이빗 호스트명 허용 (예: 커스텀 Tailscale 호스트명):

```sh
pnpm paperclipai allowed-hostname dotta-macbook-pro
```

## 원커맨드 로컬 실행

처음 로컬 설치 시, 한 명령으로 부트스트랩하고 실행할 수 있습니다:

```sh
pnpm paperclipai run
```

`paperclipai run`은 다음을 수행합니다:

1. 설정이 없으면 자동 온보드
2. 수리 활성화로 `paperclipai doctor` 실행
3. 검사 통과 시 서버 시작

## Docker 퀵스타트 (로컬 Node 설치 불필요)

Docker에서 Paperclip을 빌드하고 실행:

```sh
docker build -t paperclip-local .
docker run --name paperclip \
  -p 3100:3100 \
  -e HOST=0.0.0.0 \
  -e PAPERCLIP_HOME=/paperclip \
  -v "$(pwd)/data/docker-paperclip:/paperclip" \
  paperclip-local
```

또는 Compose 사용:

```sh
docker compose -f docker/docker-compose.quickstart.yml up --build
```

API 키 연결(`OPENAI_API_KEY` / `ANTHROPIC_API_KEY`) 및 영속성 세부사항은 `doc/DOCKER.md`를 참조하세요.

## 비신뢰 PR 리뷰를 위한 Docker

Docker 볼륨에 `codex`/`claude` 로그인 상태를 유지하고 PR을 격리된 스크래치 워크스페이스에 체크아웃하는 별도의 리뷰 지향 컨테이너는 `doc/UNTRUSTED-PR-REVIEW.md`를 참조하세요.

## 개발 시 데이터베이스 (자동 처리)

로컬 개발에서는 `DATABASE_URL`을 설정하지 않습니다.
서버가 자동으로 내장 PostgreSQL을 사용하고 다음에 데이터를 영속합니다:

- `~/.paperclip/instances/default/db`

홈 및 인스턴스 오버라이드:

```sh
PAPERCLIP_HOME=/custom/path PAPERCLIP_INSTANCE_ID=dev pnpm paperclipai run
```

이 모드에는 Docker나 외부 데이터베이스가 필요하지 않습니다.

## 개발 시 저장소 (자동 처리)

로컬 개발에서 기본 저장소 제공자는 `local_disk`이며, 업로드된 이미지/첨부 파일을 다음에 영속합니다:

- `~/.paperclip/instances/default/data/storage`

저장소 제공자/설정 구성:

```sh
pnpm paperclipai configure --section storage
```

## 기본 에이전트 워크스페이스

로컬 에이전트 실행에 해결된 프로젝트/세션 워크스페이스가 없는 경우, Paperclip은 인스턴스 루트 아래의 에이전트 홈 워크스페이스로 폴백합니다:

- `~/.paperclip/instances/default/workspaces/<agent-id>`

이 경로는 비기본 설정에서 `PAPERCLIP_HOME` 및 `PAPERCLIP_INSTANCE_ID`를 준수합니다.

`codex_local`의 경우, Paperclip은 인스턴스 루트 아래에 회사별 Codex 홈도 관리하고 공유 Codex 로그인/설정 홈(`$CODEX_HOME` 또는 `~/.codex`)에서 시드합니다:

- `~/.paperclip/instances/default/companies/<company-id>/codex-home`

`codex` CLI가 설치되지 않았거나 `PATH`에 없으면, `codex_local` 에이전트 실행은 실행 시점에 명확한 어댑터 오류와 함께 실패합니다. 할당량 폴링은 단기 `codex app-server` 서브프로세스를 사용합니다: `codex`를 생성할 수 없으면 해당 제공자는 집계된 할당량 결과에서 `ok: false`를 보고하고 API 서버는 계속 실행됩니다(누락된 바이너리에서 종료되면 안 됩니다).

## 워크트리 로컬 인스턴스

여러 git 워크트리에서 개발할 때, 두 Paperclip 서버를 같은 내장 PostgreSQL 데이터 디렉토리에 지정하지 마세요.

대신, 워크트리를 위한 저장소 로컬 Paperclip 설정과 격리된 인스턴스를 생성하세요:

```sh
paperclipai worktree init
# 또는 git 워크트리를 생성하고 한 단계에서 초기화:
pnpm paperclipai worktree:make paperclip-pr-432
```

이 명령은:

- `.paperclip/config.json` 및 `.paperclip/.env`에 저장소 로컬 파일을 작성
- `~/.paperclip-worktrees/instances/<worktree-id>/` 아래에 격리된 인스턴스를 생성
- 연결된 git 워크트리 내에서 실행되면, 유효한 git 훅을 해당 워크트리의 프라이빗 git 디렉토리에 미러링
- 사용 가능한 앱 포트와 내장 PostgreSQL 포트를 선택
- 기본적으로 현재 유효한 Paperclip 인스턴스/설정(있는 경우 저장소 로컬 워크트리 설정, 그렇지 않으면 기본 인스턴스)에서 논리적 SQL 스냅샷을 통해 `minimal` 모드로 격리된 DB를 시드

시드 모드:

- `minimal`은 회사, 프로젝트, 이슈, 댓글, 승인, 인증 상태와 같은 핵심 앱 상태를 유지하고, 모든 테이블의 스키마를 보존하지만, 하트비트 실행, 웨이크 요청, 활동 로그, 런타임 서비스, 에이전트 세션 상태와 같은 무거운 운영 이력의 행 데이터를 생략
- `full`은 소스 인스턴스의 전체 논리 복제를 수행
- `--no-seed`는 빈 격리 인스턴스를 생성

`worktree init` 후, 서버와 CLI 모두 해당 워크트리 내에서 실행 시 저장소 로컬 `.paperclip/.env`를 자동 로드하므로, `pnpm dev`, `paperclipai doctor`, `paperclipai db:backup`과 같은 일반 명령이 워크트리 인스턴스에 범위가 지정됩니다.

프로비전된 git 워크트리는 기본적으로 격리된 워크트리 데이터베이스의 모든 시드된 루틴을 일시 중지합니다. 이를 통해 복사된 일일/cron 루틴이 개발 중 새 워크스페이스 인스턴스에서 예기치 않게 실행되는 것을 방지합니다.

해당 저장소 로컬 env는 또한 다음을 설정합니다:

- `PAPERCLIP_IN_WORKTREE=true`
- `PAPERCLIP_WORKTREE_NAME=<worktree-name>`
- `PAPERCLIP_WORKTREE_COLOR=<hex-color>`

서버/UI는 이 값을 상단 배너 및 동적으로 색상이 지정된 파비콘과 같은 워크트리별 브랜딩에 사용합니다.

필요할 때 셸 내보내기를 명시적으로 출력:

```sh
paperclipai worktree env
# 또는:
eval "$(paperclipai worktree env)"
```

### 워크트리 CLI 참조

**`pnpm paperclipai worktree init [options]`** -- 현재 워크트리를 위한 저장소 로컬 설정/env 및 격리된 인스턴스를 생성합니다.

| 옵션 | 설명 |
|---|---|
| `--name <name>` | 인스턴스 id를 파생하는 데 사용되는 표시 이름 |
| `--instance <id>` | 명시적 격리 인스턴스 id |
| `--home <path>` | 워크트리 인스턴스의 홈 루트 (기본값: `~/.paperclip-worktrees`) |
| `--from-config <path>` | 시드할 소스 config.json |
| `--from-data-dir <path>` | 소스 설정 파생 시 사용되는 소스 PAPERCLIP_HOME |
| `--from-instance <id>` | 소스 인스턴스 id (기본값: `default`) |
| `--server-port <port>` | 선호 서버 포트 |
| `--db-port <port>` | 선호 내장 Postgres 포트 |
| `--seed-mode <mode>` | 시드 프로필: `minimal` 또는 `full` (기본값: `minimal`) |
| `--no-seed` | 소스 인스턴스에서 데이터베이스 시딩 건너뛰기 |
| `--force` | 기존 저장소 로컬 설정 및 격리된 인스턴스 데이터 대체 |

예시:

```sh
paperclipai worktree init --no-seed
paperclipai worktree init --seed-mode full
paperclipai worktree init --from-instance default
paperclipai worktree init --from-data-dir ~/.paperclip
paperclipai worktree init --force
```

이미 생성된 저장소 관리형 워크트리를 수리하고 메인 기본 설치에서 격리된 인스턴스를 다시 시드:

```sh
cd ~/.paperclip/worktrees/PAP-884-ai-commits-component
pnpm paperclipai worktree init --force --seed-mode minimal \
  --name PAP-884-ai-commits-component \
  --from-config ~/.paperclip/instances/default/config.json
```

이것은 워크트리 로컬 `.paperclip/config.json` + `.paperclip/.env`를 다시 작성하고, `~/.paperclip-worktrees/instances/<worktree-id>/` 아래에 격리된 인스턴스를 다시 생성하며, git 워크트리 내용 자체는 보존합니다.

기존 저장소 로컬 설정/env를 유지하면서 격리된 데이터베이스만 덮어쓰려면 `worktree reseed`를 대신 사용하세요. 명령이 안전하게 DB를 교체할 수 있도록 대상 워크트리의 Paperclip 서버를 먼저 중지하세요.

**`pnpm paperclipai worktree reseed [options]`** -- 대상 워크트리의 현재 설정, 포트, 인스턴스 ID를 유지하면서 다른 Paperclip 인스턴스 또는 워크트리에서 기존 워크트리 로컬 인스턴스를 다시 시드합니다.

| 옵션 | 설명 |
|---|---|
| `--from <worktree>` | 소스 워크트리 경로, 디렉토리 이름, 브랜치 이름, 또는 `current` |
| `--to <worktree>` | 대상 워크트리 경로, 디렉토리 이름, 브랜치 이름, 또는 `current` (기본값: `current`) |
| `--from-config <path>` | 시드할 소스 config.json |
| `--from-data-dir <path>` | 소스 설정 파생 시 사용되는 소스 `PAPERCLIP_HOME` |
| `--from-instance <id>` | 소스 설정 파생 시 소스 인스턴스 id |
| `--seed-mode <mode>` | 시드 프로필: `minimal` 또는 `full` (기본값: `full`) |
| `--yes` | 파괴적 확인 프롬프트 건너뛰기 |
| `--allow-live-target` | 대상 워크트리 DB가 먼저 중지되어야 하는 가드를 오버라이드 |

예시:

```sh
# 메인 저장소에서 현재 기본/master 인스턴스로부터 워크트리를 다시 시드.
cd /path/to/paperclip
pnpm paperclipai worktree reseed \
  --from current \
  --to PAP-1132-assistant-ui-pap-1131-make-issues-comments-be-like-a-chat \
  --seed-mode full \
  --yes

# 워크트리 내에서 기본 인스턴스 설정으로부터 다시 시드.
cd /path/to/paperclip/.paperclip/worktrees/PAP-1132-assistant-ui-pap-1131-make-issues-comments-be-like-a-chat
pnpm paperclipai worktree reseed \
  --from-instance default \
  --seed-mode full
```

**`pnpm paperclipai worktree:make <name> [options]`** -- `~/NAME`을 git 워크트리로 생성한 다음, 그 안에 격리된 Paperclip 인스턴스를 초기화합니다. `git worktree add`와 `worktree init`을 한 단계로 결합합니다.

| 옵션 | 설명 |
|---|---|
| `--start-point <ref>` | 새 브랜치의 기반이 되는 원격 ref (예: `origin/main`) |
| `--instance <id>` | 명시적 격리 인스턴스 id |
| `--home <path>` | 워크트리 인스턴스의 홈 루트 (기본값: `~/.paperclip-worktrees`) |
| `--from-config <path>` | 시드할 소스 config.json |
| `--from-data-dir <path>` | 소스 PAPERCLIP_HOME |
| `--from-instance <id>` | 소스 인스턴스 id (기본값: `default`) |
| `--server-port <port>` | 선호 서버 포트 |
| `--db-port <port>` | 선호 내장 Postgres 포트 |
| `--seed-mode <mode>` | 시드 프로필: `minimal` 또는 `full` (기본값: `minimal`) |
| `--no-seed` | 시딩 건너뛰기 |
| `--force` | 기존 설정 및 데이터 대체 |

예시:

```sh
pnpm paperclipai worktree:make paperclip-pr-432
pnpm paperclipai worktree:make my-feature --start-point origin/main
pnpm paperclipai worktree:make experiment --no-seed
```

**`pnpm paperclipai worktree env [options]`** -- 현재 워크트리 로컬 Paperclip 인스턴스의 셸 내보내기를 출력합니다.

| 옵션 | 설명 |
|---|---|
| `-c, --config <path>` | 설정 파일 경로 |
| `--json` | 셸 내보내기 대신 JSON 출력 |

예시:

```sh
pnpm paperclipai worktree env
pnpm paperclipai worktree env --json
eval "$(pnpm paperclipai worktree env)"
```

프로젝트 실행 워크트리의 경우, Paperclip은 격리된 git 워크트리를 생성하거나 재사용한 후 프로젝트 정의 프로비전 명령을 실행할 수도 있습니다. 프로젝트의 실행 워크스페이스 정책(`workspaceStrategy.provisionCommand`)에서 이를 구성합니다. 명령은 파생된 워크트리 내에서 실행되며 `PAPERCLIP_WORKSPACE_*`, `PAPERCLIP_PROJECT_ID`, `PAPERCLIP_AGENT_ID`, `PAPERCLIP_ISSUE_*` 환경 변수를 받아 각 저장소가 원하는 방식으로 부트스트랩할 수 있습니다.

## 빠른 상태 확인

다른 터미널에서:

```sh
curl http://localhost:3100/api/health
curl http://localhost:3100/api/companies
```

예상:

- `/api/health`는 `{"status":"ok"}`를 반환
- `/api/companies`는 JSON 배열을 반환

## 로컬 개발 데이터베이스 초기화

로컬 개발 데이터를 초기화하고 새로 시작하려면:

```sh
rm -rf ~/.paperclip/instances/default/db
pnpm dev
```

## 선택사항: 외부 Postgres 사용

`DATABASE_URL`을 설정하면 서버가 내장 PostgreSQL 대신 그것을 사용합니다.

## 자동 DB 백업

Paperclip은 타이머로 자동 DB 백업을 실행할 수 있습니다. 기본값:

- 활성화
- 60분마다
- 30일 보관
- 백업 디렉토리: `~/.paperclip/instances/default/data/backups`

다음에서 구성:

```sh
pnpm paperclipai configure --section database
```

수동으로 일회성 백업 실행:

```sh
pnpm paperclipai db:backup
# 또는:
pnpm db:backup
```

환경 오버라이드:

- `PAPERCLIP_DB_BACKUP_ENABLED=true|false`
- `PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES=<minutes>`
- `PAPERCLIP_DB_BACKUP_RETENTION_DAYS=<days>`
- `PAPERCLIP_DB_BACKUP_DIR=/absolute/or/~/path`

## 개발 시 비밀

에이전트 환경 변수는 이제 비밀 참조를 지원합니다. 기본적으로 비밀 값은 로컬 암호화로 저장되고 비밀 참조만 에이전트 설정에 영속됩니다.

- 기본 로컬 키 경로: `~/.paperclip/instances/default/secrets/master.key`
- 키 자료 직접 오버라이드: `PAPERCLIP_SECRETS_MASTER_KEY`
- 키 파일 경로 오버라이드: `PAPERCLIP_SECRETS_MASTER_KEY_FILE`

엄격 모드 (로컬 신뢰 머신 외부에서 권장):

```sh
PAPERCLIP_SECRETS_STRICT_MODE=true
```

엄격 모드가 활성화되면, 민감한 환경 키(예: `*_API_KEY`, `*_TOKEN`, `*_SECRET`)는 인라인 일반 값 대신 비밀 참조를 사용해야 합니다.

CLI 구성 지원:

- `pnpm paperclipai onboard`는 기본 `secrets` 설정 섹션(`local_encrypted`, 엄격 모드 비활성, 키 파일 경로 설정)을 작성하고 필요 시 로컬 키 파일을 생성합니다.
- `pnpm paperclipai configure --section secrets`로 제공자/엄격 모드/키 경로를 업데이트하고 필요 시 로컬 키 파일을 생성합니다.
- `pnpm paperclipai doctor`는 비밀 어댑터 구성을 검증하고 `--repair`로 누락된 로컬 키 파일을 생성할 수 있습니다.

기존 인라인 환경 비밀 마이그레이션 헬퍼:

```sh
pnpm secrets:migrate-inline-env         # 드라이 런
pnpm secrets:migrate-inline-env --apply # 마이그레이션 적용
```

## 회사 삭제 토글

회사 삭제는 개발/디버그 기능으로 의도되었으며 런타임에 비활성화할 수 있습니다:

```sh
PAPERCLIP_ENABLE_COMPANY_DELETION=false
```

기본 동작:

- `local_trusted`: 활성화
- `authenticated`: 비활성화

## CLI 클라이언트 작업

Paperclip CLI는 이제 설정 명령 외에 클라이언트 측 컨트롤 플레인 명령을 포함합니다.

빠른 예시:

```sh
pnpm paperclipai issue list --company-id <company-id>
pnpm paperclipai issue create --company-id <company-id> --title "Investigate checkout conflict"
pnpm paperclipai issue update <issue-id> --status in_progress --comment "Started triage"
```

컨텍스트 프로파일로 한 번 기본값 설정:

```sh
pnpm paperclipai context set --api-base http://localhost:3100 --company-id <company-id>
```

그러면 플래그를 반복하지 않고 명령 실행:

```sh
pnpm paperclipai issue list
pnpm paperclipai dashboard get
```

전체 명령 참조는 `doc/CLI.md`를 참조하세요.

## OpenClaw 초대 온보딩 엔드포인트

에이전트 지향 초대 온보딩은 이제 기계 판독 가능한 API 문서를 노출합니다:

- `GET /api/invites/:token`은 초대 요약과 온보딩 및 스킬 인덱스 링크를 반환합니다.
- `GET /api/invites/:token/onboarding`은 온보딩 매니페스트 세부사항(등록 엔드포인트, 클레임 엔드포인트 템플릿, 스킬 설치 힌트)을 반환합니다.
- `GET /api/invites/:token/onboarding.txt`는 선택적 초대자 메시지와 제안된 네트워크 호스트 후보를 포함한, 운영자와 에이전트 모두를 위한 평문 온보딩 문서(llm.txt 스타일 핸드오프)를 반환합니다.
- `GET /api/skills/index`는 사용 가능한 스킬 문서를 나열합니다.
- `GET /api/skills/paperclip`은 Paperclip 하트비트 스킬 마크다운을 반환합니다.

## OpenClaw 참여 스모크 테스트

엔드투엔드 OpenClaw 참여 스모크 하네스를 실행합니다:

```sh
pnpm smoke:openclaw-join
```

검증하는 것:

- 에이전트 전용 참여를 위한 초대 생성
- `adapterType=openclaw`를 사용한 에이전트 참여 요청
- 보드 승인 + 일회성 API 키 클레임 의미론
- 도커화된 OpenClaw 스타일 웹훅 수신기로의 웨이크업 시 콜백 전달

필수 권한:

- 이 스크립트는 보드 관리 동작(초대 생성, 참여 승인, 다른 에이전트 웨이크업)을 수행합니다.
- 인증 모드에서는 `PAPERCLIP_AUTH_HEADER` 또는 `PAPERCLIP_COOKIE`를 통해 보드 인증으로 실행합니다.

선택적 인증 플래그 (인증 모드용):

- `PAPERCLIP_AUTH_HEADER` (예: `Bearer ...`)
- `PAPERCLIP_COOKIE` (세션 쿠키 헤더 값)

## OpenClaw Docker UI 원커맨드 스크립트

한 명령으로 Docker에서 OpenClaw를 부팅하고 호스트 브라우저 대시보드 URL을 출력하려면:

```sh
pnpm smoke:openclaw-docker-ui
```

이 스크립트는 `scripts/smoke/openclaw-docker-ui.sh`에 있으며, Compose 기반 로컬 OpenClaw UI 테스팅을 위한 clone/build/config/start를 자동화합니다.

이 스모크 스크립트의 페어링 동작:

- 기본 `OPENCLAW_DISABLE_DEVICE_AUTH=1` (로컬 스모크에서 Control UI 페어링 프롬프트 없음; 추가 페어링 환경 변수 불필요)
- 표준 디바이스 페어링을 요구하려면 `OPENCLAW_DISABLE_DEVICE_AUTH=0`으로 설정

이 스모크 스크립트의 모델 동작:

- 기본적으로 Anthropic 인증이 필요 없도록 OpenAI 모델(`openai/gpt-5.2` + OpenAI 폴백)을 사용

이 스모크 스크립트의 상태 동작:

- 기본적으로 격리된 설정 디렉토리 `~/.openclaw-paperclip-smoke` 사용
- 오래된 제공자/인증 드리프트를 피하기 위해 기본적으로 매 실행마다 스모크 에이전트 상태를 초기화(`OPENCLAW_RESET_STATE=1`)

이 스모크 스크립트의 네트워킹 동작:

- OpenClaw Docker 내에서 도달 가능한 Paperclip 호스트 URL을 자동 감지하고 출력
- 기본 컨테이너 측 호스트 별칭은 `host.docker.internal` (`PAPERCLIP_HOST_FROM_CONTAINER` / `PAPERCLIP_HOST_PORT`로 오버라이드)
- Paperclip이 인증/프라이빗 모드에서 컨테이너 호스트명을 거부하면, `pnpm paperclipai allowed-hostname host.docker.internal`로 `host.docker.internal`을 허용하고 Paperclip을 재시작
