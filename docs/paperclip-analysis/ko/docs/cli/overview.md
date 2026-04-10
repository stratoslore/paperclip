---
title: CLI Overview
summary: CLI 설치 및 설정
---

Paperclip CLI는 인스턴스 설정, 진단, 컨트롤 플레인 작업을 처리합니다.

## 사용법

```sh
pnpm paperclipai --help
```

## 글로벌 옵션

모든 명령어에서 지원됩니다:

| 플래그 | 설명 |
|------|-------------|
| `--data-dir <path>` | 로컬 Paperclip 데이터 루트 (`~/.paperclip`와 격리) |
| `--api-base <url>` | API 기본 URL |
| `--api-key <token>` | API 인증 토큰 |
| `--context <path>` | 컨텍스트 파일 경로 |
| `--profile <name>` | 컨텍스트 프로필 이름 |
| `--json` | JSON으로 출력 |

회사 범위 명령어는 `--company-id <id>`도 지원합니다.

깨끗한 로컬 인스턴스의 경우, 실행하는 명령어에 `--data-dir`을 전달하세요:

```sh
pnpm paperclipai run --data-dir ./tmp/paperclip-dev
```

## 컨텍스트 프로필

플래그 반복을 피하기 위해 기본값을 저장합니다:

```sh
# 기본값 설정
pnpm paperclipai context set --api-base http://localhost:3100 --company-id <id>

# 현재 컨텍스트 조회
pnpm paperclipai context show

# 프로필 목록 조회
pnpm paperclipai context list

# 프로필 전환
pnpm paperclipai context use default
```

컨텍스트에 시크릿 저장을 피하려면 환경 변수를 사용하세요:

```sh
pnpm paperclipai context set --api-key-env-var-name PAPERCLIP_API_KEY
export PAPERCLIP_API_KEY=...
```

컨텍스트는 `~/.paperclip/context.json`에 저장됩니다.

## 명령어 카테고리

CLI에는 두 가지 카테고리가 있습니다:

1. **[설정 명령어](/cli/setup-commands)** -- 인스턴스 부트스트랩, 진단, 설정
2. **[컨트롤 플레인 명령어](/cli/control-plane-commands)** -- 이슈, 에이전트, 승인, 활동
