---
title: Docker
summary: Docker Compose 빠른 시작
---

Node나 pnpm을 로컬에 설치하지 않고 Docker에서 Paperclip을 실행합니다.

## Compose 빠른 시작 (권장)

```sh
docker compose -f docker/docker-compose.quickstart.yml up --build
```

[http://localhost:3100](http://localhost:3100)을 엽니다.

기본값:

- 호스트 포트: `3100`
- 데이터 디렉토리: `./data/docker-paperclip`

환경 변수로 재정의:

```sh
PAPERCLIP_PORT=3200 PAPERCLIP_DATA_DIR=../data/pc \
  docker compose -f docker/docker-compose.quickstart.yml up --build
```

**참고:** `PAPERCLIP_DATA_DIR`은 compose 파일(`docker/`)을 기준으로 해석되므로, `../data/pc`는 프로젝트 루트의 `data/pc`에 매핑됩니다.

## 수동 Docker 빌드

```sh
docker build -t paperclip-local .
docker run --name paperclip \
  -p 3100:3100 \
  -e HOST=0.0.0.0 \
  -e PAPERCLIP_HOME=/paperclip \
  -v "$(pwd)/data/docker-paperclip:/paperclip" \
  paperclip-local
```

## 데이터 영속성

모든 데이터는 바인드 마운트(`./data/docker-paperclip`) 아래에 저장됩니다:

- 임베디드 PostgreSQL 데이터
- 업로드된 에셋
- 로컬 시크릿 키
- 에이전트 워크스페이스 데이터

## Docker에서의 Claude 및 Codex 어댑터

Docker 이미지에는 다음이 사전 설치됩니다:

- `claude` (Anthropic Claude Code CLI)
- `codex` (OpenAI Codex CLI)

컨테이너 내에서 로컬 어댑터 실행을 활성화하려면 API 키를 전달하세요:

```sh
docker run --name paperclip \
  -p 3100:3100 \
  -e HOST=0.0.0.0 \
  -e PAPERCLIP_HOME=/paperclip \
  -e OPENAI_API_KEY=sk-... \
  -e ANTHROPIC_API_KEY=sk-... \
  -v "$(pwd)/data/docker-paperclip:/paperclip" \
  paperclip-local
```

API 키 없이도 앱은 정상 실행됩니다 -- 어댑터 환경 검사에서 누락된 사전 요구 사항을 표시합니다.
