---
title: 어댑터 개요
summary: 어댑터란 무엇이며 에이전트를 Paperclip에 연결하는 방법
---

어댑터는 Paperclip의 오케스트레이션 계층과 에이전트 런타임 사이의 다리입니다. 각 어댑터는 특정 유형의 AI 에이전트를 호출하고 결과를 캡처하는 방법을 알고 있습니다.

## 어댑터 작동 방식

하트비트가 발생하면 Paperclip은:

1. 에이전트의 `adapterType`과 `adapterConfig`를 조회합니다
2. 실행 컨텍스트와 함께 어댑터의 `execute()` 함수를 호출합니다
3. 어댑터가 에이전트 런타임을 생성하거나 호출합니다
4. 어댑터가 stdout을 캡처하고, 사용량/비용 데이터를 파싱하며, 구조화된 결과를 반환합니다

## 내장 어댑터

| 어댑터 | 타입 키 | 설명 |
|---------|----------|-------------|
| [Claude Local](/adapters/claude-local) | `claude_local` | Claude Code CLI를 로컬에서 실행 |
| [Codex Local](/adapters/codex-local) | `codex_local` | OpenAI Codex CLI를 로컬에서 실행 |
| [Gemini Local](/adapters/gemini-local) | `gemini_local` | Gemini CLI를 로컬에서 실행 (실험적 -- 어댑터 패키지는 존재하나, 아직 안정 타입 enum에 미포함) |
| OpenCode Local | `opencode_local` | OpenCode CLI를 로컬에서 실행 (멀티 프로바이더 `provider/model`) |
| Cursor | `cursor` | Cursor를 백그라운드 모드로 실행 |
| Pi Local | `pi_local` | 임베디드 Pi 에이전트를 로컬에서 실행 |
| Hermes Local | `hermes_local` | Hermes CLI를 로컬에서 실행 (`hermes-paperclip-adapter`) |
| OpenClaw Gateway | `openclaw_gateway` | OpenClaw 게이트웨이 엔드포인트에 연결 |
| [Process](/adapters/process) | `process` | 임의의 셸 명령 실행 |
| [HTTP](/adapters/http) | `http` | 외부 에이전트에 웹훅 전송 |

### 외부 (플러그인) 어댑터

이 어댑터들은 독립형 npm 패키지로 제공되며 플러그인 시스템을 통해 설치됩니다:

| 어댑터 | 패키지 | 타입 키 | 설명 |
|---------|---------|----------|-------------|
| Droid Local | `@henkey/droid-paperclip-adapter` | `droid_local` | Factory Droid를 로컬에서 실행 |

## 외부 어댑터

Paperclip의 소스 코드를 변경하지 않고 독립형 패키지로 어댑터를 구축하고 배포할 수 있습니다. 외부 어댑터는 플러그인 시스템을 통해 시작 시 로드됩니다.

```sh
# Install from npm via API
curl -X POST http://localhost:3102/api/adapters \
  -d '{"packageName": "my-paperclip-adapter"}'

# Or link from a local directory
curl -X POST http://localhost:3102/api/adapters \
  -d '{"localPath": "/home/user/my-adapter"}'
```

전체 가이드는 [External Adapters](/adapters/external-adapters)를 참조하세요.

## 어댑터 아키텍처

각 어댑터는 세 가지 레지스트리에서 사용되는 모듈을 가진 패키지입니다:

```
my-adapter/
  src/
    index.ts            # Shared metadata (type, label, models)
    server/
      execute.ts        # Core execution logic
      parse.ts          # Output parsing
      test.ts           # Environment diagnostics
    ui-parser.ts        # Self-contained UI transcript parser (for external adapters)
    cli/
      format-event.ts   # Terminal output for `paperclipai run --watch`
```

| 레지스트리 | 역할 | 출처 |
|----------|-------------|--------|
| **Server** | 에이전트 실행, 결과 캡처 | 패키지 루트의 `createServerAdapter()` |
| **UI** | 실행 트랜스크립트 렌더링, 설정 폼 제공 | `ui-parser.js` (동적) 또는 정적 import (내장형) |
| **CLI** | 실시간 감시를 위한 터미널 출력 포맷팅 | 정적 import |

## 어댑터 선택

- **코딩 에이전트가 필요한가요?** `claude_local`, `codex_local`, `opencode_local`, `hermes_local`을 사용하거나, 외부 플러그인으로 `droid_local`을 설치하세요
- **스크립트나 명령을 실행해야 하나요?** `process`를 사용하세요
- **외부 서비스를 호출해야 하나요?** `http`를 사용하세요
- **커스텀이 필요한가요?** [자체 어댑터를 만들거나](/adapters/creating-an-adapter) [외부 어댑터 플러그인을 구축](/adapters/external-adapters)하세요

## UI Parser Contract

외부 어댑터는 Paperclip 웹 UI에 stdout을 렌더링하는 방법을 알려주는 독립형 UI 파서를 제공할 수 있습니다. 이것이 없으면 UI는 일반 셸 파서를 사용합니다. 자세한 내용은 [UI Parser Contract](/adapters/adapter-ui-parser)를 참조하세요.
