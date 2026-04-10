# Paperclip 릴리스

npm, GitHub, 웹사이트 변경 로그 등 Paperclip 배포를 위한 유지보수 담당자 운영 매뉴얼.

릴리스 모델은 이제 커밋 기반입니다:

1. `master`에 대한 모든 푸시는 자동으로 카나리를 배포합니다.
2. 안정(stable) 릴리스는 선택된 테스트 완료 커밋 또는 카나리 태그에서 수동으로 승격됩니다.
3. 안정 릴리스 노트는 `releases/vYYYY.MDD.P.md`에 위치합니다.
4. GitHub Releases는 안정 릴리스에서만 생성됩니다.

## 버전 관리 모델

Paperclip은 semver 구문에 맞는 캘린더 버전을 사용합니다:

- stable: `YYYY.MDD.P`
- canary: `YYYY.MDD.P-canary.N`

예시:

- 2026년 3월 18일 첫 번째 안정 릴리스: `2026.318.0`
- 2026년 3월 18일 두 번째 안정 릴리스: `2026.318.1`
- `2026.318.1` 라인의 네 번째 카나리: `2026.318.1-canary.3`

중요한 제약 사항:

- 중간 숫자 슬롯은 `MDD`이며, `M`은 UTC 월, `DD`는 0으로 채워진 UTC 일입니다
- 3월 3일에는 `2026.33.0`이 아닌 `2026.303.0`을 사용합니다
- `2026.0318.0`과 같이 앞에 0을 붙이지 마세요
- `2026.3.18.1`과 같이 4개의 숫자 세그먼트를 사용하지 마세요
- semver 호환 카나리 형식은 `2026.318.0-canary.1`입니다

## 릴리스 배포 대상

모든 안정 릴리스에는 4가지 배포 대상이 있습니다:

1. **검증** -- 해당 git SHA가 타입 체크, 테스트, 빌드를 통과합니다
2. **npm** -- `paperclipai` 및 공개 워크스페이스 패키지가 배포됩니다
3. **GitHub** -- 안정 릴리스에 git 태그와 GitHub Release가 생성됩니다
4. **웹사이트 / 공지** -- 안정 변경 로그가 외부에 게시되고 공지됩니다

안정 릴리스는 4가지 배포 대상이 모두 처리되어야만 완료됩니다.

카나리는 처음 두 가지 배포 대상과 내부 추적용 태그만 포함합니다.

## 핵심 불변 조건

- 카나리는 `master`에서 배포합니다
- 안정 릴리스는 명시적으로 선택된 소스 ref에서 배포합니다
- 태그는 생성된 릴리스 커밋이 아닌 원본 소스 커밋을 가리킵니다
- 안정 릴리스 노트는 항상 `releases/vYYYY.MDD.P.md`입니다
- 카나리는 GitHub Releases를 생성하지 않습니다
- 카나리는 변경 로그 생성이 필요하지 않습니다

## 요약

### 카나리

`master`에 대한 모든 푸시는 [`.github/workflows/release.yml`](../.github/workflows/release.yml) 내의 카나리 경로를 실행합니다.

수행 작업:

- 푸시된 커밋을 검증합니다
- 현재 UTC 날짜에 대한 카나리 버전을 계산합니다
- npm dist-tag `canary`로 배포합니다
- git 태그 `canary/vYYYY.MDD.P-canary.N`을 생성합니다

사용자는 다음과 같이 카나리를 설치합니다:

```bash
npx paperclipai@canary onboard
# or
npx paperclipai@canary onboard --data-dir "$(mktemp -d /tmp/paperclip-canary.XXXXXX)"
```

### 안정 릴리스

Actions 탭에서 수동 `workflow_dispatch` 입력과 함께 [`.github/workflows/release.yml`](../.github/workflows/release.yml)을 사용합니다.

[여기에서 액션 실행](https://github.com/paperclipai/paperclip/actions/workflows/release.yml)

입력값:

- `source_ref`
  - 커밋 SHA, 브랜치 또는 태그
- `stable_date`
  - 선택적 UTC 날짜 오버라이드 (`YYYY-MM-DD` 형식)
  - `2026.318.0`과 같은 버전이 아닌 `2026-03-18`과 같은 날짜를 입력합니다
- `dry_run`
  - true일 때 미리보기만 수행

안정 릴리스 실행 전:

1. 신뢰할 수 있는 카나리 커밋 또는 태그를 선택합니다
2. `./scripts/release.sh stable --date "$(date +%F)" --print-version`으로 대상 안정 버전을 확인합니다
3. 해당 소스 ref에서 `releases/vYYYY.MDD.P.md`를 생성하거나 업데이트합니다
4. 해당 소스 ref에서 안정 워크플로우를 실행합니다

예시:

- `source_ref`: `master`
- `stable_date`: `2026-03-18`
- 결과 안정 버전: `2026.318.0`

워크플로우 수행 작업:

- 정확한 소스 ref를 재검증합니다
- 선택된 UTC 날짜에 대한 다음 안정 패치 슬롯을 계산합니다
- npm dist-tag `latest`로 `YYYY.MDD.P`를 배포합니다
- git 태그 `vYYYY.MDD.P`를 생성합니다
- `releases/vYYYY.MDD.P.md`에서 GitHub Release를 생성하거나 업데이트합니다

## 로컬 명령어

### 로컬에서 카나리 미리보기

```bash
./scripts/release.sh canary --dry-run
```

### 로컬에서 안정 릴리스 미리보기

```bash
./scripts/release.sh stable --dry-run
```

### 로컬에서 안정 릴리스 배포

이는 주로 긴급/수동 사용을 위한 것입니다. 일반적인 경로는 GitHub 워크플로우입니다.

```bash
./scripts/release.sh stable
git push public-gh refs/tags/vYYYY.MDD.P
PUBLISH_REMOTE=public-gh ./scripts/create-github-release.sh YYYY.MDD.P
```

## 안정 변경 로그 워크플로우

안정 변경 로그 파일 위치:

- `releases/vYYYY.MDD.P.md`

카나리는 변경 로그 파일을 생성하지 않습니다.

권장 로컬 생성 흐름:

```bash
VERSION="$(./scripts/release.sh stable --date 2026-03-18 --print-version)"
claude --print --output-format stream-json --verbose --dangerously-skip-permissions --model claude-opus-4-6 "Use the release-changelog skill to draft or update releases/v${VERSION}.md for Paperclip. Read doc/RELEASING.md and .agents/skills/release-changelog/SKILL.md, then generate the stable changelog for v${VERSION} from commits since the last stable tag. Do not create a canary changelog."
```

저장소에서 이를 GitHub Actions를 통해 실행하지 않는 이유:

- 카나리가 너무 빈번합니다
- 안정 릴리스 노트만이 LLM 도움이 필요한 공개 설명 대상입니다
- 유지보수 담당자의 LLM 토큰은 Actions에 있어서는 안 됩니다

## 스모크 테스트

카나리의 경우:

```bash
PAPERCLIPAI_VERSION=canary ./scripts/docker-onboard-smoke.sh
```

현재 안정 릴리스의 경우:

```bash
PAPERCLIPAI_VERSION=latest ./scripts/docker-onboard-smoke.sh
```

유용한 격리 변형:

```bash
HOST_PORT=3232 DATA_DIR=./data/release-smoke-canary PAPERCLIPAI_VERSION=canary ./scripts/docker-onboard-smoke.sh
HOST_PORT=3233 DATA_DIR=./data/release-smoke-stable PAPERCLIPAI_VERSION=latest ./scripts/docker-onboard-smoke.sh
```

자동화된 브라우저 스모크 테스트도 사용 가능합니다:

```bash
gh workflow run release-smoke.yml -f paperclip_version=canary
gh workflow run release-smoke.yml -f paperclip_version=latest
```

최소 확인 항목:

- `npx paperclipai@canary onboard`가 설치됩니다
- 온보딩이 충돌 없이 완료됩니다
- 스모크 자격 증명으로 인증된 로그인이 작동합니다
- 새 인스턴스에서 브라우저가 온보딩 화면에 도달합니다
- 회사 생성이 성공합니다
- 첫 번째 CEO 에이전트가 생성됩니다
- 첫 번째 CEO heartbeat 실행이 트리거됩니다

## 롤백

롤백은 버전을 비공개로 전환하지 않습니다.

`latest` dist-tag를 이전 안정 릴리스로 되돌리기만 합니다:

```bash
./scripts/rollback-latest.sh 2026.318.0 --dry-run
./scripts/rollback-latest.sh 2026.318.0
```

이후 새 안정 패치 슬롯 또는 릴리스 날짜로 수정 사항을 적용합니다.

## 장애 대응 플레이북

### 카나리가 배포되었지만 스모크 테스트가 실패한 경우

안정 릴리스를 실행하지 마세요.

대신:

1. `master`에서 문제를 수정합니다
2. 수정 사항을 병합합니다
3. 다음 자동 카나리를 기다립니다
4. 스모크 테스트를 다시 실행합니다

### 안정 npm 배포는 성공했지만 태그 푸시 또는 GitHub Release 생성이 실패한 경우

이는 부분 릴리스입니다. npm은 이미 활성 상태입니다.

즉시 수행:

1. 누락된 태그를 푸시합니다
2. `PUBLISH_REMOTE=public-gh ./scripts/create-github-release.sh YYYY.MDD.P`를 다시 실행합니다
3. GitHub Release 노트가 `releases/vYYYY.MDD.P.md`를 가리키는지 확인합니다

동일한 버전을 다시 배포하지 마세요.

### 안정 배포 후 `latest`가 망가진 경우

dist-tag를 롤백합니다:

```bash
./scripts/rollback-latest.sh YYYY.MDD.P
```

이후 새 안정 릴리스로 수정 사항을 적용합니다.

## 관련 파일

- [`scripts/release.sh`](../scripts/release.sh)
- [`scripts/release-package-map.mjs`](../scripts/release-package-map.mjs)
- [`scripts/create-github-release.sh`](../scripts/create-github-release.sh)
- [`scripts/rollback-latest.sh`](../scripts/rollback-latest.sh)
- [`doc/PUBLISHING.md`](PUBLISHING.md)
- [`doc/RELEASE-AUTOMATION-SETUP.md`](RELEASE-AUTOMATION-SETUP.md)
