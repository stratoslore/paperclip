# 플러그인 작성 가이드

이 가이드는 이 저장소에서 Paperclip 플러그인을 만드는 현재 구현된 방법을 설명합니다.

이 가이드는 의도적으로 [PLUGIN_SPEC.md](./PLUGIN_SPEC.md)보다 범위가 좁습니다. 스펙에는 미래 아이디어가 포함되어 있지만, 이 가이드는 현재 존재하는 알파 수준의 API만 다룹니다.

## 현재 상황

- 플러그인 워커와 플러그인 UI를 신뢰할 수 있는 코드로 취급합니다.
- 플러그인 UI는 메인 Paperclip 앱 내부에서 same-origin JavaScript로 실행됩니다.
- 워커 측 호스트 API는 capability로 제한됩니다.
- 플러그인 UI는 매니페스트 capability로 샌드박싱되지 않습니다.
- 플러그인용 호스트 제공 공유 React 컴포넌트 키트는 아직 없습니다.
- `ctx.assets`는 현재 런타임에서 지원되지 않습니다.

## 플러그인 스캐폴딩

스캐폴드 패키지를 사용합니다:

```bash
pnpm --filter @paperclipai/create-paperclip-plugin build
node packages/plugins/create-paperclip-plugin/dist/index.js @yourscope/plugin-name --output ./packages/plugins/examples
```

Paperclip 저장소 외부에 존재하는 플러그인의 경우:

```bash
pnpm --filter @paperclipai/create-paperclip-plugin build
node packages/plugins/create-paperclip-plugin/dist/index.js @yourscope/plugin-name \
  --output /absolute/path/to/plugin-repos \
  --sdk-path /absolute/path/to/paperclip/packages/plugins/sdk
```

이렇게 하면 다음 파일들이 포함된 패키지가 생성됩니다:

- `src/manifest.ts`
- `src/worker.ts`
- `src/ui/index.tsx`
- `tests/plugin.spec.ts`
- `esbuild.config.mjs`
- `rollup.config.mjs`

이 모노레포 내부에서 스캐폴드는 `@paperclipai/plugin-sdk`에 대해 `workspace:*`를 사용합니다.

이 모노레포 외부에서 스캐폴드는 로컬 Paperclip 체크아웃에서 `@paperclipai/plugin-sdk`를 `.paperclip-sdk/` tarball로 스냅샷하여 npm에 먼저 게시하지 않고도 플러그인을 빌드하고 테스트할 수 있게 합니다.

## 권장 로컬 워크플로우

생성된 플러그인 폴더에서:

```bash
pnpm install
pnpm typecheck
pnpm test
pnpm build
```

로컬 개발 시, 플러그인 매니저나 API를 통해 절대 로컬 경로에서 Paperclip에 설치합니다. 서버는 로컬 파일시스템 설치를 지원하며, 로컬 경로 플러그인의 파일 변경을 감시하여 리빌드 후 워커가 자동으로 재시작됩니다.

예시:

```bash
curl -X POST http://127.0.0.1:3100/api/plugins/install \
  -H "Content-Type: application/json" \
  -d '{"packageName":"/absolute/path/to/your-plugin","isLocalPath":true}'
```

## 지원되는 알파 API

워커:

- config
- events
- jobs
- launchers
- http
- secrets
- activity
- state
- entities
- projects 및 project workspaces
- companies
- issues 및 comments
- agents 및 agent sessions
- goals
- data/actions
- streams
- tools
- metrics
- logger

UI:

- `usePluginData`
- `usePluginAction`
- `usePluginStream`
- `usePluginToast`
- `useHostContext`
- `@paperclipai/plugin-sdk/ui`의 타입된 slot props

현재 호스트에 연결된 마운트 영역:

- `page`
- `settingsPage`
- `dashboardWidget`
- `sidebar`
- `sidebarPanel`
- `detailTab`
- `taskDetailView`
- `projectSidebarItem`
- `globalToolbarButton`
- `toolbarButton`
- `contextMenuItem`
- `commentAnnotation`
- `commentContextMenuItem`

## 회사 라우트

플러그인은 `routePath`가 있는 `page` 슬롯을 선언하여 다음과 같은 회사 라우트를 소유할 수 있습니다:

```text
/:companyPrefix/<routePath>
```

규칙:

- `routePath`는 단일 소문자 slug여야 합니다
- 예약된 호스트 라우트와 충돌할 수 없습니다
- 다른 설치된 플러그인 페이지 라우트와 중복될 수 없습니다

## 배포 가이드

- npm 패키지를 배포 아티팩트로 사용합니다.
- 저장소 내 로컬 예제 설치는 개발 워크플로우로만 취급합니다.
- 플러그인 UI를 패키지 내에 자체 포함하는 것을 선호합니다.
- 호스트 디자인 시스템 컴포넌트나 문서화되지 않은 앱 내부 구현에 의존하지 마세요.
- GitHub 저장소 설치는 현재 공식 워크플로우가 아닙니다. 로컬 개발에는 체크아웃된 로컬 경로를 사용하세요. 프로덕션에는 npm 또는 프라이빗 npm 호환 레지스트리에 배포하세요.

## 인수 전 검증

최소 요건:

```bash
pnpm --filter <your-plugin-package> typecheck
pnpm --filter <your-plugin-package> test
pnpm --filter <your-plugin-package> build
```

호스트 통합도 변경한 경우 추가로 실행합니다:

```bash
pnpm -r typecheck
pnpm test:run
pnpm build
```
