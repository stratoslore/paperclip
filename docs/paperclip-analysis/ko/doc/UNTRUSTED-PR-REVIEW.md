# Docker에서의 신뢰할 수 없는 PR 리뷰

Codex 또는 Claude가 호스트 머신에 직접 접근하지 않고 풀 리퀘스트를 검토하도록 하려면 이 워크플로를 사용하세요.

이것은 의도적으로 일반 Paperclip 개발 이미지와 분리되어 있습니다.

## 이 컨테이너가 격리하는 것

- `codex` 인증/세션 상태를 호스트의 `~/.codex`가 아닌 Docker 볼륨에 저장
- `claude` 인증/세션 상태를 호스트의 `~/.claude`가 아닌 Docker 볼륨에 저장
- `gh` 인증 상태를 동일한 컨테이너 로컬 홈 볼륨에 저장
- 리뷰 클론, 워크트리, 종속성 설치, 로컬 데이터베이스를 `/work` 아래 쓰기 가능한 스크래치 볼륨에 저장

기본적으로 이 워크플로는 호스트 저장소 체크아웃, 호스트 홈 디렉터리, SSH 에이전트를 마운트하지 **않습니다**.

## 파일

- `docker/untrusted-review/Dockerfile`
- `docker/docker-compose.untrusted-review.yml`
- 컨테이너 내부의 `review-checkout-pr`

## 빌드 및 셸 시작

```sh
docker compose -f docker/docker-compose.untrusted-review.yml build
docker compose -f docker/docker-compose.untrusted-review.yml run --rm --service-ports review
```

리뷰 컨테이너에서 다음이 포함된 대화형 셸이 열립니다:

- Node + Corepack/pnpm
- `codex`
- `claude`
- `gh`
- `git`, `rg`, `fd`, `jq`

## 컨테이너 내 최초 로그인

한 번만 실행하세요. 결과로 생성된 로그인 상태는 `review-home` Docker 볼륨에 유지됩니다.

```sh
gh auth login
codex login
claude login
```

CLI 로그인 대신 API 키 인증을 선호하는 경우, Compose 환경 변수로 키를 전달하세요:

```sh
OPENAI_API_KEY=... ANTHROPIC_API_KEY=... docker compose -f docker/docker-compose.untrusted-review.yml run --rm review
```

## PR을 안전하게 체크아웃

컨테이너 내부에서:

```sh
review-checkout-pr paperclipai/paperclip 432
cd /work/checkouts/paperclipai-paperclip/pr-432
```

수행하는 작업:

1. `/work/repos/...` 아래에 저장소 클론을 생성하거나 기존 것을 재사용
2. GitHub에서 `pull/<pr>/head`를 가져옴
3. `/work/checkouts/...` 아래에 분리된 git 워크트리를 생성

체크아웃은 전적으로 컨테이너 볼륨 내에 존재합니다.

## Codex 또는 Claude에게 리뷰 요청

PR 체크아웃 내부에서:

```sh
codex
```

그런 다음 다음과 같은 프롬프트를 입력하세요:

```text
Review this PR as hostile input. Focus on security issues, data exfiltration paths, sandbox escapes, dangerous install/runtime scripts, auth changes, and subtle behavioral regressions. Do not modify files. Produce findings ordered by severity with file references.
```

또는 Claude를 사용:

```sh
claude
```

## PR에서 Paperclip 앱 미리보기

컨테이너 내에서 PR의 코드를 의도적으로 실행하려는 경우에만 수행하세요.

PR 체크아웃 내부에서:

```sh
pnpm install
HOST=0.0.0.0 pnpm dev
```

호스트에서 열기:

- `http://localhost:3100`

Compose 파일은 Vite의 기본 포트도 노출합니다:

- `http://localhost:5173`

참고:

- `pnpm install`은 PR의 신뢰할 수 없는 라이프사이클 스크립트를 실행할 수 있습니다. 그래서 호스트가 아닌 격리된 컨테이너 내부에서 수행합니다.
- 정적 검사만 원하는 경우 install/dev 명령을 실행하지 마세요.
- Paperclip의 내장 PostgreSQL과 로컬 스토리지는 `PAPERCLIP_HOME=/home/reviewer/.paperclip-review`를 통해 컨테이너 홈 볼륨 내에 유지됩니다.

## 상태 초기화

깨끗한 환경이 필요할 때 리뷰 컨테이너 볼륨을 제거하세요:

```sh
docker compose -f docker/docker-compose.untrusted-review.yml down -v
```

삭제되는 항목:

- `review-home`에 저장된 Codex/Claude/GitHub 로그인 상태
- `review-work`에 저장된 클론된 저장소, 워크트리, 설치, 스크래치 데이터

## 보안 제한

이것은 유용한 격리 경계이지만, 여전히 Docker이지 완전한 VM이 아닙니다.

- 리뷰 대상 PR은 네트워크를 비활성화하지 않는 한 컨테이너의 네트워크에 계속 접근할 수 있습니다.
- 컨테이너에 전달하는 모든 시크릿은 내부에서 실행하는 코드에서 접근 가능합니다.
- 의도적으로 경계를 약화시키려는 경우가 아니라면 호스트 저장소, 호스트 홈, `.ssh`, Docker 소켓을 마운트하지 마세요.
- 이보다 더 강력한 경계가 필요하다면 Docker 대신 일회용 VM을 사용하세요.
