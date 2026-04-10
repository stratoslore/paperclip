---
title: Setup Commands
summary: 온보딩, 실행, 진단, 설정
---

인스턴스 설정 및 진단 명령어입니다.

## `paperclipai run`

단일 명령어로 부트스트랩 및 시작:

```sh
pnpm paperclipai run
```

수행 내용:

1. 설정이 없는 경우 자동 온보딩
2. 복구가 활성화된 상태로 `paperclipai doctor` 실행
3. 검사를 통과하면 서버 시작

특정 인스턴스 선택:

```sh
pnpm paperclipai run --instance dev
```

## `paperclipai onboard`

대화형 최초 설정:

```sh
pnpm paperclipai onboard
```

Paperclip이 이미 설정된 경우, `onboard`를 다시 실행하면 기존 설정이 유지됩니다. 기존 설치의 설정을 변경하려면 `paperclipai configure`를 사용하세요.

첫 번째 프롬프트:

1. `Quickstart` (권장): 로컬 기본값 (임베디드 데이터베이스, LLM 공급자 없음, 로컬 디스크 스토리지, 기본 시크릿)
2. `Advanced setup`: 전체 대화형 설정

온보딩 직후 즉시 시작:

```sh
pnpm paperclipai onboard --run
```

비대화형 기본값 + 즉시 시작 (서버 리스닝 시 브라우저 열림):

```sh
pnpm paperclipai onboard --yes
```

기존 설치에서 `--yes`는 현재 설정을 유지하고 해당 설정으로 Paperclip을 시작합니다.

## `paperclipai doctor`

선택적 자동 복구가 포함된 상태 검사:

```sh
pnpm paperclipai doctor
pnpm paperclipai doctor --repair
```

검증 항목:

- 서버 설정
- 데이터베이스 연결
- 시크릿 어댑터 설정
- 스토리지 설정
- 누락된 키 파일

## `paperclipai configure`

설정 섹션 업데이트:

```sh
pnpm paperclipai configure --section server
pnpm paperclipai configure --section secrets
pnpm paperclipai configure --section storage
```

## `paperclipai env`

해석된 환경 설정 표시:

```sh
pnpm paperclipai env
```

## `paperclipai allowed-hostname`

인증/프라이빗 모드에서 프라이빗 호스트명 허용:

```sh
pnpm paperclipai allowed-hostname my-tailscale-host
```

## 로컬 스토리지 경로

| 데이터 | 기본 경로 |
|------|-------------|
| 설정 | `~/.paperclip/instances/default/config.json` |
| 데이터베이스 | `~/.paperclip/instances/default/db` |
| 로그 | `~/.paperclip/instances/default/logs` |
| 스토리지 | `~/.paperclip/instances/default/data/storage` |
| 시크릿 키 | `~/.paperclip/instances/default/secrets/master.key` |

다음으로 재정의:

```sh
PAPERCLIP_HOME=/custom/home PAPERCLIP_INSTANCE_ID=dev pnpm paperclipai run
```

또는 모든 명령어에 `--data-dir`을 직접 전달:

```sh
pnpm paperclipai run --data-dir ./tmp/paperclip-dev
pnpm paperclipai doctor --data-dir ./tmp/paperclip-dev
```
