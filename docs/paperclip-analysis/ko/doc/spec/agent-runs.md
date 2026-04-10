# 에이전트 실행 서브시스템 스펙

상태: 초안
날짜: 2026-02-17
대상: 제품 + 엔지니어링
범위: 에이전트 실행 런타임, 어댑터 프로토콜, 웨이크업 오케스트레이션, 실시간 상태 전달

## 1. 문서 역할

이 스펙은 Paperclip이 런타임에 구애받지 않으면서 실제로 에이전트를 실행하는 방법을 정의합니다.

- `doc/SPEC-implementation.md`는 V1 기준 계약으로 유지됩니다.
- 이 문서는 로컬 CLI 어댑터, 런타임 상태 영속성, 웨이크업 스케줄링, 브라우저 실시간 업데이트를 포함한 에이전트 실행에 대한 구체적인 서브시스템 세부사항을 추가합니다.
- 이 문서가 코드의 현재 런타임 동작과 충돌하는 경우, 이 문서가 향후 구현의 목표 동작입니다.

## 2. 포착된 의도 (요청에서)

다음 의도는 이 스펙에 명시적으로 보존됩니다:

1. Paperclip은 어댑터에 구애받지 않습니다. 핵심은 프로토콜이지 특정 런타임이 아닙니다.
2. 시스템을 즉시 유용하게 만들기 위해 기본 내장 기능이 여전히 필요합니다.
3. 첫 두 내장 기능은 `claude-local`과 `codex-local`입니다.
4. 이 어댑터는 호스트 머신에서 샌드박싱 없이 직접 로컬 CLI를 실행합니다.
5. 에이전트 구성에는 작업 디렉토리와 초기/기본 프롬프트가 포함됩니다.
6. 하트비트는 구성된 어댑터 프로세스를 실행하고, Paperclip이 수명주기를 관리하며, 종료 시 Paperclip이 JSON 출력을 파싱하고 상태를 업데이트합니다.
7. 세션 ID와 토큰 사용량은 이후 하트비트가 재개할 수 있도록 영속되어야 합니다.
8. 어댑터는 상태 업데이트(짧은 메시지 + 색상)와 선택적 스트리밍 로그를 지원해야 합니다.
9. UI는 변수 삽입을 위한 프롬프트 템플릿 "필"을 지원해야 합니다.
10. CLI 오류는 UI에서 전체(또는 가능한 한 많이) 표시되어야 합니다.
11. 상태 변경은 서버 푸시를 통해 작업 및 에이전트 뷰에서 실시간으로 업데이트되어야 합니다.
12. 웨이크업 트리거는 최소한 다음을 포함하는 하트비트/웨이크업 서비스에 의해 중앙 집중화되어야 합니다:
   - 타이머 간격
   - 작업 할당 시 깨우기
   - 명시적 핑/요청

## 3. 목표와 비목표

### 3.1 목표

1. 여러 런타임을 지원하는 안정적인 어댑터 프로토콜을 정의합니다.
2. Claude CLI와 Codex CLI를 위한 프로덕션 사용 가능한 로컬 어댑터를 출시합니다.
3. 어댑터 런타임 상태(세션 ID, 토큰/비용 사용량, 마지막 오류)를 영속합니다.
4. 웨이크업 결정과 큐잉을 하나의 서비스에 중앙 집중화합니다.
5. 브라우저에 실시간 실행/작업/에이전트 업데이트를 제공합니다.
6. Postgres를 비대하게 하지 않으면서 배포별 전체 로그 저장을 지원합니다.
7. 회사 범위 지정과 기존 거버넌스 불변성을 보존합니다.

### 3.2 비목표 (이 서브시스템 단계에서)

1. 여러 호스트에 걸친 분산 실행 워커.
2. 서드파티 어댑터 마켓플레이스/플러그인 SDK.
3. 비용을 보고하지 않는 제공자에 대한 완벽한 비용 회계.
4. 기본 보관 이상의 장기 로그 아카이빙 전략.

## 4. 기준선과 격차 (2026-02-17 기준)

현재 코드에 이미 있는 것:

- `adapterType` + `adapterConfig`가 있는 `agents`.
- 기본 상태 추적이 있는 `heartbeat_runs`.
- `process`와 `http`를 호출하는 인프로세스 `heartbeatService`.
- 활성 실행에 대한 취소 엔드포인트.

이 스펙이 다루는 현재 격차:

1. 세션 재개를 위한 에이전트별 영속 런타임 상태 없음.
2. 큐/웨이크업 추상화 없음 (호출이 즉각적).
3. 할당 트리거 또는 타이머 트리거 중앙 집중 웨이크업 없음.
4. 브라우저로의 websocket/SSE 푸시 경로 없음.
5. 영속된 실행 이벤트 타임라인 또는 외부 전체 로그 저장 계약 없음.
6. Claude/Codex 세션 및 사용량 추출을 위한 타입된 로컬 어댑터 계약 없음.
7. 에이전트 설정의 프롬프트 템플릿 변수/필 시스템 없음.
8. 전체 실행 로그 저장을 위한 배포 인식 어댑터 없음 (디스크/객체 저장소/등).

## 5. 아키텍처 개요

이 서브시스템은 여섯 개의 협력 컴포넌트를 도입합니다:

1. `Adapter Registry`
   - `adapter_type`을 구현에 매핑합니다.
   - capability 메타데이터와 설정 검증을 노출합니다.

2. `Wakeup Coordinator`
   - 모든 웨이크업(`timer`, `assignment`, `on_demand`, `automation`)을 위한 단일 진입점.
   - 중복 제거/통합 및 큐 규칙을 적용합니다.

3. `Run Executor`
   - 큐에 있는 웨이크업을 클레임합니다.
   - `heartbeat_runs`를 생성합니다.
   - 로컬 어댑터를 위해 자식 프로세스를 생성/모니터링합니다.
   - 타임아웃/취소/우아한 종료를 처리합니다.

4. `Runtime State Store`
   - 에이전트별 재개 가능한 어댑터 상태를 영속합니다.
   - 실행 사용량 요약과 경량 실행 이벤트 타임라인을 영속합니다.

5. `Run Log Store`
   - 플러거블 저장소 어댑터를 통해 전체 stdout/stderr 스트림을 영속합니다.
   - 검색을 위한 안정적인 `logRef`를 반환합니다 (로컬 경로, 객체 키, 또는 DB 참조).

6. `Realtime Event Hub`
   - websocket을 통해 실행/에이전트/작업 업데이트를 게시합니다.
   - 회사별 선택적 구독을 지원합니다.

제어 흐름 (정상 경로):

1. 트리거 도착 (`timer`, `assignment`, `on_demand`, 또는 `automation`).
2. 웨이크업 코디네이터가 웨이크 요청을 큐에 넣거나 병합.
3. 실행기가 요청을 클레임하고, 실행 행을 생성하고, 에이전트를 `running`으로 표시.
4. 어댑터가 실행하고, 상태/로그/사용량 이벤트를 발행.
5. 전체 로그는 `RunLogStore`로 스트리밍; 메타데이터/이벤트는 DB에 영속되고 websocket 구독자에게 푸시.
6. 프로세스 종료, 출력 파서가 실행 결과 + 런타임 상태를 업데이트.
7. 에이전트가 `idle` 또는 `error`로 돌아감; UI가 실시간으로 업데이트.

## 6. 에이전트 실행 프로토콜 (버전 `agent-run/v1`)

이 프로토콜은 런타임에 구애받지 않으며 모든 어댑터에 의해 구현됩니다.

```ts
type RunOutcome = "succeeded" | "failed" | "cancelled" | "timed_out";
type StatusColor = "neutral" | "blue" | "green" | "yellow" | "red";

interface TokenUsage {
  inputTokens: number;
  outputTokens: number;
  cachedInputTokens?: number;
  cachedOutputTokens?: number;
}

interface AdapterInvokeInput {
  protocolVersion: "agent-run/v1";
  companyId: string;
  agentId: string;
  runId: string;
  wakeupSource: "timer" | "assignment" | "on_demand" | "automation";
  triggerDetail?: "manual" | "ping" | "callback" | "system";
  cwd: string;
  prompt: string;
  adapterConfig: Record<string, unknown>;
  runtimeState: Record<string, unknown>;
  env: Record<string, string>;
  timeoutSec: number;
}

interface AdapterHooks {
  status?: (update: { message: string; color?: StatusColor }) => Promise<void>;
  log?: (event: { stream: "stdout" | "stderr" | "system"; chunk: string }) => Promise<void>;
  usage?: (usage: TokenUsage) => Promise<void>;
  event?: (eventType: string, payload: Record<string, unknown>) => Promise<void>;
}

interface AdapterInvokeResult {
  outcome: RunOutcome;
  exitCode: number | null;
  errorMessage?: string | null;
  summary?: string | null;
  sessionId?: string | null;
  usage?: TokenUsage | null;
  provider?: string | null;
  model?: string | null;
  costUsd?: number | null;
  runtimeStatePatch?: Record<string, unknown>;
  rawResult?: Record<string, unknown> | null;
}

interface AgentRunAdapter {
  type: string;
  protocolVersion: "agent-run/v1";
  capabilities: {
    resumableSession: boolean;
    statusUpdates: boolean;
    logStreaming: boolean;
    tokenUsage: boolean;
  };
  validateConfig(config: unknown): { ok: true } | { ok: false; errors: string[] };
  invoke(input: AdapterInvokeInput, hooks: AdapterHooks, signal: AbortSignal): Promise<AdapterInvokeResult>;
}
```

### 6.1 필수 동작

1. `validateConfig`는 저장 또는 호출 전에 실행됩니다.
2. `invoke`는 주어진 설정 + 런타임 상태 + 프롬프트에 대해 결정적이어야 합니다.
3. 어댑터는 DB를 직접 변경해서는 안 됩니다; 결과/이벤트를 통해서만 데이터를 반환합니다.
4. 어댑터는 오류를 디버깅할 수 있는 충분한 컨텍스트를 발행해야 합니다.
5. `invoke`가 예외를 발생시키면, 실행기는 캡처된 오류 텍스트와 함께 실행을 `failed`로 기록합니다.

### 6.2 선택적 동작

어댑터는 상태/로그 훅을 생략할 수 있습니다. 생략하면 런타임은 시스템 수명주기 상태(`queued`, `running`, `finished`)를 발행합니다.

### 6.3 실행 로그 저장 프로토콜

전체 실행 로그는 에이전트 어댑터가 아닌 별도의 플러거블 저장소에 의해 관리됩니다.

```ts
type RunLogStoreType = "local_file" | "object_store" | "postgres";

interface RunLogHandle {
  store: RunLogStoreType;
  logRef: string; // 불투명 제공자 참조 (경로, 키, uri, 행 id)
}

interface RunLogStore {
  begin(input: { companyId: string; agentId: string; runId: string }): Promise<RunLogHandle>;
  append(
    handle: RunLogHandle,
    event: { stream: "stdout" | "stderr" | "system"; chunk: string; ts: string },
  ): Promise<void>;
  finalize(
    handle: RunLogHandle,
    summary: { bytes: number; sha256?: string; compressed: boolean },
  ): Promise<void>;
  read(
    handle: RunLogHandle,
    opts?: { offset?: number; limitBytes?: number },
  ): Promise<{ content: string; nextOffset?: number }>;
  delete?(handle: RunLogHandle): Promise<void>;
}
```

V1 배포 기본값:

1. 개발/로컬 기본값: `local_file` (`data/run-logs/...`에 쓰기).
2. 클라우드/서버리스 기본값: `object_store` (S3/R2/GCS 호환).
3. 선택적 폴백: 엄격한 크기 제한이 있는 `postgres`.

### 6.4 어댑터 ID와 호환성

V1 출시에서 어댑터 ID는 명시적입니다:

- `claude_local`
- `codex_local`
- `process` (기존 범용 동작)
- `http` (기존 범용 동작)

`claude_local`과 `codex_local`은 임의의 `process` 래퍼가 아닙니다; 알려진 파서/재개 의미론을 가진 타입된 어댑터입니다.

## 7. 내장 어댑터 (1단계)

## 7.1 `claude-local`

로컬 `claude` CLI를 직접 실행합니다.

### 설정

```json
{
  "cwd": "/absolute/or/relative/path",
  "promptTemplate": "You are agent {{agent.id}} ...",
  "model": "optional-model-id",
  "maxTurnsPerRun": 1000,
  "dangerouslySkipPermissions": true,
  "env": {"KEY": "VALUE"},
  "extraArgs": [],
  "timeoutSec": 1800,
  "graceSec": 20
}
```

### 호출

- 기본 명령: `claude --print <prompt> --output-format json`
- 재개: 런타임 상태에 세션 ID가 있으면 `--resume <sessionId>` 추가
- 샌드박싱 없는 모드: 활성화 시 `--dangerously-skip-permissions` 추가

### 출력 파싱

1. stdout JSON 객체를 파싱합니다.
2. 재개를 위해 `session_id`를 추출합니다.
3. 사용량 필드를 추출합니다:
   - `usage.input_tokens`
   - `usage.cache_read_input_tokens` (있는 경우)
   - `usage.output_tokens`
4. 있는 경우 `total_cost_usd`를 추출합니다.
5. 비정상 종료 시: 여전히 파싱을 시도합니다; 파싱 성공 시 추출된 상태를 유지하고 어댑터가 명시적으로 성공을 보고하지 않는 한 실행을 실패로 표시합니다.

## 7.2 `codex-local`

로컬 `codex` CLI를 직접 실행합니다.

### 설정

```json
{
  "cwd": "/absolute/or/relative/path",
  "promptTemplate": "You are agent {{agent.id}} ...",
  "model": "optional-model-id",
  "search": false,
  "dangerouslyBypassApprovalsAndSandbox": true,
  "env": {"KEY": "VALUE"},
  "extraArgs": [],
  "timeoutSec": 1800,
  "graceSec": 20
}
```

### 호출

- 기본 명령: `codex exec --json <prompt>`
- 재개 형태: `codex exec --json resume <sessionId> <prompt>`
- 샌드박싱 없는 모드: 활성화 시 `--dangerously-bypass-approvals-and-sandbox` 추가
- 선택적 검색 모드: `--search` 추가

### 출력 파싱

Codex는 JSONL 이벤트를 발행합니다. 줄별로 파싱하고 추출:

1. `thread.started.thread_id` -> 세션 ID
2. 항목 유형이 `agent_message`인 `item.completed` -> 출력 텍스트
3. `turn.completed.usage`:
   - `input_tokens`
   - `cached_input_tokens`
   - `output_tokens`

Codex JSONL은 현재 비용을 포함하지 않을 수 있습니다; 토큰 사용량을 저장하고 사용 가능하지 않으면 비용을 null/unknown으로 둡니다.

## 7.3 공통 로컬 어댑터 프로세스 처리

두 로컬 어댑터 모두:

1. `spawn(command, args, { shell: false, stdio: "pipe" })`를 사용해야 합니다.
2. stdout/stderr를 스트림 청크로 캡처하고 `RunLogStore`로 전달해야 합니다.
3. DB 진단 필드를 위해 메모리에 롤링 stdout/stderr 테일 발췌를 유지해야 합니다.
4. websocket 구독자에게 실시간 로그 이벤트를 발행해야 합니다 (스로틀링/청킹 선택 가능).
5. 우아한 취소를 지원해야 합니다: `SIGTERM`, 그 후 `graceSec` 후 `SIGKILL`.
6. 어댑터 `timeoutSec`을 사용하여 타임아웃을 적용해야 합니다.
7. 종료 코드 + 파싱된 결과 + 진단 stderr를 반환해야 합니다.

## 8. 하트비트 및 웨이크업 코디네이터

## 8.1 웨이크업 소스

지원되는 소스:

1. `timer`: 에이전트별 주기적 하트비트.
2. `assignment`: 에이전트에게 이슈가 할당/재할당됨.
3. `on_demand`: 명시적 웨이크 요청 경로 (보드/수동 클릭 또는 API 핑).
4. `automation`: 비대화형 웨이크 경로 (외부 콜백 또는 내부 시스템 자동화).

## 8.2 중앙 API

모든 소스가 하나의 내부 서비스를 호출합니다:

```ts
enqueueWakeup({
  companyId,
  agentId,
  source,
  triggerDetail, // 선택: manual|ping|callback|system
  reason,
  payload,
  requestedBy,
  idempotencyKey?
})
```

어떤 소스도 어댑터를 직접 호출하지 않습니다.

## 8.3 큐 의미론

1. 에이전트당 최대 활성 실행은 `1`로 유지됩니다.
2. 에이전트에 이미 `queued`/`running` 실행이 있는 경우:
   - 중복 웨이크업을 통합
   - `coalescedCount`를 증가
   - 최신 이유/소스 메타데이터를 보존
3. 큐는 재시작 안전을 위해 DB 기반입니다.
4. 코디네이터는 `requested_at`에 의한 FIFO를 사용하며, 선택적 우선순위:
   - `on_demand` > `assignment` > `timer`/`automation`

## 8.4 에이전트 하트비트 정책 필드

에이전트 수준 컨트롤 플레인 설정 (어댑터별이 아님):

```json
{
  "heartbeat": {
    "enabled": true,
    "intervalSec": 300,
    "wakeOnAssignment": true,
    "wakeOnOnDemand": true,
    "wakeOnAutomation": true,
    "cooldownSec": 10
  }
}
```

기본값:

- `enabled: true`
- `intervalSec: null` (명시적으로 설정될 때까지 타이머 없음) 또는 전역적으로 원하는 경우 제품 기본값 `300`
- `wakeOnAssignment: true`
- `wakeOnOnDemand: true`
- `wakeOnAutomation: true`

## 8.5 트리거 통합 규칙

1. 타이머 검사는 서버 워커 간격으로 실행되고 기한이 된 에이전트를 큐에 넣습니다.
2. 이슈 할당 변경은 담당자가 변경되고 대상 에이전트가 `wakeOnAssignment=true`인 경우 웨이크업을 큐에 넣습니다.
3. 온디맨드 엔드포인트는 `wakeOnOnDemand=true`인 경우 `source=on_demand`와 `triggerDetail=manual|ping`으로 웨이크업을 큐에 넣습니다.
4. 콜백/시스템 자동화는 `wakeOnAutomation=true`인 경우 `source=automation`과 `triggerDetail=callback|system`으로 웨이크업을 큐에 넣습니다.
5. 일시 중지/종료된 에이전트는 새 웨이크업을 받지 않습니다.
6. 하드 예산 중지된 에이전트는 새 웨이크업을 받지 않습니다.

## 9. 영속성 모델

모든 테이블은 회사 범위로 유지됩니다.

## 9.0 `agents` 변경사항

1. `adapter_type` 도메인을 확장하여 `claude_local`과 `codex_local`을 포함합니다 (기존 `process`, `http`와 함께).
2. `adapter_config`를 어댑터 소유 구성(CLI 플래그, cwd, 프롬프트 템플릿, 환경 오버라이드)으로 유지합니다.
3. 컨트롤 플레인 스케줄링 정책을 위한 `runtime_config` jsonb를 추가합니다:
   - 하트비트 활성화/간격
   - 할당 시 깨우기
   - 온디맨드 시 깨우기
   - 자동화 시 깨우기
   - 쿨다운

이 분리는 하트비트 서비스가 일관된 스케줄링 로직을 적용할 수 있게 하면서 어댑터 설정을 런타임에 구애받지 않게 유지합니다.

## 9.1 새 테이블: `agent_runtime_state`

집계 런타임 카운터와 레거시 호환성을 위한 에이전트당 하나의 행.

- `agent_id` uuid pk fk `agents.id`
- `company_id` uuid fk not null
- `adapter_type` text not null
- `session_id` text null
- `state_json` jsonb not null default `{}`
- `last_run_id` uuid fk `heartbeat_runs.id` null
- `last_run_status` text null
- `total_input_tokens` bigint not null default `0`
- `total_output_tokens` bigint not null default `0`
- `total_cached_input_tokens` bigint not null default `0`
- `total_cost_cents` bigint not null default `0`
- `last_error` text null
- `updated_at` timestamptz not null

불변성: 에이전트당 정확히 하나의 런타임 상태 행.

## 9.1.1 새 테이블: `agent_task_sessions`

재개 가능한 세션 상태를 위한 `(company_id, agent_id, adapter_type, task_key)`당 하나의 행.

- `id` uuid pk
- `company_id` uuid fk not null
- `agent_id` uuid fk not null
- `adapter_type` text not null
- `task_key` text not null
- `session_params_json` jsonb null (어댑터 정의 형태)
- `session_display_id` text null (UI/디버그용)
- `last_run_id` uuid fk `heartbeat_runs.id` null
- `last_error` text null
- `created_at` timestamptz not null
- `updated_at` timestamptz not null

불변성: `(company_id, agent_id, adapter_type, task_key)` 고유.

## 9.2 새 테이블: `agent_wakeup_requests`

웨이크업을 위한 큐 + 감사.

- `id` uuid pk
- `company_id` uuid fk not null
- `agent_id` uuid fk not null
- `source` text not null (`timer|assignment|on_demand|automation`)
- `trigger_detail` text null (`manual|ping|callback|system`)
- `reason` text null
- `payload` jsonb null
- `status` text not null (`queued|claimed|coalesced|skipped|completed|failed|cancelled`)
- `coalesced_count` int not null default `0`
- `requested_by_actor_type` text null (`user|agent|system`)
- `requested_by_actor_id` text null
- `idempotency_key` text null
- `run_id` uuid fk `heartbeat_runs.id` null
- `requested_at` timestamptz not null
- `claimed_at` timestamptz null
- `finished_at` timestamptz null
- `error` text null

## 9.3 새 테이블: `heartbeat_run_events`

실행별 경량 이벤트 타임라인 추가 전용 (전체 원시 로그 청크 아님).

- `id` bigserial pk
- `company_id` uuid fk not null
- `run_id` uuid fk `heartbeat_runs.id` not null
- `agent_id` uuid fk `agents.id` not null
- `seq` int not null
- `event_type` text not null (`lifecycle|status|usage|error|structured`)
- `stream` text null (`system|stdout|stderr`) (요약된 이벤트만, 전체 스트림 청크 아님)
- `level` text null (`info|warn|error`)
- `color` text null
- `message` text null
- `payload` jsonb null
- `created_at` timestamptz not null

## 9.4 `heartbeat_runs` 변경사항

결과 및 진단에 필요한 필드 추가:

- `wakeup_request_id` uuid fk `agent_wakeup_requests.id` null
- `exit_code` int null
- `signal` text null
- `usage_json` jsonb null
- `result_json` jsonb null
- `session_id_before` text null
- `session_id_after` text null
- `log_store` text null (`local_file|object_store|postgres`)
- `log_ref` text null (불투명 제공자 참조; 경로/키/uri/행 id)
- `log_bytes` bigint null
- `log_sha256` text null
- `log_compressed` boolean not null default false
- `stderr_excerpt` text null
- `stdout_excerpt` text null
- `error_code` text null

전체 로그를 Postgres에 저장하지 않으면서 실행별 진단을 쿼리 가능하게 유지합니다.

## 9.5 로그 저장소 어댑터 설정

런타임 로그 저장소는 배포 수준에서 설정됩니다 (기본적으로 에이전트별이 아님).

```json
{
  "runLogStore": {
    "type": "local_file | object_store | postgres",
    "basePath": "./data/run-logs",
    "bucket": "paperclip-run-logs",
    "prefix": "runs/",
    "compress": true,
    "maxInlineExcerptBytes": 32768
  }
}
```

규칙:

1. `log_ref`는 API 경계에서 불투명하고 제공자 중립이어야 합니다.
2. UI/API는 로컬 파일시스템 의미론을 가정하면 안 됩니다.
3. 제공자별 비밀/자격 증명은 에이전트 설정이 아닌 서버 설정에 유지됩니다.

## 10-16. (이후 섹션은 프롬프트 템플릿 시스템, 실시간 상태 전달, 오류 처리, API 변경, 구현 계획, 수락 기준, 미해결 질문을 다룹니다 - 원본 문서의 전체 내용은 영문 원본을 참조하세요)

## 15. 수락 기준

1. `claude-local` 또는 `codex-local`이 있는 에이전트가 실행하고, 종료하고, 실행 결과를 영속할 수 있습니다.
2. 세션 매개변수가 작업 범위별로 영속되고 동일 작업 재개 시 자동으로 재사용됩니다.
3. 토큰 사용량이 실행별로 영속되고 에이전트 런타임 상태별로 누적됩니다.
4. 타이머, 할당, 온디맨드, 자동화 웨이크업 모두 하나의 코디네이터를 통해 큐에 들어갑니다.
5. 일시 중지/종료가 실행 중인 로컬 프로세스를 중단하고 새 웨이크업을 방지합니다.
6. 브라우저가 실행 상태/로그 및 작업/에이전트 변경에 대한 실시간 websocket 업데이트를 받습니다.
7. 실패한 실행이 발췌가 즉시 사용 가능하고 전체 로그가 `RunLogStore`를 통해 검색 가능한 풍부한 CLI 진단을 UI에 노출합니다.
8. 모든 동작이 회사 범위 지정되고 감사 가능하게 유지됩니다.

## 16. 미해결 질문

1. 타이머 기본값은 `null` (활성화될 때까지 꺼짐)이어야 하나, 기본 `300`초여야 하나?
2. 전체 로그 객체와 Postgres 메타데이터의 기본 보관 정책은 어떻게 되어야 하나?
3. 에이전트 API 자격 증명이 프롬프트 템플릿에서 기본적으로 허용되어야 하나, 명시적 옵트인 토글을 요구해야 하나?
4. Websocket이 유일한 실시간 채널이어야 하나, 더 단순한 클라이언트를 위해 SSE도 노출해야 하나?
