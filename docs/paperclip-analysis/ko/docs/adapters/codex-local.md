---
title: Codex Local
summary: OpenAI Codex 로컬 어댑터 설정 및 구성
---

`codex_local` 어댑터는 OpenAI의 Codex CLI를 로컬에서 실행합니다. `previous_response_id` 체이닝을 통한 세션 지속성과 글로벌 Codex 스킬 디렉토리를 통한 스킬 주입을 지원합니다.

## 사전 요구 사항

- Codex CLI 설치 (`codex` 명령어 사용 가능)
- 환경 변수 또는 에이전트 설정에 `OPENAI_API_KEY` 설정

## 구성 필드

| 필드 | 타입 | 필수 | 설명 |
|-------|------|----------|-------------|
| `cwd` | string | 예 | 에이전트 프로세스의 작업 디렉토리 (절대 경로; 권한이 허용되면 없을 시 자동 생성) |
| `model` | string | 아니오 | 사용할 모델 |
| `promptTemplate` | string | 아니오 | 모든 실행에 사용되는 프롬프트 |
| `env` | object | 아니오 | 환경 변수 (시크릿 참조 지원) |
| `timeoutSec` | number | 아니오 | 프로세스 타임아웃 (0 = 타임아웃 없음) |
| `graceSec` | number | 아니오 | 강제 종료 전 유예 기간 |
| `dangerouslyBypassApprovalsAndSandbox` | boolean | 아니오 | 안전 검사 건너뛰기 (개발용) |

## 세션 지속성

Codex는 세션 연속성을 위해 `previous_response_id`를 사용합니다. 어댑터는 이를 하트비트 간에 직렬화하고 복원하여 에이전트가 대화 컨텍스트를 유지할 수 있도록 합니다.

## 스킬 주입

어댑터는 Paperclip 스킬을 글로벌 Codex 스킬 디렉토리(`~/.codex/skills`)에 심볼릭 링크합니다. 기존 사용자 스킬은 덮어쓰지 않습니다.

Paperclip이 관리형 worktree 인스턴스 내에서 실행될 때(`PAPERCLIP_IN_WORKTREE=true`), 어댑터는 대신 Paperclip 인스턴스 하위의 worktree 격리된 `CODEX_HOME`을 사용하여 Codex 스킬, 세션, 로그 및 기타 런타임 상태가 체크아웃 간에 누출되지 않도록 합니다. 공유 인증/구성 연속성을 위해 사용자의 메인 Codex 홈에서 해당 격리된 홈을 시드합니다.

하트비트 실행 외부에서 수동 로컬 CLI를 사용하는 경우 (예: `codexcoder`로 직접 실행), 다음을 사용하세요:

```sh
pnpm paperclipai agent local-cli codexcoder --company-id <company-id>
```

이 명령은 누락된 스킬을 설치하고, 에이전트 API 키를 생성하며, 해당 에이전트로 실행하기 위한 셸 export 명령을 출력합니다.

## 지시사항 해석

`instructionsFilePath`가 구성된 경우, Paperclip은 해당 파일을 읽고 매 실행마다 `codex exec`에 전송되는 stdin 프롬프트 앞에 추가합니다.

이는 Codex 자체가 실행 `cwd`에서 수행하는 워크스페이스 수준의 지시사항 탐색과는 별개입니다. Paperclip은 Codex 네이티브 저장소 지시사항 파일을 비활성화하지 않으므로, Paperclip이 관리하는 에이전트 지시사항 외에도 저장소 로컬 `AGENTS.md`가 Codex에 의해 로드될 수 있습니다.

## 환경 테스트

환경 테스트는 다음을 확인합니다:

- Codex CLI가 설치되어 있고 접근 가능한지
- 작업 디렉토리가 절대 경로이고 사용 가능한지 (권한이 허용되면 없을 시 자동 생성)
- 인증 신호 (`OPENAI_API_KEY` 존재 여부)
- CLI가 실제로 실행 가능한지 확인하기 위한 실시간 hello 프로브 (`codex exec --json -`에 `Respond with hello.` 프롬프트)
