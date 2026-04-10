# Docker에서 OpenClaw 실행하기 (로컬 개발)

로컬 개발 및 Paperclip OpenClaw 어댑터 통합 테스트를 위해 Docker 컨테이너에서 OpenClaw를 실행하는 방법입니다.

## 자동화된 Join 스모크 테스트 (권장 첫 단계)

Paperclip에는 엔드투엔드 join 스모크 하네스가 포함되어 있습니다:

```bash
pnpm smoke:openclaw-join
```

하네스가 자동화하는 내용:

- 초대 생성 (`allowedJoinTypes=agent`)
- OpenClaw 에이전트 join 요청 (`adapterType=openclaw`)
- 이사회 승인
- 일회성 API 키 클레임 (유효하지 않은/재생 클레임 검사 포함)
- Docker화된 OpenClaw 스타일 웹훅 수신기로의 wakeup 콜백 전달

기본적으로 사전 구성된 Docker 수신기 이미지(`docker/openclaw-smoke`)를 사용하므로 실행이 결정론적이며 수동 OpenClaw 구성 편집이 필요하지 않습니다.

권한 참고:

- 하네스는 이사회가 관리하는 작업(초대 생성, join 승인, 새 에이전트 wakeup)을 수행합니다.
- 인증 모드에서는 이사회/운영자 인증을 제공하거나, 그렇지 않으면 실행이 명시적 권한 오류와 함께 조기 종료됩니다.

## 원커맨드 OpenClaw Gateway UI (수동 Docker 흐름)

Docker에서 OpenClaw를 시작하고 호스트 브라우저 대시보드 URL을 출력하는 하나의 명령:

```bash
pnpm smoke:openclaw-docker-ui
```

기본 동작은 제로 플래그입니다: 페어링 관련 환경 변수 없이 명령을 그대로 실행할 수 있습니다.

이 명령이 하는 일:

- `/tmp/openclaw-docker`에서 `openclaw/openclaw`를 클론/업데이트
- `openclaw:local` 빌드 (`OPENCLAW_BUILD=0`이 아닌 경우)
- `~/.openclaw-paperclip-smoke/openclaw.json`과 Docker `.env`에 격리된 스모크 구성 작성
- 에이전트 모델 기본값을 OpenAI로 고정 (`openai/gpt-5.2` OpenAI 폴백 포함)
- Compose를 통해 `openclaw-gateway` 시작 (필수 `/tmp` tmpfs 재정의 포함)
- OpenClaw Docker 내부에서 도달 가능한 Paperclip 호스트 URL을 프로빙 및 출력
- 상태 확인 후 다음을 출력:
  - `http://127.0.0.1:18789/#token=...`
- 로컬 스모크 편의를 위해 기본적으로 Control UI 디바이스 페어링 비활성화

환경 설정:

- `OPENAI_API_KEY` (필수; 환경 또는 `~/.secrets`에서 로드)
- `OPENCLAW_DOCKER_DIR` (기본값 `/tmp/openclaw-docker`)
- `OPENCLAW_GATEWAY_PORT` (기본값 `18789`)
- `OPENCLAW_GATEWAY_TOKEN` (기본값 랜덤)
- `OPENCLAW_BUILD=0` -- 재빌드 건너뛰기
- `OPENCLAW_OPEN_BROWSER=1` -- macOS에서 URL 자동 열기
- `OPENCLAW_DISABLE_DEVICE_AUTH=1` (기본값) -- 로컬 스모크용 Control UI 디바이스 페어링 비활성화
- `OPENCLAW_DISABLE_DEVICE_AUTH=0` -- 페어링 활성화 유지 (그런 다음 `devices` CLI 명령으로 브라우저 승인)
- `OPENCLAW_MODEL_PRIMARY` (기본값 `openai/gpt-5.2`)
- `OPENCLAW_MODEL_FALLBACK` (기본값 `openai/gpt-5.2-chat-latest`)
- `OPENCLAW_CONFIG_DIR` (기본값 `~/.openclaw-paperclip-smoke`)
- `OPENCLAW_RESET_STATE=1` (기본값) -- 오래된 인증/세션 드리프트를 방지하기 위해 매 실행마다 스모크 에이전트 상태 리셋
- `PAPERCLIP_HOST_PORT` (기본값 `3100`)
- `PAPERCLIP_HOST_FROM_CONTAINER` (기본값 `host.docker.internal`)

### 인증 모드

Paperclip 배포가 `authenticated`인 경우, 인증 컨텍스트를 제공하세요:

```bash
PAPERCLIP_AUTH_HEADER="Bearer <token>" pnpm smoke:openclaw-join
# or
PAPERCLIP_COOKIE="your_session_cookie=..." pnpm smoke:openclaw-join
```

### 네트워크 토폴로지 팁

- 로컬 동일 호스트 스모크: 기본 콜백은 `http://127.0.0.1:<port>/webhook`을 사용합니다.
- OpenClaw Docker 내부에서 `127.0.0.1`은 호스트 Paperclip 서버가 아닌 컨테이너 자체를 가리킵니다.
- Docker 내 OpenClaw가 사용하는 초대/온보딩 URL에는 스크립트가 출력한 Paperclip URL(일반적으로 `http://host.docker.internal:3100`)을 사용하세요.
- Paperclip이 컨테이너에서 보이는 호스트를 호스트명 오류로 거부하면 호스트에서 허용하세요:

```bash
pnpm paperclipai allowed-hostname host.docker.internal
```

그런 다음 Paperclip을 재시작하고 스모크 스크립트를 다시 실행하세요.
- Docker/원격 OpenClaw: 도달 가능한 호스트명(Docker 호스트 별칭, Tailscale 호스트명 또는 공개 도메인)을 선호합니다.
- 인증/비공개 모드: 필요한 경우 호스트명이 허용 목록에 있는지 확인하세요:

```bash
pnpm paperclipai allowed-hostname <host>
```

## 사전 요구 사항

- **Docker Desktop v29+** (Docker Sandbox 지원 포함)
- **2 GB+ RAM** -- Docker 이미지 빌드에 필요
- **API 키** -- `~/.secrets`에 (최소 `OPENAI_API_KEY`)

## 옵션 A: Docker Sandbox (권장)

Docker Sandbox는 Docker Compose보다 더 나은 격리(마이크로VM 기반)와 더 간단한 설정을 제공합니다. Docker Desktop v29+ / Docker Sandbox v0.12+ 필요.

```bash
# 1. Clone the OpenClaw repo and build the image
git clone https://github.com/openclaw/openclaw.git /tmp/openclaw-docker
cd /tmp/openclaw-docker
docker build -t openclaw:local -f Dockerfile .

# 2. Create the sandbox using the built image
docker sandbox create --name openclaw -t openclaw:local shell ~/.openclaw/workspace

# 3. Allow network access to OpenAI API
docker sandbox network proxy openclaw \
  --allow-host api.openai.com \
  --allow-host localhost

# 4. Write the config inside the sandbox
docker sandbox exec openclaw sh -c '
mkdir -p /home/node/.openclaw/workspace /home/node/.openclaw/identity /home/node/.openclaw/credentials
cat > /home/node/.openclaw/openclaw.json << INNEREOF
{
  "gateway": {
    "mode": "local",
    "port": 18789,
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "sandbox-dev-token-12345"
    },
    "controlUi": { "enabled": true }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai/gpt-5.2",
        "fallbacks": ["openai/gpt-5.2-chat-latest"]
      },
      "workspace": "/home/node/.openclaw/workspace"
    }
  }
}
INNEREOF
chmod 600 /home/node/.openclaw/openclaw.json
'

# 5. Start the gateway (pass your API key from ~/.secrets)
source ~/.secrets
docker sandbox exec -d \
  -e OPENAI_API_KEY="$OPENAI_API_KEY" \
  -w /app openclaw \
  node dist/index.js gateway --bind loopback --port 18789

# 6. Wait ~15 seconds, then verify
sleep 15
docker sandbox exec openclaw curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
# Should print: 200

# 7. Check status
docker sandbox exec -e OPENAI_API_KEY="$OPENAI_API_KEY" -w /app openclaw \
  node dist/index.js status
```

### Sandbox 관리

```bash
# List sandboxes
docker sandbox ls

# Shell into the sandbox
docker sandbox exec -it openclaw bash

# Stop the sandbox (preserves state)
docker sandbox stop openclaw

# Remove the sandbox
docker sandbox rm openclaw

# Check sandbox version
docker sandbox version
```

## 옵션 B: Docker Compose (폴백)

Docker Sandbox를 사용할 수 없는 경우(Docker Desktop < v29) 이 방법을 사용하세요.

```bash
# 1. Clone the OpenClaw repo
git clone https://github.com/openclaw/openclaw.git /tmp/openclaw-docker
cd /tmp/openclaw-docker

# 2. Build the Docker image (~5-10 min on first run)
docker build -t openclaw:local -f Dockerfile .

# 3. Create config directories
mkdir -p ~/.openclaw/workspace ~/.openclaw/identity ~/.openclaw/credentials
chmod 700 ~/.openclaw ~/.openclaw/credentials

# 4. Generate a gateway token
export OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)
echo "Your gateway token: $OPENCLAW_GATEWAY_TOKEN"

# 5. Create the config file
cat > ~/.openclaw/openclaw.json << EOF
{
  "gateway": {
    "mode": "local",
    "port": 18789,
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "$OPENCLAW_GATEWAY_TOKEN"
    },
    "controlUi": {
      "enabled": true,
      "allowedOrigins": ["http://127.0.0.1:18789"]
    }
  },
  "env": {
    "OPENAI_API_KEY": "\${OPENAI_API_KEY}"
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai/gpt-5.2",
        "fallbacks": ["openai/gpt-5.2-chat-latest"]
      },
      "workspace": "/home/node/.openclaw/workspace"
    }
  }
}
EOF
chmod 600 ~/.openclaw/openclaw.json

# 6. Create the .env file (load API keys from ~/.secrets)
source ~/.secrets
cat > .env << EOF
OPENCLAW_CONFIG_DIR=$HOME/.openclaw
OPENCLAW_WORKSPACE_DIR=$HOME/.openclaw/workspace
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_BRIDGE_PORT=18790
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_TOKEN=$OPENCLAW_GATEWAY_TOKEN
OPENCLAW_IMAGE=openclaw:local
OPENAI_API_KEY=$OPENAI_API_KEY
OPENCLAW_EXTRA_MOUNTS=
OPENCLAW_HOME_VOLUME=
OPENCLAW_DOCKER_APT_PACKAGES=
EOF

# 7. Add tmpfs to docker-compose.yml (required — see Known Issues)
# Add to BOTH openclaw-gateway and openclaw-cli services:
#   tmpfs:
#     - /tmp:exec,size=512M

# 8. Start the gateway
docker compose up -d openclaw-gateway

# 9. Wait ~15 seconds for startup, then get the dashboard URL
sleep 15
docker compose run --rm openclaw-cli dashboard --no-open
```

대시보드 URL은 다음과 같은 형태입니다: `http://127.0.0.1:18789/#token=<your-token>`

### Docker Compose 관리

```bash
cd /tmp/openclaw-docker

# Stop
docker compose down

# Start again (no rebuild needed)
docker compose up -d openclaw-gateway

# View logs
docker compose logs -f openclaw-gateway

# Check status
docker compose run --rm openclaw-cli status

# Get dashboard URL
docker compose run --rm openclaw-cli dashboard --no-open
```

## 알려진 문제 및 해결 방법

### 컨테이너 시작 시 "no space left on device"

Docker Desktop의 가상 디스크가 가득 찼을 수 있습니다.

```bash
docker system df                   # check usage
docker system prune -f             # remove stopped containers, unused networks
docker image prune -f              # remove dangling images
```

### "Unable to create fallback OpenClaw temp dir: /tmp/openclaw-1000" (Compose만 해당)

컨테이너가 `/tmp`에 쓸 수 없습니다. `docker-compose.yml`에서 **두 서비스 모두**에 `tmpfs` 마운트를 추가하세요:

```yaml
services:
  openclaw-gateway:
    tmpfs:
      - /tmp:exec,size=512M
  openclaw-cli:
    tmpfs:
      - /tmp:exec,size=512M
```

이 문제는 Docker Sandbox 방식에는 영향을 미치지 않습니다.

### 커뮤니티 템플릿 이미지의 Node 버전 불일치

일부 커뮤니티 빌드 sandbox 템플릿(예: `olegselajev241/openclaw-dmr:latest`)은 Node 20을 포함하지만, OpenClaw는 Node >=22.12.0이 필요합니다. Node 22를 포함하는 로컬 빌드 `openclaw:local` 이미지를 sandbox 템플릿으로 사용하세요.

### Gateway 시작 후 ~15초간 응답 없음

Node.js gateway가 초기화하는 데 시간이 필요합니다. `http://127.0.0.1:18789/`에 접근하기 전 15초를 기다리세요.

### CLAUDE_AI_SESSION_KEY 경고 (Compose만 해당)

이 Docker Compose 경고는 무해하며 무시할 수 있습니다:
```
level=warning msg="The \"CLAUDE_AI_SESSION_KEY\" variable is not set. Defaulting to a blank string."
```

## 구성

구성 파일: `~/.openclaw/openclaw.json` (JSON5 형식)

주요 설정:
- `gateway.auth.token` -- 웹 UI 및 API용 인증 토큰
- `agents.defaults.model.primary` -- AI 모델 (`openai/gpt-5.2` 이상 사용)
- `env.OPENAI_API_KEY` -- `OPENAI_API_KEY` 환경 변수 참조 (Compose 방식)

API 키는 `~/.secrets`에 저장되며 환경 변수를 통해 컨테이너에 전달됩니다.

## 참고 자료

- [OpenClaw Docker docs](https://docs.openclaw.ai/install/docker)
- [OpenClaw Configuration Reference](https://docs.openclaw.ai/gateway/configuration-reference)
- [Docker blog: Run OpenClaw Securely in Docker Sandboxes](https://www.docker.com/blog/run-openclaw-securely-in-docker-sandboxes/)
- [Docker Sandbox docs](https://docs.docker.com/ai/sandboxes)
- [OpenAI Models](https://platform.openai.com/docs/models) -- 현재 모델: gpt-5.2, gpt-5.2-chat-latest, gpt-5.2-pro
