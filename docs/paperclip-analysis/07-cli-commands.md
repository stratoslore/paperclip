# 07. CLI 명령어

> 소스: `cli/src/commands/`

Paperclip CLI는 `paperclipai` (npm 패키지)로 배포되며, Commander.js 기반이다.

## 설치

```bash
# npx로 직접 실행
npx paperclipai <command>

# 또는 글로벌 설치
npm install -g paperclipai
```

## 명령어 목록

### `onboard` — 초기 설정 마법사

> 소스: `cli/src/commands/onboard.ts`

Paperclip 인스턴스를 처음 설정하는 인터랙티브 마법사.

```bash
# 기본값으로 빠른 설정
npx paperclipai onboard --yes

# 인터랙티브 설정
npx paperclipai onboard
```

**설정 항목:**

| 단계 | 설명 | 기본값 |
|------|------|--------|
| Setup mode | quickstart / advanced | quickstart |
| Database | embedded / external PostgreSQL | embedded |
| LLM provider | Claude / OpenAI | Claude |
| Logging | file / cloud | file |
| Server host | 바인드 호스트 | 127.0.0.1 |
| Server port | 리스닝 포트 | 3100 |
| Deployment mode | local_trusted / authenticated | local_trusted |
| Auth | base URL, public URL | auto |
| Storage | local disk / S3 | local disk |
| Secrets | local encrypted / AWS / GCP / Vault | local encrypted |

**출력:**
- `~/.paperclip/instances/{id}/config.json` — 설정 파일
- `.env` — JWT 시크릿 등
- 내장 PostgreSQL 디렉토리 구조

**환경변수 지원:**
- `PAPERCLIP_*` 환경변수로 기본값 프리필 가능
- `PAPERCLIP_MIGRATION_AUTO_APPLY=true`로 비대화형 모드

---

### `run` — 서버 시작

> 소스: `cli/src/commands/run.ts`

Paperclip 서버를 시작한다.

```bash
npx paperclipai run
npx paperclipai run --instance my-instance
npx paperclipai run --config /path/to/config.json
npx paperclipai run --repair
```

**실행 흐름:**
1. 인스턴스 ID 및 경로 결정
2. 설정 로드 (없으면 onboard 실행)
3. doctor 검사 (--repair 시 자동 복구)
4. 서버 모듈 임포트 및 `startServer()` 호출
5. authenticated 모드 + 내장 DB 시: CEO 초대 생성

**옵션:**

| 옵션 | 설명 |
|------|------|
| `--config` | 설정 파일 경로 |
| `--instance` | 인스턴스 ID |
| `--repair` | 자동 복구 활성화 |
| `--yes` | 비대화형 모드 |

---

### `configure` — 설정 변경

> 소스: `cli/src/commands/configure.ts`

기존 Paperclip 인스턴스의 설정을 수정한다.

```bash
npx paperclipai configure
```

---

### `doctor` — 헬스체크 및 복구

> 소스: `cli/src/commands/doctor.ts`

Paperclip 인스턴스의 상태를 진단하고 문제를 복구한다.

```bash
npx paperclipai doctor
npx paperclipai doctor --repair
```

**검사 항목:**
- 설정 파일 유효성
- 데이터베이스 연결
- 마이그레이션 상태
- 시크릿 키 파일
- 디스크 공간
- 포트 사용 가능 여부

---

### `heartbeat-run` — 수동 하트비트 트리거

> 소스: `cli/src/commands/heartbeat-run.ts`

에이전트의 하트비트를 수동으로 트리거한다.

```bash
npx paperclipai heartbeat-run --agent-id <uuid>
```

---

### `routines` — 루틴 관리

> 소스: `cli/src/commands/routines.ts`

CLI에서 루틴을 관리한다.

```bash
npx paperclipai routines list
npx paperclipai routines trigger <routine-id>
```

---

### `db-backup` — 데이터베이스 백업

> 소스: `cli/src/commands/db-backup.ts`

```bash
npx paperclipai db-backup
npx paperclipai db-backup --output /path/to/backup
```

---

### `env` — 환경변수 설정

> 소스: `cli/src/commands/env.ts`

```bash
npx paperclipai env
```

---

### `worktree` — Git Worktree 관리

> 소스: `cli/src/commands/worktree.ts`

에이전트 실행 워크스페이스의 git worktree를 관리한다.

```bash
npx paperclipai worktree list
npx paperclipai worktree cleanup
```

---

### `auth-bootstrap-ceo` — CEO 초대

> 소스: `cli/src/commands/auth-bootstrap-ceo.ts`

authenticated 모드에서 최초 CEO 사용자를 초대한다.

```bash
npx paperclipai auth-bootstrap-ceo
```

---

### `allowed-hostname` — 허용 호스트네임 관리

> 소스: `cli/src/commands/allowed-hostname.ts`

private 모드에서 접근 허용할 호스트네임을 관리한다.

```bash
npx paperclipai allowed-hostname add example.com
npx paperclipai allowed-hostname list
npx paperclipai allowed-hostname remove example.com
```

## 개발 모드 명령어

`package.json` 스크립트로 실행:

```bash
pnpm dev              # 전체 개발 모드 (API + UI, 파일 감시)
pnpm dev:watch        # dev와 동일
pnpm dev:once         # 파일 감시 없이 한 번 실행
pnpm dev:server       # 서버만 실행
pnpm dev:ui           # UI만 실행
pnpm dev:list         # 실행 중인 개발 서비스 목록
pnpm dev:stop         # 개발 서비스 중지
pnpm build            # 전체 빌드
pnpm typecheck        # 타입 검사
pnpm test:run         # 테스트 실행 (vitest)
pnpm test:e2e         # E2E 테스트 (Playwright)
pnpm db:generate      # DB 마이그레이션 생성
pnpm db:migrate       # DB 마이그레이션 적용
pnpm db:backup        # DB 백업
pnpm docs:dev         # Mintlify 문서 로컬 실행
```
