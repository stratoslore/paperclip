---
title: Secrets
summary: 시크릿 CRUD
---

에이전트가 환경 설정에서 참조하는 암호화된 시크릿을 관리합니다.

## 시크릿 목록 조회

```
GET /api/companies/{companyId}/secrets
```

시크릿 메타데이터를 반환합니다 (복호화된 값은 포함하지 않음).

## 시크릿 생성

```
POST /api/companies/{companyId}/secrets
{
  "name": "anthropic-api-key",
  "value": "sk-ant-..."
}
```

값은 저장 시 암호화됩니다. 시크릿 ID와 메타데이터만 반환됩니다.

## 시크릿 업데이트

```
PATCH /api/secrets/{secretId}
{
  "value": "sk-ant-new-value..."
}
```

시크릿의 새 버전을 생성합니다. `"version": "latest"`를 참조하는 에이전트는 다음 하트비트에서 자동으로 새 값을 가져옵니다.

## 에이전트 설정에서 시크릿 사용

인라인 값 대신 에이전트 어댑터 설정에서 시크릿을 참조하세요:

```json
{
  "env": {
    "ANTHROPIC_API_KEY": {
      "type": "secret_ref",
      "secretId": "{secretId}",
      "version": "latest"
    }
  }
}
```

서버는 런타임에 시크릿 참조를 확인하고 복호화하여 에이전트 프로세스 환경에 실제 값을 주입합니다.
