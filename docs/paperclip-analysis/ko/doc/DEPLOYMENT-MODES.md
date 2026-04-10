# 배포 모드

상태: 정식 배포 및 인증 모드 모델
날짜: 2026-02-23

## 1. 목적

Paperclip은 두 가지 런타임 모드를 지원합니다:

1. `local_trusted`
2. `authenticated`

`authenticated`는 두 가지 노출 정책을 지원합니다:

1. `private`
2. `public`

이를 통해 하나의 인증 스택을 유지하면서 저마찰 프라이빗 네트워크 기본값과 인터넷 노출 강화 요구사항을 분리합니다.

## 2. 정식 모델

| 런타임 모드 | 노출 | 사용자 인증 | 주요 용도 |
|---|---|---|---|
| `local_trusted` | 해당 없음 | 로그인 불필요 | 단일 운영자 로컬 머신 워크플로우 |
| `authenticated` | `private` | 로그인 필요 | 프라이빗 네트워크 접근 (예: Tailscale/VPN/LAN) |
| `authenticated` | `public` | 로그인 필요 | 인터넷 노출/클라우드 배포 |

## 3. 보안 정책

## `local_trusted`

- 루프백 전용 호스트 바인딩
- 사용자 로그인 플로우 없음
- 가장 빠른 로컬 시작에 최적화

## `authenticated + private`

- 로그인 필요
- 저마찰 URL 처리 (`auto` 기본 URL 모드)
- 프라이빗 호스트 신뢰 정책 필요

## `authenticated + public`

- 로그인 필요
- 명시적 공개 URL 필요
- doctor에서 더 엄격한 배포 검사 및 실패

## 4. 온보딩 UX 계약

기본 온보딩은 대화형이며 플래그 없이 유지됩니다:

```sh
pnpm paperclipai onboard
```

서버 프롬프트 동작:

1. 모드를 묻고, 기본값은 `local_trusted`
2. 옵션 설명:
- `local_trusted`: "로컬 설정에 가장 쉬움 (로그인 없음, localhost 전용)"
- `authenticated`: "로그인 필요; 프라이빗 네트워크 또는 공개 호스팅에 사용"
3. `authenticated`인 경우, 노출을 묻습니다:
- `private`: "프라이빗 네트워크 접근 (예: Tailscale), 낮은 설정 마찰"
- `public`: "인터넷 노출 배포, 더 엄격한 보안 요구사항"
4. `authenticated + public`에서만 명시적 공개 URL을 묻습니다

`configure --section server`도 동일한 대화형 동작을 따릅니다.

## 5. Doctor UX 계약

기본 doctor는 플래그 없이 유지됩니다:

```sh
pnpm paperclipai doctor
```

Doctor는 설정된 모드/노출을 읽고 모드별 검사를 적용합니다. 선택적 오버라이드 플래그는 보조적입니다.

## 6. 보드/사용자 통합 계약

보드 ID는 사용자 기반 기능이 일관되게 작동하려면 실제 DB 사용자 주체로 표현되어야 합니다.

필수 통합 포인트:

- 보드 ID를 위한 `authUsers`의 실제 사용자 행
- 보드 관리자 권한을 위한 `instance_user_roles` 항목
- 사용자 수준 작업 할당 및 접근을 위한 `company_memberships` 통합

`assigneeUserId`에 대해 사용자 할당 경로가 활성 멤버십을 검증하므로 이것이 필요합니다.

## 7. Local Trusted -> Authenticated 클레임 플로우

`authenticated` 모드를 실행할 때, 유일한 인스턴스 관리자가 `local-board`인 경우, Paperclip은 일회성 고엔트로피 클레임 URL과 함께 시작 경고를 표시합니다.

- URL 형식: `/board-claim/<token>?code=<code>`
- 용도: 로그인한 사용자가 보드 소유권을 클레임
- 클레임 동작:
  - 현재 로그인한 사용자를 `instance_admin`으로 승격
  - `local-board` 관리자 역할을 강등
  - 클레임하는 사용자에 대해 기존 회사들의 활성 소유자 멤버십을 보장

이를 통해 사용자가 장기 실행된 로컬 신뢰 모드에서 인증 모드로 마이그레이션할 때 잠김을 방지합니다.

## 8. 현재 코드 실상 (2026-02-23 기준)

- 런타임 값은 `local_trusted | authenticated`
- `authenticated`는 Better Auth 세션과 부트스트랩 초대 플로우를 사용
- `local_trusted`는 `instance_user_roles` 관리자 접근이 있는 `authUsers`의 실제 로컬 보드 사용자 주체를 보장
- 회사 생성은 사용자 할당/접근 플로우가 일관되도록 `company_memberships`에서 생성자 멤버십을 보장

## 9. 명명 및 호환성 정책

- 정식 명명은 `local_trusted`와 `authenticated`이며 `private/public` 노출을 포함
- 폐기된 명명 변형에 대한 장기 호환성 별칭 레이어 없음

## 10. 다른 문서와의 관계

- 구현 계획: `doc/plans/deployment-auth-mode-consolidation.md`
- V1 계약: `doc/SPEC-implementation.md`
- 운영자 워크플로우: `doc/DEVELOPING.md` 및 `doc/CLI.md`
