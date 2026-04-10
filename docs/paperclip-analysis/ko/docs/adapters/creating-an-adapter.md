---
title: 어댑터 만들기
summary: 커스텀 어댑터 구축 가이드
---

Paperclip을 모든 에이전트 런타임에 연결하기 위한 커스텀 어댑터를 만드세요.

<Tip>
Claude Code를 사용하는 경우, `.agents/skills/create-agent-adapter` 스킬이 전체 어댑터 생성 과정을 대화형으로 안내할 수 있습니다. Claude에게 새 어댑터를 만들어달라고 요청하면 각 단계를 안내해 줍니다.
</Tip>

## 두 가지 경로

| | 내장형 | 외부 플러그인 |
|---|---|---|
| 소스 | `paperclip-fork` 내부 | 별도의 npm 패키지 |
| 배포 | Paperclip과 함께 제공 | 독립적인 npm 배포 |
| UI 파서 | 정적 import | API에서 동적 로드 |
| 등록 | 3개의 레지스트리 편집 | 시작 시 자동 로드 |
| 적합한 경우 | 핵심 어댑터, 기여자 | 서드파티 어댑터, 내부 도구 |

대부분의 경우 **외부 어댑터 플러그인을 구축**하세요. 더 깔끔하고, 독립적으로 버전 관리되며, Paperclip의 소스를 수정할 필요가 없습니다. 전체 가이드는 [External Adapters](/adapters/external-adapters)를 참조하세요.

이 페이지의 나머지 부분은 두 경로 모두에서 사용하는 공통 내부 구조를 다룹니다.

## 패키지 구조

```
packages/adapters/<name>/    # built-in
  ── or ──
my-adapter/                   # external plugin
  package.json
  tsconfig.json
  src/
    index.ts            # Shared metadata
    server/
      index.ts          # Server exports (createServerAdapter)
      execute.ts        # Core execution logic
      parse.ts          # Output parsing
      test.ts           # Environment diagnostics
    ui/
      index.ts          # UI exports (built-in only)
      parse-stdout.ts   # Transcript parser (built-in only)
      build-config.ts   # Config builder
    ui-parser.ts        # Self-contained UI parser (external — see [UI Parser Contract](/adapters/adapter-ui-parser))
    cli/
      index.ts          # CLI exports
      format-event.ts   # Terminal formatter
```

## 1단계: 루트 메타데이터

`src/index.ts`는 세 가지 소비자 모두에서 가져옵니다. 의존성 없이 유지하세요.

```ts
export const type = "my_agent";        // snake_case, globally unique
export const label = "My Agent (local)";
export const models = [
  { id: "model-a", label: "Model A" },
];
export const agentConfigurationDoc = `# my_agent configuration
Use when: ...
Don't use when: ...
Core fields: ...
`;

// Required for external adapters (plugin-loader convention)
export { createServerAdapter } from "./server/index.js";
```

## 2단계: 서버 실행

`src/server/execute.ts`가 핵심입니다. `AdapterExecutionContext`를 받아 `AdapterExecutionResult`를 반환합니다.

주요 역할:

1. `@paperclipai/adapter-utils/server-utils`의 안전한 헬퍼(`asString`, `asNumber` 등)를 사용하여 설정 읽기
2. `buildPaperclipEnv(agent)`와 컨텍스트 변수로 환경 구축
3. `runtime.sessionParams`에서 세션 상태 해석
4. `renderTemplate(template, data)`로 프롬프트 렌더링
5. `runChildProcess()`로 프로세스 생성 또는 `fetch()`로 호출
6. 사용량, 비용, 세션 상태, 오류에 대한 출력 파싱
7. 알 수 없는 세션 오류 처리 (새 세션으로 재시도, `clearSession: true` 설정)

### 사용 가능한 헬퍼

| 헬퍼 | 출처 | 용도 |
|--------|--------|---------|
| `runChildProcess(cmd, opts)` | `@paperclipai/adapter-utils/server-utils` | 타임아웃, 유예, 스트리밍과 함께 생성 |
| `buildPaperclipEnv(agent)` | `@paperclipai/adapter-utils/server-utils` | `PAPERCLIP_*` 환경 변수 주입 |
| `renderTemplate(tpl, data)` | `@paperclipai/adapter-utils/server-utils` | `{{variable}}` 치환 |
| `asString(v)` | `@paperclipai/adapter-utils` | 안전한 설정 값 추출 |
| `asNumber(v)` | `@paperclipai/adapter-utils` | 안전한 숫자 추출 |

### AdapterExecutionContext

```ts
interface AdapterExecutionContext {
  runId: string;
  agent: { id: string; companyId: string; name: string; adapterConfig: unknown };
  runtime: { sessionId: string | null; sessionParams: Record<string, unknown> | null };
  config: Record<string, unknown>;      // agent's adapterConfig
  context: Record<string, unknown>;      // task, wake reason, etc.
  onLog: (stream: "stdout" | "stderr", chunk: string) => Promise<void>;
  onMeta?: (meta: AdapterInvocationMeta) => Promise<void>;
  onSpawn?: (meta: { pid: number; startedAt: string }) => Promise<void>;
}
```

### AdapterExecutionResult

```ts
interface AdapterExecutionResult {
  exitCode: number | null;
  signal: string | null;
  timedOut: boolean;
  errorMessage?: string | null;
  usage?: { inputTokens: number; outputTokens: number };
  sessionParams?: Record<string, unknown> | null;  // persist across heartbeats
  sessionDisplayId?: string | null;
  provider?: string | null;
  model?: string | null;
  costUsd?: number | null;
  clearSession?: boolean;  // set true to force fresh session on next wake
}
```

## 3단계: 환경 테스트

`src/server/test.ts`는 실행 전에 어댑터 구성을 검증합니다.

구조화된 진단 결과를 반환합니다:

| 레벨 | 의미 | 효과 |
|-------|---------|--------|
| `error` | 유효하지 않거나 사용 불가능한 설정 | 실행 차단 |
| `warn` | 차단하지 않는 문제 | 노란색 표시기로 표시 |
| `info` | 성공적인 확인 | 테스트 결과에 표시 |

```ts
export async function testEnvironment(
  ctx: AdapterEnvironmentTestContext,
): Promise<AdapterEnvironmentTestResult> {
  return {
    adapterType: ctx.adapterType,
    status: "pass",  // "pass" | "warn" | "fail"
    checks: [
      { level: "info", message: "CLI v1.2.0 detected", code: "cli_detected" },
      { level: "warn", message: "No API key found", hint: "Set ANTHROPIC_API_KEY", code: "no_key" },
    ],
    testedAt: new Date().toISOString(),
  };
}
```

## 4단계: UI 모듈 (내장형 전용)

Paperclip 소스에 등록된 내장 어댑터의 경우:

- `parse-stdout.ts` -- stdout 라인을 실행 뷰어용 `TranscriptEntry[]`로 변환
- `build-config.ts` -- 폼 값을 `adapterConfig` JSON으로 변환
- `ui/src/adapters/<name>/config-fields.tsx`의 설정 필드 React 컴포넌트

외부 어댑터의 경우, 독립형 `ui-parser.ts`를 대신 사용하세요. [UI Parser Contract](/adapters/adapter-ui-parser)를 참조하세요.

## 5단계: CLI 모듈

`format-event.ts` -- `picocolors`를 사용하여 `paperclipai run --watch`에서 stdout를 보기 좋게 출력합니다.

```ts
export function formatStdoutEvent(line: string, debug: boolean): void {
  if (line.startsWith("[tool-done]")) {
    console.log(chalk.green(`  ✓ ${line}`));
  } else {
    console.log(`  ${line}`);
  }
}
```

## 6단계: 등록 (내장형 전용)

세 가지 레지스트리 모두에 어댑터를 추가합니다:

1. `server/src/adapters/registry.ts`
2. `ui/src/adapters/registry.ts`
3. `cli/src/adapters/registry.ts`

외부 어댑터의 경우 등록은 자동입니다 -- 플러그인 로더가 처리합니다.

## 세션 지속성

에이전트 런타임이 하트비트 간 대화 연속성을 지원하는 경우:

1. `execute()`에서 `sessionParams`를 반환합니다 (예: `{ sessionId: "abc123" }`)
2. 다음 깨어남 시 `runtime.sessionParams`를 읽어 재개합니다
3. 선택적으로 검증 및 표시를 위한 `sessionCodec`을 구현합니다

```ts
export const sessionCodec: AdapterSessionCodec = {
  deserialize(raw) { /* validate raw session data */ },
  serialize(params) { /* serialize for storage */ },
  getDisplayId(params) { /* human-readable session label */ },
};
```

## 스킬 주입

에이전트의 작업 디렉토리에 쓰지 않고 에이전트 런타임에서 Paperclip 스킬을 검색할 수 있게 합니다:

1. **최선: tmpdir + 플래그** -- tmpdir 생성, 스킬 심볼릭 링크, CLI 플래그로 전달, 완료 후 정리
2. **허용: 글로벌 설정 디렉토리** -- 런타임의 글로벌 플러그인 디렉토리에 심볼릭 링크
3. **허용: 환경 변수** -- 스킬 경로 환경 변수를 저장소의 `skills/` 디렉토리로 지정
4. **최후의 수단: 프롬프트 주입** -- 프롬프트 템플릿에 스킬 내용 포함

## 보안

- 에이전트 출력을 신뢰하지 않는 것으로 취급합니다 (방어적으로 파싱, 절대 실행하지 않음)
- 프롬프트가 아닌 환경 변수를 통해 시크릿을 주입합니다
- 런타임이 지원하는 경우 네트워크 접근 제어를 구성합니다
- 항상 타임아웃과 유예 기간을 적용합니다
- UI 파서 모듈은 브라우저 샌드박스에서 실행됩니다 -- 런타임 import 없음, 부작용 없음

## 다음 단계

- [External Adapters](/adapters/external-adapters) -- 독립형 어댑터 플러그인 구축
- [UI Parser Contract](/adapters/adapter-ui-parser) -- 커스텀 실행 로그 파서 제공
- [How Agents Work](/guides/agent-developer/how-agents-work) -- 하트비트 라이프사이클
