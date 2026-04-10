---
title: 배포 모드
summary: local_trusted vs authenticated (private/public)
---

Paperclip은 서로 다른 보안 프로필을 가진 두 가지 런타임 모드를 지원합니다.

## `local_trusted`

기본 모드. 단일 운영자 로컬 사용에 최적화되어 있습니다.

- **호스트 바인딩**: 루프백만 (localhost)
- **인증**: 로그인 불필요
- **사용 사례**: 로컬 개발, 단독 실험
- **이사회 신원**: 자동 생성된 로컬 이사회 사용자

```sh
# Set during onboard
pnpm paperclipai onboard
# Choose "local_trusted"
```

## `authenticated`

로그인 필요. 두 가지 노출 정책을 지원합니다.

### `authenticated` + `private`

프라이빗 네트워크 접근(Tailscale, VPN, LAN)용.

- **인증**: Better Auth를 통한 로그인 필요
- **URL 처리**: 자동 기본 URL 모드 (더 낮은 마찰)
- **호스트 신뢰**: 프라이빗 호스트 신뢰 정책 필요

```sh
pnpm paperclipai onboard
# Choose "authenticated" -> "private"
```

사용자 정의 Tailscale 호스트명 허용:

```sh
pnpm paperclipai allowed-hostname my-machine
```

### `authenticated` + `public`

인터넷 노출 배포용.

- **인증**: 로그인 필요
- **URL**: 명시적 공개 URL 필요
- **보안**: doctor에서 더 엄격한 배포 검사

```sh
pnpm paperclipai onboard
# Choose "authenticated" -> "public"
```

## 이사회 클레임 흐름

`local_trusted`에서 `authenticated`로 마이그레이션할 때, Paperclip은 시작 시 일회성 클레임 URL을 출력합니다:

```
/board-claim/<token>?code=<code>
```

로그인한 사용자가 이 URL을 방문하여 이사회 소유권을 클레임합니다. 이를 통해:

- 현재 사용자를 인스턴스 관리자로 승격
- 자동 생성된 로컬 이사회 관리자를 강등
- 클레임하는 사용자의 활성 회사 멤버십 보장

## 모드 변경

배포 모드를 업데이트합니다:

```sh
pnpm paperclipai configure --section server
```

환경 변수를 통한 런타임 재정의:

```sh
PAPERCLIP_DEPLOYMENT_MODE=authenticated pnpm paperclipai run
```
