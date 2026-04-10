---
title: Adapter UI Parser 계약
summary: Paperclip UI가 Adapter의 출력을 올바르게 렌더링하도록 커스텀 실행 로그 파서를 제공하세요
---

Paperclip이 에이전트를 실행하면, stdout이 실시간으로 UI에 스트리밍됩니다. UI는 원시 stdout 라인을 구조화된 트랜스크립트 항목(tool call, tool result, assistant 메시지, 시스템 이벤트)으로 변환하기 위해 **파서**가 필요합니다. 커스텀 파서가 없으면, UI는 모든 비시스템 라인을 `assistant` 출력으로 처리하는 범용 shell 파서로 대체됩니다 — tool 명령이 일반 텍스트로 노출되고, 소요 시간이 사라지고, 에러가 보이지 않습니다.

## 문제

대부분의 에이전트 CLI는 tool call, 진행 표시기, 다중 라인 출력이 포함된 구조화된 stdout을 출력합니다. 예를 들어:

```
[hermes] Session resumed: abc123
┊ 💬 Thinking about how to approach this...
┊ $ ls /home/user/project
┊ [done] $ ls /home/user/project — /src /README.md  0.3s
┊ 💬 I see the project structure. Let me read the README.
┊ read /home/user/project/README.md
┊ [done] read — Project Overview: A CLI tool for...  1.2s
The project is a CLI tool. Here's what I found:
- It uses TypeScript
- Tests are in /tests
```

파서가 없으면, UI는 이 모든 것을 원시 `assistant` 텍스트로 표시합니다 — tool call과 결과가 에이전트의 실제 응답과 구별할 수 없습니다.

파서가 있으면, UI는 다음과 같이 렌더링합니다:

- `Thinking about how to approach this...`를 접을 수 있는 thinking 블록으로
- `$ ls /home/user/project`를 tool call 카드(접힌 상태)로
- `0.3s` 소요 시간을 tool result 카드로
- `The project is a CLI tool...`을 assistant의 응답으로

## 작동 방식

```
┌──────────────────┐     package.json        ┌──────────────────┐
│  Adapter Package  │─── exports["./ui-parser"] ──→│  dist/ui-parser.js │
│  (npm / local)    │                          │  (zero imports)  │
└──────────────────┘                          └────────┬─────────┘
                                                       │ plugin-loader reads at startup
                                                       ▼
┌──────────────────┐   GET /api/:type/ui-parser.js   ┌──────────────────┐
│  Paperclip Server  │◄────────────────────────────────│  uiParserCache    │
│  (in-memory)      │                                 └──────────────────┘
└────────┬─────────┘
         │ serves JS to browser
         ▼
┌──────────────────┐   fetch() + eval   ┌──────────────────┐
│  Paperclip UI     │─────────────────────→│  parseStdoutLine │
│  (dynamic loader) │   registers parser  │  (per-adapter)   │
└──────────────────┘                     └──────────────────┘
```

1. **빌드 시** — `src/ui-parser.ts`를 `dist/ui-parser.js`로 컴파일합니다 (런타임 import 없음)
2. **서버 시작 시** — 플러그인 로더가 파일을 읽고 메모리에 캐시합니다
3. **UI 로드 시** — 사용자가 실행을 열면, UI가 `GET /api/:type/ui-parser.js`에서 파서를 가져옵니다
4. **런타임** — 가져온 모듈이 eval되고 등록됩니다. 이후 모든 라인은 실제 파서를 사용합니다

## 계약: package.json

### 1. `paperclip.adapterUiParser` — 계약 버전

```json
{
  "paperclip": {
    "adapterUiParser": "1.0.0"
  }
}
```

Paperclip 호스트가 이 필드를 확인합니다. 주 버전이 지원되지 않으면, 호스트는 경고를 로그에 남기고 잠재적으로 호환되지 않는 코드를 실행하는 대신 범용 파서로 대체합니다.

| 호스트 기대 | Adapter 선언 | 결과 |
|---|---|---|
| `1.x` | `1.0.0` | 파서 로드 |
| `1.x` | `2.0.0` | 경고 로그, 범용 파서 사용 |
| `1.x` | (없음) | 파서 로드 (유예 기간 — 향후 버전에서 필수가 될 수 있음) |

### 2. `exports["./ui-parser"]` — 파일 경로

```json
{
  "exports": {
    ".": "./dist/server/index.js",
    "./ui-parser": "./dist/ui-parser.js"
  }
}
```

## 계약: 모듈 내보내기

`dist/ui-parser.js`는 **최소 하나**를 내보내야 합니다:

### `parseStdoutLine(line: string, ts: string): TranscriptEntry[]`

정적 파서. Adapter stdout의 각 라인에 대해 호출됩니다.

```ts
export function parseStdoutLine(line: string, ts: string): TranscriptEntry[] {
  if (line.startsWith("[my-agent]")) {
    return [{ kind: "system", ts, text: line }];
  }
  return [{ kind: "assistant", ts, text: line }];
}
```

### `createStdoutParser(): { parseLine(line, ts): TranscriptEntry[]; reset(): void }`

상태 유지 파서 팩토리. 다중 라인 연속, 명령 중첩, 또는 기타 호출 간 상태를 추적해야 하는 경우 선호됩니다.

```ts
let counter = 0;

export function createStdoutParser() {
  let suppressContinuation = false;

  function parseLine(line: string, ts: string): TranscriptEntry[] {
    const trimmed = line.trim();
    if (!trimmed) return [];

    if (suppressContinuation) {
      if (/^[\d.]+s$/.test(trimmed)) {
        suppressContinuation = false;
        return [];
      }
      return []; // swallow continuation lines
    }

    if (trimmed.startsWith("[tool-done]")) {
      const id = `tool-${++counter}`;
      suppressContinuation = true;
      return [
        { kind: "tool_call", ts, name: "shell", input: {}, toolUseId: id },
        { kind: "tool_result", ts, toolUseId: id, content: trimmed, isError: false },
      ];
    }

    return [{ kind: "assistant", ts, text: trimmed }];
  }

  function reset() {
    suppressContinuation = false;
  }

  return { parseLine, reset };
}
```

두 가지 모두 내보내면, `createStdoutParser`가 우선합니다.

## 계약: TranscriptEntry

각 항목은 다음 구별 유니온 형태 중 하나와 일치해야 합니다:

```ts
// Assistant 메시지
{ kind: "assistant"; ts: string; text: string; delta?: boolean }

// Thinking / 추론
{ kind: "thinking"; ts: string; text: string; delta?: boolean }

// 사용자 메시지 (드물게 — 보통 에이전트가 시작한 프롬프트에서)
{ kind: "user"; ts: string; text: string }

// Tool 호출
{ kind: "tool_call"; ts: string; name: string; input: unknown; toolUseId?: string }

// Tool 결과
{ kind: "tool_result"; ts: string; toolUseId: string; content: string; isError: boolean }

// 시스템 / Adapter 메시지
{ kind: "system"; ts: string; text: string }

// Stderr / 에러
{ kind: "stderr"; ts: string; text: string }

// 원시 stdout (대체)
{ kind: "stdout"; ts: string; text: string }
```

### tool call과 결과 연결

`toolUseId`를 사용하여 `tool_call`과 `tool_result` 항목을 연결합니다. UI에서 접을 수 있는 카드로 렌더링됩니다.

```ts
const id = `my-tool-${++counter}`;
return [
  { kind: "tool_call", ts, name: "read", input: { path: "/src/main.ts" }, toolUseId: id },
  { kind: "tool_result", ts, toolUseId: id, content: "const main = () => {...}", isError: false },
];
```

### 에러 처리

tool result에 `isError: true`를 설정하면 빨간색 표시기가 나타납니다:

```ts
{ kind: "tool_result", ts, toolUseId: id, content: "ENOENT: no such file", isError: true }
```

## 제약 조건

1. **런타임 import 없음.** 파일은 브라우저에서 `URL.createObjectURL` + 동적 `import()`로 로드됩니다. `import`, `require`, 최상위 `await` 모두 불가합니다.

2. **DOM / Node.js API 없음.** 브라우저 샌드박스에서 실행됩니다. 바닐라 JS (ES2020+)만 사용하세요.

3. **부수 효과 없음.** 모듈 수준 코드는 전역을 수정하거나, `window`에 접근하거나, I/O를 수행하면 안 됩니다. 함수만 선언하고 내보내세요.

4. **결정적.** 동일한 `(line, ts)` 입력에 대해 동일한 출력이 생성되어야 합니다. 로그 재생에 중요합니다.

5. **에러 내성.** 절대 throw하지 마세요. 파싱할 수 없는 라인에 대해 트랜스크립트를 중단하는 대신 `[{ kind: "stdout", ts, text: line }]`을 반환하세요.

6. **파일 크기.** 50 KB 미만으로 유지하세요. 요청마다 제공되고 브라우저에서 eval됩니다.

## 생명주기

| 이벤트 | 동작 |
|---|---|
| 서버 시작 | 플러그인 로더가 `exports["./ui-parser"]`를 읽고, 파일을 읽어 메모리에 캐시 |
| UI에서 실행 열기 | `getUIAdapter(type)` 호출. 내장 파서가 없으면 비동기 `fetch(/api/:type/ui-parser.js)` 시작 |
| 첫 라인 도착 | 범용 process 파서가 즉시 처리 (블로킹 없음). 동적 파서는 백그라운드에서 로드 |
| 파서 로드 | `registerUIAdapter()` 호출. 이후 모든 라인 파싱은 실제 파서 사용 |
| 파서 실패 (404, eval 에러) | 콘솔에 경고 로그. 범용 파서 계속 사용. 실패한 타입 캐시 — 재시도 없음 |
| 서버 재시작 | 인메모리 캐시가 Adapter 패키지에서 다시 채워짐 |

## 에러 동작

| 실패 | 동작 |
|---|---|
| 모듈 구문 에러 (import 실패) | 캐치, 로그, 범용 파서로 대체. 재시도 없음. |
| 잘못된 형태 반환 | 누락된 필드가 있는 개별 항목은 트랜스크립트 빌더에서 무시됨. |
| 런타임에서 throw | 라인별로 캐치. 해당 라인은 범용으로 대체. 파서는 이후 라인에 대해 등록 유지. |
| 404 (ui-parser export 없음) | 타입이 실패 로드 세트에 추가. 첫 호출부터 범용 파서. |
| 계약 버전 불일치 | 서버가 경고 로그, 로딩 건너뜀. 범용 파서 사용. |

## 빌드

```sh
# TypeScript를 JavaScript로 컴파일
tsc src/ui-parser.ts --outDir dist --target ES2020 --module ES2020 --declaration false
```

`tsconfig.json`이 이것을 자동으로 처리할 수 있습니다 — `ui-parser.ts`가 빌드에 포함되고 `dist/ui-parser.js`로 출력되는지 확인하세요.

## 테스트

샘플 stdout에 대해 파서를 로컬에서 테스트하세요:

```ts
// test-parser.ts
import { createStdoutParser } from "./dist/ui-parser.js";

const parser = createStdoutParser();
const sampleLines = [
  "[my-agent] Starting session abc123",
  "Thinking about the task...",
  "$ ls /home/user/project",
  "[done] $ ls — /src /README.md  0.3s",
  "I'll read the README now.",
  "Error: file not found",
];

for (const line of sampleLines) {
  const entries = parser.parseLine(line, new Date().toISOString());
  for (const entry of entries) {
    console.log(`  ${entry.kind}:`, entry.text ?? entry.name ?? entry.content);
  }
}
```

실행: `npx tsx test-parser.ts`

## UI Parser 생략

Adapter의 stdout이 간단한 경우 (tool 마커 없음, 특수 형식 없음), UI 파서를 완전히 생략할 수 있습니다. 범용 `process` 파서가 처리합니다 — 모든 비시스템 라인이 `assistant` 출력이 됩니다. 다음과 같은 경우 적합합니다:

- 일반 텍스트 응답을 출력하는 에이전트
- 결과만 출력하는 커스텀 스크립트
- 구조화된 출력이 없는 간단한 CLI

생략하려면, `package.json`에 `exports["./ui-parser"]`를 포함하지 마세요.

## 다음 단계

- [External Adapter](/adapters/external-adapters) — Adapter 패키지 구축 전체 가이드
- [Adapter 만들기](/adapters/creating-an-adapter) — Adapter 내부 구조 및 내장 통합
