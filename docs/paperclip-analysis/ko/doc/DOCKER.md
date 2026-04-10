# Docker 퀵스타트

Node나 pnpm을 로컬에 설치하지 않고 Docker에서 Paperclip을 실행합니다.

아래 모든 명령은 `docker/` 내부가 아닌 **프로젝트 루트** (`package.json`이 포함된 디렉토리)에서 실행한다고 가정합니다.

## 이미지 빌드

```sh
docker build -t paperclip-local .
```

Dockerfile은 일반 에이전트 도구(`git`, `gh`, `curl`, `wget`, `ripgrep`, `python3`)와 Claude, Codex, OpenCode CLI를 설치합니다.

빌드 인수:

| 인수 | 기본값 | 용도 |
|-----|---------|---------|
| `USER_UID` | `1000` | 컨테이너 `node` 사용자의 UID (바인드 마운트의 권한 문제를 피하려면 호스트 UID와 일치시킵니다) |
| `USER_GID` | `1000` | 컨테이너 `node` 그룹의 GID |

```sh
docker build -t paperclip-local \
  --build-arg USER_UID=$(id -u) --build-arg USER_GID=$(id -g) .
```

## 한 줄 실행 (빌드 + 실행)

```sh
docker build -t paperclip-local . && \
docker run --name paperclip \
  -p 3100:3100 \
  -e HOST=0.0.0.0 \
  -e PAPERCLIP_HOME=/paperclip \
  -e BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
  -v "$(pwd)/data/docker-paperclip:/paperclip" \
  paperclip-local
```

열기: `http://localhost:3100`

데이터 영속성:

- 내장 PostgreSQL 데이터
- 업로드된 자산
- 로컬 비밀 키
- 로컬 에이전트 워크스페이스 데이터

모두 바인드 마운트(위 예시의 `./data/docker-paperclip`) 아래에 영속됩니다.

## Docker Compose

### 퀵스타트 (내장 SQLite)

단일 컨테이너, 외부 데이터베이스 없음. 바인드 마운트를 통해 데이터가 영속됩니다.

```sh
BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
  docker compose -f docker/docker-compose.quickstart.yml up --build
```

기본값:

- 호스트 포트: `3100`
- 영속 데이터 디렉토리: `./data/docker-paperclip`

선택적 오버라이드:

```sh
PAPERCLIP_PORT=3200 PAPERCLIP_DATA_DIR=../data/pc \
  docker compose -f docker/docker-compose.quickstart.yml up --build
```

**참고:** `PAPERCLIP_DATA_DIR`은 compose 파일(`docker/`) 기준으로 확인되므로, `../data/pc`는 프로젝트 루트의 `data/pc`에 매핑됩니다.

호스트 포트를 변경하거나 비로컬 도메인을 사용하는 경우, `PAPERCLIP_PUBLIC_URL`을 브라우저/인증 플로우에서 사용할 외부 URL로 설정하세요.

로컬 어댑터 실행을 활성화하려면 `OPENAI_API_KEY` 및/또는 `ANTHROPIC_API_KEY`를 전달합니다.

### 풀 스택 (PostgreSQL 포함)

Paperclip 서버 + PostgreSQL 17. 데이터베이스는 서버 시작 전에 헬스 체크됩니다.

```sh
BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
  docker compose -f docker/docker-compose.yml up --build
```

PostgreSQL 데이터는 명명된 Docker 볼륨(`pgdata`)에 영속됩니다. Paperclip 데이터는 `paperclip-data`에 영속됩니다.

### 비신뢰 PR 리뷰

호스트 머신을 노출하지 않고 Codex 또는 Claude로 비신뢰 풀 리퀘스트를 리뷰하기 위한 격리된 컨테이너. 전체 워크플로우는 `doc/UNTRUSTED-PR-REVIEW.md`를 참조하세요.

```sh
docker compose -f docker/docker-compose.untrusted-review.yml build
docker compose -f docker/docker-compose.untrusted-review.yml run --rm --service-ports review
```

## 인증 Compose (단일 공개 URL)

인증된 배포의 경우, 하나의 정식 공개 URL을 설정하고 Paperclip이 인증/콜백 기본값을 파생하도록 합니다:

```yaml
services:
  paperclip:
    environment:
      PAPERCLIP_DEPLOYMENT_MODE: authenticated
      PAPERCLIP_DEPLOYMENT_EXPOSURE: private
      PAPERCLIP_PUBLIC_URL: https://desk.koker.net
```

`PAPERCLIP_PUBLIC_URL`은 다음의 기본 소스로 사용됩니다:

- 인증 공개 기본 URL
- Better Auth 기본 URL 기본값
- 부트스트랩 초대 URL 기본값
- 호스트명 허용 목록 기본값 (URL에서 호스트명 추출)

필요한 경우 세부 오버라이드 사용 가능 (`PAPERCLIP_AUTH_PUBLIC_BASE_URL`, `BETTER_AUTH_URL`, `BETTER_AUTH_TRUSTED_ORIGINS`, `PAPERCLIP_ALLOWED_HOSTNAMES`).

공개 URL 호스트 외에 추가 호스트명이 필요한 경우(예: Tailscale/LAN 별칭 또는 다중 프라이빗 호스트명)에만 `PAPERCLIP_ALLOWED_HOSTNAMES`를 명시적으로 설정하세요.

## Docker에서 Claude + Codex 로컬 어댑터

이미지에 사전 설치:

- `claude` (Anthropic Claude Code CLI)
- `codex` (OpenAI Codex CLI)

컨테이너 내에서 로컬 어댑터를 실행하려면 컨테이너 시작 시 API 키를 전달합니다:

```sh
docker run --name paperclip \
  -p 3100:3100 \
  -e HOST=0.0.0.0 \
  -e PAPERCLIP_HOME=/paperclip \
  -e OPENAI_API_KEY=... \
  -e ANTHROPIC_API_KEY=... \
  -v "$(pwd)/data/docker-paperclip:/paperclip" \
  paperclip-local
```

참고:

- API 키 없이도 앱은 정상적으로 실행됩니다.
- Paperclip의 어댑터 환경 검사가 누락된 인증/CLI 전제조건을 표시합니다.

## Podman Quadlet (systemd)

`docker/quadlet/` 디렉토리에는 Podman Quadlet을 통해 Paperclip + PostgreSQL을 systemd 서비스로 실행하기 위한 유닛 파일이 포함되어 있습니다.

| 파일 | 용도 |
|------|---------|
| `docker/quadlet/paperclip.pod` | Pod 정의 — 컨테이너를 공유 네트워크 네임스페이스로 그룹화 |
| `docker/quadlet/paperclip.container` | Paperclip 서버 — pod에 참여, `127.0.0.1`에서 Postgres에 연결 |
| `docker/quadlet/paperclip-db.container` | PostgreSQL 17 — pod에 참여, 헬스 체크 |

### 설정

1. 이미지를 빌드합니다 (위 참조).

2. quadlet 파일을 systemd 디렉토리에 복사합니다:

   ```sh
   # Rootless (권장)
   cp docker/quadlet/*.pod docker/quadlet/*.container \
     ~/.config/containers/systemd/

   # 또는 rootful
   sudo cp docker/quadlet/*.pod docker/quadlet/*.container \
     /etc/containers/systemd/
   ```

3. 비밀 env 파일을 생성합니다 (버전 관리에서 제외):

   ```sh
   cat > ~/.config/containers/systemd/paperclip.env <<EOL
   BETTER_AUTH_SECRET=$(openssl rand -hex 32)
   POSTGRES_USER=paperclip
   POSTGRES_PASSWORD=paperclip
   POSTGRES_DB=paperclip
   DATABASE_URL=postgres://paperclip:paperclip@127.0.0.1:5432/paperclip
   # OPENAI_API_KEY=sk-...
   # ANTHROPIC_API_KEY=sk-...
   EOL
   ```

4. 데이터 디렉토리를 생성하고 시작합니다:

   ```sh
   mkdir -p ~/.local/share/paperclip
   systemctl --user daemon-reload
   systemctl --user start paperclip-pod
   ```

### Quadlet 관리

```sh
journalctl --user -u paperclip -f        # 앱 로그
journalctl --user -u paperclip-db -f     # DB 로그
systemctl --user status paperclip-pod    # Pod 상태
systemctl --user restart paperclip-pod   # 전체 재시작
systemctl --user stop paperclip-pod      # 전체 중지
```

### Quadlet 참고

- **첫 부팅**: Docker Compose의 `condition: service_healthy`와 달리, Quadlet의 `After=`는 DB 유닛이 *시작*되기만을 기다리고 PostgreSQL이 준비될 때까지 기다리지 않습니다. 콜드 첫 부팅에서 PostgreSQL이 초기화되는 동안 `journalctl --user -u paperclip`에서 한두 번의 재시작 시도가 보일 수 있습니다 — 이는 예상된 것이며 `Restart=on-failure`를 통해 자동으로 해결됩니다.
- pod의 컨테이너는 `localhost`를 공유하므로, Paperclip은 `127.0.0.1:5432`에서 Postgres에 도달합니다.
- PostgreSQL 데이터는 `paperclip-pgdata` 명명된 볼륨에 영속됩니다.
- Paperclip 데이터는 `~/.local/share/paperclip`에 영속됩니다.
- rootful quadlet 배포의 경우, `%h` 접두사를 제거하고 절대 경로를 사용합니다.

## 온보드 스모크 테스트 (Ubuntu + npm만)

Ubuntu + npm만 있는 깨끗한 머신을 흉내내서 다음을 검증하고 싶을 때 사용합니다:

- `npx paperclipai onboard --yes`가 완료됨
- 서버가 `0.0.0.0:3100`에 바인딩되어 호스트 접근이 가능
- 온보드/실행 배너와 시작 로그가 터미널에 표시됨

빌드 + 실행:

```sh
./scripts/docker-onboard-smoke.sh
```

열기: `http://localhost:3131` (기본 스모크 호스트 포트)

유용한 오버라이드:

```sh
HOST_PORT=3200 PAPERCLIPAI_VERSION=latest ./scripts/docker-onboard-smoke.sh
PAPERCLIP_DEPLOYMENT_MODE=authenticated PAPERCLIP_DEPLOYMENT_EXPOSURE=private ./scripts/docker-onboard-smoke.sh
SMOKE_DETACH=true SMOKE_METADATA_FILE=/tmp/paperclip-smoke.env PAPERCLIPAI_VERSION=latest ./scripts/docker-onboard-smoke.sh
```

참고:

- 기본적으로 `./data/docker-onboard-smoke`에 영속 데이터가 마운트됩니다.
- 컨테이너 런타임 사용자 id는 기본적으로 로컬 `id -u`로 설정되어 마운트된 데이터 디렉토리가 루트 런타임을 피하면서 쓰기 가능합니다.
- 스모크 스크립트는 기본적으로 `authenticated/private` 모드로 설정되어 `HOST=0.0.0.0`이 호스트에 노출될 수 있습니다.
- 스모크 스크립트는 기본 호스트 포트를 `3131`로 설정하여 `3100`의 로컬 Paperclip과의 충돌을 피합니다.
- 스모크 스크립트는 또한 `PAPERCLIP_PUBLIC_URL`을 `http://localhost:<HOST_PORT>`로 기본 설정하여 부트스트랩 초대 URL과 인증 콜백이 컨테이너 내부의 `3100` 대신 접근 가능한 호스트 포트를 사용합니다.
- 인증 모드에서 스모크 스크립트는 기본적으로 `SMOKE_AUTO_BOOTSTRAP=true`로 설정하고 실제 부트스트랩 경로를 자동으로 수행합니다: 실제 사용자를 가입시키고, 컨테이너 내에서 `paperclipai auth bootstrap-ceo`를 실행하여 실제 부트스트랩 초대를 발행하고, HTTP를 통해 해당 초대를 수락하고, 보드 세션 접근을 검증합니다.
- 온보딩 플로우를 확인하려면 포그라운드에서 스크립트를 실행하고, 검증 후 `Ctrl+C`로 중지합니다.
- 자동화를 위해 컨테이너를 실행 상태로 두려면 `SMOKE_DETACH=true`로 설정하고 선택적으로 셸 준비 메타데이터를 `SMOKE_METADATA_FILE`에 기록합니다.
- 이미지 정의는 `docker/Dockerfile.onboard-smoke`에 있습니다.

## 일반 참고

- `docker-entrypoint.sh`는 시작 시 컨테이너 `node` 사용자의 UID/GID를 `USER_UID`/`USER_GID`를 통해 전달된 값에 맞게 조정하여 바인드 마운트된 볼륨의 권한 문제를 방지합니다.
- Paperclip 데이터는 Docker 볼륨/바인드 마운트(compose) 또는 `~/.local/share/paperclip`(quadlet)을 통해 영속됩니다.
