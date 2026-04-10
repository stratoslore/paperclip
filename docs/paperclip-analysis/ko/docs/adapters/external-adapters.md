---
title: 외부 어댑터
summary: Paperclip 소스를 수정하지 않고 플러그인으로 어댑터를 구축, 패키징 및 배포
---

Paperclip은 npm 패키지나 로컬 디렉토리에서 설치할 수 있는 외부 어댑터 플러그인을 지원합니다. 외부 어댑터는 내장 어댑터와 동일하게 작동합니다 -- 에이전트를 실행하고, 출력을 파싱하고, 트랜스크립트를 렌더링합니다 -- 하지만 자체 패키지에 존재하며 Paperclip의 소스 코드를 변경할 필요가 없습니다.

## 내장형 vs 외부

| | 내장형 | 외부 |
|---|---|---|
| 소스 위치 | `paperclip-fork/packages/adapters/` 내부 | 별도의 npm 패키지 또는 로컬 디렉토리 |
| 등록 | 세 가지 레지스트리에 하드코딩 | 플러그인 시스템을 통해 시작 시 로드 |
| UI 파서 | 빌드 시 정적 import | API에서 동적으로 로드 ([UI Parser](/adapters/adapter-ui-parser) 참조) |
| 배포 | Paperclip과 함께 제공 | npm에 배포 또는 `file:`로 링크 |
| 업데이트 | Paperclip 릴리스 필요 | 독립적인 버전 관리 |

## 빠른 시작

### 최소 패키지 구조

```
my-adapter/
  package.json
  tsconfig.json
  src/
    index.ts            # Shared metadata (type, label, models)
    server/
      index.ts          # createServerAdapter() factory
      execute.ts        # Core execution logic
      parse.ts          # Output parsing
      test.ts           # Environment diagnostics
    ui-parser.ts        # Self-contained UI transcript parser
```

### package.json

```json
{
  "name": "my-paperclip-adapter",
  "version": "1.0.0",
  "type": "module",
  "license": "MIT",
  "paperclip": {
    "adapterUiParser": "1.0.0"
  },
  "exports": {
    ".": "./dist/index.js",
    "./server": "./dist/server/index.js",
    "./ui-parser": "./dist/ui-parser.js"
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc"
  },
  "dependencies": {
    "@paperclipai/adapter-utils": "^2026.325.0",
    "picocolors": "^1.1.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "typescript": "^5.7.0"
  }
}
```

주요 필드:

| 필드 | 용도 |
|-------|---------|
| `exports["."]` | 진입점 -- `createServerAdapter`를 내보내야 함 |
| `exports["./ui-parser"]` | 독립형 UI 파서 모듈 (선택 사항이지만 권장) |
| `paperclip.adapterUiParser` | UI 파서의 계약 버전 (`"1.0.0"`) |
| `files` | 배포 대상 제한 -- `dist/`만 포함 |

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

## 서버 모듈

플러그인 로더는 패키지 루트에서 `createServerAdapter()`를 호출합니다. 이 함수는 `ServerAdapterModule`을 반환해야 합니다.

### src/index.ts

```ts
export const type = "my_adapter";     // snake_case, globally unique
export const label = "My Agent (local)";

export const models = [
  { id: "model-a", label: "Model A" },
];

export const agentConfigurationDoc = `# my_adapter configuration
Use when: ...
Don't use when: ...
`;

// Required by plugin-loader convention
export { createServerAdapter } from "./server/index.js";
```

### src/server/index.ts

```ts
import type { ServerAdapterModule } from "@paperclipai/adapter-utils";
import { type, models, agentConfigurationDoc } from "../index.js";
import { execute } from "./execute.js";
import { testEnvironment } from "./test.js";

export function createServerAdapter(): ServerAdapterModule {
  return {
    type,
    execute,
    testEnvironment,
    models,
    agentConfigurationDoc,
  };
}
```

### src/server/execute.ts

핵심 실행 함수입니다. `AdapterExecutionContext`를 받아 `AdapterExecutionResult`를 반환합니다.

```ts
import type {
  AdapterExecutionContext,
  AdapterExecutionResult,
} from "@paperclipai/adapter-utils";

import {
  runChildProcess,
  buildPaperclipEnv,
  renderTemplate,
} from "@paperclipai/adapter-utils/server-utils";

export async function execute(
  ctx: AdapterExecutionContext,
): Promise<AdapterExecutionResult> {
  const { config, agent, runtime, context, onLog, onMeta } = ctx;

  // 1. Read config with safe helpers
  const cwd = String(config.cwd ?? "/tmp");
  const command = String(config.command ?? "my-agent");
  const timeoutSec = Number(config.timeoutSec ?? 300);

  // 2. Build environment with Paperclip vars injected
  const env = buildPaperclipEnv(agent);

  // 3. Render prompt template
  const prompt = config.promptTemplate
    ? renderTemplate(String(config.promptTemplate), {
        agentId: agent.id,
        agentName: agent.name,
        companyId: agent.companyId,
        runId: ctx.runId,
        taskId: context.taskId ?? "",
        taskTitle: context.taskTitle ?? "",
      })
    : "Continue your work.";

  // 4. Spawn process
  const result = await runChildProcess(command, {
    args: [prompt],
    cwd,
    env,
    timeout: timeoutSec * 1000,
    graceMs: 10_000,
    onStdout: (chunk) => onLog("stdout", chunk),
    onStderr: (chunk) => onLog("stderr", chunk),
  });

  // 5. Return structured result
  return {
    exitCode: result.exitCode,
    timedOut: result.timedOut,
    // Include session state for persistence
    sessionParams: { /* ... */ },
  };
}
```

#### `@paperclipai/adapter-utils`의 사용 가능한 헬퍼

| 헬퍼 | 용도 |
|--------|---------|
| `runChildProcess(command, opts)` | 타임아웃, 유예 기간, 스트리밍 콜백과 함께 자식 프로세스 생성 |
| `buildPaperclipEnv(agent)` | `PAPERCLIP_*` 환경 변수 주입 |
| `renderTemplate(template, data)` | 프롬프트 템플릿에서 `{{variable}}` 치환 |
| `asString(v)`, `asNumber(v)`, `asBoolean(v)` | 안전한 설정 값 추출 |

### src/server/test.ts

실행 전에 어댑터 구성을 검증합니다. 구조화된 진단 결과를 반환합니다.

```ts
import type {
  AdapterEnvironmentTestContext,
  AdapterEnvironmentTestResult,
} from "@paperclipai/adapter-utils";

export async function testEnvironment(
  ctx: AdapterEnvironmentTestContext,
): Promise<AdapterEnvironmentTestResult> {
  const checks = [];

  // Example: check CLI is installed
  checks.push({
    level: "info",
    message: "My Agent CLI v1.2.0 detected",
    code: "cli_detected",
  });

  // Example: check working directory
  const cwd = String(ctx.config.cwd ?? "");
  if (!cwd.startsWith("/")) {
    checks.push({
      level: "error",
      message: `Working directory must be absolute: "${cwd}"`,
      hint: "Use /home/user/project or /workspace",
      code: "invalid_cwd",
    });
  }

  return {
    adapterType: ctx.adapterType,
    status: checks.some(c => c.level === "error") ? "fail" : "pass",
    checks,
    testedAt: new Date().toISOString(),
  };
}
```

체크 레벨:

| 레벨 | 의미 | 효과 |
|-------|---------|--------|
| `info` | 정보성 | 테스트 결과에 표시 |
| `warn` | 차단하지 않는 문제 | 노란색 표시기로 표시 |
| `error` | 실행 차단 | 에이전트 실행 방지 |

## 설치

### npm에서

```sh
# Via the Paperclip UI
# Settings → Adapters → Install from npm → "my-paperclip-adapter"

# Or via API
curl -X POST http://localhost:3102/api/adapters \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"packageName": "my-paperclip-adapter"}'
```

### 로컬 디렉토리에서

```sh
curl -X POST http://localhost:3102/api/adapters \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"localPath": "/home/user/my-adapter"}'
```

로컬 어댑터는 Paperclip의 어댑터 디렉토리에 심볼릭 링크됩니다. 소스 변경 사항은 서버 재시작 시 반영됩니다.

### adapter-plugins.json을 통해

개발 시 `~/.paperclip/adapter-plugins.json`을 직접 편집할 수도 있습니다:

```json
[
  {
    "packageName": "my-paperclip-adapter",
    "localPath": "/home/user/my-adapter",
    "type": "my_adapter",
    "installedAt": "2026-03-30T12:00:00.000Z"
  }
]
```

## 선택 사항: 세션 지속성

에이전트 런타임이 세션(하트비트 간 대화 연속성)을 지원하는 경우, 세션 코덱을 구현하세요:

```ts
import type { AdapterSessionCodec } from "@paperclipai/adapter-utils";

export const sessionCodec: AdapterSessionCodec = {
  deserialize(raw) {
    if (typeof raw !== "object" || raw === null) return null;
    const r = raw as Record<string, unknown>;
    return r.sessionId ? { sessionId: String(r.sessionId) } : null;
  },
  serialize(params) {
    return params?.sessionId ? { sessionId: String(params.sessionId) } : null;
  },
  getDisplayId(params) {
    return params?.sessionId ? String(params.sessionId) : null;
  },
};
```

`createServerAdapter()`에 포함합니다:

```ts
return { type, execute, testEnvironment, sessionCodec, /* ... */ };
```

## 선택 사항: 스킬 동기화

에이전트 런타임이 스킬/플러그인을 지원하는 경우, `listSkills`와 `syncSkills`를 구현하세요:

```ts
return {
  type,
  execute,
  testEnvironment,
  async listSkills(ctx) {
    return {
      adapterType: ctx.adapterType,
      supported: true,
      mode: "ephemeral",
      desiredSkills: [],
      entries: [],
      warnings: [],
    };
  },
  async syncSkills(ctx, desiredSkills) {
    // Install desired skills into the runtime
    return { /* same shape as listSkills */ };
  },
};
```

## 선택 사항: 모델 감지

런타임에 기본 모델을 지정하는 로컬 설정 파일이 있는 경우:

```ts
async function detectModel() {
  // Read ~/.my-agent/config.yaml or similar
  return {
    model: "anthropic/claude-sonnet-4",
    provider: "anthropic",
    source: "~/.my-agent/config.yaml",
    candidates: ["anthropic/claude-sonnet-4", "openai/gpt-4o"],
  };
}

return { type, execute, testEnvironment, detectModel: () => detectModel() };
```

## 배포

```sh
npm run build
npm publish
```

다른 Paperclip 사용자는 UI 또는 API에서 패키지 이름으로 어댑터를 설치할 수 있습니다.

## 보안

- 에이전트 출력을 신뢰하지 않는 것으로 취급합니다 -- 방어적으로 파싱하고, 절대 에이전트 출력을 `eval()`하지 마세요
- 프롬프트가 아닌 환경 변수를 통해 시크릿을 주입합니다
- 런타임이 지원하는 경우 네트워크 접근 제어를 구성합니다
- 항상 타임아웃과 유예 기간을 적용합니다 -- 에이전트가 무한히 실행되게 두지 마세요
- UI 파서 모듈은 브라우저 샌드박스에서 실행됩니다 -- 런타임 import 없음, 부작용 없음이어야 합니다

## 다음 단계

- [UI Parser Contract](/adapters/adapter-ui-parser) -- UI가 어댑터의 출력을 올바르게 렌더링하도록 커스텀 실행 로그 파서 추가
- [Creating an Adapter](/adapters/creating-an-adapter) -- 어댑터 내부 구조의 전체 안내
- [How Agents Work](/guides/agent-developer/how-agents-work) -- 어댑터가 제공하는 하트비트 라이프사이클 이해
