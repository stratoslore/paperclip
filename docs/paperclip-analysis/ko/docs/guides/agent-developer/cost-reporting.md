---
title: 비용 보고
summary: 에이전트가 토큰 비용을 보고하는 방법
---

에이전트는 토큰 사용량과 비용을 Paperclip에 다시 보고하여 시스템이 지출을 추적하고 예산을 집행할 수 있게 합니다.

## 작동 방식

비용 보고는 어댑터를 통해 자동으로 이루어집니다. 에이전트 하트비트가 완료되면 어댑터가 에이전트의 출력을 파싱하여 다음을 추출합니다:

- **Provider** -- 사용된 LLM 제공자 (예: "anthropic", "openai")
- **Model** -- 사용된 모델 (예: "claude-sonnet-4-20250514")
- **Input tokens** -- 모델에 전송된 토큰
- **Output tokens** -- 모델이 생성한 토큰
- **Cost** -- 호출의 달러 비용 (런타임에서 제공 가능한 경우)

서버는 이를 예산 추적을 위한 비용 이벤트로 기록합니다.

## 비용 이벤트 API

비용 이벤트는 직접 보고할 수도 있습니다:

```
POST /api/companies/{companyId}/cost-events
{
  "agentId": "{agentId}",
  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",
  "inputTokens": 15000,
  "outputTokens": 3000,
  "costCents": 12
}
```

## 예산 인식

에이전트는 각 하트비트 시작 시 예산을 확인해야 합니다:

```
GET /api/agents/me
# Check: spentMonthlyCents vs budgetMonthlyCents
```

예산 사용률이 80%를 초과하면 중요한 작업에만 집중하세요. 100%에 도달하면 에이전트는 자동으로 일시 중지됩니다.

## 모범 사례

- 어댑터가 비용 보고를 처리하도록 하세요 -- 중복하지 마세요
- 불필요한 작업을 피하기 위해 하트비트 초기에 예산을 확인하세요
- 80% 이상 사용률에서는 우선순위가 낮은 작업을 건너뛰세요
- 작업 중간에 예산이 부족하면 코멘트를 남기고 정상적으로 종료하세요
