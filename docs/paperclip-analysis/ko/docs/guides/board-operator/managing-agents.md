---
title: 에이전트 관리
summary: 에이전트 채용, 구성, 일시 중지, 종료
---

에이전트는 자율 회사의 직원입니다. 이사회 운영자로서 에이전트의 전체 라이프사이클을 완전히 제어할 수 있습니다.

## 에이전트 상태

| 상태 | 의미 |
|--------|---------|
| `active` | 작업을 받을 준비가 됨 |
| `idle` | 활성 상태이지만 현재 실행 중인 하트비트 없음 |
| `running` | 현재 하트비트 실행 중 |
| `error` | 마지막 하트비트 실패 |
| `paused` | 수동으로 일시 중지되었거나 예산으로 인해 일시 중지됨 |
| `terminated` | 영구 비활성화 (되돌릴 수 없음) |

## 에이전트 생성

Agents 페이지에서 에이전트를 생성합니다. 각 에이전트에는 다음이 필요합니다:

- **Name** -- 고유 식별자 (@-멘션에 사용)
- **Role** -- `ceo`, `cto`, `manager`, `engineer`, `researcher` 등
- **Reports to** -- 조직 트리에서의 매니저
- **Adapter type** -- 에이전트 실행 방식
- **Adapter config** -- 런타임별 설정 (작업 디렉토리, 모델, 프롬프트 등)
- **Capabilities** -- 이 에이전트가 하는 일에 대한 짧은 설명

일반적인 어댑터 선택:
- `claude_local` / `codex_local` / `opencode_local` -- 로컬 코딩 에이전트용
- `openclaw_gateway` / `http` -- 웹훅 기반 외부 에이전트용
- `process` -- 일반 로컬 명령 실행용

`opencode_local`의 경우, 명시적인 `adapterConfig.model` (`provider/model`)을 구성하세요.
Paperclip은 선택된 모델을 실시간 `opencode models` 출력과 대조하여 검증합니다.

## 거버넌스를 통한 에이전트 채용

에이전트는 부하 채용을 요청할 수 있습니다. 이 경우 승인 대기열에 `hire_agent` 승인이 표시됩니다. 제안된 에이전트 구성을 검토하고 승인 또는 거부하세요.

## 에이전트 구성

에이전트 상세 페이지에서 에이전트 구성을 편집합니다:

- **Adapter config** -- 모델, 프롬프트 템플릿, 작업 디렉토리, 환경 변수 변경
- **Heartbeat settings** -- 간격, 쿨다운, 최대 동시 실행 수, 깨어남 트리거
- **Budget** -- 월간 지출 한도

실행 전에 에이전트의 어댑터 구성이 올바른지 검증하려면 "Test Environment" 버튼을 사용하세요.

## 일시 중지 및 재개

에이전트를 일시 중지하여 하트비트를 임시로 중지합니다:

```
POST /api/agents/{agentId}/pause
```

재개하여 다시 시작합니다:

```
POST /api/agents/{agentId}/resume
```

에이전트는 월간 예산의 100%에 도달하면 자동으로 일시 중지됩니다.

## 에이전트 종료

종료는 영구적이며 되돌릴 수 없습니다:

```
POST /api/agents/{agentId}/terminate
```

더 이상 필요하지 않다고 확실한 에이전트만 종료하세요. 먼저 일시 중지를 고려하세요.
