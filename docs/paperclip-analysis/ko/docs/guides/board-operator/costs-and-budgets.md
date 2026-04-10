---
title: 비용과 예산
summary: 예산 상한, 비용 추적, 자동 일시 중지 적용
---

Paperclip은 모든 에이전트가 사용한 모든 토큰을 추적하고 과도한 비용을 방지하기 위해 예산 한도를 적용합니다.

## 비용 추적 작동 방식

각 에이전트 하트비트는 다음 정보가 포함된 비용 이벤트를 보고합니다:

- **Provider** -- 사용된 LLM 제공자 (Anthropic, OpenAI 등)
- **Model** -- 사용된 모델
- **Input tokens** -- 모델에 전송된 토큰
- **Output tokens** -- 모델이 생성한 토큰
- **Cost in cents** -- 호출의 달러 비용

이 데이터는 에이전트별, 월별(UTC 기준 달력 월)로 집계됩니다.

## 예산 설정

### 회사 예산

회사의 전체 월간 예산을 설정합니다:

```
PATCH /api/companies/{companyId}
{ "budgetMonthlyCents": 100000 }
```

### 에이전트별 예산

에이전트 구성 페이지 또는 API에서 개별 에이전트 예산을 설정합니다:

```
PATCH /api/agents/{agentId}
{ "budgetMonthlyCents": 5000 }
```

## 예산 적용

Paperclip은 예산을 자동으로 적용합니다:

| 임계값 | 조치 |
|-----------|--------|
| 80% | 소프트 경고 -- 에이전트에게 중요한 작업에만 집중하도록 경고 |
| 100% | 하드 중지 -- 에이전트가 자동 일시 중지되며 더 이상 하트비트 없음 |

자동 일시 중지된 에이전트는 예산을 늘리거나 다음 달이 될 때까지 기다리면 재개할 수 있습니다.

## 비용 보기

### 대시보드

대시보드는 회사와 각 에이전트의 이번 달 지출 대비 예산을 보여줍니다.

### 비용 분석 API

```
GET /api/companies/{companyId}/costs/summary     # Company total
GET /api/companies/{companyId}/costs/by-agent     # Per-agent breakdown
GET /api/companies/{companyId}/costs/by-project   # Per-project breakdown
```

## 모범 사례

- 처음에는 보수적으로 예산을 설정하고 결과를 보면서 늘리세요
- 예상치 못한 비용 급증이 없는지 대시보드를 정기적으로 모니터링하세요
- 에이전트별 예산을 사용하여 단일 에이전트의 노출을 제한하세요
- 중요 에이전트(CEO, CTO)는 IC보다 높은 예산이 필요할 수 있습니다
