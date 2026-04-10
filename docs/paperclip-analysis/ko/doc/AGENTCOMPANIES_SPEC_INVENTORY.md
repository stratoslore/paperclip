# Agent Companies 스펙 인벤토리

이 문서는 Paperclip 코드베이스에서 [Agent Companies Specification](docs/companies/companies-spec.md) (`agentcompanies/v1-draft`)을 참조하는 모든 부분을 색인합니다.

다음과 같은 경우에 사용하세요:

1. **스펙 업데이트 시** — 어떤 구현 코드가 함께 변경되어야 하는지 파악합니다.
2. **스펙 관련 코드 변경 시** — 관련된 모든 파일을 빠르게 찾습니다.
3. **정합성 유지 시** — 구현이 스펙과 일치하는지 감사합니다.

---

## 1. 사양 및 설계 문서

| 파일 | 역할 |
|---|---|
| `docs/companies/companies-spec.md` | **규범적 스펙** — 마크다운 우선 패키지 형식(COMPANY.md, TEAM.md, AGENTS.md, PROJECT.md, TASK.md, SKILL.md), 예약 파일, frontmatter 스키마, 벤더 확장 규약(`.paperclip.yaml`)을 정의합니다. |
| `doc/plans/2026-03-13-company-import-export-v2.md` | 마크다운 우선 패키지 모델 전환을 위한 구현 계획 — 단계, API 변경, UI 계획 및 롤아웃 전략. |
| `doc/SPEC-implementation.md` | V1 구현 계약; 이식성 시스템과 `.paperclip.yaml` 사이드카 형식을 참조합니다. |
| `docs/specs/cliphub-plan.md` | 이전 블루프린트 번들 계획; 마크다운 우선 스펙에 의해 부분적으로 대체됨(v2 계획에 명시). |
| `doc/plans/2026-02-16-module-system.md` | 모듈 시스템 계획; JSON 전용 회사 템플릿 섹션이 마크다운 우선 모델로 대체됨. |
| `doc/plans/2026-03-14-skills-ui-product-plan.md` | Skills UI 계획; 이식 가능한 스킬 파일과 `.paperclip.yaml`을 참조합니다. |
| `doc/plans/2026-03-14-adapter-skill-sync-rollout.md` | 어댑터 스킬 동기화 롤아웃; v2 가져오기/내보내기 계획의 보완 문서. |

## 2. 공유 타입 및 검증기

서버, CLI, UI 간의 계약을 정의합니다.

| 파일 | 정의 내용 |
|---|---|
| `packages/shared/src/types/company-portability.ts` | TypeScript 인터페이스: `CompanyPortabilityManifest`, `CompanyPortabilityFileEntry`, `CompanyPortabilityEnvInput`, 내보내기/가져오기/미리보기 요청 및 결과 타입, 에이전트, 스킬, 프로젝트, 이슈, 반복 루틴, 회사에 대한 매니페스트 항목 타입. |
| `packages/shared/src/validators/company-portability.ts` | 모든 이식성 요청/응답 형태에 대한 Zod 스키마 — 서버 라우트와 CLI 모두에서 사용됩니다. |
| `packages/shared/src/types/index.ts` | 이식성 타입을 재내보냅니다. |
| `packages/shared/src/validators/index.ts` | 이식성 검증기를 재내보냅니다. |

## 3. 서버 — 서비스

| 파일 | 책임 |
|---|---|
| `server/src/services/company-portability.ts` | **핵심 이식성 서비스.** 내보내기(매니페스트 생성, 마크다운 파일 출력, `.paperclip.yaml` 사이드카), 가져오기(그래프 해석, 충돌 처리, 엔티티 생성), 미리보기(계획된 동작 요약). 스킬 키 파생, 반복 작업 <-> 루틴 매핑, 레거시 반복 마이그레이션, 패키지 README 생성을 처리합니다. `agentcompanies/v1` 버전 문자열을 참조합니다. |
| `server/src/services/routines.ts` | Paperclip 루틴 런타임 서비스. 이식성이 이제 루틴을 반복 `TASK.md` 항목으로 내보내고, 반복 작업을 이 서비스를 통해 다시 가져옵니다. |
| `server/src/services/company-export-readme.ts` | 내보낸 회사 패키지에 대한 `README.md` 및 Mermaid 조직도를 생성합니다. |
| `server/src/services/index.ts` | `companyPortabilityService`를 재내보냅니다. |

## 4. 서버 — 라우트

| 파일 | 엔드포인트 |
|---|---|
| `server/src/routes/companies.ts` | `POST /api/companies/:companyId/export` — 레거시 내보내기 번들<br>`POST /api/companies/:companyId/exports/preview` — 내보내기 미리보기<br>`POST /api/companies/:companyId/exports` — 패키지 내보내기<br>`POST /api/companies/import/preview` — 가져오기 미리보기<br>`POST /api/companies/import` — 가져오기 실행 |

라우트 등록은 `server/src/app.ts`에서 `companyRoutes(db, storage)`를 통해 이루어집니다.

## 5. 서버 — 테스트

| 파일 | 커버리지 |
|---|---|
| `server/src/__tests__/company-portability.test.ts` | 이식성 서비스에 대한 유닛 테스트(내보내기, 가져오기, 미리보기, 매니페스트 형태, `agentcompanies/v1` 버전). |
| `server/src/__tests__/company-portability-routes.test.ts` | 이식성 HTTP 엔드포인트에 대한 통합 테스트. |

## 6. CLI

| 파일 | 명령어 |
|---|---|
| `cli/src/commands/client/company.ts` | `company export` — 회사 패키지를 디스크로 내보냅니다(플래그: `--out`, `--include`, `--projects`, `--issues`, `--projectIssues`).<br>`company import <fromPathOrUrl>` — 파일 또는 폴더에서 회사 패키지를 가져옵니다(플래그: 위치 인수 소스 경로/URL 또는 GitHub 약어, `--include`, `--target`, `--companyId`, `--newCompanyName`, `--agents`, `--collision`, `--ref`, `--dryRun`).<br>이식 가능한 파일 항목을 읽고/쓰며 `.paperclip.yaml` 필터링을 처리합니다. |

## 7. UI — 페이지

| 파일 | 역할 |
|---|---|
| `ui/src/pages/CompanyExport.tsx` | 내보내기 UI: 미리보기, 매니페스트 표시, 파일 트리 시각화, ZIP 아카이브 생성 및 다운로드. 선택에 따라 `.paperclip.yaml`을 필터링합니다. 에디터에서 매니페스트와 README를 표시합니다. |
| `ui/src/pages/CompanyImport.tsx` | 가져오기 UI: 소스 입력(업로드/폴더/GitHub URL/일반 URL), ZIP 읽기, 의존성 트리가 있는 미리보기 패널, 엔티티 선택 체크박스, 신뢰/라이선스 경고, 시크릿 요구사항, 충돌 전략, 어댑터 설정. |

## 8. UI — 컴포넌트

| 파일 | 역할 |
|---|---|
| `ui/src/components/PackageFileTree.tsx` | 가져오기와 내보내기 모두에서 사용되는 재사용 가능한 파일 트리 컴포넌트. `CompanyPortabilityFileEntry` 항목으로 트리를 구축하고, frontmatter를 파싱하며, 동작 표시기(생성/업데이트/건너뛰기)를 보여주고, frontmatter 필드 레이블을 매핑합니다. |

## 9. UI — 라이브러리

| 파일 | 역할 |
|---|---|
| `ui/src/lib/portable-files.ts` | 이식 가능한 파일 항목을 위한 헬퍼: `getPortableFileText`, `getPortableFileDataUrl`, `getPortableFileContentType`, `isPortableImageFile`. |
| `ui/src/lib/zip.ts` | ZIP 아카이브 생성(`createZipArchive`) 및 읽기(`readZipArchive`) — 회사 패키지를 위한 ZIP 형식을 처음부터 구현합니다. CRC32, DOS 날짜/시간 인코딩. |
| `ui/src/lib/zip.test.ts` | ZIP 유틸리티 테스트; 이식성 파일 항목과 `.paperclip.yaml` 콘텐츠로 왕복 테스트를 실행합니다. |

## 10. UI — API 클라이언트

| 파일 | 함수 |
|---|---|
| `ui/src/api/companies.ts` | `companiesApi.exportBundle`, `companiesApi.exportPreview`, `companiesApi.exportPackage`, `companiesApi.importPreview`, `companiesApi.importBundle` — 이식성 엔드포인트에 대한 타입이 지정된 fetch 래퍼. |

## 11. 스킬 및 에이전트 지시사항

| 파일 | 관련성 |
|---|---|
| `skills/paperclip/references/company-skills.md` | 회사 스킬 라이브러리 워크플로에 대한 참조 문서 — 설치, 검사, 업데이트, 할당. 스킬 패키지는 agent companies 스펙의 하위 집합입니다. |
| `server/src/services/company-skills.ts` | 회사 스킬 관리 서비스 — SKILL.md 기반 가져오기와 회사 수준 스킬 라이브러리를 처리합니다. |
| `server/src/services/agent-instructions.ts` | 에이전트 지시사항 서비스 — 에이전트 지시사항 로딩을 위한 AGENTS.md 경로를 해석합니다. |

## 12. 스펙 개념별 빠른 상호 참조

| 스펙 개념 | 주요 구현 파일 |
|---|---|
| `COMPANY.md` frontmatter 및 본문 | `company-portability.ts` (내보내기 발행기 + 가져오기 파서) |
| `AGENTS.md` frontmatter 및 본문 | `company-portability.ts`, `agent-instructions.ts` |
| `PROJECT.md` frontmatter 및 본문 | `company-portability.ts` |
| `TASK.md` frontmatter 및 본문 | `company-portability.ts` |
| `SKILL.md` 패키지 | `company-portability.ts`, `company-skills.ts` |
| `.paperclip.yaml` 벤더 사이드카 | `company-portability.ts`, `routines.ts`, `CompanyExport.tsx`, `company.ts` (CLI) |
| `manifest.json` | `company-portability.ts` (생성), 공유 타입 (스키마) |
| ZIP 패키지 형식 | `zip.ts` (UI), `company.ts` (CLI 파일 I/O) |
| 충돌 해결 | `company-portability.ts` (서버), `CompanyImport.tsx` (UI) |
| 환경변수/시크릿 선언 | 공유 타입 (`CompanyPortabilityEnvInput`), `CompanyImport.tsx` (UI) |
| README + 조직도 | `company-export-readme.ts` |
