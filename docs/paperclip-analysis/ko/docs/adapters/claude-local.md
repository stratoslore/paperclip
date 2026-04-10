---
title: Claude Local
summary: Claude Code 로컬 어댑터 설정 및 구성
---

`claude_local` 어댑터는 Anthropic의 Claude Code CLI를 로컬에서 실행합니다. 세션 지속성, 스킬 주입, 구조화된 출력 파싱을 지원합니다.

## 사전 요구 사항

- Claude Code CLI 설치 (`claude` 명령어 사용 가능)
- 환경 변수 또는 에이전트 설정에 `ANTHROPIC_API_KEY` 설정

## 구성 필드

| 필드 | 타입 | 필수 | 설명 |
|-------|------|----------|-------------|
| `cwd` | string | 예 | 에이전트 프로세스의 작업 디렉토리 (절대 경로; 권한이 허용되면 없을 시 자동 생성) |
| `model` | string | 아니오 | 사용할 Claude 모델 (예: `claude-opus-4-6`) |
| `promptTemplate` | string | 아니오 | 모든 실행에 사용되는 프롬프트 |
| `env` | object | 아니오 | 환경 변수 (시크릿 참조 지원) |
| `timeoutSec` | number | 아니오 | 프로세스 타임아웃 (0 = 타임아웃 없음) |
| `graceSec` | number | 아니오 | 강제 종료 전 유예 기간 |
| `maxTurnsPerRun` | number | 아니오 | 하트비트당 최대 에이전트 턴 수 (기본값: `300`) |
| `dangerouslySkipPermissions` | boolean | 아니오 | 권한 프롬프트 건너뛰기 (기본값: `true`); 대화형 승인이 불가능한 헤드리스 실행에 필요 |

## 프롬프트 템플릿

템플릿은 `{{variable}}` 치환을 지원합니다:

| 변수 | 값 |
|----------|-------|
| `{{agentId}}` | 에이전트 ID |
| `{{companyId}}` | 회사 ID |
| `{{runId}}` | 현재 실행 ID |
| `{{agent.name}}` | 에이전트 이름 |
| `{{company.name}}` | 회사 이름 |

## 세션 지속성

어댑터는 하트비트 간에 Claude Code 세션 ID를 유지합니다. 다음 깨어남 시 기존 대화를 이어서 에이전트가 전체 컨텍스트를 유지합니다.

세션 재개는 cwd를 인식합니다: 마지막 실행 이후 에이전트의 작업 디렉토리가 변경된 경우, 새 세션이 시작됩니다.

알 수 없는 세션 오류로 재개에 실패하면, 어댑터는 자동으로 새 세션으로 재시도합니다.

## 스킬 주입

어댑터는 Paperclip 스킬에 대한 심볼릭 링크가 있는 임시 디렉토리를 생성하고 `--add-dir`을 통해 전달합니다. 이를 통해 에이전트의 작업 디렉토리를 오염시키지 않고 스킬을 검색할 수 있습니다.

하트비트 실행 외부에서 수동 로컬 CLI를 사용하는 경우 (예: `claudecoder`로 직접 실행), 다음을 사용하세요:

```sh
pnpm paperclipai agent local-cli claudecoder --company-id <company-id>
```

이 명령은 `~/.claude/skills`에 Paperclip 스킬을 설치하고, 에이전트 API 키를 생성하며, 해당 에이전트로 실행하기 위한 셸 export 명령을 출력합니다.

## 환경 테스트

UI의 "Test Environment" 버튼을 사용하여 어댑터 구성을 검증합니다. 다음을 확인합니다:

- Claude CLI가 설치되어 있고 접근 가능한지
- 작업 디렉토리가 절대 경로이고 사용 가능한지 (권한이 허용되면 없을 시 자동 생성)
- API 키/인증 모드 힌트 (`ANTHROPIC_API_KEY` vs 구독 로그인)
- CLI 준비 상태를 확인하기 위한 실시간 hello 프로브 (`claude --print - --output-format stream-json --verbose`에 `Respond with hello.` 프롬프트)
