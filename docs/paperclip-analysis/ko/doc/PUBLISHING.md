# npm에 퍼블리싱하기

Paperclip 패키지를 준비하고 npm에 퍼블리싱하는 방법에 대한 저수준 참조 문서입니다.

메인테이너 워크플로우는 [doc/RELEASING.md](RELEASING.md)를 참고하세요. 이 문서는 패키징 내부 구조에 초점을 맞추고 있습니다.

## 현재 릴리스 진입점

다음 스크립트를 사용하세요:

- [`scripts/release.sh`](../scripts/release.sh) - canary 및 stable 퍼블리싱 플로우용
- [`scripts/create-github-release.sh`](../scripts/create-github-release.sh) - stable 태그를 푸시한 후 사용
- [`scripts/rollback-latest.sh`](../scripts/rollback-latest.sh) - `latest`를 이전 버전으로 되돌릴 때 사용
- [`scripts/build-npm.sh`](../scripts/build-npm.sh) - CLI 패키징 빌드용

Paperclip은 더 이상 릴리스 브랜치나 Changesets를 퍼블리싱에 사용하지 않습니다.

## CLI에 특별한 패키징이 필요한 이유

CLI 패키지 `paperclipai`는 다음과 같은 워크스페이스 패키지의 코드를 임포트합니다:

- `@paperclipai/server`
- `@paperclipai/db`
- `@paperclipai/shared`
- `packages/adapters/` 하위의 어댑터 패키지들

이러한 워크스페이스 참조는 개발 환경에서는 유효하지만, 퍼블리싱 가능한 npm 패키지에서는 사용할 수 없습니다. 릴리스 플로우에서 버전을 임시로 재작성한 후, 퍼블리싱 가능한 CLI 번들을 빌드합니다.

## `build-npm.sh`

실행 방법:

```bash
./scripts/build-npm.sh
```

이 스크립트는 다음을 수행합니다:

1. `--skip-checks`가 지정되지 않은 경우 금지된 토큰 검사를 실행합니다
2. `pnpm -r typecheck`를 실행합니다
3. esbuild로 CLI 진입점을 `cli/dist/index.js`에 번들링합니다
4. `node --check`로 번들된 진입점을 검증합니다
5. `cli/package.json`을 퍼블리싱 가능한 npm 매니페스트로 재작성하고 개발용 사본을 `cli/package.dev.json`으로 저장합니다
6. npm 메타데이터를 위해 저장소의 `README.md`를 `cli/README.md`로 복사합니다

릴리스 스크립트가 종료된 후, 개발용 매니페스트와 임시 파일은 자동으로 복원됩니다.

## 패키지 탐색 및 버전 관리

공개 패키지는 다음 경로에서 탐색됩니다:

- `packages/`
- `server/`
- `ui/`
- `cli/`

버전 재작성 단계는 이제 [`scripts/release-package-map.mjs`](../scripts/release-package-map.mjs)를 사용하며, 이 스크립트는 다음을 수행합니다:

- 모든 공개 패키지를 찾습니다
- 내부 의존성을 기반으로 토폴로지 정렬합니다
- 각 패키지 버전을 대상 릴리스 버전으로 재작성합니다
- 내부 `workspace:*` 의존성 참조를 정확한 대상 버전으로 재작성합니다
- CLI의 표시 버전 문자열을 업데이트합니다

이러한 재작성은 임시적입니다. 퍼블리싱 또는 dry-run 후에 작업 트리가 복원됩니다.

## `@paperclipai/ui` 패키징

UI 패키지는 소스 워크스페이스가 아닌 사전 빌드된 정적 에셋을 퍼블리싱합니다.

`ui` 패키지는 `prepack` 단계에서 [`scripts/generate-ui-package-json.mjs`](../scripts/generate-ui-package-json.mjs)를 사용하여 간소화된 퍼블리싱 매니페스트로 교체합니다. 이 매니페스트는:

- 릴리스 관리 대상인 `name`과 `version`을 유지합니다
- `dist/`만 퍼블리싱합니다
- 다운스트림 설치 시 소스 전용 의존성 그래프를 생략합니다

패킹 또는 퍼블리싱 후, `postpack`이 자동으로 개발용 매니페스트를 복원합니다.

### `@paperclipai/ui` 수동 최초 퍼블리싱

UI 패키지만 수동으로 한 번 퍼블리싱해야 하는 경우, 실제 패키지 이름을 사용하세요:

- `@paperclipai/ui`

저장소 루트에서의 권장 플로우:

```bash
# 선택적 사전 확인: 최초 퍼블리싱 전까지 404가 반환됩니다
npm view @paperclipai/ui version

# dist 페이로드가 최신 상태인지 확인합니다
pnpm --filter @paperclipai/ui build

# 실제 퍼블리싱 전에 로컬 npm 인증을 확인합니다
npm whoami

# 정확한 퍼블리싱 페이로드의 안전한 미리보기
cd ui
pnpm publish --dry-run --no-git-checks --access public

# 실제 퍼블리싱
pnpm publish --no-git-checks --access public
```

참고:

- `ui/`에서 퍼블리싱하세요. 저장소 루트에서 하지 마세요.
- `prepack`이 자동으로 `ui/package.json`을 간소화된 퍼블리싱 매니페스트로 재작성하고, `postpack`이 명령 완료 후 개발용 매니페스트를 복원합니다.
- `npm view @paperclipai/ui version`이 이미 [`ui/package.json`](../ui/package.json)에 있는 것과 동일한 버전을 반환하는 경우, 다시 퍼블리싱하지 마세요. 버전을 올리거나 [`scripts/release.sh`](../scripts/release.sh)의 일반적인 저장소 전체 릴리스 플로우를 사용하세요.

최초 실제 퍼블리싱에서 npm `E404`가 반환되면, 재시도하기 전에 npm 측 사전 조건을 확인하세요:

- `npm whoami`가 먼저 성공해야 합니다. 만료되었거나 누락된 npm 로그인은 퍼블리싱을 차단합니다.
- `@paperclipai/ui`와 같은 조직 범위 패키지의 경우, `paperclipai` npm 조직이 존재해야 하며 퍼블리셔가 해당 범위에 퍼블리싱할 권한이 있는 멤버여야 합니다.
- 공개 범위 패키지의 최초 퍼블리싱에는 `--access public`이 포함되어야 합니다.
- npm은 퍼블리싱을 위한 계정 2FA 또는 2FA를 우회할 수 있는 세분화된 토큰이 필요합니다.

### `@paperclipai/mcp-server` 수동 최초 퍼블리싱

MCP 서버 패키지만 수동으로 한 번 퍼블리싱해야 하는 경우, 다음을 사용하세요:

- `@paperclipai/mcp-server`

저장소 루트에서의 권장 플로우:

```bash
# 선택적 사전 확인: 최초 퍼블리싱 전까지 404가 반환됩니다
npm view @paperclipai/mcp-server version

# 빌드 출력이 최신 상태인지 확인합니다
pnpm --filter @paperclipai/mcp-server build

# 실제 퍼블리싱 전에 로컬 npm 인증을 확인합니다
npm whoami

# 정확한 퍼블리싱 페이로드의 안전한 미리보기
cd packages/mcp-server
pnpm publish --dry-run --no-git-checks --access public

# 실제 퍼블리싱
pnpm publish --no-git-checks --access public
```

참고:

- `packages/mcp-server/`에서 퍼블리싱하세요. 저장소 루트에서 하지 마세요.
- `npm view @paperclipai/mcp-server version`이 이미 [`packages/mcp-server/package.json`](../packages/mcp-server/package.json)에 있는 것과 동일한 버전을 반환하는 경우, 다시 퍼블리싱하지 마세요. 버전을 올리거나 [`scripts/release.sh`](../scripts/release.sh)의 일반적인 저장소 전체 릴리스 플로우를 사용하세요.
- 위와 동일한 npm 측 사전 조건이 적용됩니다: 유효한 npm 인증, `@paperclipai` 범위에 퍼블리싱할 권한, `--access public`, 그리고 필요한 퍼블리싱 인증/2FA 정책.

## 버전 형식

Paperclip은 캘린더 버전을 사용합니다:

- stable: `YYYY.MDD.P`
- canary: `YYYY.MDD.P-canary.N`

예시:

- stable: `2026.318.0`
- canary: `2026.318.1-canary.2`

## 퍼블리싱 모델

### Canary

Canary는 npm dist-tag `canary`로 퍼블리싱됩니다.

예시:

- `paperclipai@2026.318.1-canary.2`

이를 통해 기본 설치 경로는 변경하지 않으면서 다음과 같이 명시적 설치가 가능합니다:

```bash
npx paperclipai@canary onboard
```

### Stable

Stable 퍼블리싱은 npm dist-tag `latest`를 사용합니다.

예시:

- `paperclipai@2026.318.0`

Stable 퍼블리싱은 릴리스 커밋을 생성하지 않습니다. 대신:

- 패키지 버전이 임시로 재작성됩니다
- 선택된 소스 커밋에서 패키지가 퍼블리싱됩니다
- git 태그 `vYYYY.MDD.P`가 해당 원본 커밋을 가리킵니다

## Trusted publishing

의도된 CI 모델은 GitHub OIDC를 통한 npm trusted publishing입니다.

이는 다음을 의미합니다:

- 저장소 시크릿에 장기 `NPM_TOKEN`이 불필요합니다
- GitHub Actions가 단기 퍼블리싱 자격 증명을 획득합니다
- trusted publisher 규칙은 워크플로우 파일별로 구성됩니다

GitHub/npm 설정 단계는 [doc/RELEASE-AUTOMATION-SETUP.md](RELEASE-AUTOMATION-SETUP.md)를 참고하세요.

## 롤백 모델

롤백은 아무것도 unpublish하지 않습니다.

`latest` dist-tag를 이전 stable 버전으로 재지정합니다:

```bash
./scripts/rollback-latest.sh 2026.318.0
```

이는 stable 릴리스에 문제가 있을 때 기본 설치 경로를 복원하는 가장 빠른 방법입니다.

## 관련 파일

- [`scripts/build-npm.sh`](../scripts/build-npm.sh)
- [`scripts/generate-npm-package-json.mjs`](../scripts/generate-npm-package-json.mjs)
- [`scripts/generate-ui-package-json.mjs`](../scripts/generate-ui-package-json.mjs)
- [`scripts/release-package-map.mjs`](../scripts/release-package-map.mjs)
- [`cli/esbuild.config.mjs`](../cli/esbuild.config.mjs)
- [`doc/RELEASING.md`](RELEASING.md)
