---
title: 배포 개요
summary: 배포 모드 한눈에 보기
---

Paperclip은 마찰 없는 로컬부터 인터넷 노출 프로덕션까지 세 가지 배포 구성을 지원합니다.

## 배포 모드

| 모드 | 인증 | 적합한 용도 |
|------|------|----------|
| `local_trusted` | 로그인 불필요 | 단일 운영자 로컬 머신 |
| `authenticated` + `private` | 로그인 필요 | 프라이빗 네트워크 (Tailscale, VPN, LAN) |
| `authenticated` + `public` | 로그인 필요 | 인터넷 노출 클라우드 배포 |

## 빠른 비교

### Local Trusted (기본값)

- 루프백 전용 호스트 바인딩 (localhost)
- 사람의 로그인 흐름 없음
- 가장 빠른 로컬 시작
- 적합: 단독 개발 및 실험

### Authenticated + Private

- Better Auth를 통한 로그인 필요
- 네트워크 접근을 위해 모든 인터페이스에 바인딩
- 자동 기본 URL 모드 (더 낮은 마찰)
- 적합: Tailscale 또는 로컬 네트워크를 통한 팀 접근

### Authenticated + Public

- 로그인 필요
- 명시적 공개 URL 필요
- 더 엄격한 보안 검사
- 적합: 클라우드 호스팅, 인터넷 노출 배포

## 모드 선택

- **Paperclip을 시험해보는 중?** `local_trusted` 사용 (기본값)
- **프라이빗 네트워크에서 팀과 공유?** `authenticated` + `private` 사용
- **클라우드에 배포?** `authenticated` + `public` 사용

온보딩 중에 모드를 설정합니다:

```sh
pnpm paperclipai onboard
```

또는 나중에 업데이트합니다:

```sh
pnpm paperclipai configure --section server
```
