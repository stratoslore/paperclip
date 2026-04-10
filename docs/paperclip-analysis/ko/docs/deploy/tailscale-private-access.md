---
title: Tailscale 프라이빗 접근
summary: Tailscale 친화적 호스트 바인딩으로 Paperclip을 실행하고 다른 디바이스에서 연결하기
---

`localhost`만이 아닌 Tailscale(또는 프라이빗 LAN/VPN)을 통해 Paperclip에 접근하려 할 때 사용합니다.

## 1. 프라이빗 인증 모드로 Paperclip 시작

```sh
pnpm dev --tailscale-auth
```

다음이 구성됩니다:

- `PAPERCLIP_DEPLOYMENT_MODE=authenticated`
- `PAPERCLIP_DEPLOYMENT_EXPOSURE=private`
- `PAPERCLIP_AUTH_BASE_URL_MODE=auto`
- `HOST=0.0.0.0` (모든 인터페이스에 바인딩)

동등한 플래그:

```sh
pnpm dev --authenticated-private
```

## 2. 도달 가능한 Tailscale 주소 찾기

Paperclip을 실행 중인 머신에서:

```sh
tailscale ip -4
```

Tailscale MagicDNS 호스트명(예: `my-macbook.tailnet.ts.net`)도 사용할 수 있습니다.

## 3. 다른 디바이스에서 Paperclip 열기

Tailscale IP 또는 MagicDNS 호스트와 Paperclip 포트를 사용합니다:

```txt
http://<tailscale-host-or-ip>:3100
```

예시:

```txt
http://my-macbook.tailnet.ts.net:3100
```

## 4. 필요 시 사용자 정의 프라이빗 호스트명 허용

사용자 정의 프라이빗 호스트명으로 Paperclip에 접근하는 경우, 허용 목록에 추가하세요:

```sh
pnpm paperclipai allowed-hostname my-macbook.tailnet.ts.net
```

## 5. 서버 도달 가능 여부 확인

원격 Tailscale 연결 디바이스에서:

```sh
curl http://<tailscale-host-or-ip>:3100/api/health
```

예상 결과:

```json
{"status":"ok"}
```

## 문제 해결

- 프라이빗 호스트명에서 로그인 또는 리다이렉트 오류: `paperclipai allowed-hostname`으로 추가하세요.
- 앱이 `localhost`에서만 작동: `--tailscale-auth`로 시작했는지 확인하세요 (또는 프라이빗 모드에서 `HOST=0.0.0.0` 설정).
- 로컬에서는 연결되지만 원격에서는 안 됨: 두 디바이스가 동일한 Tailscale 네트워크에 있고 포트 `3100`이 도달 가능한지 확인하세요.
