# 릴리스 자동화 설정

이 문서는 현재 Paperclip 릴리스 모델에 필요한 GitHub 및 npm 설정을 다룹니다:

- `master`에서의 자동 canary
- 선택된 소스 ref에서의 수동 stable 프로모션
- GitHub OIDC를 통한 npm trusted publishing
- 공개 저장소에서의 보호된 릴리스 인프라

이 설정에 의존하는 저장소 측 파일:

- `.github/workflows/release.yml`
- `.github/CODEOWNERS`

참고:

- 릴리스 워크플로우는 의도적으로 `pnpm install --no-frozen-lockfile`을 사용합니다
- 이는 매니페스트 변경이 `master`에 반영된 후 GitHub 자동화가 `pnpm-lock.yaml`을 갱신하는 저장소의 현재 정책과 일치합니다
- 퍼블리싱 작업은 `scripts/release.sh`를 실행하기 전에 `pnpm-lock.yaml`을 복원하므로 릴리스 스크립트는 여전히 깨끗한 작업 트리를 볼 수 있습니다

## 1. 저장소 변경 사항 먼저 병합하기

GitHub 또는 npm 설정을 변경하기 전에, 참조되는 워크플로우 파일명이 기본 브랜치에 이미 존재하도록 릴리스 자동화 코드를 먼저 병합하세요.

필요한 파일:

- `.github/workflows/release.yml`
- `.github/CODEOWNERS`

## 2. npm Trusted Publishing 구성

Paperclip이 퍼블리싱하는 모든 공개 패키지에 대해 이 작업을 수행하세요.

최소한 다음이 포함됩니다:

- `paperclipai`
- `@paperclipai/server`
- `@paperclipai/ui`
- `packages/` 하위의 공개 패키지

### 2.1. npm에서 각 패키지 설정 페이지 열기

각 패키지에 대해:

1. 패키지 소유자로 npm을 엽니다
2. 패키지 설정 / 퍼블리싱 접근 권한 영역으로 이동합니다
3. GitHub 저장소 `paperclipai/paperclip`에 대한 trusted publisher를 추가합니다

### 2.2. 패키지당 하나의 trusted publisher 항목 추가

npm은 현재 패키지당 하나의 trusted publisher 구성을 허용합니다.

구성 내용:

- workflow: `.github/workflows/release.yml`

저장소:

- `paperclipai/paperclip`

환경 이름:

- npm trusted-publisher 환경 필드는 비워 두세요

이유:

- 단일 `release.yml` 워크플로우가 canary와 stable 퍼블리싱을 모두 처리합니다
- GitHub 환경 `npm-canary`와 `npm-stable`은 여전히 GitHub 측에서 서로 다른 승인 규칙을 적용합니다

### 2.3. 기존 인증 제거 전에 trusted publishing 확인

워크플로우가 가동된 후:

1. canary 퍼블리싱을 실행합니다
2. `NPM_TOKEN` 없이 npm 퍼블리싱이 성공하는지 확인합니다
3. stable dry-run을 실행합니다
4. 실제 stable 퍼블리싱을 한 번 실행합니다

그 후에만 기존 토큰 기반 접근 권한을 제거해야 합니다.

## 3. 레거시 npm 토큰 제거

Trusted publishing이 작동한 후:

1. 퍼블리싱에 사용된 저장소 또는 조직 `NPM_TOKEN` 시크릿을 폐기합니다
2. 이전에 Paperclip을 퍼블리싱하는 데 사용된 개인 자동화 토큰을 폐기합니다
3. npm이 퍼블리싱을 trusted publisher로 제한하는 패키지 수준 설정을 제공하는 경우, 이를 활성화합니다

목표:

- GitHub Actions에 장기 npm 퍼블리싱 토큰이 남아 있어서는 안 됩니다

## 4. GitHub 환경 생성

GitHub 저장소에 두 개의 환경을 생성합니다:

- `npm-canary`
- `npm-stable`

경로:

1. GitHub 저장소
2. `Settings`
3. `Environments`
4. `New environment`

## 5. `npm-canary` 구성

`npm-canary`의 권장 설정:

- 환경 이름: `npm-canary`
- 필수 리뷰어: 없음
- 대기 타이머: 없음
- 배포 브랜치 및 태그:
  - 선택된 브랜치만
  - `master` 허용

이유:

- `master`에 대한 모든 푸시가 자동으로 canary를 퍼블리싱할 수 있어야 합니다
- canary에는 사람의 승인이 필요하지 않아야 합니다

## 6. `npm-stable` 구성

`npm-stable`의 권장 설정:

- 환경 이름: `npm-stable`
- 필수 리뷰어: 가능한 경우 워크플로우를 트리거하는 사람 외에 최소 한 명의 메인테이너
- 셀프 리뷰 방지: 활성화
- 관리자 우회: 팀이 감당할 수 있는 경우 비활성화
- 대기 타이머: 선택 사항
- 배포 브랜치 및 태그:
  - 선택된 브랜치만
  - `master` 허용

이유:

- stable 퍼블리싱은 명시적인 사람의 승인 게이트가 필요합니다
- 워크플로우는 수동이지만, 환경이 여전히 실제 통제 지점이어야 합니다

## 7. `master` 보호

`master`의 브랜치 보호 설정을 엽니다.

권장 규칙:

1. 병합 전 pull request 필수
2. 병합 전 상태 검사 통과 필수
3. 코드 소유자의 리뷰 필수
4. 새 커밋이 푸시되면 이전 승인 해제
5. `master`에 직접 푸시할 수 있는 사람 제한

최소한 워크플로우 및 릴리스 스크립트 변경이 리뷰 없이 반영되지 않도록 하세요.

## 8. CODEOWNERS 리뷰 적용

이 저장소에는 이제 `.github/CODEOWNERS`가 포함되어 있지만, GitHub는 브랜치 보호가 코드 소유자 리뷰를 요구하는 경우에만 이를 적용합니다.

`master`의 브랜치 보호에서 다음을 활성화하세요:

- `Require review from Code Owners`

그런 다음 소유자 항목이 실제 메인테이너 구성과 일치하는지 확인하세요.

현재 파일:

- `.github/CODEOWNERS`

`@cryppadotta`가 공개 저장소에서 올바른 리뷰어 ID가 아닌 경우, 적용을 활성화하기 전에 변경하세요.

## 9. 릴리스 인프라 별도 보호

다음 파일은 항상 코드 소유자 리뷰를 트리거해야 합니다:

- `.github/workflows/release.yml`
- `scripts/release.sh`
- `scripts/release-lib.sh`
- `scripts/release-package-map.mjs`
- `scripts/create-github-release.sh`
- `scripts/rollback-latest.sh`
- `doc/RELEASING.md`
- `doc/PUBLISHING.md`

더 강력한 통제를 원하는 경우, 다음에 대한 직접 푸시를 명시적으로 차단하는 저장소 규칙셋을 추가하세요:

- `.github/workflows/**`
- `scripts/release*`

## 10. GitHub Actions에 Claude 토큰을 저장하지 마세요

자동 변경 로그 생성을 위해 개인 Claude 또는 Anthropic 토큰을 추가하지 마세요.

권장 정책:

- stable 변경 로그 생성은 신뢰할 수 있는 메인테이너 머신에서 로컬로 수행합니다
- canary는 절대 변경 로그를 생성하지 않습니다

이렇게 하면 LLM 지출이 의도적으로 유지되고 고가치 토큰이 Actions에 방치되는 것을 방지합니다.

## 11. Canary 워크플로우 확인

설정 후:

1. `master`에 무해한 커밋을 병합합니다
2. 해당 푸시로 트리거된 `Release` 워크플로우 실행을 엽니다
3. 검증을 통과하는지 확인합니다
4. `npm-canary` 환경에서 퍼블리싱이 성공하는지 확인합니다
5. npm에 새로운 `canary` 릴리스가 표시되는지 확인합니다
6. `canary/vYYYY.MDD.P-canary.N`이라는 이름의 git 태그가 푸시되었는지 확인합니다

설치 경로 확인:

```bash
npx paperclipai@canary onboard
```

## 12. Stable 워크플로우 확인

최소 하나의 정상적인 canary가 존재한 후:

1. `./scripts/release.sh stable --date YYYY-MM-DD --print-version`으로 대상 stable 버전을 확인합니다
2. 프로모션하려는 소스 커밋에 `releases/vYYYY.MDD.P.md`를 준비합니다
3. `Actions` -> `Release`를 엽니다
4. 다음 설정으로 실행합니다:
   - `source_ref`: 테스트된 커밋 SHA 또는 canary 태그 소스 커밋
   - `stable_date`: 비워 두거나 `2026-03-18`과 같은 의도된 UTC 날짜를 설정합니다
     `2026.318.0`과 같은 버전을 입력하지 마세요; 워크플로우가 날짜에서 이를 계산합니다
   - `dry_run`: `true`
5. dry-run이 성공하는지 확인합니다
6. `dry_run: false`로 다시 실행합니다
7. 프롬프트가 나타나면 `npm-stable` 환경을 승인합니다
8. npm `latest`가 새 stable 버전을 가리키는지 확인합니다
9. git 태그 `vYYYY.MDD.P`가 존재하는지 확인합니다
10. GitHub Release가 생성되었는지 확인합니다

구현 참고:

- GitHub Actions stable 워크플로우는 `PUBLISH_REMOTE=origin`으로 `create-github-release.sh`를 호출합니다
- 로컬 메인테이너 사용 시 필요에 따라 `PUBLISH_REMOTE=public-gh`를 명시적으로 전달할 수 있습니다

## 13. 권장 메인테이너 정책

앞으로 이 정책을 사용하세요:

- canary는 자동이고 비용이 적습니다
- stable은 수동이고 승인이 필요합니다
- stable만 공개 노트와 공지를 받습니다
- 릴리스 노트는 stable 퍼블리싱 전에 커밋됩니다
- 롤백은 unpublish가 아닌 `npm dist-tag`를 사용합니다

## 14. 문제 해결

### Trusted publishing이 인증 오류로 실패하는 경우

확인 사항:

1. GitHub의 워크플로우 파일명이 npm에 구성된 파일명과 정확히 일치하는지
2. 패키지에 올바른 저장소에 대한 trusted publisher 항목이 있는지
3. 작업에 `id-token: write`가 있는지
4. 작업이 포크가 아닌 예상된 저장소에서 실행되고 있는지

### Stable 워크플로우가 실행되지만 승인을 요청하지 않는 경우

확인 사항:

1. `publish` 작업이 환경 `npm-stable`을 사용하는지
2. 환경에 실제로 필수 리뷰어가 구성되어 있는지
3. 워크플로우가 포크가 아닌 정식 저장소에서 실행되고 있는지

### CODEOWNERS가 트리거되지 않는 경우

확인 사항:

1. `.github/CODEOWNERS`가 기본 브랜치에 있는지
2. `master`의 브랜치 보호가 코드 소유자 리뷰를 요구하는지
3. 파일의 소유자 ID가 저장소 접근 권한이 있는 유효한 리뷰어인지

## 관련 문서

- [doc/RELEASING.md](RELEASING.md)
- [doc/PUBLISHING.md](PUBLISHING.md)
- [doc/plans/2026-03-17-release-automation-and-versioning.md](plans/2026-03-17-release-automation-and-versioning.md)
