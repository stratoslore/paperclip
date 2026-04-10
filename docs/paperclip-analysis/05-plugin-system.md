# 05. 플러그인 시스템

> 소스: `packages/plugins/sdk/src/`, `server/src/services/plugin-*.ts` (14개 파일)

Paperclip은 강력한 플러그인 시스템을 제공하여 기능을 확장할 수 있다.

## 아키텍처 개요

```
┌────────────────────────────────────────────────┐
│                Paperclip Server                 │
│                                                │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Plugin Loader │  │ Plugin Worker Manager │   │
│  │  (디스커버리) │  │  (프로세스 관리)      │   │
│  └──────┬───────┘  └──────────┬───────────┘   │
│         │                     │                │
│         ▼                     ▼                │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │  Event Bus   │  │  Job Scheduler       │   │
│  │  (이벤트)    │  │  (백그라운드 작업)    │   │
│  └──────────────┘  └──────────────────────┘   │
│                                                │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Host Services│  │ Tool Dispatcher      │   │
│  │  (서비스)    │  │  (도구 디스패치)      │   │
│  └──────────────┘  └──────────────────────┘   │
└─────────────────────┬──────────────────────────┘
                      │ JSON-RPC
                      ▼
┌────────────────────────────────────────────────┐
│           Plugin Worker Process                 │
│                                                │
│  definePlugin({                                │
│    setup(ctx) { ... },                         │
│    onHealth() { ... },                         │
│    validateConfig(config) { ... },             │
│  })                                            │
└────────────────────────────────────────────────┘
```

## 플러그인 정의

```typescript
import { definePlugin, runWorker } from '@paperclipai/plugin-sdk';

export default definePlugin({
  // 초기화
  async setup(ctx: PluginContext) {
    // 이벤트 핸들러 등록
    ctx.events.on('issue.created', async (event) => {
      ctx.logger.info('New issue created', { issueId: event.data.id });
    });

    // 백그라운드 잡 등록
    ctx.jobs.register('full-sync', async (job) => {
      // 동기화 로직
    });
  },

  // 헬스체크
  async onHealth(): Promise<PluginHealthDiagnostics> {
    return { status: 'ok', message: 'All systems operational' };
  },

  // 설정 검증
  async validateConfig(config: unknown) {
    return { valid: true };
  },
});

// 워커 실행
runWorker();
```

## PluginContext API

플러그인이 사용할 수 있는 전체 API:

### 로깅

```typescript
ctx.logger.info('message', { key: 'value' });
ctx.logger.warn('warning');
ctx.logger.error('error', { err });
```

### 이벤트 구독

```typescript
// 이벤트 타입
type PluginEventType =
  | 'issue.created'
  | 'issue.updated'
  | 'issue.commented'
  | 'agent.created'
  | 'agent.updated'
  | 'heartbeat.completed'
  | 'project.created'
  // ... 20+ 이벤트 타입

ctx.events.on('issue.created', async (event) => {
  const { id, title, status } = event.data;
  // 처리 로직
});
```

### 백그라운드 잡

```typescript
// 잡 등록
ctx.jobs.register('my-job', async (job: PluginJobContext) => {
  ctx.logger.info('Running', { runId: job.runId });
  // 잡 로직
});

// 수동 트리거
await ctx.jobs.trigger('my-job', { companyId: '...' });
```

### 영속 상태 (State Store)

```typescript
// 상태 저장
await ctx.state.set({
  scopeKind: 'company',       // 'company' | 'agent' | 'global'
  scopeId: companyId,
  stateKey: 'last-sync-at',
  value: new Date().toISOString(),
});

// 상태 조회
const result = await ctx.state.get({
  scopeKind: 'company',
  scopeId: companyId,
  stateKey: 'last-sync-at',
});
```

### HTTP 클라이언트

```typescript
// 자동 인증 포함 HTTP 요청
const response = await ctx.http.fetch('https://api.example.com/data', {
  method: 'GET',
  headers: { 'Accept': 'application/json' },
});
```

### 시크릿 관리

```typescript
// 암호화된 시크릿 참조 해제
const apiKey = await ctx.secrets.resolve('my-api-key-ref');
```

### Paperclip 리소스 접근

```typescript
// 이슈 조회/수정
const issues = await ctx.issues.list(companyId, { status: 'todo' });
await ctx.issues.update(issueId, { status: 'in_progress' });

// 에이전트 조회
const agents = await ctx.agents.list(companyId);

// 프로젝트 조회
const projects = await ctx.projects.list(companyId);

// 회사 조회
const company = await ctx.companies.getById(companyId);

// 목표 조회
const goals = await ctx.goals.list(companyId);
```

### 에이전트 세션 (스트리밍 상호작용)

```typescript
const session = await ctx.agentSessions.create(agentId, {
  prompt: 'Analyze this data...',
});
```

### 데이터 프로바이더 (UI 확장)

```typescript
ctx.data.registerProvider('my-dashboard-data', async (params) => {
  return {
    items: [...],
    total: 42,
  };
});
```

### 도구 등록

```typescript
ctx.tools.register('my-tool', {
  description: 'My custom tool',
  parameters: { ... },
  handler: async (params) => {
    return { result: 'done' };
  },
});
```

## 플러그인 설치 위치

```
~/.paperclip/instances/{instance-id}/plugins/
  ├── my-plugin/
  │   ├── manifest.json     # 플러그인 메타데이터
  │   ├── dist/
  │   │   └── worker.js     # 빌드된 워커 코드
  │   └── package.json
  └── another-plugin/
      └── ...
```

## 플러그인 스캐폴딩

```bash
npx create-paperclip-plugin my-plugin
cd my-plugin
pnpm install
pnpm build
```

## 플러그인 라이프사이클

```
1. 디스커버리: pluginLoader가 plugins/ 디렉토리 스캔
   │
2. 로딩: manifest.json 파싱 및 설정 스키마 검증
   │
3. 워커 생성: 격리된 서브프로세스 실행
   │
4. 초기화: JSON-RPC로 initialize({config}) 호출
   │
5. 셋업: definePlugin의 setup(ctx) 실행
   │  ├─ 이벤트 핸들러 등록
   │  ├─ 잡 등록
   │  └─ 데이터 프로바이더 등록
   │
6. 실행: 이벤트 수신, 잡 실행, API 호출
   │
7. 헬스체크: 주기적 onHealth() 호출
   │
8. 종료: dispose → 워커 프로세스 종료
```

## DB 테이블

| 테이블 | 설명 |
|--------|------|
| `plugins` | 플러그인 레지스트리 |
| `pluginConfig` | 인스턴스 수준 설정 |
| `pluginState` | 영속 상태 스토어 |
| `pluginJobs` | 잡 정의 |
| `pluginJobRuns` | 잡 실행 기록 |

## 서버 사이드 플러그인 서비스 (14개)

| 서비스 | 역할 |
|--------|------|
| `plugin-loader` | 플러그인 디스커버리 및 초기화 |
| `plugin-worker-manager` | 워커 프로세스 관리 |
| `plugin-event-bus` | 이벤트 발행/구독 |
| `plugin-job-scheduler` | 잡 스케줄링 |
| `plugin-job-coordinator` | 잡 실행 조율 |
| `plugin-tool-dispatcher` | 도구 디스패치 |
| `plugin-lifecycle` | 라이프사이클 관리 |
| `plugin-sandbox` | 샌드박스 격리 |
| `plugin-config` | 설정 관리 |
| `plugin-state` | 상태 관리 |
| `plugin-secrets` | 시크릿 접근 |
| `plugin-entities` | 커스텀 엔티티 |
| `plugin-metrics` | 메트릭 수집 |
| `plugin-streams` | 스트림 관리 |
