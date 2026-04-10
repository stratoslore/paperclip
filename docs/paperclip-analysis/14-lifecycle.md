# Paperclip 라이프사이클 관리

## 개요

Paperclip의 4가지 핵심 엔티티(에이전트, 스킬, 플러그인, 어댑터 스킬)의 생성부터 종료까지 전체 라이프사이클과 데이터 저장 방식을 정리한다.

---

## 1. 에이전트 (Agent)

### 라이프사이클 상태

```
생성(idle) → 활성(idle) ↔ 일시정지(paused) → 종료(terminated)
                              │
                      pauseReason, pausedAt 설정
```

| 상태 | 설명 |
|---|---|
| `idle` | 기본 상태. 작업 할당 가능, Heartbeat 대기 |
| `paused` | 일시 정지. 사유와 시각 기록. resume으로 복귀 가능 |
| `terminated` | 영구 비활성화. 모든 관련 데이터 cascade 삭제 |

### 생성 방법

- **API**: `POST /api/companies/{companyId}/agents`
- **UI**: 에이전트 페이지 > "+ New Agent"
- **온보딩**: 회사 생성 위저드에서 자동 생성

### 데이터 저장

| 테이블 | 용도 |
|---|---|
| `agents` | 핵심 설정 (name, role, status, adapterType, adapterConfig, budgetMonthlyCents 등) |
| `agent_config_revisions` | 설정 변경 이력 (누가, 언제, 무엇을 변경했는지) |
| `agent_api_keys` | 에이전트-서버 인증용 API 키 |
| `agent_runtime_state` | Heartbeat 간 지속 세션/상태 |
| `agent_task_sessions` | 현재 체크아웃한 작업 |
| `agent_wakeup_requests` | 예약된 wake 트리거 |

- 파일 시스템에는 저장되지 않음. 전부 PostgreSQL DB

### Heartbeat 라이프사이클

```
트리거(스케줄러/수동/이벤트)
  → heartbeat_runs 레코드 생성 (status: pending)
  → 어댑터가 에이전트 프로세스 실행 (status: running)
  → 스킬 주입, 작업 수행
  → 프로세스 종료 (status: completed/failed)
  → contextSnapshot 저장 (다음 Heartbeat 연속성)
```

| 필드 | 설명 |
|---|---|
| `status` | pending, running, completed, failed |
| `invocationSource` | timer, manual, assignment, mention |
| `sessionIdBefore/After` | 세션 연속성 추적 |
| `exitCode`, `signal` | 프로세스 종료 정보 |
| `logStore`, `logRef` | 실행 로그 |
| `contextSnapshot` | 다음 Heartbeat을 위한 상태 보존 (JSONB) |
| `processLossRetryCount` | 프로세스 손실 재시도 횟수 |

### 종료 시 정리

`terminated` 상태로 변경 시 cascade 삭제:
- 모든 Heartbeat 실행 기록
- API 키
- 런타임 상태
- 작업 세션
- 예약 트리거

---

## 2. Company Skills (회사 스킬)

### 라이프사이클

```
발견(Discovery) → 임포트(Import) → DB 저장 → 에이전트 할당 → 런타임 Sync → 주입
```

### 소스 유형

| sourceType | 설명 | 예시 |
|---|---|---|
| `github` | GitHub 저장소 | `owner/repo/skills/my-skill` |
| `skills_sh` | skills.sh 레지스트리 | `skills.sh/search-tool` |
| `local_path` | 로컬 파일 경로 | `/workspace/skills/my-skill` |
| `url` | 웹 URL | `https://example.com/skill.tar.gz` |
| `catalog` | 내장 카탈로그 | Paperclip 기본 제공 |

### 임포트 API

```
POST /api/companies/{companyId}/skills/import
```

흐름:
1. 소스에서 SKILL.md + 부속 파일 가져오기
2. YAML front-matter 파싱 (name, description)
3. 스킬 키 생성 (소스 기반 정규화)
4. `company_skills` 테이블에 upsert

### 데이터 저장

| 필드 | 설명 |
|---|---|
| `id` | UUID |
| `companyId` | 소속 회사 |
| `key` | 정규 식별자 (예: `paperclipai/paperclip/skill-name`) |
| `slug` | 짧은 이름 |
| `name`, `description` | 메타데이터 |
| `markdown` | SKILL.md 전체 내용 |
| `sourceType` | github, skills_sh, local_path, url |
| `sourceLocator` | 원본 위치 |
| `sourceRef` | 브랜치/태그/커밋 (VCS용) |
| `trustLevel` | `markdown_only`, `assets`, `scripts_executables` |
| `compatibility` | `compatible`, `unknown`, `invalid` |
| `fileInventory` | 파일 목록 (경로, 종류, 해시) |

### 에이전트 할당

에이전트의 `adapterConfig`에서 설정:

```json
{
  "paperclipSkillSync": {
    "desiredSkills": ["paperclip", "my-custom-skill"]
  }
}
```

- API: `POST /api/agents/{agentId}/skills/sync`
- 필수 스킬(`required: true`)은 자동 포함
- 원하는 스킬만 `desiredSkills`에 추가

### 런타임 주입

| 어댑터 | 방식 | 정리 |
|---|---|---|
| claude_local | temp 디렉토리에 심링크 → `--add-dir` 플래그 | 실행 후 자동 삭제 |
| codex_local | 전역 스킬 디렉토리 | 수동 |
| process | 워크스페이스 `./skills/` 하위 | 수동 |

### 스킬 파일 구조

```
skills/my-skill/
├── SKILL.md          # 메인 스킬 문서 (필수)
├── references/       # 참조 파일
│   └── api-spec.md
└── scripts/          # 실행 스크립트 (trustLevel에 따라)
    └── setup.sh
```

### 삭제

- API: `DELETE /api/companies/{companyId}/skills/{skillId}`
- 삭제 전까지 DB에 영구 보존
- 런타임 파일은 매 Heartbeat마다 새로 생성되므로 별도 정리 불필요

---

## 3. 플러그인 (Plugin)

### 라이프사이클 상태

```
installed → ready ──→ disabled ──→ uninstalled
    │         │ ↕          │
    │         ↓            │
    └→ error ←─┘           │
         ↓                 │
    upgrade_pending ───────┘
```

| 상태 | 설명 |
|---|---|
| `installed` | 패키지 다운로드 및 검증 완료 |
| `ready` | Worker 프로세스 가동, 헬스체크 통과, 도구 등록 완료 |
| `disabled` | 운영자가 수동 비활성화. Worker 정상 종료 |
| `error` | Worker 크래시, 헬스체크 실패, 런타임 예외 |
| `upgrade_pending` | 새 버전에 Capability 변경 있음. 운영자 승인 대기 |
| `uninstalled` | 제거됨 (soft-delete 또는 hard-delete) |

### 유효 상태 전이

```
installed     → ready, error, uninstalled
ready         → disabled, error, upgrade_pending, uninstalled
disabled      → ready, uninstalled
error         → ready(재시도), uninstalled
upgrade_pending → ready, error, uninstalled
uninstalled   → installed(재설치)
```

### 설치 방법

| 방법 | 예시 |
|---|---|
| npm 패키지 | `npm install paperclip-plugin-linear` |
| 로컬 경로 | `POST /api/plugins/install { "source": "/path/to/plugin", "isLocalPath": true }` |

### 데이터 저장

| 테이블 | 용도 |
|---|---|
| `plugins` | 핵심 정보 (pluginKey, version, status, manifestJson, packagePath) |
| `plugin_config` | 플러그인별 설정 |
| `plugin_state` | 상태 저장 (company/agent/run 범위) |
| `plugin_jobs` | 스케줄된/큐잉된 작업 |
| `plugin_logs` | 감사 로그 |
| `plugin_webhook_deliveries` | 수신 webhook 기록 |

### 파일 위치

- 기본: `~/.paperclip/plugins/`
- npm: `node_modules/paperclip-plugin-*`
- 로컬 개발: 지정된 절대 경로

### Worker 관리

- 플러그인이 `ready` 상태가 되면 Worker 프로세스 자동 시작
- `ready`에서 벗어나면 Worker 자동 중지
- 코드 변경 시 `restartWorker()` 호출 필요 (자동 hot-reload 없음)

### 라이프사이클 이벤트

| 이벤트 | 발생 시점 |
|---|---|
| `plugin.loaded` | 패키지 로드 완료 |
| `plugin.enabled` | ready 상태 진입 |
| `plugin.disabled` | disabled 상태 진입 |
| `plugin.unloaded` | 언로드 |
| `plugin.status_changed` | 모든 상태 변경 |
| `plugin.error` | 오류 발생 |
| `plugin.worker_started` | Worker 시작 |
| `plugin.worker_stopped` | Worker 중지 |

---

## 4. 어댑터 스킬 (Adapter Runtime Skills)

### 어댑터별 모드

| 어댑터 | 모드 | 설명 |
|---|---|---|
| `claude_local` | **Ephemeral** | 매 Heartbeat마다 temp 디렉토리 생성 → 심링크 → 실행 후 삭제 |
| `codex_local` | **Persistent** | 전역 스킬 디렉토리에 영구 보존 |
| `process` | **Persistent** | 워크스페이스 내 `./skills/` 에 보존 |

### Ephemeral 모드 흐름 (Claude)

```
Heartbeat 시작
  → readPaperclipRuntimeSkillEntries() — 사용 가능 스킬 목록 조회
  → resolvePaperclipDesiredSkillNames() — 에이전트 설정에서 원하는 스킬 필터
  → fs.mkdtemp() — 임시 디렉토리 생성
  → fs.symlink() — 스킬 파일 심링크
  → claude --add-dir {tmp}/.claude/skills — 에이전트에 주입
  → 실행 완료
  → 임시 디렉토리 삭제
```

### Persistent 모드 흐름 (Codex/Process)

```
스킬 Sync 요청
  → 전역/워크스페이스 스킬 디렉토리에 파일 배치
  → 이후 모든 Heartbeat에서 자동 사용
  → 수동 삭제 전까지 유지
```

### 스킬 Sync 설정

에이전트의 `adapterConfig`에서 관리:

```typescript
// 읽기
readPaperclipSkillSyncPreference(config)
// → { explicit: boolean, desiredSkills: ["skill-1", "skill-2"] }

// 해석
resolvePaperclipDesiredSkillNames(config, availableEntries)
// → required 스킬 + desired 스킬 병합
```

### AdapterSkillEntry 구조

```typescript
{
  key: "paperclipai/paperclip/my-skill",  // 정규 식별자
  runtimeName: "my-skill",                // 런타임 이름
  desired: true,                          // 에이전트가 원하는지
  managed: true,                          // Paperclip이 관리하는지
  required: false,                        // 필수 스킬인지
  state: "configured",                    // available | configured | installed | missing
  origin: "company_managed",              // company_managed | paperclip_required | user_installed
  sourcePath: "/path/to/skill",           // 원본 위치
  targetPath: "/tmp/skills/my-skill",     // 런타임 위치
}
```

---

## 5. Execution Workspace (실행 워크스페이스)

### 라이프사이클

```
생성(active) → 사용 중(active) → 보관(archived) → 종료(closed)
```

| 상태 | 설명 |
|---|---|
| `active` | 코드 수정 가능, 런타임 서비스 관리 가능 |
| `archived` | 접근 가능하나 비활성 |
| `closed` | 런타임 서비스 중지, Git worktree 제거, 아티팩트 정리 |

### 워크스페이스 모드

| 모드 | 설명 |
|---|---|
| `isolated` | 이슈별 격리 (ephemeral, git worktree) |
| `reuse` | 여러 이슈가 공유 |
| `primary` | 프로젝트 메인 워크스페이스 |

### 데이터 저장

| 필드 | 설명 |
|---|---|
| `cwd` | 로컬 파일시스템 경로 |
| `repoUrl`, `baseRef`, `branchName` | Git 저장소 정보 |
| `providerType` | `local_fs`, `git_worktree` |
| `mode` | `isolated`, `reuse`, `primary` |
| `metadata.config` | provisionCommand, teardownCommand, cleanupCommand |

### Runtime Service 관리

```typescript
// workspace_runtime_services 테이블
{
  serviceName: "dev-server",
  command: "npm run dev",
  cwd: "/workspace/my-app",
  port: 3000,
  url: "http://localhost:3000",
  provider: "local_process",       // local_process | adapter_managed
  status: "running",               // starting | running | stopped | failed
  lifecycle: "shared",             // shared | ephemeral
  healthStatus: "healthy",
  pid: 12345,
}
```

- UI에서 수동 시작/중지
- 서버 재시작 시 자동 복구 없음
- 워크스페이스 종료 시 서비스도 중지

---

## 6. 전체 엔티티 비교표

| 엔티티 | 생성 | 저장 | 라이프사이클 | 정리 |
|---|---|---|---|---|
| **Agent** | API/UI | DB: `agents` | idle→paused↔idle→terminated | cascade 삭제 |
| **Skill** | Import API | DB: `company_skills` + 런타임 임시 파일 | 발견→임포트→할당→주입 | DB에서 DELETE |
| **Plugin** | npm/로컬 | DB: `plugins` + 파일시스템 | installed→ready→disabled→uninstalled | soft/hard delete |
| **Adapter Skill** | 자동 (Heartbeat) | 임시 디렉토리 또는 전역 디렉토리 | 매 실행 생성→사용→정리 | ephemeral: 자동, persistent: 수동 |
| **Workspace** | 이슈 할당 | DB: `execution_workspaces` + Git worktree | active→archived→closed | worktree 제거, 정책별 정리 |
| **Runtime Service** | 수동 UI/Plugin | DB: `workspace_runtime_services` + 프로세스 | starting→running→stopped | 워크스페이스 종료 시 |

---

## 7. 앱 개발과 도구 호출 통합 시나리오

### 에이전트가 앱을 개발하고, 개발된 앱을 도구로 호출하는 흐름

```
1. 프로젝트 생성
   └─ Execution Workspace 생성 (cwd: /workspace/my-app)

2. 에이전트가 코드 작성
   └─ Git worktree에서 개발 (feature/api 브랜치)
   └─ npm install, npm run build 실행

3. Runtime Service 등록
   └─ npm run dev → port 3000에서 실행
   └─ DB에 서비스 기록 (url: http://localhost:3000)

4. Skill 작성
   └─ SKILL.md: "POST http://localhost:3000/api/action" 호출 방법

5. 다른 에이전트가 Skill 사용
   └─ Heartbeat → Skill 주입 → localhost:3000 호출 → 결과 처리

6. Plugin Tool로 고도화
   └─ Plugin worker에서 localhost:3000 직접 호출
   └─ 에러 핸들링, 인증, 데이터 변환 캡슐화
```

### 외부 API 연동 시나리오

```
[Paperclip Agent]
  │
  ├─ Skill: REST API 호출 가이드 (마크다운)
  │   └─ 에이전트가 curl로 외부 API 호출
  │
  ├─ Plugin Tool: Worker에서 직접 호출
  │   └─ fetch("https://api.slack.com/...")
  │   └─ Capability 기반 보안 (http.outbound)
  │
  ├─ MCP Server: Paperclip 데이터를 외부에 노출
  │   └─ 외부 Claude.ai가 Paperclip MCP 서버에 접속
  │
  └─ HTTP Adapter: 외부 서비스가 에이전트 역할
      └─ Paperclip → POST webhook → 외부 서비스
      └─ 외부 서비스 → Paperclip REST API로 결과 반환
```
