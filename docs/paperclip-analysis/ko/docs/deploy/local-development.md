---
title: 로컬 개발
summary: 로컬 개발을 위한 Paperclip 설정
---

외부 의존성 없이 Paperclip을 로컬에서 실행합니다.

## 사전 요구 사항

- Node.js 20+
- pnpm 9+

## 개발 서버 시작

```sh
pnpm install
pnpm dev
```

다음이 시작됩니다:

- **API 서버** -- `http://localhost:3100`
- **UI** -- 개발 미들웨어 모드로 API 서버가 제공 (동일 오리진)

Docker나 외부 데이터베이스가 필요하지 않습니다. Paperclip은 자동으로 임베디드 PostgreSQL을 사용합니다.

## 원커맨드 부트스트랩

최초 설치 시:

```sh
pnpm paperclipai run
```

이 명령은:

1. 구성이 없으면 자동 온보딩
2. 복구 기능 활성화 상태로 `paperclipai doctor` 실행
3. 검사 통과 시 서버 시작

## Tailscale/프라이빗 인증 개발 모드

네트워크 접근을 위해 `authenticated/private` 모드로 실행하려면:

```sh
pnpm dev --tailscale-auth
```

이렇게 하면 프라이빗 네트워크 접근을 위해 서버가 `0.0.0.0`에 바인딩됩니다.

별칭:

```sh
pnpm dev --authenticated-private
```

추가 프라이빗 호스트명 허용:

```sh
pnpm paperclipai allowed-hostname dotta-macbook-pro
```

전체 설정 및 문제 해결은 [Tailscale Private Access](/deploy/tailscale-private-access)를 참조하세요.

## 상태 확인

```sh
curl http://localhost:3100/api/health
# -> {"status":"ok"}

curl http://localhost:3100/api/companies
# -> []
```

## 개발 데이터 리셋

로컬 데이터를 지우고 새로 시작하려면:

```sh
rm -rf ~/.paperclip/instances/default/db
pnpm dev
```

## 데이터 위치

| 데이터 | 경로 |
|------|------|
| 구성 | `~/.paperclip/instances/default/config.json` |
| 데이터베이스 | `~/.paperclip/instances/default/db` |
| 저장소 | `~/.paperclip/instances/default/data/storage` |
| 시크릿 키 | `~/.paperclip/instances/default/secrets/master.key` |
| 로그 | `~/.paperclip/instances/default/logs` |

환경 변수로 재정의:

```sh
PAPERCLIP_HOME=/custom/path PAPERCLIP_INSTANCE_ID=dev pnpm paperclipai run
```
